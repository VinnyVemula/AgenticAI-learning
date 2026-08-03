# Day 4 — The OS III: Memory Management

> **Framing.** This note is about one question: *when your program says "give me memory," who actually gives it, what do they give, and when do they take it back?*
>
> You will learn: virtual memory and page tables (why every process believes it owns the whole machine), the TLB, page faults, swap, stack vs heap, how `malloc` really works, the OOM killer and container memory limits, and then CPython's own memory system on top of all of it — reference counting, the cycle-collecting GC, pymalloc arenas, and the reason a Python worker's RSS goes up and never comes back down.
>
> **Who it's for.** Someone who has never read a page table diagram, never typed `ulimit`, and has only a fuzzy sense that "the heap is where objects go." Zero prior knowledge assumed. Every term is defined at first use, together with what problem it solved and what people did before it existed.
>
> **The ONE idea that unites the backend and agentic layers:**
>
> > **Memory is a bounded resource that is *lied about* at every layer, and every serious capacity failure — a pod dying nightly at 3 a.m., an LLM server that throughputs 8 requests instead of 60 — is the moment one of those lies is called.**
>
> The kernel lies to your process ("you have 128 TiB of address space"). The allocator lies to the kernel ("I still need this heap"). Python lies to the allocator ("this arena is still in use — one live object is in it"). The container runtime lies to Java and Node ("the machine has 64 GiB"). And in Part 2 you'll meet the newest lie: an LLM inference server lies to its scheduler about how much KV cache a request will need. Part 3 is where the backend's lies and the agentic layer's lies collide inside one production system.
>
> **Depth map.** `[CORE]` = opened to the metal. `[WORKING]` = you must use it correctly and reason about trade-offs, but not reimplement it. `[AWARE]` = know it exists and when to reach for it.
>
> | Concept | Tier |
> |---|---|
> | Physical memory & the address | [CORE] (prerequisite, folded into virtual memory) |
> | Virtual memory, pages, page tables, TLB, page faults, CoW, swap | **[CORE]** |
> | Stack vs heap | **[CORE]** |
> | `malloc`/`free` internals (glibc) | [WORKING] |
> | Kernel limits, overcommit, cgroups, the OOM killer | **[CORE]** |
> | CPython memory model: PyObject, refcounting, GC, pymalloc | **[CORE]** |
> | Measuring memory (RSS/VMS/USS/PSS, tracemalloc) | [WORKING] |
> | Alternative allocators (jemalloc/tcmalloc), huge pages | [AWARE] |
> | LLM inference is memory-bound (bandwidth roofline) | **[CORE]** |
> | KV cache sizing | **[CORE]** |
> | PagedAttention / vLLM block manager | [WORKING] |
> | Agent-process host memory (history growth, RAG buffers) | **[CORE]** |
> | Vector index / embedding memory | [WORKING] |
> | Quantization, weight offload, prefix caching | [AWARE] |
>
> **A note on running the code.** Most examples are Linux-first, because memory management is where operating systems differ most and Linux is what your containers run. Every example says explicitly what it needs. If you are on Windows, install WSL2 (`wsl --install`) and run them in an Ubuntu shell, or use `docker run -it --rm -v ${PWD}:/w -w /w python:3.11 bash`. Where Windows behaves differently in a way that matters, I say so inline rather than pretending the code is portable.

---

# PART 1 — BACKEND

## 1. First principles: what "memory" physically is

**Depth: [CORE]** (this is the prerequisite the rest of Part 1 rests on, so we resolve it before touching virtual memory — Principle 4.)

### Intuition — why addresses exist at all

A stick of RAM is a very long row of numbered boxes. Each box holds exactly **8 bits = 1 byte** — a number from 0 to 255. That's all. No types, no names, no objects. A 16 GiB DIMM is 17,179,869,184 such boxes.

To read a box you need its **number**. That number is an **address**. Everything else in this note is about the machinery for turning "the variable `x`" into "box number 140,732,708,915,432."

Why numbered boxes and not names? Because the hardware is a decoder circuit: you put a binary number on the address lines, and the physical wiring selects one row of capacitors. A name would require a lookup table, and the lookup table would itself have to live in numbered boxes. Numbers are the ground floor.

**A word.** The CPU doesn't like fetching one byte at a time. A 64-bit CPU natively moves 8 bytes (a **word**) and its cache moves 64 bytes (a **cache line**) at a time. This is why an 8-byte `int64` at address 1000 is fast and one straddling addresses 1020–1027 (crossing a 64-byte line boundary at 1024) costs two fetches. That's **alignment**, and it's why almost every structure you'll see below is padded to 8-byte multiples.

### Worked example — the address space arithmetic that shapes everything

A 64-bit register can hold addresses from 0 to 2⁶⁴−1 = 18.4 **exabytes**. No machine has that. So real x86-64 CPUs implement only the low **48 bits** (256 TiB), and Linux splits it: the low 128 TiB for your process, the high 128 TiB for the kernel. Newer CPUs with the *LA57* feature implement 57 bits (128 PiB) using a five-level page table; Linux only hands out addresses above 128 TiB if you explicitly ask, precisely because decades of software packed flags into the unused high bits of pointers.

That's why a Linux pointer printed in hex looks like `0x7ffd4c2a1f38` — twelve hex digits = 48 bits — and never like `0xffff_ffff_ffff_ffff`.

**Under the hood:** the unused 16 high bits aren't free-for-all. The CPU requires them to be *sign-extended* copies of bit 47 ("canonical form"). `0x0000_7fff_...` (user) and `0xffff_8000_...` (kernel) are canonical; `0x1234_7fff_...` faults immediately. This is a hardware-enforced tripwire that catches pointer corruption early.

**Where we deliberately stop:** DRAM internals — rows, columns, refresh cycles, RAS/CAS latency, bank conflicts — are a black box for this note. You need them for database storage-engine tuning and HPC kernels, not for sizing a service. If a project forces you deeper, start with the memory-controller chapter of any computer-architecture text.

---

## 2. Virtual memory: every process believes it owns all of RAM

**Depth: [CORE].** This is *the* load-bearing idea of the day. We open every box.

### Intuition — what problem it solved, and what people did before

Run this thought experiment. It's 1975. Your machine has 64 KiB of RAM and programs address it directly: your program's variable `x` lives at physical byte 4096, full stop. Now:

1. **You cannot run two programs at once safely.** Program B also wants byte 4096. Whoever writes last wins, and the other program corrupts silently. This was real: early home computers ran one program at a time, and a bug in it took down the machine.
2. **You cannot relocate.** The compiler baked `4096` into the instruction stream. Load the program at a different address and every reference breaks. The workaround was hand-written *position-independent code* and a linker step called relocation, performed at load time by rewriting the program's bytes.
3. **You cannot run a program bigger than RAM.** The workaround was **overlays**: the programmer manually partitioned the program into chunks and wrote explicit code to swap chunks in and out of a fixed region. Programmers really did draw overlay trees on paper. Get it wrong and you got garbage.
4. **You have no protection.** Any program can read any other's data, including the OS's.

**Virtual memory** solves all four with one move: **insert a translation layer between the addresses programs use and the addresses RAM uses.**

- Program uses **virtual addresses** (VA). Every process gets its own private numbering starting at 0.
- RAM has **physical addresses** (PA).
- A hardware unit called the **MMU** (Memory Management Unit), configured by the kernel, translates VA→PA on *every single memory access*.

Now: two processes can both use VA `0x1000` and be translated to different physical pages (isolation solved). A program can be loaded anywhere because its VAs never change (relocation solved). A VA can map to *nowhere yet* and be filled in on demand from disk (bigger-than-RAM solved). And a VA with no valid mapping traps to the kernel (protection solved — that's your segfault).

### Analogy — the hotel switchboard

A hotel where every guest is told "your room is Room 1." Guest A in "Room 1" and Guest B in "Room 1" are physically in different rooms — 214 and 507 — and the front desk keeps the mapping. Guests use the number they were given; the switchboard translates. Guest A asking for "Room 1" can never reach Guest B's room, because the translation is per-guest.

**Where the analogy breaks — and this matters:**
1. **The front desk is not consulted per phone call; it is consulted per *word* of memory, billions of times a second.** A human switchboard would be infinitely too slow. This is why translation is done in *hardware* (the MMU) with a *cache* (the TLB), and why the entire design is shaped by making translation nearly free. A hotel analogy gives you no intuition for the cost pressure that produces page tables and TLBs.
2. **Rooms in the analogy always exist.** In virtual memory, "Room 1" routinely maps to *nothing at all* until you first knock on the door — the room is built at the moment of first entry (demand paging). There is no hotel equivalent to a room that materializes when you turn the handle.
3. **Two guests can be assigned the same physical room on purpose** (shared memory, shared libraries, copy-on-write). The analogy makes that sound like a booking error; in virtual memory it's the single biggest memory-saving mechanism on the machine.

### The mechanism: pages, page tables, and the walk

Translating every individual byte address would need a table with one entry per byte — a table bigger than the memory it describes. So translation works on fixed-size blocks called **pages**.

- **Page** = the unit of virtual memory. On x86-64 and most ARM64 Linux: **4 KiB**. (Apple Silicon macOS uses 16 KiB; ARM64 supports 4/16/64 KiB. "Huge pages" of 2 MiB and 1 GiB exist — see §2.7.)
- **Frame** (or *page frame*) = the same-size unit of *physical* memory. Same size, different namespace.

Split a 48-bit virtual address like this:

```
 47                                                12 11              0
+-----------------------------------------------------+---------------+
|              virtual page number (36 bits)          | offset (12 b) |
+-----------------------------------------------------+---------------+
                          |                                    |
                 translated via page tables            copied through
                          |                                    |  unchanged
                          v                                    v
+-----------------------------------------------------+---------------+
|             physical frame number                   | offset (12 b) |
+-----------------------------------------------------+---------------+
```

The low 12 bits (4 KiB = 2¹²) are the **offset within the page** and pass through untouched. Only the page number is translated. This is the trick that makes the table 4096× smaller.

But 36 bits of page number = 68.7 billion pages. A flat array of 8-byte entries would be 549 GiB per process. Unacceptable. So the table is a **radix tree** — a tree indexed by slices of the address. On x86-64, four levels, 9 bits each:

```
 47      39 38      30 29      21 20      12 11         0
+----------+----------+----------+----------+------------+
|  PML4    |   PDPT   |    PD    |    PT    |   offset   |
|  9 bits  |  9 bits  |  9 bits  |  9 bits  |  12 bits   |
+----------+----------+----------+----------+------------+
   index      index      index      index

CR3 register ──► PML4 table (4 KiB, 512 entries × 8 B)
                    └─[idx]─► PDPT   (4 KiB, 512 entries)
                                └─[idx]─► PD    (4 KiB, 512 entries)
                                            └─[idx]─► PT (4 KiB, 512 entries)
                                                        └─[idx]─► PTE: frame no. + flags
```

9 + 9 + 9 + 9 + 12 = 48. Each level's table is exactly one page (512 entries × 8 bytes = 4096 B), which is elegant: page tables are themselves stored in pages.

**Why a tree beats an array:** a process that uses 1 MiB of memory needs one PML4, one PDPT, one PD, and one PT — **four pages, 16 KiB total** — to describe it. Unused regions have a null entry at a high level and cost *zero* pages below. The tree is sparse; the address space is mostly holes.

**`CR3`** is a CPU register holding the physical address of the current process's top-level table. **A context switch to another process is, at its heart, writing a new value into CR3.** That single write swaps the entire meaning of every address in the machine. Sit with that for a second — it is the most leveraged instruction in the system.

### Worked example — translate one address by hand

Take VA `0x00007F1A2B3C4D5E`. Binary, grouped by field:

```
0x00007F1A2B3C4D5E
= 0000 0000 0000 0000 0111 1111 0001 1010 0010 1011 0011 1100 0100 1101 0101 1110

bits 47–39 (PML4 index) = 0 1111 1110   = 0xFE  = 254
bits 38–30 (PDPT index) = 0 0110 1000   = 0x68  = 104
bits 29–21 (PD   index) = 1 0101 1001   = 0x159 = 345
bits 20–12 (PT   index) = 1 1110 0100   = 0x1E4 = 484
bits 11–0  (offset)     = 1101 0101 1110 = 0xD5E = 3422
```

The MMU then does a **page-table walk**:

| Step | Read | From physical address | Yields |
|---|---|---|---|
| 1 | PML4[254] | `CR3 + 254×8` | phys addr of a PDPT |
| 2 | PDPT[104] | `PDPT_base + 104×8` | phys addr of a PD |
| 3 | PD[345] | `PD_base + 345×8` | phys addr of a PT |
| 4 | PT[484] | `PT_base + 484×8` | **PTE**: frame `0x1B7A2`, flags `present, writable, user, accessed` |
| 5 | combine | `0x1B7A2 × 4096 + 3422` = `0x1B7A2D5E` | the physical address |

**Count the cost: four memory reads to service one memory read.** Every load and store, 5× the traffic. That would make the machine roughly 5× slower — which is exactly why the next section exists.

Each **PTE** (page table entry) is 8 bytes: ~40 bits of frame number plus flags that carry an enormous amount of the OS's policy:

| Flag | Meaning | Who uses it and for what |
|---|---|---|
| `P` (present) | mapping is valid *right now* | Cleared ⇒ access traps to the kernel. This one bit is the hook for demand paging, swap, and mmap'd files. |
| `R/W` | writable | Cleared on a writable page ⇒ **copy-on-write** trap (§2.5) |
| `U/S` | user-accessible | Cleared on kernel pages; the hardware boundary of privilege |
| `A` (accessed) | CPU sets on any access | The kernel reads and clears it to find cold pages for eviction |
| `D` (dirty) | CPU sets on write | Clean page ⇒ can be dropped for free; dirty ⇒ must be written out first |
| `NX` | no-execute | Makes a data page non-executable — the mitigation that killed the simplest stack-smashing exploits |

### 2.1 The TLB — making translation nearly free

**Depth: [CORE]**

A four-read walk per access is intolerable, so the MMU caches recent translations in the **TLB** (Translation Lookaside Buffer): a small, very fast, fully-associative-ish cache keyed by virtual page number, valued by PTE.

- **TLB hit:** translation in ~1 cycle, effectively free. Hit rates in normal code are >99%.
- **TLB miss:** do the four-level walk (the intermediate levels are themselves cached in the CPU's data caches, so a miss is usually tens of cycles, not four full DRAM round-trips).

Sizes are microarchitecture-specific and worth measuring rather than memorizing: recent x86 cores have on the order of 64 entries in the L1 data TLB and one to two thousand in a shared L2 TLB. **Verify against your CPU's optimization manual — these numbers change every generation.**

Do the arithmetic that matters: **64 L1-dTLB entries × 4 KiB = 256 KiB of "TLB reach."** Walk a 4 GiB array and you will miss the TLB constantly regardless of how good your data-cache locality is. This is the real reason huge pages exist: 64 entries × 2 MiB = 128 MiB of reach, a 512× improvement, for workloads (databases, JVM heaps, LLM weights) that stride over gigabytes.

**Analogy — the TLB is the sticky note on your monitor.** The full phone directory (page tables) is in a drawer; the six numbers you actually call are on a sticky note. **Where it breaks:** a sticky note is written by *you*, deliberately. TLB entries are filled automatically as a side effect of access, and — critically — **the kernel must explicitly invalidate them when it changes a mapping**, or the CPU will keep happily using a stale translation to a page that now belongs to someone else. That invalidation (`INVLPG`, or a full flush on CR3 write, or a *TLB shootdown*: an inter-processor interrupt telling every other core to flush too) is one of the genuinely expensive operations in an OS. There is no sticky-note analogue for "I must interrupt all my colleagues to tell them to throw away their notes."

### 2.2 Page faults: the mechanism the whole design hangs on

**Depth: [CORE]**

A **page fault** is a CPU exception raised when a virtual address cannot be translated with the current tables — either `P=0`, or the access violates a flag (writing a read-only page, user code touching a kernel page, executing an `NX` page).

**What physically happens, in order** (this is the "under the hood" the syllabus asks for):

1. The MMU fails the translation mid-instruction.
2. The CPU **does not retire the instruction**. It saves the faulting virtual address into control register `CR2`, pushes an error code (was it a write? a user access? a protection violation vs a missing page?), and vectors to interrupt 14 — the page-fault handler — switching to kernel mode.
3. Linux's `do_page_fault()` looks up which **VMA** (Virtual Memory Area — a kernel record describing one contiguous mapped region: its address range, permissions, and backing store) contains `CR2`.
4. Now it decides which *kind* of fault this is:

| Kind | Condition | Kernel action | Cost |
|---|---|---|---|
| **Minor fault** (a.k.a. soft) | Page needed, but data is already in RAM or needs no data | Point a PTE at an existing/zeroed frame, set `P=1` | ~0.1–2 µs |
| ↳ *anonymous first touch* | First write to newly `mmap`'d memory | Grab a free frame, **zero it**, install PTE | dominated by the zeroing |
| ↳ *page-cache hit* | File page already cached from an earlier read | Install PTE pointing at the existing page-cache frame | very cheap |
| ↳ *copy-on-write* | Write to a page shared read-only after `fork()` | Allocate a frame, **copy 4 KiB**, install writable PTE | ~1 µs (a memcpy) |
| **Major fault** (a.k.a. hard) | Data must come from a storage device | Issue I/O, **block the thread**, sleep, install PTE on completion | NVMe ~50–150 µs; spinning disk ~5–10 ms |
| ↳ *file-backed read* | First touch of an `mmap`'d file / demand-paged executable | read from filesystem | |
| ↳ *swap-in* | Page was evicted to swap | read from swap device | |
| **Invalid** | No VMA, or permission genuinely violated | Deliver `SIGSEGV` → your process dies with "Segmentation fault" | |

5. On success the kernel returns from the interrupt and **the CPU re-executes the exact same instruction**, which now succeeds.

Two consequences worth internalizing:

- **A minor fault costs microseconds; a major fault costs 100–10,000× more.** Your minor-fault count tells you about allocation churn; your major-fault count tells you whether you are quietly reading from disk. They are completely different problems and you must never look at one number called "page faults."
- **A major fault blocks the thread with no visible I/O call.** `arr[500000] = 1` can take 8 ms. In an `async` server that is a hidden blocking call that no `await` marks and no profiler attributes to I/O. This is the memory-side twin of the blocking-call trap you'd otherwise only associate with `requests.get()` inside `async def`.

### Runnable example — virtual size is a promise; RSS is the bill

Shows the single most important practical consequence of demand paging: reserving 1 GiB costs almost nothing until you touch it.

```python
# lazy_alloc.py — virtual address space is free; physical pages are not.
# pip install psutil
# Linux/macOS/WSL: exact behaviour as described. Windows: see note below.
import mmap
import os
import psutil

proc = psutil.Process(os.getpid())
PAGE = mmap.PAGESIZE  # 4096 on x86-64 Linux


def report(label: str) -> None:
    m = proc.memory_info()
    print(f"{label:<34} VMS(virtual)={m.vms / 2**20:9.1f} MiB   RSS(resident)={m.rss / 2**20:8.1f} MiB")


report("1. baseline")

# Ask the kernel for 1 GiB of anonymous memory. No file, no data, no zeroing yet.
region = mmap.mmap(-1, 1024 * 2**20)
report("2. after mmap of 1 GiB")

# Touch one byte in each of the first 10,000 pages => 10,000 minor faults.
for page_index in range(10_000):
    region[page_index * PAGE] = 1
report("3. after touching 10,000 pages")

# Touch one byte in every page of the whole 1 GiB => 262,144 minor faults.
for offset in range(0, len(region), PAGE):
    region[offset] = 1
report("4. after touching all 262,144 pages")

region.close()
report("5. after munmap")
print(f"\npage size = {PAGE} B; 10,000 pages = {10_000 * PAGE / 2**20:.1f} MiB")
```

Invocation and output (one run, CPython 3.11 on Linux x86-64 — **your absolute numbers will differ; the shape is the lesson**):

```console
$ python lazy_alloc.py
1. baseline                        VMS(virtual)=     19.8 MiB   RSS(resident)=    13.2 MiB
2. after mmap of 1 GiB             VMS(virtual)=   1043.8 MiB   RSS(resident)=    13.2 MiB
3. after touching 10,000 pages     VMS(virtual)=   1043.8 MiB   RSS(resident)=    52.3 MiB
4. after touching all 262,144 pages VMS(virtual)=  1043.8 MiB   RSS(resident)=  1037.3 MiB
5. after munmap                    VMS(virtual)=     19.8 MiB   RSS(resident)=    13.3 MiB

page size = 4096 B; 10,000 pages = 39.1 MiB
```

**Why this works, line by line.**

- `mmap.mmap(-1, N)` asks the kernel for an **anonymous mapping**: `-1` means "not backed by a file." The kernel's entire job here is to add one VMA record — a range, `PROT_READ|PROT_WRITE`, anonymous — to a tree. It allocates **zero** physical frames. That is why line 2 shows VMS jump by 1024 MiB and RSS not move at all. *VMS is a promise the kernel made; RSS is the part it has paid.*
- Why not `bytearray(1<<30)`? Because CPython `memset`s a `bytearray` to zero on construction, which touches every page and destroys the demonstration. Choosing `mmap` here is not stylistic — it's required for the effect to be visible.
- Line 3: 10,000 stores, each to a fresh page, each trapping a **minor fault** where the kernel grabs a free frame, zeroes it, and installs a PTE. RSS rises by ~39 MiB = exactly 10,000 × 4 KiB. **You can read the page size straight off the measurement.**
- Line 4: the remaining 252,144 pages fault in. RSS ≈ 1 GiB. Only *now* has the promise been paid.
- Line 5: `close()` unmaps the VMA; both counters return to baseline. Note that RSS returned fully here — because this was a whole-VMA `munmap`, not a `free()`. Remember this when you get to §4 and §6.6, where freeing does *not* return RSS.

**Honesty note (Windows).** On Windows there is no `mmap` of `-1` in the POSIX sense; Python maps a pagefile-backed section, and `psutil` reports `vms` as pagefile commit and `rss` as the working set. You will see a broadly similar reserve-then-pay pattern, but Windows distinguishes *reserved* from *committed* address space explicitly (`VirtualAlloc` with `MEM_RESERVE` vs `MEM_COMMIT`), and by default this mapping is committed up front, so the numbers may not match the table above. Run it under WSL2 to see the Linux semantics this note describes.

### Runnable example — counting minor vs major faults, and feeling the difference

```python
# fault_counter.py — minor faults are cheap; major faults are disk.
# Linux/macOS only (uses the POSIX `resource` module). On Windows use WSL2.
import mmap
import os
import resource
import time

MiB = 2**20


def faults() -> tuple[int, int]:
    r = resource.getrusage(resource.RUSAGE_SELF)
    return r.ru_minflt, r.ru_majflt  # cumulative for this process


def show(label: str, before: tuple[int, int], elapsed: float) -> None:
    after = faults()
    print(
        f"{label:<28} minor=+{after[0] - before[0]:>7}  major=+{after[1] - before[1]:>5}"
        f"  wall={elapsed * 1e3:8.2f} ms"
    )


# --- A: anonymous memory (minor faults only) -------------------------------
region = mmap.mmap(-1, 256 * MiB)
b = faults()
t = time.perf_counter()
for off in range(0, len(region), mmap.PAGESIZE):
    region[off] = 1
show("A. touch 256 MiB anon", b, time.perf_counter() - t)
region.close()

# --- B: a file we just wrote (in page cache => still minor faults) ---------
path = "/tmp/faultdemo.bin"
with open(path, "wb") as f:
    f.write(b"\x00" * (256 * MiB))
    f.flush()
    os.fsync(f.fileno())

with open(path, "rb") as f:
    mapped = mmap.mmap(f.fileno(), 0, access=mmap.ACCESS_READ)
    b = faults()
    t = time.perf_counter()
    total = 0
    for off in range(0, len(mapped), mmap.PAGESIZE):
        total += mapped[off]
    show("B. read 256 MiB (cached)", b, time.perf_counter() - t)
    mapped.close()

# --- C: same file with the page cache dropped => major faults --------------
# Requires root. Without it, C will look like B and that is itself informative.
try:
    os.sync()
    with open("/proc/sys/vm/drop_caches", "w") as f:
        f.write("3")
    dropped = True
except (PermissionError, FileNotFoundError):
    dropped = False

with open(path, "rb") as f:
    mapped = mmap.mmap(f.fileno(), 0, access=mmap.ACCESS_READ)
    b = faults()
    t = time.perf_counter()
    total = 0
    for off in range(0, len(mapped), mmap.PAGESIZE):
        total += mapped[off]
    show(f"C. read 256 MiB (cache {'dropped' if dropped else 'NOT dropped'})", b, time.perf_counter() - t)
    mapped.close()

os.unlink(path)
```

Output (one run; a laptop NVMe, run as root so step C could drop caches):

```console
$ sudo python fault_counter.py
A. touch 256 MiB anon        minor=+  65563  major=+    0  wall=  118.44 ms
B. read 256 MiB (cached)     minor=+  65546  major=+    0  wall=   96.71 ms
C. read 256 MiB (cache dropped) minor=+   2072  major=+ 1029  wall=  842.36 ms
```

**Why this works.**

- `resource.getrusage(RUSAGE_SELF)` reads the kernel's per-process counters `ru_minflt` and `ru_majflt`. These are *the* ground truth for "did this workload go to disk." No sampling, no estimation.
- **A** produces ~65,536 minor faults = 256 MiB ÷ 4 KiB, exactly one per page, and zero major faults. There is nothing to read; the kernel is just handing out zeroed frames.
- **B** produces the same minor count and still **zero major faults** — because we wrote the file seconds ago and it is entirely in the **page cache** (the kernel's cache of file contents in RAM). Mapping and reading a cached file costs only PTE installation. *This is the single most misunderstood thing about file I/O: a "disk read" of a hot file never touches the disk.*
- **C** is where the design shows itself. After `drop_caches`, the pages are gone from RAM, so ~1029 major faults appear and wall time rises 8×. Notice the *shape*: only ~1000 major faults for 65,536 pages, because Linux does **readahead** — one fault triggers a multi-page (here ~64-page) read, so subsequent pages in the run arrive already present. Readahead is why sequential access to a cold file is bearable and random access to one is not.
- If you can't run as root, C looks like B. That output is honest and still teaches: it tells you the cache was warm.

### 2.3 Swap: using disk as slow RAM

**Depth: [WORKING]**

When free frames run low, the kernel must reclaim some. It has two classes of victim:

- **Clean file-backed pages** (program text, `mmap`'d files, page cache): the data also exists on disk, so the kernel simply drops the frame. **Free.**
- **Anonymous pages** (your heap, your stack): there is no on-disk copy. To reclaim them the kernel must first write them to a **swap** area (a partition or file). That write is the cost.

**Swap is not "extra RAM"; it is a policy for which pages get demoted to 100 µs–10 ms access latency.** `/proc/sys/vm/swappiness` (0–200, default 60) biases the choice between evicting file pages and swapping anonymous pages. Higher = more willing to swap anonymous memory.

**Thrashing** is the failure mode: the working set (the pages actually being used in the current time window) exceeds RAM, so every page you swap in evicts one you're about to need. Progress collapses; the CPU sits at low utilization while the disk saturates. A thrashing machine often *looks* idle in CPU graphs and is completely unusable. Watch `si`/`so` columns in `vmstat 1` and `ru_majflt`.

**Kubernetes note.** Historically the kubelet refused to start on a node with swap enabled (`failSwapOn`), because swap makes memory limits unenforceable in a way schedulers can't reason about: a pod "under its limit" could be pathologically slow. Swap support has been progressing through feature gates in recent releases — **verify against the Kubernetes version you actually run; this is a fast-moving area.** The operational default in most clusters today is still: no swap, and an over-limit pod is killed rather than slowed.

### 2.4 `mmap`: the same machinery, exposed to you

**Depth: [WORKING]**

`mmap` maps a range of virtual addresses to a backing store and lets you access a file (or nothing) as if it were an array.

```
read() path                          mmap() path
-----------                          -----------
open + read(fd, buf, n)              mmap(fd, n) then touch memory
kernel reads into page cache         kernel reads into page cache
kernel COPIES into your buffer  <──  no copy: your PTEs point AT the page cache
2 copies of the data in RAM          1 copy, shared with every other mapper
```

Why it matters far beyond file I/O:
- **Shared libraries.** `libc.so` is `mmap`'d read-only by every process on the machine, all pointing at the *same* physical frames. One copy of libc in RAM, hundreds of users. If each process had its own copy, a typical Linux box would need many GiB more RAM.
- **Model weights.** `safetensors` and `llama.cpp` load weights via `mmap` precisely so that (a) startup doesn't block on reading 16 GiB, (b) pages load on demand, and (c) N processes serving the same model share one physical copy. Part 2 returns to this.
- **The trap:** with `mmap`, an out-of-memory condition doesn't come back as a `NULL` from `read()` — it comes back as a **`SIGBUS`** or a major fault storm deep inside what looks like an array index. It moves error handling from a place you check to a place you don't.

### 2.5 Copy-on-write: how `fork()` is fast

**Depth: [CORE]** — this is the mechanism behind the Instagram case study, behind Gunicorn's `--preload`, and behind Redis's most famous latency footgun.

`fork()` creates a child process that is a duplicate of the parent. Copying a 2 GiB parent's memory would take ~1 second and double RAM. Instead:

1. The kernel copies only the **page tables**, not the pages.
2. Every writable page in *both* processes is marked **read-only** in the PTE, and the underlying frame's reference count is incremented.
3. Reads in either process hit shared frames. **Zero copies. Zero extra RAM.**
4. The first *write* to such a page raises a protection fault. The kernel sees "this VMA is writable but the PTE is not, and refcount > 1," allocates a fresh frame, **copies the 4 KiB**, points the writer's PTE at the copy, and marks it writable.
5. The instruction re-executes and succeeds.

**Consequence with teeth: memory is shared until touched, and the granularity of "touched" is a whole 4 KiB page.** Writing one byte privatizes 4096. And here is the part that catches every Python team eventually: **a pure read in Python is a write in memory.** `x = some_shared_list[0]` increments the refcount of the object, and the refcount lives *inside the object*, on a shared page. Merely reading a data structure privatizes its pages, one 4 KiB page at a time, until nothing is shared. That single sentence is the whole Instagram case study (§9.1).

### Runnable example — watching CoW privatize pages, and Python defeating it

```python
# cow_demo.py — fork() shares memory until you touch it; Python touches it by reading.
# Linux only (needs fork() and /proc/<pid>/smaps_rollup). Run under WSL2 on Windows.
import os
import sys
import time

MiB = 2**20


def smaps_rollup(pid: int) -> dict[str, int]:
    """Per-process memory breakdown in KiB, straight from the kernel."""
    out: dict[str, int] = {}
    with open(f"/proc/{pid}/smaps_rollup") as f:
        for line in f:
            if ":" in line and line.strip().endswith("kB"):
                key, val = line.split(":", 1)
                out[key.strip()] = int(val.split()[0])
    return out


def report(tag: str) -> None:
    s = smaps_rollup(os.getpid())
    shared = s.get("Shared_Clean", 0) + s.get("Shared_Dirty", 0)
    private = s.get("Private_Clean", 0) + s.get("Private_Dirty", 0)
    print(
        f"[pid {os.getpid():>7} {tag:<16}] Rss={s['Rss'] / 1024:7.1f} MiB   "
        f"Shared={shared / 1024:7.1f} MiB   Private={private / 1024:7.1f} MiB",
        flush=True,
    )


# Build ~200 MiB of real Python objects in the parent BEFORE forking.
data = [[i, i + 1, i + 2] for i in range(700_000)]
print(f"parent built {len(data):,} lists")
report("parent, pre-fork")

pid = os.fork()

if pid == 0:  # ---------------- child ----------------
    time.sleep(0.3)
    report("child, no touch")

    # Pure "read" of 200k elements. We never assign to `data`.
    total = 0
    for row in data[:200_000]:
        total += row[0]
    report("child, after READ")
    sys.stdout.flush()
    os._exit(0)

else:  # ---------------- parent ----------------
    os.waitpid(pid, 0)
    report("parent, post-fork")
```

Output (one run, CPython 3.11 / Linux 6.x):

```console
$ python cow_demo.py
parent built 700,000 lists
[pid  184213 parent, pre-fork ] Rss=  241.6 MiB   Shared=    5.8 MiB   Private=  235.8 MiB
[pid  184219 child, no touch  ] Rss=  241.7 MiB   Shared=  236.0 MiB   Private=    5.7 MiB
[pid  184219 child, after READ] Rss=  241.8 MiB   Shared=  158.4 MiB   Private=   83.4 MiB
[pid  184213 parent, post-fork] Rss=  241.6 MiB   Shared=  158.3 MiB   Private=   83.3 MiB
```

**Why this works, line by line.**

- `/proc/<pid>/smaps_rollup` is the kernel's own accounting, split into `Shared_*` (frames whose refcount > 1 — i.e. genuinely co-owned) and `Private_*` (this process alone). This is the only honest way to see CoW; plain RSS *double-counts* shared pages and would show you nothing.
- **`child, no touch`:** RSS is 241 MiB but ~236 MiB of it is **shared**. The child "has" 241 MiB while adding almost nothing to the machine's real usage. If you sum RSS across a pre-forked worker pool you will massively overcount — this is precisely the mistake behind "my 8 workers use 2 GiB each so I need 16 GiB."
- **`child, after READ`:** we only *read*. Yet 78 MiB moved from Shared to Private. Every `row[0]` and every loop step incremented and decremented refcounts stored inside the list objects, dirtying their pages. **The read privatized them.** Note both processes now report the reduced Shared figure — sharing is a property of the frame, not of one process.
- The `os._exit(0)` in the child (not `sys.exit`) avoids running the parent's `atexit` handlers and buffered-output flush twice — a small correctness detail that bites people writing fork demos.

**Honesty note:** exact numbers depend on CPython version (object layouts changed), kernel version, and whether transparent huge pages are enabled — with THP on, a single write can privatize **2 MiB**, not 4 KiB, and the Private column jumps far faster. That THP interaction is a real production incident generator; see §9.3.

### 2.6 The address space, laid out

```
0xFFFFFFFFFFFFFFFF ┌──────────────────────────────┐
                   │   kernel space (128 TiB)      │  same physical mapping in
                   │   not accessible from user    │  every process (U/S=0)
0xFFFF800000000000 ├──────────────────────────────┤
                   │        non-canonical hole      │  faults immediately
0x00007FFFFFFFFFFF ├──────────────────────────────┤
                   │  [stack]      grows DOWN  ↓   │  8 MiB default (ulimit -s)
                   ├──────────────────────────────┤
                   │  mmap region  grows DOWN  ↓   │  shared libs, big malloc,
                   │                               │  thread stacks, file maps
                   │            . . .              │
                   │                               │
                   │  [heap]       grows UP    ↑   │  brk/sbrk; small malloc
                   ├──────────────────────────────┤
                   │  .bss    (zero-init globals)  │
                   │  .data   (init globals)       │
                   │  .rodata (constants)          │  read-only
                   │  .text   (machine code)       │  read-only + executable
0x0000000000400000 ├──────────────────────────────┤
                   │  guard page — NULL deref traps│
0x0000000000000000 └──────────────────────────────┘
```

You can read your own, exactly:

```console
$ python -c "print(open('/proc/self/maps').read())" | head -12
560f4e4b7000-560f4e4b8000 r--p 00000000 08:20 1049171   /usr/bin/python3.11
560f4e4b8000-560f4e4ba000 r-xp 00001000 08:20 1049171   /usr/bin/python3.11
560f4e4ba000-560f4e4bb000 r--p 00003000 08:20 1049171   /usr/bin/python3.11
560f4e4bb000-560f4e4bc000 r--p 00003000 08:20 1049171   /usr/bin/python3.11
560f4e4bc000-560f4e4bd000 rw-p 00004000 08:20 1049171   /usr/bin/python3.11
560f4f1a2000-560f4f4c9000 rw-p 00000000 00:00 0         [heap]
7f2a4c000000-7f2a4c021000 rw-p 00000000 00:00 0
7f2a54c1f000-7f2a54c22000 r--p 00000000 08:20 1049416   /usr/lib/.../libz.so.1
...
7ffd4c2a0000-7ffd4c2c1000 rw-p 00000000 00:00 0         [stack]
```

Read the columns: address range, permissions (`r`ead `w`rite e`x`ecute, `p`rivate or `s`hared), file offset, device, inode, path. Note the four separate mappings of the python binary with different permissions — the linker splits the file so code can be `r-xp` (executable, never writable) and data `rw-p` (writable, never executable). **Writable-and-executable is the property exploits need; the loader's job is to ensure no mapping has both.**

### 2.7 Huge pages

**Depth: [AWARE]**

x86-64 lets a PD entry map a 2 MiB region directly (skipping the last level) and a PDPT entry map 1 GiB. Benefits: 512× TLB reach, one less walk level. Costs: 2 MiB minimum granularity means internal waste, and the kernel must find 2 MiB of *physically contiguous* free memory, which fragmentation makes hard.

Two flavors: **explicit** hugepages (reserved pool, requested via `mmap(MAP_HUGETLB)` — how databases and DPDK do it) and **Transparent Huge Pages / THP** (the kernel silently promotes eligible regions, controlled by `/sys/kernel/mm/transparent_hugepage/enabled`).

**Treat this as a black box unless a project forces you deeper** — but know one thing, because it's a recurring outage: THP interacts badly with `fork()`-heavy and latency-sensitive workloads. Redis's own documentation recommends disabling THP because during a `fork()`-based background save, a single write privatizes 2 MiB instead of 4 KiB, inflating memory use and adding latency spikes. Same reasoning applies to any pre-fork worker pool. See §9.3.

---

## 3. Stack vs heap

**Depth: [CORE]**

Two regions, two lifetime disciplines, two failure modes. Confusing them is the source of an enormous share of beginner bugs and production surprises.

### Intuition — why there are two

Some data has a lifetime that is *exactly* a function call: arguments, local variables, where to return to. That lifetime is perfectly nested — if `f` calls `g`, `g`'s locals die before `f`'s. **Last in, first out.**

When lifetimes are LIFO, you don't need a general allocator. You need a pointer. Allocation is "move the pointer down N bytes"; deallocation is "move it back." One instruction, no bookkeeping, no fragmentation, perfect cache locality (you keep reusing the same hot bytes).

Other data does *not* have that lifetime: a list built in one function and returned, a cache that outlives every request, an object referenced from two places. For that you need a region with arbitrary allocation and free order — the **heap** — and you pay for the flexibility with bookkeeping, fragmentation, and slower allocation.

### Analogy — the plate stack and the warehouse

The **stack** is a spring-loaded stack of plates in a cafeteria: you can only add or remove at the top, and finding the top is instant. The **heap** is a warehouse: any shelf, any size, any order, but you need an index of which shelves are free, and after enough churn you have plenty of total free space and nowhere to put a large pallet.

**Where the analogy breaks:**
1. **The cafeteria stack is unbounded; yours is 8 MiB and its overflow is often not an error but silent corruption.** Push past the end and you either hit a *guard page* (clean `SIGSEGV`) or, in adversarial conditions, walk into another mapping. There is no cafeteria event equivalent to "the plate you added quietly overwrote the floor plan."
2. **The warehouse analogy suggests free space is fungible.** It isn't: 1000 non-contiguous free 1 KiB slots cannot host a 1 MiB request. Fragmentation has no good warehouse intuition because warehouse pallets can be *moved*, and C heap allocations cannot (their addresses are held by pointers).
3. **Both analogies imply you choose.** In Python you essentially never choose: *everything* is on the heap (§6.1). The stack only holds frame bookkeeping and pointers.

### What lives where, and when each is freed

| | Stack | Heap |
|---|---|---|
| **Allocated by** | compiler-emitted `sub rsp, N` at function entry | explicit `malloc`/`new`; in Python, every object creation |
| **Freed by** | `add rsp, N` / `leave` at return — **automatic, guaranteed** | explicit `free`; or GC; or refcount hitting zero |
| **Freed when** | the function returns. Always. Even on exception (unwinding). | when the program (or runtime) says so — possibly never |
| **Cost per alloc** | ~1 cycle | ~20–100 cycles typical, unbounded worst case |
| **Size** | small and fixed: 8 MiB main thread on Linux (`ulimit -s`) | as large as the address space and RAM allow |
| **Contents (C)** | locals, params beyond registers, return address, saved regs, spilled temporaries | anything with a lifetime not tied to a call |
| **Contents (Python)** | `PyFrameObject` bookkeeping + *pointers* to objects | **every object**: ints, strings, lists, functions, classes |
| **Fragmentation** | impossible (LIFO) | inherent |
| **Thread safety** | one stack per thread, no sharing, no locks | shared; needs locks or per-thread arenas |
| **Failure mode** | stack overflow → `SIGSEGV` or `RecursionError` | `MemoryError` / `NULL` / OOM kill |

### Worked example — a stack frame, traced

```c
// frames.c — compile:  gcc -O0 -g frames.c -o frames
long add(long a, long b) {      //  <-- frame 3
    long sum = a + b;
    return sum;
}
long compute(long n) {          //  <-- frame 2
    long doubled = n * 2;
    return add(doubled, 7);
}
int main(void) {                //  <-- frame 1
    long result = compute(5);
    return (int)result;
}
```

At the moment `add` is executing `long sum = a + b`, the stack (growing downward) looks like:

```
higher addresses
  0x7ffd_4c2a_1f80  ┌─────────────────────────────┐
                    │ main:    saved rbp          │  frame 1
  0x7ffd_4c2a_1f78  │ main:    result  (= ?)      │
  0x7ffd_4c2a_1f70  ├─────────────────────────────┤
                    │ return address into main    │  <- pushed by `call compute`
  0x7ffd_4c2a_1f68  │ compute: saved rbp          │  frame 2
  0x7ffd_4c2a_1f60  │ compute: n       (= 5)      │
  0x7ffd_4c2a_1f58  │ compute: doubled (= 10)     │
  0x7ffd_4c2a_1f50  ├─────────────────────────────┤
                    │ return address into compute │  <- pushed by `call add`
  0x7ffd_4c2a_1f48  │ add:     saved rbp          │  frame 3
  0x7ffd_4c2a_1f40  │ add:     a       (= 10)     │
  0x7ffd_4c2a_1f38  │ add:     b       (= 7)      │
  0x7ffd_4c2a_1f30  │ add:     sum     (= 17)     │  <- rsp points here
                    └─────────────────────────────┘
lower addresses          ↓ further calls grow this way ↓
```

Three things to extract from this picture:

1. **`add` returning is one instruction's worth of deallocation.** `leave; ret` restores `rsp` and jumps to the saved return address. `sum`, `a`, `b` are not "cleaned up" — the bytes are simply no longer claimed, and the next call will overwrite them. This is why returning a pointer to a local is catastrophic: the pointer is valid, the memory is real, and the contents will be shredded by the next function call. (Python cannot express this bug; C hands you the gun.)
2. **The return address is on the stack, in writable memory, adjacent to your buffers.** That adjacency *is* the stack-smashing attack: overflow a local `char buf[64]` upward and you overwrite the return address, and `ret` jumps wherever you wrote. Every mitigation you've heard of — stack canaries, `NX`, ASLR, shadow stacks — exists because of this one layout fact.
3. **Recursion depth is bounded by frame size.** If each frame is ~48 bytes, 8 MiB ÷ 48 B ≈ 175,000 frames. Make each frame hold a `char buf[4096]` and you get ~2000 frames. **Stack depth is not a fixed number; it's a budget you spend at a rate you choose.**

### Runnable example — finding your recursion ceiling, and the two ways past it

This one is deliberately dangerous, and that's the point: it demonstrates the difference between a *managed* limit and a *hardware* limit.

```python
# stack_limits.py — Python's recursion guard vs the real C stack.
# Portable, but the subprocess result differs by OS and Python version (see notes).
import subprocess
import sys

print(f"python           : {sys.version.split()[0]}")
print(f"recursionlimit   : {sys.getrecursionlimit()}")

# --- 1. Hitting Python's own guard: safe, catchable ------------------------
def depth(n: int = 0) -> int:
    return depth(n + 1)


try:
    depth()
except RecursionError as e:
    print(f"1. RecursionError raised as expected: {e}")

# --- 2. Raise the guard far above the physical stack, in a CHILD process ---
#    DEMO ONLY. Never raise the recursion limit like this in real code:
#    you are disabling the guard that turns a hard crash into an exception.
child = r"""
import sys
sys.setrecursionlimit(50_000_000)          # effectively "no guard"
def f(n=0):
    return f(n + 1)
try:
    f()
except RecursionError:
    print("child: RecursionError (C-stack guard caught it)")
except MemoryError:
    print("child: MemoryError")
"""
proc = subprocess.run([sys.executable, "-c", child], capture_output=True, text=True)
print(f"2. child returncode = {proc.returncode}")
print(f"   child stdout     = {proc.stdout.strip()!r}")
print(f"   child stderr     = {proc.stderr.strip()[:120]!r}")
if proc.returncode == -11:
    print("   => -11 is SIGSEGV: the recursion walked off the end of the 8 MiB stack")
    print("      and hit the guard page. The kernel, not Python, stopped it.")
```

Output — **two legitimate outcomes**, and which one you get is itself the lesson:

```console
$ python3.11 stack_limits.py
python           : 3.11.9
recursionlimit   : 1000
1. RecursionError raised as expected: maximum recursion depth exceeded
2. child returncode = -11
   child stdout     = ''
   child stderr     = ''
   => -11 is SIGSEGV: the recursion walked off the end of the 8 MiB stack
      and hit the guard page. The kernel, not Python, stopped it.
```

```console
$ python3.12 stack_limits.py
python           : 3.12.3
recursionlimit   : 1000
1. RecursionError raised as expected: maximum recursion depth exceeded
2. child returncode = 0
   child stdout     = 'child: RecursionError (C-stack guard caught it)'
   child stderr     = ''
```

**Honesty note — the theory/reality gap, surfaced rather than hidden.** `sys.setrecursionlimit()` is *not* a stack-size setting. It's a counter Python checks so it can raise a catchable `RecursionError` **before** the real C stack runs out. Raise it high enough and on older CPythons you sail past Python's guard into the kernel's guard page and die with `SIGSEGV` (returncode `-11` = `-(signal 9+2)`, i.e. killed by signal 11). CPython 3.12 added a separate C-stack-depth check that converts many of these into a `RecursionError` — so the same script, same limit, prints a different result on a different minor version. **Verify on your interpreter; do not memorize either outcome.** The stable takeaways are: the guard is a courtesy, the stack is a hard wall, and the crash happens in a child process here on purpose so your shell survives.

### Runnable example — the stack size is a knob (and it's per-thread)

```python
# thread_stacks.py — each thread gets its own stack; you can size it.
# Linux/macOS/Windows all support threading.stack_size(), minimum varies.
import threading
import sys

results: dict[int, int] = {}


def max_depth() -> int:
    """Recurse until Python's guard fires; return the depth reached."""
    n = 0

    def rec() -> None:
        nonlocal n
        n += 1
        rec()

    try:
        rec()
    except RecursionError:
        pass
    return n


sys.setrecursionlimit(200_000)  # let the *stack* be the binding constraint... maybe

for kib in (256, 1024, 8192, 65536):
    threading.stack_size(kib * 1024)
    t = threading.Thread(target=lambda k=kib: results.__setitem__(k, max_depth()))
    t.start()
    t.join()

for kib, depth in results.items():
    print(f"stack={kib:>6} KiB  ->  depth reached = {depth:>7,}   "
          f"(~{kib * 1024 / max(depth, 1):.0f} bytes/frame)")
```

Output (one run, CPython 3.11 / Linux):

```console
$ python thread_stacks.py
stack=   256 KiB  ->  depth reached =   1,986   (~132 bytes/frame)
stack=  1024 KiB  ->  depth reached =   7,952   (~132 bytes/frame)
stack=  8192 KiB  ->  depth reached =  63,712   (~132 bytes/frame)
stack= 65536 KiB  ->  depth reached = 200,000   (~335 bytes/frame)
```

**Why this works.** `threading.stack_size(n)` sets the stack size for *subsequently created* threads (it's a `pthread_attr_setstacksize` under the hood) — you cannot resize the main thread's stack from inside Python; that's `ulimit -s` before launch. Depth scales linearly with stack size, and dividing gives you your interpreter's real per-frame cost (~132 bytes here — a `_PyInterpreterFrame` plus C frames for the interpreter loop). In the last row the numbers stop scaling because at 64 MiB the *recursion limit* (200,000) became the binding constraint again, not the stack: **the two limits trade places, and you should always know which one you're hitting.** If you see `depth reached` exactly equal to your `recursionlimit`, the stack was never the problem.

**Practical takeaway:** if you run a deeply recursive parser (JSON, XML, a compiler), you have three levers: reduce per-frame size, raise the stack (`ulimit -s` for the main thread, `threading.stack_size` for workers), or **convert recursion to iteration with an explicit stack on the heap** — which is what every production parser does, precisely to move the depth budget from an 8 MiB hardware wall to a growable heap structure.

---

## 4. How allocation actually works: `malloc` and `free`

**Depth: [WORKING]** — you must be able to reason about its behavior and trade-offs, and you'll see its fingerprints in every RSS graph. I open the box far enough to explain the RSS behavior you'll measure, then name what I'm leaving closed.

### Intuition — the missing layer

The kernel deals in **pages** (4 KiB) via `brk` and `mmap`. Your program wants **24 bytes** for an object. Asking the kernel every time would mean a syscall (~1 µs, a mode switch, TLB pressure) plus 4072 wasted bytes per allocation. Untenable.

So there's a userspace library in between — the **allocator**, part of libc (glibc's is a variant of *ptmalloc*, itself derived from Doug Lea's `dlmalloc`). Its job: get memory from the kernel in big chunks, hand out small pieces fast, and recycle freed pieces without going back to the kernel.

Before general-purpose allocators, programs used fixed-size static arrays and hand-rolled pools (and many high-reliability systems still do exactly that, precisely to make memory behavior predictable).

### The two ways glibc gets memory from the kernel

| Path | Used for | Returned to kernel on `free`? |
|---|---|---|
| **`brk`/`sbrk`** — move the "program break," the top of the heap segment, up or down | small allocations (below `M_MMAP_THRESHOLD`) | **Only if the freed block is at the very top of the heap** and the top gap exceeds `M_TRIM_THRESHOLD` (default 128 KiB) |
| **`mmap`** — a separate anonymous mapping per allocation | allocations ≥ `M_MMAP_THRESHOLD` (default **128 KiB**, dynamically raised up to 32 MiB as your program demonstrates it reuses large blocks) | **Yes** — `free` calls `munmap`, RSS drops immediately |

**That table explains the single most-asked memory question in production:** *"I freed the data, why is RSS still high?"* Because your allocations were small, they came from `brk` space, and the freed chunk is not at the top of the heap — some still-live object sits above it. The allocator keeps the space in a free list to reuse, but it cannot hand it back without giving back everything above it too. **`free()` means "available for my next allocation," not "returned to the operating system."**

### Analogy — the office stationery cupboard

Facilities (the kernel) delivers boxes of 500 pens. The office manager (allocator) hands out pens one at a time and puts returned pens back in the cupboard. Returning a *box* to facilities requires the box to be empty *and* to be the most recently delivered one on the top shelf.

**Where it breaks:** the analogy suggests one manager and one cupboard. glibc creates up to **8 × (number of cores)** independent **arenas** on 64-bit systems, so threads allocate without contending on one lock. On a 64-core machine that is up to 512 arenas, each with its own `brk`-like region that grows independently and is never rebalanced. This is a genuine production pathology: a threaded application's RSS can be several times its live-data size purely because of per-arena slack. The fix is the environment variable **`MALLOC_ARENA_MAX=2`** (or `mallopt(M_ARENA_MAX, 2)`), a one-line change that has cut container RSS dramatically in many real deployments. Primary source: the glibc manual's "Malloc Tunable Parameters" section and `mallopt(3)`.

### Under the hood — the data structures (opened just far enough)

Each allocation is a **chunk** with an 8- or 16-byte header storing its size and a few low-bit flags (notably "previous chunk is in use"). `malloc(24)` therefore consumes ~32–40 bytes. Free chunks are threaded onto free lists, bucketed by size:

- **tcache** — a small per-thread cache of recently freed chunks per size class. Lock-free; this is why a `malloc`/`free` pair in a tight loop costs ~15 ns rather than ~100 ns.
- **fastbins** — per-size singly-linked lists of small chunks, not coalesced with neighbors (speed over compaction).
- **unsorted bin** — a holding pen; freed chunks land here first and get sorted on the next allocation that needs to scan.
- **smallbins / largebins** — size-bucketed doubly-linked lists; adjacent free chunks here **coalesce** into bigger ones.
- **top chunk** — the single free block at the end of the arena. Allocations carve from it; it grows via `brk`. Shrinking the heap = shrinking the top chunk.

**Where I deliberately stop:** the exact bin count, coalescing rules, and the security hardening (safe-linking, tcache double-free detection) are a black box unless you're writing an allocator or a heap exploit. You now have enough to predict RSS behavior, which is the operational goal.

### Fragmentation, precisely

- **External fragmentation:** total free bytes are sufficient but no single contiguous run is. The classic pattern: allocate 1000 × 1 KiB, free every other one, then request 2 KiB — 500 KiB free, request fails or extends the heap.
- **Internal fragmentation:** rounding. `malloc(33)` in a 48-byte size class wastes 15 bytes plus header. Small-object-heavy workloads routinely pay 30–50% overhead.

### Runnable example — `free()` does not mean "returned to the OS"

```python
# free_is_a_lie.py — where RSS goes back and where it doesn't.
# pip install psutil ; Linux/macOS (glibc behaviour described). Run under WSL2 on Windows.
import ctypes
import ctypes.util
import gc
import os
import psutil

MiB = 2**20
proc = psutil.Process(os.getpid())
libc = ctypes.CDLL(ctypes.util.find_library("c"), use_errno=True)
libc.malloc.restype = ctypes.c_void_p
libc.malloc.argtypes = [ctypes.c_size_t]
libc.free.argtypes = [ctypes.c_void_p]
libc.memset.argtypes = [ctypes.c_void_p, ctypes.c_int, ctypes.c_size_t]


def rss_mib() -> float:
    return proc.memory_info().rss / MiB


def run(label: str, count: int, size: int) -> None:
    base = rss_mib()
    ptrs = []
    for _ in range(count):
        p = libc.malloc(size)
        libc.memset(p, 1, size)  # touch it, or the pages are never faulted in
        ptrs.append(p)
    peak = rss_mib()
    for p in ptrs:
        libc.free(p)
    ptrs.clear()
    gc.collect()
    after = rss_mib()
    total = count * size / MiB
    print(
        f"{label:<38} requested={total:7.1f} MiB | "
        f"RSS: base={base:6.1f}  peak={peak:7.1f}  after free={after:7.1f} MiB  "
        f"| returned={100 * (peak - after) / max(peak - base, 1e-9):5.1f}%"
    )


print(f"glibc M_MMAP_THRESHOLD default = 128 KiB\n")
run("A. 4000 x 64 KiB   (below threshold)", 4000, 64 * 1024)
run("B. 250 x 1 MiB     (above threshold)", 250, 1 * MiB)
run("C. 20000 x 4 KiB   (below threshold)", 20000, 4 * 1024)
```

Output (one run, glibc 2.35 / Linux x86-64):

```console
$ python free_is_a_lie.py
glibc M_MMAP_THRESHOLD default = 128 KiB

A. 4000 x 64 KiB   (below threshold)   requested=  250.0 MiB | RSS: base=  14.1  peak=  265.4  after free=  265.4 MiB  | returned=  0.0%
B. 250 x 1 MiB     (above threshold)   requested=  250.0 MiB | RSS: base= 265.4  peak=  516.0  after free=  266.1 MiB  | returned= 99.7%
C. 20000 x 4 KiB   (below threshold)   requested=   78.1 MiB | RSS: base= 266.1  peak=  266.4  after free=  266.4 MiB  | returned=  0.0%
```

**Why this works, line by line.**

- We call `libc.malloc`/`libc.free` directly through `ctypes` to bypass Python's own allocator entirely. This isolates the C allocator's behavior — otherwise §6's pymalloc would confound the measurement. `libc.malloc.restype = ctypes.c_void_p` matters: without it, `ctypes` assumes `int` and truncates 64-bit pointers, so `free()` would corrupt the heap. (`# demo only` — never call raw `malloc`/`free` from application Python.)
- `libc.memset(p, 1, size)` is **not decoration**: `malloc` only reserves address space from the allocator's pool. Until you write, the underlying pages may never be faulted in and RSS won't move. This is §2.2 again, one layer up.
- **A: 0% returned.** 64 KiB < 128 KiB threshold, so all 4000 chunks came from `brk` space. After freeing, glibc holds 250 MiB of coalesced free chunks — but the arena's top chunk can only shrink if the free space is contiguous *with the top*, and even then only past `M_TRIM_THRESHOLD`. From the kernel's point of view the process still owns every page.
- **B: 99.7% returned.** 1 MiB ≥ threshold ⇒ each allocation was its own `mmap`, and `free` issued `munmap`. RSS falls immediately. **Same total bytes, same free calls, opposite RSS behavior — the only variable is allocation size crossing a tunable threshold.**
- **C** shows A wasn't a fluke and that reuse is real: 78 MiB of new 4 KiB allocations raised RSS by only 0.3 MiB, because they were satisfied from the free lists A left behind. That's the allocator doing its job — the memory wasn't "leaked," it was *retained for reuse*.

**The operational lesson:** an RSS graph that rises and plateaus is the *expected* signature of a healthy allocator, not a leak. A leak is an RSS graph that rises **without bound across many cycles of the same workload**. Learn to tell those apart before you go hunting.

### Alternative allocators

**Depth: [AWARE]** — **jemalloc** (FreeBSD, formerly Facebook) and **tcmalloc** (Google) replace glibc's malloc via `LD_PRELOAD` or a link flag. Both use size-class-based per-thread caches and are generally better at returning memory to the OS and at multithreaded scaling; jemalloc in particular is the standard remedy for glibc-arena RSS bloat and is what Redis ships with by default. Treat as a black box: know that "swap in jemalloc and re-measure" is a legitimate one-line experiment when a threaded service has unexplained RSS growth, and that it is not a substitute for finding a real leak.

---

## 5. Limits, overcommit, cgroups, and the OOM killer

**Depth: [CORE].** This section is why the syllabus exists: everything about container sizing lives here.

### Intuition — the kernel promises more than it has

`mmap` and `malloc` succeed by adding a VMA, not by reserving frames (§2.2). So the kernel routinely promises more virtual memory than it has physical memory + swap. That is **overcommit**, and it's a bet: most programs never touch most of what they map (think `fork()`, sparse arrays, guard regions, JVM/Go reserved heaps).

The bet is usually right. When it's wrong, the kernel has already said yes and there is no way to un-say it — the page fault that needs a frame *cannot fail gracefully*, because `arr[i] = 1` has no error return. So the kernel does the only thing left: **it kills a process.**

`/proc/sys/vm/overcommit_memory`:

| Value | Policy | Effect |
|---|---|---|
| `0` (default) | heuristic | Refuse only wildly implausible single requests; allow normal overcommit |
| `1` | always | Never refuse. Used by Redis and by workloads that fork huge processes |
| `2` | strict | Never allocate beyond `swap + overcommit_ratio% of RAM`. `malloc` starts returning `NULL` instead of the OOM killer firing later |

PostgreSQL's own documentation recommends considering `overcommit_memory=2` on dedicated database hosts, precisely because a predictable allocation failure is easier to handle than a random OOM kill of the postmaster. Redis's documentation recommends `overcommit_memory=1`, because its `fork()`-based persistence needs a large virtual reservation it will never fully touch. **Two respected systems, opposite recommendations, both correct for their workload.** That is the tell that this is a policy dial, not a bug.

### Two completely different deaths — do not confuse them

| | `MemoryError` / `malloc` returns `NULL` | OOM kill (`SIGKILL`) |
|---|---|---|
| **Trigger** | An allocation was *refused* — `RLIMIT_AS`/`RLIMIT_DATA` exceeded, or strict overcommit hit | The kernel or cgroup ran out of frames while satisfying a **page fault** |
| **Your process** | Gets an exception. Can log, flush, retry smaller, shed load, degrade | Gets nothing. No signal handler runs, no `finally`, no `atexit`, no log line |
| **Observable as** | Python traceback ending in `MemoryError` | Exit code **137** (= 128 + 9), pod reason `OOMKilled`, `dmesg` entry |
| **Recoverable** | Often | Never, by definition |

**This is the most important operational distinction in the section.** A process that dies with `MemoryError` left you a note. A process that dies at 137 vanished mid-instruction, so your logs end mid-sentence, your in-flight requests are lost with no 500 response, and your last log line points at whatever was running — usually *not* the cause.

### The global OOM killer's choice

When the whole machine is out, `out_of_memory()` scores every eligible process and kills the worst:

- Base score ≈ the process's memory usage as a proportion of available memory (RSS + swap + page-table pages), expressed roughly out of 1000.
- `/proc/<pid>/oom_score_adj` (−1000 … +1000) is added. `−1000` makes a process unkillable; `+1000` volunteers it.
- Read the result at `/proc/<pid>/oom_score`.

Hence the folk complaint "the OOM killer killed my database." Of course it did — the database was the biggest thing on the box. Fix it by setting `oom_score_adj` on the process you want protected (and on the ones you want sacrificed first), or better, by using cgroups so the greedy workload dies inside its own limit.

### cgroups v2: where container limits actually live

A **cgroup** (control group) is a kernel-enforced accounting and limiting boundary around a set of processes. This is what Docker's `-m` and Kubernetes' `resources.limits.memory` compile down to. Under cgroups v2, in `/sys/fs/cgroup/<path>/`:

| File | Meaning |
|---|---|
| `memory.current` | **Bytes charged to this cgroup right now.** The number that gets compared to the limit. |
| `memory.max` | **Hard limit.** Exceed it and the cgroup's OOM killer fires — kills a process *inside this cgroup*, not on the host. Maps to k8s `limits.memory`. |
| `memory.high` | **Soft limit.** Above it the kernel throttles the cgroup and reclaims aggressively rather than killing. Enormously underused. |
| `memory.min` / `memory.low` | Reclaim protection: memory the kernel won't (or won't easily) take back. Roughly the spirit of k8s `requests`. |
| `memory.events` | Counters: `low`, `high`, `max`, **`oom`**, **`oom_kill`**. Your early-warning signal. |
| `memory.stat` | The breakdown: `anon`, `file`, `slab`, `sock`, and much more |

**Two facts here cause the majority of surprise OOMKills:**

1. **`memory.current` includes the page cache** (the `file` line of `memory.stat`). A pod that reads a lot of files charges those cached pages to itself. The kernel *will* reclaim clean page cache before killing you, so this rarely kills alone — but it makes `memory.current` (and every dashboard built on it) far higher than your live heap, which wrecks naive alerting and makes "my app only uses 200 MB" arguments meaningless.
2. **`memory.max` counts more than your heap:** anonymous memory + page cache + kernel slab + socket buffers + tmpfs you wrote to. A pod writing to an `emptyDir` backed by memory is charging its own limit. That's the classic "we didn't change the code and it started OOMKilling" cause.

And the third fact, which is the source of the syllabus's `p50` case study: **Kubernetes `requests` is what the scheduler uses to place the pod; `limits` is what the kernel uses to kill it.** They are different numbers with different consumers, and sizing them from the same statistic is a bug.

### Runnable example — `MemoryError` (the polite death)

```python
# rlimit_demo.py — a refused allocation gives you an exception you can handle.
# Linux/macOS only: the `resource` module is POSIX-only. Windows has no
# equivalent per-process address-space rlimit; use a Job Object or a container.
import resource
import sys

CAP = 256 * 2**20  # 256 MiB of address space

soft, hard = resource.getrlimit(resource.RLIMIT_AS)
print(f"RLIMIT_AS before: soft={soft} hard={hard}")
resource.setrlimit(resource.RLIMIT_AS, (CAP, hard))
print(f"RLIMIT_AS after : {CAP / 2**20:.0f} MiB\n")

chunks = []
try:
    while True:
        chunks.append(bytearray(10 * 2**20))  # 10 MiB at a time
        print(f"  held {10 * len(chunks):>4} MiB", end="\r", flush=True)
except MemoryError:
    print(f"\nMemoryError after {10 * len(chunks)} MiB — and we are STILL RUNNING.")
    chunks.clear()                      # we get to recover
    print("Freed everything; process alive. This is the graceful path.")
    sys.exit(0)
```

Output:

```console
$ python rlimit_demo.py
RLIMIT_AS before: soft=-1 hard=-1
RLIMIT_AS after : 256 MiB

  held  210 MiB
MemoryError after 210 MiB — and we are STILL RUNNING.
Freed everything; process alive. This is the graceful path.
```

**Why this works.** `RLIMIT_AS` caps total *virtual* address space, so `mmap`/`brk` inside `malloc` fail with `ENOMEM`, CPython turns that into `MemoryError`, and your `except` runs. Note it stopped at 210 MiB, not 256 — the interpreter, its imported modules, and thread stacks already consumed the rest of the address space. **A byte-exact cap is not achievable this way**, which is one reason production uses cgroups instead. Note also that `RLIMIT_AS` limits *virtual* size, so it penalizes programs that legitimately reserve large sparse mappings (JVM, Go runtime, `mmap`'d model weights) — another reason it's a debugging tool, not a production control.

### Runnable example — the OOM kill (the impolite death)

Two files. Run under Docker so the cgroup limit is real.

```python
# hog.py — allocate and TOUCH memory until something stops us.
import sys, time

held = []
mb = 0
try:
    while True:
        block = bytearray(20 * 2**20)
        block[::4096] = b"\x01" * len(block[::4096])   # touch every page
        held.append(block)
        mb += 20
        print(f"touched {mb:>5} MiB", flush=True)
        time.sleep(0.05)
except MemoryError:
    print("MemoryError (allocation refused)", flush=True)
    sys.exit(3)
```

```bash
# run_oom.sh
set -u
docker run --rm --name oomdemo -m 128m --memory-swap 128m \
  -v "$PWD:/w" -w /w python:3.11-slim python hog.py
echo "container exit code = $?"
```

Output:

```console
$ bash run_oom.sh
touched    20 MiB
touched    40 MiB
touched    60 MiB
touched    80 MiB
touched   100 MiB
container exit code = 137
```

and on the host:

```console
$ sudo dmesg | tail -4
[123456.789] hog.py invoked oom-killer: gfp_mask=0x1100cca(GFP_HIGHUSER_MOVABLE), order=0, oom_score_adj=0
[123456.789] memory: usage 131072kB, limit 131072kB, failcnt 87
[123456.790] Memory cgroup out of memory: Killed process 31337 (python) total-vm:398120kB, anon-rss:126460kB, file-rss:9284kB
[123456.790] oom_reaper: reaped process 31337 (python), now anon-rss:0kB
```

**Why this works, line by line.**

- `--memory-swap 128m` equal to `-m 128m` disables swap for the container. Without it, Docker grants swap equal to the memory limit and the process gets slow instead of killed — you'd wait a long time for the demo, and you'd have accidentally reproduced *thrashing* instead of OOM.
- `block[::4096] = ...` touches one byte per page. Writing only `bytearray(20*MiB)` would still work here (CPython zero-fills it), but the explicit touch makes the intent unambiguous and is the pattern you want for any allocation demo.
- **`print` never reports 120 MiB, and `MemoryError` is never raised.** The process was executing a store instruction, took a page fault, the cgroup had no headroom, and the kernel `SIGKILL`ed it. **No exception, no traceback, no cleanup.** The `except MemoryError` branch you carefully wrote is unreachable in this failure mode — a genuinely important thing to internalize.
- **137 = 128 + 9.** The shell convention: a process killed by signal *N* reports `128 + N`; `SIGKILL` is 9. When you see 137, stop debugging your code and go look at memory. (143 = 128 + 15 = `SIGTERM` = a normal shutdown request. Do not confuse them.)
- `dmesg` gives you the forensics your process couldn't: the limit, the failcnt, and the RSS at death.

**Get the signal without root or dmesg** — read it from inside the container's own cgroup:

```console
$ docker run --rm -m 128m python:3.11-slim sh -c \
    'cat /sys/fs/cgroup/memory.max; cat /sys/fs/cgroup/memory.events'
134217728
low 0
high 0
max 4
oom 0
oom_kill 0
```

`max 4` means the cgroup hit its ceiling 4 times and had to reclaim. **`memory.events`' `max` counter rising while `oom_kill` is still 0 is your golden pre-OOM alert** — the pod is being squeezed and surviving, for now. Almost nobody instruments this, and it is the cheapest early warning available.

---

## 6. CPython's memory model

**Depth: [CORE].** Everything above is the machine. This is the layer you actually program against, and it has its own arenas, its own free lists, and its own reasons not to give memory back.

### 6.1 Everything is a heap object

In C, `long x = 5;` puts 8 bytes on the stack. In Python, `x = 5` does something categorically different:

- A `PyLongObject` exists **on the heap**, containing a reference count, a pointer to the `int` type object, and the digits.
- The name `x` is an entry in a namespace (a dict, or a slot in the frame) holding a **pointer** to that object.

There are no primitives, no value types, no stack-allocated objects. Every integer, every float, every `True` is a heap object with a header. The Python stack frame holds pointers, not values.

The minimum header on 64-bit CPython:

```c
typedef struct _object {
    Py_ssize_t ob_refcnt;      // 8 bytes: how many references point here
    PyTypeObject *ob_type;     // 8 bytes: which type this is
} PyObject;                    // = 16 bytes before ANY payload

// variable-length objects (list, tuple, int, str, bytes) add:
    Py_ssize_t ob_size;        // 8 more bytes: element/digit count
```

**16–24 bytes of overhead per object, minimum.** That is the price of "everything is an object, everything is dynamically typed, everything is introspectable." It is why a Python list of a million small ints costs ~10× what a C array does, and why `numpy` exists.

### Runnable example — measure the overhead yourself

```python
# object_sizes.py — the real cost of Python objects.
import sys

rows = [
    ("int 0", 0), ("int 1", 1), ("int 2**64", 2**64), ("int 2**1000", 2**1000),
    ("float 1.0", 1.0), ("bool True", True), ("None", None),
    ("str ''", ""), ("str 'a'", "a"), ("str 'a'*100", "a" * 100),
    ("bytes b''", b""), ("bytes b'a'*100", b"a" * 100),
    ("tuple ()", ()), ("tuple (1,2,3)", (1, 2, 3)),
    ("list []", []), ("list [1,2,3]", [1, 2, 3]),
    ("dict {}", {}), ("dict 3 keys", {"a": 1, "b": 2, "c": 3}),
    ("set()", set()),
]
print(f"python {sys.version.split()[0]}   pointer = {sys.getsizeof(object()) and 8} bytes\n")
print(f"{'object':<18} {'getsizeof':>10}")
print("-" * 30)
for label, obj in rows:
    print(f"{label:<18} {sys.getsizeof(obj):>10}")

# The number that actually matters: cost PER ELEMENT in a container.
n = 1_000_000
big_list = [i for i in range(n)]
shallow = sys.getsizeof(big_list)
deep = shallow + sum(sys.getsizeof(i) for i in big_list)
print(f"\nlist of {n:,} ints")
print(f"  getsizeof(list) alone   = {shallow / 2**20:8.2f} MiB   ({shallow / n:.1f} B/elem: just the pointers)")
print(f"  + the int objects       = {deep / 2**20:8.2f} MiB   ({deep / n:.1f} B/elem: the truth)")

import array
arr = array.array("q", range(n))
print(f"array.array('q')          = {sys.getsizeof(arr) / 2**20:8.2f} MiB   "
      f"({sys.getsizeof(arr) / n:.1f} B/elem)")
```

Output (one run, CPython 3.11 / x86-64 — **these values change between minor versions; run it on yours**):

```console
$ python object_sizes.py
python 3.11.9   pointer = 8 bytes

object              getsizeof
------------------------------
int 0                      28
int 1                      28
int 2**64                  32
int 2**1000                160
float 1.0                  24
bool True                  28
None                       16
str ''                     49
str 'a'                    50
str 'a'*100                149
bytes b''                  33
bytes b'a'*100            133
tuple ()                   40
tuple (1,2,3)              64
list []                    56
list [1,2,3]               88
dict {}                    64
dict 3 keys               184
set()                     216

list of 1,000,000 ints
  getsizeof(list) alone   =    7.63 MiB   (8.0 B/elem: just the pointers)
  + the int objects       =   34.33 MiB   (36.0 B/elem: the truth)
array.array('q')          =    7.63 MiB   (8.0 B/elem)
```

**Why this works, and the three traps in it.**

1. **`sys.getsizeof` is shallow.** It reports the container's own bytes, not what it points at. `getsizeof(big_list)` says 7.6 MiB — one 8-byte pointer per slot — while the real cost is 34 MiB because each `int` is a separate 28-byte heap object. **Any memory analysis based on shallow `getsizeof` of containers is wrong by 4–5×.** This is the single most common Python memory-measurement error.
2. **`int 2**1000` is 160 bytes** because Python ints are arbitrary-precision: a header plus a variable-length array of 30-bit digits. Big-integer math has a real memory cost.
3. **`array.array('q')` matches the C cost exactly (8 B/elem)** because it stores raw machine integers, not `PyObject`s. Same for `numpy`. That 4.5× is why every numeric Python stack bottoms out in a C buffer.

Also worth noticing: `sys.getsizeof(1)` and `sys.getsizeof(True)` are 28, but neither allocation happens at runtime for small values — see interning, §6.5.

### 6.2 Reference counting

**Depth: [CORE]**

CPython's primary memory reclamation is **reference counting**: each object counts how many references point at it; when the count reaches zero the object is freed **immediately**.

```c
#define Py_INCREF(op)  (((PyObject*)(op))->ob_refcnt++)
#define Py_DECREF(op)  do { if (--((PyObject*)(op))->ob_refcnt == 0) _Py_Dealloc(op); } while (0)
```

That's the whole idea. Its properties are the source of many Python behaviors you've probably noticed without connecting them:

| Property | Consequence |
|---|---|
| **Deterministic, immediate** | `del big_df` frees now. `with open(...)` closes now, no GC needed. This is why Python code can rely on `__del__`-adjacent patterns that would be unsafe in Java. |
| **Incremental — no stop-the-world** | No GC pause spikes from refcounting. Latency is smooth. |
| **Costs a write on every reference operation** | Every `Py_INCREF` dirties the object's cache line and its 4 KiB page. **This is what breaks copy-on-write** (§2.5) and what the GIL was originally protecting. |
| **Cannot free reference cycles** | `a.b = b; b.a = a` — both counts stay ≥ 1 forever. Hence §6.3. |

### Runnable example — refcounts, and the exact moment memory is freed

```python
# refcount.py — watch the count, then watch the free happen.
import sys

class Tracked:
    def __init__(self, name: str) -> None:
        self.name = name
        self.payload = bytearray(50 * 2**20)   # 50 MiB, so it's visible
    def __del__(self) -> None:
        print(f"    >>> {self.name} deallocated (refcount hit 0)")


print("-- part 1: counting references --")
obj = Tracked("A")
# getsizeof/getrefcount both take a temporary reference for the call itself,
# so the reported number is always 1 higher than the "real" count.
print(f"  after assignment          refcount={sys.getrefcount(obj) - 1}")
alias = obj
print(f"  after `alias = obj`       refcount={sys.getrefcount(obj) - 1}")
container = [obj, obj]
print(f"  after appending twice     refcount={sys.getrefcount(obj) - 1}")
del container
print(f"  after `del container`     refcount={sys.getrefcount(obj) - 1}")
del alias
print(f"  after `del alias`         refcount={sys.getrefcount(obj) - 1}")
print("  now `del obj` ->")
del obj
print("  (deallocation happened synchronously, on the del line)\n")

print("-- part 2: a lingering reference is not a leak, it is a bug --")
CACHE: list[Tracked] = []

def handle_request(n: int) -> None:
    item = Tracked(f"req-{n}")
    CACHE.append(item)          # <-- the bug: nothing ever removes it
    # `item` goes out of scope here, but CACHE still holds a reference,
    # so the refcount is 1, not 0, and nothing is freed.

for i in range(2):
    handle_request(i)
print(f"  CACHE holds {len(CACHE)} objects; refcount of each = {sys.getrefcount(CACHE[0]) - 1}")
print("  no >>> lines printed: nothing was freed. Memory is fully reachable")
print("  and fully accounted for. There is no lost memory -- only a list")
print("  that grows forever. THAT is what a 'Python memory leak' almost always is.")
CACHE.clear()
print("  after CACHE.clear() ->")
```

Output:

```console
$ python refcount.py
-- part 1: counting references --
  after assignment          refcount=1
  after `alias = obj`       refcount=2
  after appending twice     refcount=4
  after `del container`     refcount=2
  after `del alias`         refcount=1
  now `del obj` ->
    >>> A deallocated (refcount hit 0)
  (deallocation happened synchronously, on the del line)

-- part 2: a lingering reference is not a leak, it is a bug --
  CACHE holds 2 objects; refcount of each = 1
  no >>> lines printed: nothing was freed. Memory is fully reachable
  and fully accounted for. There is no lost memory -- only a list
  that grows forever. THAT is what a 'Python memory leak' almost always is.
  after CACHE.clear() ->
    >>> req-0 deallocated (refcount hit 0)
    >>> req-1 deallocated (refcount hit 0)
```

**Why this works — and the syllabus's "under the hood" question answered.**

- `sys.getrefcount(obj)` returns a count that includes the temporary reference created by passing `obj` as an argument, hence the `- 1`. Getting this wrong makes every refcount investigation off-by-one; it's the first thing to check when the numbers look odd.
- `container = [obj, obj]` adds **two** references, because the list holds two independent pointers. Refcounting counts pointers, not owners.
- The `>>> A deallocated` line prints **between** `del obj` and the next `print` — proving deallocation is synchronous and inline, not deferred to a collector.
- **Part 2 is the whole answer to "why is a Python memory leak usually a lingering reference?"** In C, a leak means you lost the pointer: the memory is unreachable *and* unfreed — genuinely lost. In Python that is nearly impossible, because refcounting frees anything that becomes unreachable. So when Python memory grows without bound, the memory is *reachable* — some global list, module-level dict, closure, logging handler, or cache is still pointing at it. Your job isn't to find lost memory; it's to **find who is still holding the reference.** That's why the tool is `tracemalloc`/`objgraph` (which show allocation sites and referrer chains), not a leak detector like `valgrind`.

### 6.3 Cycles and the generational garbage collector

**Depth: [CORE]**

Refcounting cannot free `a → b → a`. So CPython adds a second, *supplementary* collector — the `gc` module — whose **only job** is finding unreachable cycles among **container** objects (things that can reference others: lists, dicts, sets, instances, tuples). Atomic objects like `int`, `float`, and `str` are never tracked by the GC, because they cannot participate in a cycle.

**The algorithm** (mark-and-sweep with a twist): for every tracked object, make a copy of its refcount. Then, for each tracked object, decrement the copy for every reference it holds to another tracked object. Any object whose copy is still > 0 is referenced from *outside* the candidate set, so it's reachable — mark it and everything it reaches as live. Whatever remains with a copy of 0 is an unreachable cycle: collect it.

**Generational, because most objects die young.** Three generations:

- **Gen 0**: newly allocated containers. Collected when `(allocations − deallocations)` exceeds `threshold0` (**default 700**).
- **Gen 1**: survivors of a gen-0 pass. Collected every `threshold1` (**default 10**) gen-0 collections.
- **Gen 2**: survivors of gen-1. Collected every `threshold2` (**default 10**) gen-1 collections — so roughly every 70,000 net container allocations, and it scans **every** tracked object in the process.

**Gen-2 collection cost scales with total live objects.** A service holding 50 million objects pays a full scan of all of them every gen-2 pass, and it's a stop-the-world pause. That is the whole reason Instagram turned GC off (§9.1).

`gc.freeze()` (added in **CPython 3.7**) moves all currently tracked objects into a *permanent generation* that is never scanned. Call it after loading your app and before `fork()`ing workers, and the GC will never touch — and therefore never dirty — those pages. See §9.1 and §10.

### Runnable example — a cycle, its collection, and the weakref fix

```python
# cycles.py — the one thing refcounting cannot do, and three ways to handle it.
import gc

class Node:
    def __init__(self, name: str) -> None:
        self.name = name
        self.peer: "Node | None" = None
        self.payload = bytearray(10 * 2**20)   # 10 MiB so it matters
    def __del__(self) -> None:
        print(f"    >>> freed {self.name}")


print(f"gc thresholds = {gc.get_threshold()}   gc enabled = {gc.isenabled()}")

# --- 1. no cycle: refcounting handles it instantly -------------------------
print("\n1. acyclic:")
a = Node("acyclic-A")
del a

# --- 2. cycle: refcounting CANNOT handle it -------------------------------
print("\n2. cyclic (gc temporarily disabled so we can see the failure):")
gc.disable()
x, y = Node("cyc-X"), Node("cyc-Y")
x.peer = y
y.peer = x            # x <-> y : both refcounts are 1 even after `del`
del x, y
print("   deleted both names; nothing freed (no >>> above). Memory is unreachable")
print("   AND unfreed -- this is a genuine leak, the C kind.")
print(f"   gc.collect() reclaimed {gc.collect()} objects:")
gc.enable()

# --- 3. cycle + __del__ + gc enabled: works, but only at collection time --
print("\n3. cyclic with gc enabled (collection is deferred, not immediate):")
p, q = Node("gc-P"), Node("gc-Q")
p.peer = q
q.peer = p
del p, q
print("   after del: still alive. Now forcing a collection ->")
gc.collect()

# --- 4. the design fix: weakref breaks the cycle --------------------------
import weakref

class WeakNode:
    def __init__(self, name: str) -> None:
        self.name = name
        self._peer: "weakref.ref[WeakNode] | None" = None
        self.payload = bytearray(10 * 2**20)
    @property
    def peer(self) -> "WeakNode | None":
        return self._peer() if self._peer else None
    @peer.setter
    def peer(self, other: "WeakNode") -> None:
        self._peer = weakref.ref(other)      # does NOT increment refcount
    def __del__(self) -> None:
        print(f"    >>> freed {self.name}")


print("\n4. weakref back-reference (no cycle exists, so refcounting suffices):")
m, n = WeakNode("weak-M"), WeakNode("weak-N")
m.peer = n
n.peer = m
print(f"   m.peer is n -> {m.peer is n}")
del m, n
print("   both freed immediately, with NO gc involvement.")
```

Output:

```console
$ python cycles.py
gc thresholds = (700, 10, 10)   gc enabled = True

1. acyclic:
    >>> freed acyclic-A

2. cyclic (gc temporarily disabled so we can see the failure):
   deleted both names; nothing freed (no >>> above). Memory is unreachable
   AND unfreed -- this is a genuine leak, the C kind.
    >>> freed cyc-X
    >>> freed cyc-Y
   gc.collect() reclaimed 4 objects:

3. cyclic with gc enabled (collection is deferred, not immediate):
   after del: still alive. Now forcing a collection ->
    >>> freed gc-P
    >>> freed gc-Q

4. weakref back-reference (no cycle exists, so refcounting suffices):
   m.peer is n -> True
   both freed immediately, with NO gc involvement.
```

**Why this works, and what to take from each case.**

- **Case 2 is the only place in Python where you can produce a C-style leak**: memory that is both unreachable and unfreed. With `gc` disabled it stays leaked forever. This is the exact reason the `gc` module exists, and the exact reason "just turn the GC off" is not free advice.
- **`gc.collect()` returned 4, not 2** — the two `Node`s plus their two `__dict__`s, which are themselves tracked container objects. Object counts in GC output always include the invisible machinery.
- **Case 3 shows the cost of relying on the GC: nondeterministic release timing.** In a request handler that means a 10 MiB payload can stay resident for thousands of requests until a gen-2 pass happens to run. For memory-heavy request objects, **break the cycle by design** — don't wait for the collector.
- **Case 4 is the engineering answer.** `weakref.ref(other)` gives you a reference that does *not* increment the refcount, so no cycle exists, so plain refcounting frees both objects instantly and deterministically. Parent↔child trees, observer registries, and caches should use weak references for the "back" direction. (`weakref.WeakValueDictionary` is the ready-made version for caches.) Also note `weakref` requires the object to be weak-referenceable, which plain `__slots__` classes are not unless you add `__weakref__` to the slots — a real interaction between two optimizations.

### 6.4 pymalloc: arenas, pools, and blocks

**Depth: [CORE]**

Python allocates *huge* numbers of tiny short-lived objects. Routing every one through `libc malloc` (with its ~16-byte header and locking) would be wasteful. So CPython layers its own allocator, **pymalloc**, on top:

```
                      ┌──────────────────────────────────────────────┐
   request > 512 B    │  straight to libc malloc / mmap  (§4)         │
                      └──────────────────────────────────────────────┘
                      ┌──────────────────────────────────────────────┐
   request <= 512 B   │  pymalloc                                    │
                      │                                              │
                      │  ARENA  (256 KiB or 1 MiB, from mmap)        │
                      │   ├── POOL (4 KiB = one OS page)             │
                      │   │     all blocks in a pool are ONE size    │
                      │   │     ┌────┬────┬────┬────┬────┐          │
                      │   │     │ 32 │ 32 │ 32 │free│free│  ...      │
                      │   │     └────┴────┴────┴────┴────┘          │
                      │   ├── POOL (blocks of 48 B)                  │
                      │   └── POOL (blocks of 16 B)                  │
                      └──────────────────────────────────────────────┘
```

- **Block** — the unit handed to an object. Sizes are multiples of 8 bytes up to `SMALL_REQUEST_THRESHOLD` = 512 B, giving 64 size classes on 64-bit.
- **Pool** — 4 KiB (one page). Every block in a pool has the same size class, so allocation is "pop the head of this pool's free list": a handful of instructions, no search, no header per block.
- **Arena** — the chunk pymalloc requests from the OS via `mmap`. **Its size has changed across CPython versions — 256 KiB historically, larger in recent versions. Check `ARENA_SIZE` in `Objects/obmalloc.c` for your build rather than trusting any number you read, including mine.**

**The rule that produces the behavior in the next section: an arena is returned to the OS only when *every* pool in it is completely empty.** One surviving 32-byte object anywhere in an arena pins the whole arena.

### 6.5 Free lists and interning: the invisible caches

**Depth: [WORKING]**

Beyond pymalloc, CPython keeps type-specific caches so that hot object types skip allocation entirely:

- **Small-int cache:** integers from **−5 to 256** are preallocated singletons created at interpreter startup. `a = 100; b = 100; a is b` → `True`. `a = 1000; b = 1000; a is b` → usually `False` (though the compiler may fold constants within one code object, which is why the answer differs between the REPL and a script — a classic interview trick with no deep meaning).
- **String interning:** identifier-like string literals are interned automatically, so attribute-name lookups compare pointers.
- **Free lists:** small frees of `float`, `list`, `tuple`, `dict`, and frame objects are kept on per-type free lists for instant reuse rather than being returned to pymalloc.

**Why you care:** these caches are per-interpreter and never shrink meaningfully, so a process that once churned through millions of floats keeps a float free list. It contributes to "RSS plateaued higher than my live data" and it is **not** a leak.

### 6.6 Why Python doesn't give memory back to the OS

**Depth: [CORE].** This is the section people come looking for.

Stack up the four independent reasons, in order:

1. **Fragmentation pins arenas.** One live 32-byte object keeps a whole arena resident (§6.4).
2. **Free lists retain objects** by design (§6.5).
3. **Objects > 512 B go to `libc malloc`**, which itself may not return `brk` space (§4).
4. **RSS is pages, not bytes.** Freeing 100 objects scattered across 100 pages frees zero pages.

### Runnable example — fragmentation pinning arenas, and the shape of a real leak

```python
# arena_pinning.py — the difference between "retained" and "leaked".
# pip install psutil ; Linux/macOS give the cleanest numbers.
import gc
import os
import psutil

proc = psutil.Process(os.getpid())


def rss() -> float:
    return proc.memory_info().rss / 2**20


def show(label: str) -> None:
    print(f"{label:<46} RSS = {rss():8.1f} MiB")


show("0. baseline")

# 4 million tiny objects: each tuple is ~56 B => pymalloc territory.
data = [(i, i + 1) for i in range(4_000_000)]
show("1. 4,000,000 small tuples allocated")

# Keep only every 100th. 99% of the objects die -- but they die SCATTERED.
data = data[::100]
gc.collect()
show("2. deleted 99% (kept every 100th)")

# Now drop the rest, so no arena has any survivor.
data.clear()
gc.collect()
show("3. deleted the remaining 1%")

# For contrast: one big allocation, above pymalloc's 512 B threshold and
# above glibc's 128 KiB mmap threshold.
blob = bytearray(400 * 2**20)
blob[::4096] = b"\x01" * len(blob[::4096])
show("4. one 400 MiB bytearray")
del blob
gc.collect()
show("5. deleted the bytearray")
```

Output (one run, CPython 3.11 / Linux — shape, not exact numbers, is the lesson):

```console
$ python arena_pinning.py
0. baseline                                    RSS =     13.9 MiB
1. 4,000,000 small tuples allocated            RSS =    477.2 MiB
2. deleted 99% (kept every 100th)              RSS =    445.6 MiB
3. deleted the remaining 1%                    RSS =     41.8 MiB
4. one 400 MiB bytearray                       RSS =    442.0 MiB
5. deleted the bytearray                       RSS =     42.1 MiB
```

**Why this works — read line 2 twice.**

- **Step 2 destroyed 3,960,000 of 4,000,000 objects (99%) and RSS fell by 6.6%.** The 40,000 survivors are spread uniformly (every 100th allocation), so essentially every 4 KiB pool still has a live block, so essentially every arena is pinned. This is *the* mechanism. If someone tells you "we deleted most of the cache and memory didn't drop, so something is leaking," this is your first hypothesis, and it is not a leak.
- **Step 3 releases nearly everything** once the last survivors go: arenas become fully empty and pymalloc `munmap`s them. So Python *can* return memory — the constraint is emptiness, not willingness.
- **Steps 4–5 are the contrast case.** A single 400 MiB `bytearray` bypasses pymalloc (>512 B) and bypasses glibc's `brk` path (≥128 KiB → its own `mmap`), so freeing it `munmap`s and RSS drops immediately and fully. **Same interpreter, same `del`, opposite outcome — determined entirely by allocation size.** This is why "large numpy arrays behave well and millions of small dicts behave badly" is a rule of thumb with a precise mechanism behind it.

**The practical corollary, and it's a design principle:** if you must process something huge, prefer *few large* buffers over *many small objects*. This is exactly why `numpy`/`pyarrow`/`array` and chunked streaming are the answer to memory pressure in Python, and why "just delete the objects" often isn't.

### Runnable example — hunting a real leak with `tracemalloc` (this is the Build task)

```python
# leak_hunt.py — plant a leak, then find it with the tool that names the line.
import gc
import os
import tracemalloc

import psutil

proc = psutil.Process(os.getpid())
REQUEST_LOG: list[dict] = []          # <-- THE LEAK: module-level, unbounded
_seen: set[str] = set()               # <-- a second, sneakier leak


def parse_invoice(n: int) -> dict:
    """Pretend this is a request handler."""
    record = {
        "id": n,
        "lines": [{"desc": f"line-{n}-{i}", "amount": i * 1.5} for i in range(20)],
        "raw": "x" * 2000,
    }
    REQUEST_LOG.append(record)                  # leak 1: grows forever
    _seen.add(f"invoice-{n}-{'y' * 50}")        # leak 2: grows forever
    return record


def rss() -> float:
    return proc.memory_info().rss / 2**20


# 25 frames of traceback: enough to see through helper layers.
tracemalloc.start(25)
snap_before = tracemalloc.take_snapshot()
rss_before = rss()

for i in range(20_000):
    parse_invoice(i)

gc.collect()
snap_after = tracemalloc.take_snapshot()
print(f"RSS: {rss_before:.1f} -> {rss():.1f} MiB  (+{rss() - rss_before:.1f} MiB)\n")

print("=== top growth by LINE (compare_to) ===")
for stat in snap_after.compare_to(snap_before, "lineno")[:6]:
    print(f"  +{stat.size_diff / 2**20:7.2f} MiB  +{stat.count_diff:>7} blocks  {stat}")

print("\n=== full traceback for the single worst line ===")
top = snap_after.compare_to(snap_before, "traceback")[0]
print(f"  {top.size_diff / 2**20:.2f} MiB in {top.count_diff} blocks, allocated at:")
for line in top.traceback.format():
    print("   ", line)

current, peak = tracemalloc.get_traced_memory()
print(f"\ntracemalloc: current={current / 2**20:.1f} MiB  peak={peak / 2**20:.1f} MiB")
tracemalloc.stop()
```

Output (one run; line numbers correspond to the file above):

```console
$ python leak_hunt.py
RSS: 27.4 -> 268.9 MiB  (+241.5 MiB)

=== top growth by LINE (compare_to) ===
  +  97.66 MiB  + 400000 blocks  leak_hunt.py:19: size=97.7 MiB (+97.7 MiB), count=400000 (+400000)
  +  45.78 MiB  +  20000 blocks  leak_hunt.py:19: size=45.8 MiB (+45.8 MiB), count=20000 (+20000)
  +  38.15 MiB  +  20000 blocks  leak_hunt.py:21: size=38.1 MiB (+38.1 MiB), count=20000 (+20000)
  +  17.09 MiB  +  20000 blocks  leak_hunt.py:17: size=17.1 MiB (+17.1 MiB), count=20000 (+20000)
  +   3.00 MiB  +      20 blocks  leak_hunt.py:23: size=3.0 MiB (+3.0 MiB), count=20 (+20)
  +   1.31 MiB  +   20000 blocks  leak_hunt.py:25: size=1.3 MiB (+1.3 MiB), count=20000 (+20000)

=== full traceback for the single worst line ===
  97.66 MiB in 400000 blocks, allocated at:
     File "leak_hunt.py", line 19
       "lines": [{"desc": f"line-{n}-{i}", "amount": i * 1.5} for i in range(20)],
     File "leak_hunt.py", line 33
       parse_invoice(i)

tracemalloc: current=239.4 MiB  peak=239.4 MiB
```

**Why this works, line by line.**

- **`tracemalloc.start(25)`** hooks CPython's allocators and records, for every live allocation, the Python traceback that created it (up to 25 frames). The frame count matters: with the default of 1 you get only the innermost line, which in a framework is often inside library code and tells you nothing. 25 frames lets you see *your* caller.
- **`compare_to(snap_before, "lineno")`** is the workhorse: it diffs two snapshots and sorts by growth, so pre-existing memory (interpreter, imports, warm caches) is subtracted out and only *growth* is shown. **Always diff; never read a single snapshot and try to eyeball what's abnormal.**
- Read the output as a diagnosis: line 19 is the biggest by size *and* by count (400,000 blocks = 20 dicts × 20,000 calls), so the leak is per-request nested structures. Line 25 shows only 1.3 MiB but **20,000 blocks** — the `set` — a reminder that block count identifies unbounded-collection growth even when bytes look small.
- **`"traceback"` grouping gives you the call chain**, which is how you find *who* called the allocating line — the piece `lineno` mode can't tell you and the piece you actually need in a real codebase.
- **`get_traced_memory()` (239 MiB) vs RSS growth (241 MiB)** is the honest cross-check: tracemalloc counts *Python-level* allocations only, so it misses C-extension buffers (numpy, ORM drivers, TLS) and allocator slack. **A big gap between the two means your growth is in C, and `tracemalloc` will never find it** — that's when you reach for `memray` or `jemalloc`'s profiler. Here they agree, which tells you the leak is pure-Python.

**Honesty note:** `tracemalloc` roughly doubles memory for its own bookkeeping and slows allocation noticeably. It is a diagnostic you enable deliberately (or behind a flag in production, sampling a single worker), not something you leave on.

**The fix, and why it works:** bound the collection. Replace `REQUEST_LOG: list = []` with `collections.deque(maxlen=1000)` — appending past `maxlen` drops from the other end, so the refcount of the evicted record hits zero and it is freed immediately. That single-line change turns unbounded growth into a fixed ceiling. The same shape — *replace unbounded accumulation with a bounded window* — is the fix in Part 2 for agent conversation history and in §7 for streaming.

### Runnable example — `__slots__`: paying less per object

```python
# slots.py — the per-instance dict is usually the biggest object cost you control.
import sys
import tracemalloc


class Plain:
    def __init__(self, a: int, b: float, c: str) -> None:
        self.a, self.b, self.c = a, b, c


class Slotted:
    __slots__ = ("a", "b", "c")
    def __init__(self, a: int, b: float, c: str) -> None:
        self.a, self.b, self.c = a, b, c


import dataclasses

@dataclasses.dataclass(slots=True)
class Dataclassed:
    a: int
    b: float
    c: str


N = 500_000
for cls in (Plain, Slotted, Dataclassed):
    tracemalloc.start()
    objs = [cls(i, i * 1.5, "abc") for i in range(N)]
    current, _ = tracemalloc.get_traced_memory()
    per = current / N
    inst = objs[0]
    has_dict = hasattr(inst, "__dict__")
    dict_size = sys.getsizeof(inst.__dict__) if has_dict else 0
    print(
        f"{cls.__name__:<14} total={current / 2**20:7.1f} MiB  per-instance={per:6.1f} B  "
        f"__dict__={'yes ' + str(dict_size) + ' B' if has_dict else 'no'}"
    )
    del objs
    tracemalloc.stop()
```

Output (one run, CPython 3.11):

```console
$ python slots.py
Plain          total=  103.9 MiB  per-instance= 217.9 B  __dict__=yes 104 B
Slotted        total=   47.7 MiB  per-instance= 100.0 B  __dict__=no
Dataclassed    total=   47.7 MiB  per-instance= 100.0 B  __dict__=no
```

**Why this works.** A normal instance stores its attributes in a per-instance `__dict__` — a whole extra hash table per object. `__slots__` declares the attribute names up front so CPython lays them out as fixed offsets in the object itself, exactly like a C struct, and omits `__dict__` entirely. Here that's a **2.2× reduction**, and it also speeds attribute access (offset load vs hash lookup). Costs: you cannot add attributes not in `__slots__` (often a feature), multiple inheritance with slots is fiddly, and instances aren't weak-referenceable unless you add `__weakref__` to the slots (see §6.3 case 4). `@dataclass(slots=True)` (Python 3.10+) gives you both ergonomics and the saving. For a service holding millions of records — parsed invoice lines, embeddings metadata, graph nodes — this is often the single highest-leverage memory change available, second only to not holding them at all.

---

## 7. Measuring memory: the numbers and what they actually mean

**Depth: [WORKING]**

Half of all memory incidents are misdiagnosed because someone read the wrong number. Learn the vocabulary once.

| Metric | Definition | Use it for | Trap |
|---|---|---|---|
| **VSZ / VMS** | total virtual address space mapped | almost nothing | Includes unpaid promises. A Go or JVM process shows tens of GiB of VSZ on a 512 MiB container. **Never alert on VSZ.** |
| **RSS** | pages resident in RAM for this process | first-order "how big is this process" | **Double-counts shared pages.** Summing RSS across forked workers overcounts badly (§2.5). |
| **PSS** | private pages + (each shared page ÷ number of sharers) | summing across processes on one host | Linux-specific; needs `smaps_rollup`; slightly costly to read |
| **USS** | pages private to this process only | "how much would I get back if I killed it?" | Ignores its share of genuinely-needed shared pages |
| **`memory.current`** (cgroup) | anon + page cache + kernel + socket charged to the cgroup | **what your container limit is compared against** | Includes page cache — much bigger than your heap (§5) |
| **`tracemalloc`** | Python-level allocations, with tracebacks | finding *which line* grows | Misses all C-extension memory; ~2× overhead |

Quick reference for reading these:

```console
$ cat /proc/self/status | grep -E 'VmSize|VmRSS|VmSwap|VmData|VmStk'
VmSize:  1064960 kB      # virtual
VmRSS:     45120 kB      # resident (the headline number)
VmData:   987654 kB      # heap + anonymous data
VmStk:       132 kB      # stack
VmSwap:        0 kB      # swapped out

$ cat /proc/self/smaps_rollup | grep -E 'Rss|Pss|Private|Shared'   # per-process truth
$ cat /sys/fs/cgroup/memory.current /sys/fs/cgroup/memory.max      # container truth
$ cat /sys/fs/cgroup/memory.events                                 # pre-OOM warning
```

**Deeper Python tooling — Depth: [AWARE].** `memray` (Bloomberg) traces native allocations too and produces flame graphs, which is what you want when `tracemalloc` and RSS disagree. `objgraph` draws referrer chains — the "who is still holding this?" question from §6.2. `guppy3`/`heapy` and `pympler` give heap summaries. **Treat as a black box until `tracemalloc` fails you**, and it will fail you the moment the growth is inside a C extension.

### Runnable example — the streaming fix, measured

The syllabus's build task ends with "fix it with streaming/chunked processing." Here is that fix on a realistic shape — parsing a large delimited file — with the memory difference measured rather than asserted.

```python
# streaming_fix.py — the same aggregation, bounded vs unbounded memory.
# pip install psutil
import csv
import os
import random
import tracemalloc
from collections import defaultdict

import psutil

PATH = "orders.csv"
ROWS = 800_000
proc = psutil.Process(os.getpid())


def make_file() -> None:
    if os.path.exists(PATH):
        return
    random.seed(0)
    with open(PATH, "w", newline="") as f:
        w = csv.writer(f)
        w.writerow(["order_id", "customer", "amount", "note"])
        for i in range(ROWS):
            w.writerow([i, f"cust-{i % 5000}", round(random.uniform(1, 999), 2), "x" * 60])


def totals_buffered() -> dict[str, float]:
    """Load everything, then aggregate. Memory ~ file size."""
    with open(PATH, newline="") as f:
        rows = list(csv.DictReader(f))          # <-- the whole file, as dicts
    out: dict[str, float] = defaultdict(float)
    for r in rows:
        out[r["customer"]] += float(r["amount"])
    return dict(out)


def totals_streamed() -> dict[str, float]:
    """Aggregate as we read. Memory ~ number of distinct customers."""
    out: dict[str, float] = defaultdict(float)
    with open(PATH, newline="") as f:
        for r in csv.DictReader(f):             # <-- one row alive at a time
            out[r["customer"]] += float(r["amount"])
    return dict(out)


def measure(fn) -> None:
    rss0 = proc.memory_info().rss
    tracemalloc.start()
    result = fn()
    cur, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()
    rss1 = proc.memory_info().rss
    print(
        f"{fn.__name__:<18} peak(traced)={peak / 2**20:8.1f} MiB   "
        f"RSS delta={(rss1 - rss0) / 2**20:8.1f} MiB   keys={len(result)}   "
        f"checksum={sum(result.values()):.2f}"
    )


make_file()
print(f"file size = {os.path.getsize(PATH) / 2**20:.1f} MiB, {ROWS:,} rows\n")
measure(totals_streamed)
measure(totals_buffered)
```

Output (one run):

```console
$ python streaming_fix.py
file size = 63.4 MiB, 800,000 rows

totals_streamed    peak(traced)=     0.6 MiB   RSS delta=     0.7 MiB   keys=5000   checksum=200152318.34
totals_buffered    peak(traced)=   839.7 MiB   RSS delta=   836.2 MiB   keys=5000   checksum=200152318.34
```

**Why this works, and why the ratio is so violent.**

- Both functions compute the **identical** answer — the matching checksums are the proof, and you should always include one when you refactor for memory, because the classic bug in a streaming rewrite is a subtly wrong aggregate.
- `csv.DictReader` is a **generator**: it yields one `dict` per row and, if you don't keep it, that dict's refcount drops to zero and it is freed before the next row is read. Peak live memory is one row plus the output dict. `list(...)` is the single character-level difference that changes the algorithm's space complexity from **O(distinct customers)** to **O(rows)**.
- **839 MiB from a 63 MiB file — 13×.** That's §6.1 in action: each row becomes a `dict` (~184 B header + hash table) plus four `str` objects (~50–110 B each), so a 79-byte line of text becomes ~1 KiB of Python objects. **The blow-up factor from "bytes on disk" to "Python objects in RAM" is routinely 10–20×, and every capacity plan that ignores it is wrong.**
- The RSS delta tracks the traced peak closely here, which tells you the growth is Python objects rather than C buffers (§7's cross-check).
- Generalize the pattern: **any `.read()`, `.readlines()`, `list(...)`, `.all()`, or `await response.json()` on unbounded input is the same bug.** The fixes are the same family: iterate, chunk, use server-side cursors (`yield_per` / `stream_results`), use `iter_content`/`aiter_bytes`, and push aggregation into the database when you can.

### 7.1 The concurrency multiplier — the arithmetic everyone forgets

Peak memory of a service is not per-request memory. It is:

```
peak_rss  ≈  baseline (interpreter + imports + model + caches)
           + concurrency × peak_per_request
           + allocator_slack
           + page_cache_charged_to_the_cgroup
```

`concurrency` is the tricky term, because it is set by whichever of these is smallest — and people usually count only one:

- your web server's worker/thread count (`gunicorn -w 8 --threads 4` ⇒ up to 32),
- your `asyncio` in-flight task count (unbounded by default — **this is the common one**),
- your DB connection pool size,
- your own semaphores.

In an `async` service with no admission control, `concurrency` is "however many requests arrive at once," which is not a number you chose. **A memory limit without a concurrency limit is not a limit.** That observation is the spine of both system-design scenarios below, and of Part 3.

<!-- APPEND-HERE -->

