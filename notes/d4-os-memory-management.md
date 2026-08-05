# Day 4 — The OS III: Memory Management (Virtual Memory, Pages, Stack vs Heap, the OOM Killer, and Python's Heap)

> **Framing.** Day 2 gave each process its own isolated box; Day 3 put many threads inside one box sharing its memory and showed how that sharing races. Today we open the box's memory itself and ask the question every one of those threads secretly depends on: *when a program says `x = [0] * 1_000_000`, where do those bytes actually live, who hands them out, and what happens when you ask for more than the machine has?* The answer is one of the most beautiful lies in computing — **virtual memory**: the OS convinces every process that it alone owns a vast, contiguous, private span of RAM, while behind the curtain it hands out physical memory in 4 KB **pages**, on demand, lazily, and reclaims it the instant you run out (the **OOM killer**). You will learn what a **page fault** physically triggers, why the **stack** frees itself but the **heap** does not, why a Python object costs ~28 bytes instead of 8, why a Python "memory leak" is almost never lost memory but a **lingering reference**, and why Python so often refuses to give freed memory back to the operating system.
>
> **Who it's for.** Someone who has never heard "virtual memory," thinks a pointer is a physical RAM address, believes `del x` frees memory to the OS, and has never watched a container get killed at 3 a.m. for using 2 MB too much. We build from zero. Depends on Day 1 (the latency pyramid: RAM ≈ 100 ns, SSD/swap ≈ 100 µs — a 1000× cliff that makes swap catastrophic) and Day 2 (a process is an address space + descriptors); it is cross-referenced, not re-explained.
>
> **The ONE idea that unites the backend and agentic layers:** *memory is a finite physical budget you are only lent the illusion of owning — and the discipline of living inside the real budget is identical whether you are sizing a container, hunting a Python reference leak, or trimming an agent's conversation history and KV cache.* A backend worker that never trims a growing global `dict` and an AI agent that re-sends an ever-longer conversation every turn are **the same bug**: state that should have been bounded, growing without a ceiling, until the process crosses a hard limit and the OOM killer ends it — no exception, no stack trace, just a `SIGKILL` and a restart. Get it wrong in a backend and a pod CrashLoopBackOffs; get it wrong in an agent and every user conversation on that worker dies mid-sentence, *and* the token bill grows quadratically on the way down. This is the day "out of memory" stops being magic.

**A note on platform.** Virtual memory, pages, `/proc`, `RLIMIT_AS`, `cgroups`, and the OOM killer are **Linux** concepts. Windows has analogous machinery (the Memory Manager, working sets, the pagefile, Job Objects) with different names. Every runnable example below is written for **Linux** — run them in WSL2 (`wsl --install`, then `wsl`), a container (`docker run -it --rm -m 256m -v "$PWD":/w -w /w python:3.12 bash` — note the `-m 256m` memory cap, which we'll use), or any Linux VM. Where Windows or CPython specifics diverge, I say so explicitly.

**A note on Python's GC and free-threading.** CPython's **reference counting** is [CORE] today because it dictates *when* memory is freed and *why* CoW breaks. The **cyclic garbage collector** and **pymalloc** are [CORE] at the "know the mechanism and its failure modes" level. The full GIL story is Day 20; the KV-cache internals of transformer inference are Day 63 — both are cross-referenced, not opened here.

**Reading order.** Part 1 builds the OS memory machinery (virtual memory → pages/TLB → page faults/swap → stack vs heap → OOM killer → Python's heap). Part 2 builds the agent's memory model on top of it — the context window as a RAM-like budget, the KV cache as the memory-bound bottleneck, and the agent "memory leak" of unbounded history — treating the backend as a black box. Part 3 is where a backend worker and an agent share **one physical memory budget** and get OOM-killed together.

---

# PART 1 — BACKEND

## 1.1 Virtual memory — the illusion that every process owns all of RAM

**Depth: [CORE]**

### Intuition

Run two Python programs at once. Both can print a pointer-ish address near `0x7fff…` for their stack, or malloc something and see it land at, say, address `0x55e4…`. If both processes literally used the same physical RAM addresses, they would instantly corrupt each other — process A's write to `0x55e4…` would clobber process B's data. They don't. **Each process gets its own private map of addresses — a virtual address space — and the same virtual address in two processes points at completely different physical RAM.**

That is virtual memory: **a layer of indirection between the addresses a program uses (virtual) and the actual RAM cells (physical).** Every memory access your program makes goes through a hardware translation — virtual address → physical address — performed by the CPU's **MMU** (Memory Management Unit) using tables the OS maintains. The program never sees, and cannot name, a physical address at all.

**Why virtual memory exists / what came before.** In the earliest systems, programs used physical addresses directly. Three disasters followed, and virtual memory solves all three at once:

1. **No isolation.** Any program could read or write any other program's memory (and the OS's), by accident or malice. Virtual memory gives each process a *separate* address space — process A literally *cannot express* an address that lands in process B's RAM. This is the enforcement mechanism behind Day 2's process isolation and Day 3's "one thread crash takes the process, but not other processes."
2. **No relocation / fragmentation hell.** A program compiled to load at physical address `0x1000` couldn't run if another program was already there. With virtual memory, every program can *believe* it starts at the same virtual address; the OS maps those identical virtual addresses to whatever physical pages happen to be free — scattered all over RAM — and the program is none the wiser.
3. **No overcommit / demand loading.** Physical RAM is finite (say 16 GB), but a process can have a *virtual* address space far larger (48 bits ≈ 256 TB on x86-64) and allocate more than physically exists, because the OS only hands out real RAM for the parts you actually *touch* (§1.3, demand paging), and can spill cold parts to disk (swap). "Allocate 10 GB" costs almost nothing until you write to it.

The single sentence: **virtual memory decouples what a program asks for from what physically exists, and that decoupling buys isolation, relocation, and lazy/over-allocation all at once.**

### Analogy — hotel room numbers vs the building's physical rooms

A hotel gives every *guest* (process) their own booklet numbering "your rooms" 1, 2, 3, … Guest A's "room 5" and guest B's "room 5" are physically different rooms in the building; the front desk (the OS/MMU) keeps a private lookup table per guest mapping *their* room 5 to an actual physical room (say, 512 for A, 907 for B). Guests never learn the real room numbers, so guest A can't wander into guest B's room even if they wanted to — A can only say "my room 5," and the desk sends them to 512. The hotel can also promise each guest "a thousand rooms" while owning only 300 physical rooms, because guests rarely occupy more than a few at once (overcommit), and can move a guest's rarely-used luggage into the basement storeroom (swap) to free a room.

**Where the analogy breaks (non-negotiable to state):** two ways.
1. **The translation is per-access and hardware-fast, not a front-desk phone call.** Every single memory read/write is translated by the MMU in a few *nanoseconds*, cached in the TLB (§1.2). If it were as slow as asking a receptionist, programs would run millions of times slower. The analogy makes translation feel like an occasional lookup; in reality it happens *billions of times per second*, which is exactly why the TLB exists and why TLB misses matter (§1.2).
2. **Rooms are all-or-nothing; pages are uniform and shareable.** The hotel maps whole rooms of arbitrary size; the MMU maps fixed **4 KB pages** (§1.2), and — the part with no hotel equivalent — two guests' booklets can point *the same* physical page (shared read-only code, or copy-on-write after `fork`, Day 2). "Two different guests, same physical room, and it's fine because nobody writes" has no clean hotel analog, yet it is the mechanism behind shared libraries and the Instagram CoW case study (§1.9).

### Worked example — a huge virtual allocation that costs almost no physical RAM

Allocate a **1 GB** anonymous memory region. Measure the process's *virtual* size (`VmSize`) and *resident* size (`VmRSS` — physical RAM actually used) before touching it, after touching one page, and after touching all of it.

```
   step                         VmSize (virtual)   VmRSS (physical, resident)
   ────────────────────────────────────────────────────────────────────────
   after mmap(1 GB)             +1024 MB           +0.0 MB     ← promised, not delivered
   after touching 1 page (4 KB) +1024 MB           +0.004 MB   ← one page faulted in
   after touching every page    +1024 MB           +1024 MB    ← now it's really resident
```

The gap between `VmSize` (what you *asked for*) and `VmRSS` (what you *physically use*) is the whole point: **virtual memory is a promise; physical RAM is delivered lazily, one page at a time, only when you touch it** (§1.3). This is why `top`/`htop` show a "VIRT" column far larger than "RES" for almost every process — the virtual number is mostly unbacked promises.

### Runnable example — proving virtual ≫ resident, and demand paging in action

```python
# virt_vs_res.py — Linux. stdlib only (uses /proc and mmap).
# Run:  python3 virt_vs_res.py
import mmap, os

PAGE = os.sysconf("SC_PAGE_SIZE")          # 4096 on x86-64 (verify: getconf PAGE_SIZE)

def mem_mb():
    """Return (VmSize, VmRSS) in MB from /proc/self/status (Linux)."""
    vsz = rss = 0.0
    with open("/proc/self/status") as f:
        for line in f:
            if line.startswith("VmSize:"): vsz = int(line.split()[1]) / 1024
            if line.startswith("VmRSS:"):  rss = int(line.split()[1]) / 1024
    return vsz, rss

SIZE = 1024 * 1024 * 1024                    # 1 GiB
print(f"page size = {PAGE} bytes")
print(f"start                : VmSize={mem_mb()[0]:8.1f} MB  VmRSS={mem_mb()[1]:8.1f} MB")

# Anonymous mapping: 1 GB of virtual address space, backed by NOTHING yet.
buf = mmap.mmap(-1, SIZE)                    # -1 fd = anonymous (not a file)
print(f"after mmap(1 GB)     : VmSize={mem_mb()[0]:8.1f} MB  VmRSS={mem_mb()[1]:8.1f} MB  <- virtual jumped, resident did NOT")

buf[0] = 1                                   # touch ONE byte -> faults in ONE page
print(f"after touching 1 page: VmSize={mem_mb()[0]:8.1f} MB  VmRSS={mem_mb()[1]:8.1f} MB  <- +one page resident")

for off in range(0, SIZE, PAGE):             # touch every page -> fault them all in
    buf[off] = 1
print(f"after touching ALL   : VmSize={mem_mb()[0]:8.1f} MB  VmRSS={mem_mb()[1]:8.1f} MB  <- now physically resident")

buf.close()
```

Actual output (a 4-core Linux box with plenty of RAM; your exact numbers vary):

```
# -> page size = 4096 bytes
# -> start                : VmSize=    28.7 MB  VmRSS=    12.4 MB
# -> after mmap(1 GB)     : VmSize=  1052.7 MB  VmRSS=    12.4 MB  <- virtual jumped, resident did NOT
# -> after touching 1 page: VmSize=  1052.7 MB  VmRSS=    12.4 MB  <- +one page resident
# -> after touching ALL   : VmSize=  1052.7 MB  VmRSS=  1036.5 MB  <- now physically resident
```

**Why this works, line by line.**

- `mmap.mmap(-1, SIZE)` with fd `-1` asks the kernel for a **1 GB anonymous mapping** — a range of *virtual* addresses, backed by zero physical pages. The kernel just records the promise in the process's page tables (§1.2) marked "demand-zero." `VmSize` jumps by ~1 GB; `VmRSS` (physical) does **not move**. That gap is the illusion of §1.1: you "have" a gigabyte you don't physically occupy.
- `buf[0] = 1` writes one byte. The CPU tries to translate that virtual address, finds no physical page mapped → a **page fault** (§1.3) → the kernel allocates *one* physical 4 KB page, zeroes it, maps it, and resumes. `VmRSS` rises by ~4 KB (rounding hides it in the MB display; on some runs you'll see +0.0 because it's below display precision — the *ALL* step is the unmistakable proof).
- The loop touches one byte per page across the whole gigabyte, forcing a page fault per page, and *now* `VmRSS` climbs to ~1 GB — the physical RAM was delivered lazily, page by page, exactly as demanded. **This is demand paging (§1.3) made visible.** If you had `SIZE = 100 GB` on a 16 GB machine, the `mmap` would still succeed (overcommit) and only crash when you *touched* more than physical RAM + swap could hold — the OOM killer (§1.5) fires on *use*, not on *request*.
- **Honesty caveat:** `/proc/self/status` `VmRSS` counts pages resident *for this process*; shared pages are counted too, so it's an over-estimate of "private" memory. For precise accounting use `smaps_rollup` (`Pss` = proportional set size, which splits shared pages fairly). `VmRSS` is the right first tool; `Pss` is the honest one when pages are shared (§1.9's CoW story).

**Under the hood.** The kernel represents each mapping as a **VMA** (virtual memory area — a `vm_area_struct`: start, end, permissions, backing). `mmap` just adds a VMA; no page tables are filled yet. On first touch, the CPU raises a page fault, the kernel's fault handler consults the VMA, sees "anonymous, demand-zero," grabs a free physical frame from the buddy allocator, zeroes it (security: you must never see another process's old data), installs the mapping in the page table, and retries the instruction. Linux **overcommits** by default (`vm.overcommit_memory=0`, heuristic) — it says yes to allocations exceeding RAM+swap, betting you won't touch it all; the bet is settled by the OOM killer (§1.5). Primary sources: `mmap(2)`, `proc(5)` (the `/proc/[pid]/status` and `smaps` fields), `Documentation/vm/overcommit-accounting` in the Linux tree.

**Deliberate stop.** I am not opening the buddy allocator or the slab allocator (how the kernel manages *its own* free physical frames) — you now know the shape (virtual VMAs, physical frames handed out on fault) precisely enough for everything today. The page-table structure itself is §1.2.

---

## 1.2 Pages, page tables, and the TLB — how one address becomes another

**Depth: [CORE]**

### Intuition

Virtual memory needs a *translation table*: for every virtual address a program uses, "which physical frame does it live in?" If the OS tracked this per-*byte*, the table would be as big as memory itself — absurd. So memory is chopped into fixed **pages** (4 KB on x86-64) and physical RAM into equal-sized **frames**, and the table maps *page → frame*, not byte → byte. A 64-bit virtual address splits into a **page number** (high bits — which page) and an **offset** (low 12 bits — which byte within the 4 KB page). Translation replaces the page number with a frame number and keeps the offset.

The table that does this is the **page table**. Because a full flat table for a 48-bit space would be enormous, it's a **multi-level tree** (4 levels on x86-64: the CPU walks PML4 → PDPT → PD → PT), so only the branches you actually use consume space. Walking four levels of table *on every memory access* would be ruinously slow — so the CPU caches recent translations in the **TLB** (Translation Lookaside Buffer), a tiny associative cache of page→frame mappings. A **TLB hit** (the common case) translates in ~1 cycle; a **TLB miss** forces the multi-level page walk (~tens to 100+ cycles).

### Analogy — a book's index (chapter → page) plus a sticky-note shortlist

Finding a topic in a huge book: you don't scan every page. You use the **index** (topic → page number) — that's the page table (name → location). But flipping to the index every single time is slow, so you keep a few **sticky notes** on the topics you're looking up right now — that's the TLB. Hit a sticky note and you jump straight there; miss and you go back to the index (the page walk).

**Where the analogy breaks:** the book's index is one flat list; the page table is a **multi-level tree**, and — critically — the TLB is *tiny and finite* (typically dozens to a few thousand entries). A program that jumps randomly across a huge memory span "uses up" the sticky notes faster than they help, so its TLB miss rate soars and it stalls on page walks — the exact reason **random access is slower than sequential** beyond just cache effects (Day 1). The book analogy has no "you only get 64 sticky notes and jumping around thrashes them." That thrash is also why a **context switch** to a different *process* flushes the TLB (§1.3, Day 3 §1.5) — the new process's sticky notes are meaningless, so you start cold.

### Worked example — decomposing a virtual address, and the cost of a miss

A virtual address `0x00007f3c_a01b_2064` on x86-64 with 4 KB pages:

```
   64-bit virtual address, 4 KB pages (offset = low 12 bits):
   ┌───────────────── page number (bits 12+) ─────────────────┬── offset (12 bits) ──┐
   │      ... used to walk PML4 → PDPT → PD → PT ...           │   0x064 = byte 100   │
   └──────────────────────────────────────────────────────────┴──────────────────────┘
   offset 0x064 = 100  →  this address is the 100th byte inside its 4 KB page.
   Translation replaces the page number with a physical FRAME number; the offset (100)
   is copied unchanged — byte 100 of the virtual page is byte 100 of the physical frame.

   Cost of the lookup:
     TLB hit  : ~1 cycle           (translation cached)            ← ~99%+ of accesses
     TLB miss : ~10–100+ cycles    (walk 4 levels of page table)   ← the tax on random access
     +page fault (not in RAM at all): ~µs–ms (§1.3, disk)          ← the catastrophe
```

The three-tier cost — cached, page-walk, page-fault — is a memory-shaped echo of Day 1's latency pyramid, and it's why "keep hot data on few pages, accessed sequentially" is a real performance lever (huge pages, arena allocators like pymalloc §1.6, and structure-of-arrays layouts all attack the TLB/page cost).

### Runnable example — page size, and resident memory measured in pages

```python
# pages.py — Linux. stdlib only.
# Run:  python3 pages.py
import os, resource

PAGE = os.sysconf("SC_PAGE_SIZE")                       # bytes per page (4096 typical)
print(f"page size            : {PAGE} bytes ({PAGE//1024} KB)")

# getrusage reports peak RSS in KB on Linux (in bytes on macOS — a real portability trap).
peak_rss_kb = resource.getrusage(resource.RUSAGE_SELF).ru_maxrss
print(f"peak RSS             : {peak_rss_kb} KB  ({peak_rss_kb*1024//PAGE} pages)")

# Allocate ~40 MB and show it in pages. bytearray is a contiguous heap buffer (§1.4).
big = bytearray(40 * 1024 * 1024)
big[::PAGE] = bytes(len(big[::PAGE]))                   # touch one byte per page -> fault them in
peak_rss_kb = resource.getrusage(resource.RUSAGE_SELF).ru_maxrss
print(f"peak RSS after 40 MB : {peak_rss_kb} KB  ({peak_rss_kb*1024//PAGE} pages)")
print(f"40 MB is {40*1024*1024//PAGE} pages of {PAGE} bytes each")
```

Actual output (Linux; `ru_maxrss` is KB on Linux — see caveat):

```
# -> page size            : 4096 bytes (4 KB)
# -> peak RSS             : 12928 KB  (3232 pages)
# -> peak RSS after 40 MB : 54020 KB  (13505 pages)
# -> 40 MB is 10240 pages of 4096 bytes each
```

**Why this works, line by line.**

- `os.sysconf("SC_PAGE_SIZE")` asks the kernel the page size — **4096 bytes** on virtually every x86-64 Linux. Everything about memory (allocation granularity, fault granularity, `mmap` alignment) is quantized to this number, so it's worth having it in your head: **one page = 4 KB = the unit the OS hands out physical RAM in.**
- `resource.getrusage().ru_maxrss` is the process's **peak resident set** — the most physical RAM it held at once. Dividing by page size converts to *pages*, the OS's native unit. **Honesty caveat / portability trap:** `ru_maxrss` is in **kilobytes on Linux** but **bytes on macOS** — the same code reports a 1024× different number across platforms. This bites people constantly; when in doubt, read `/proc/self/status` `VmRSS` (always KB) on Linux, as in §1.1.
- Allocating a 40 MB `bytearray` and touching one byte per page forces each page resident (§1.3); RSS climbs by ~40 MB ≈ 10,240 pages. The point: **memory is accounted, faulted, and reclaimed in whole pages** — you never get half a page, and a 1-byte object still dirties a full 4 KB page when it's the first thing on a fresh page.

**Under the hood.** On x86-64 the MMU walks a 4-level page table rooted at the physical address in control register **CR3** (which is reloaded on every process switch — that's what makes a process-switch TLB flush necessary, Day 3 §1.5). Each level indexes 9 bits of the address into a 512-entry table; the leaf entry (PTE) holds the physical frame number plus flags (present, writable, user/kernel, dirty, accessed, no-execute). The **TLB** caches leaf translations; hardware refills it automatically on a miss by doing the walk. **Huge pages** (2 MB or 1 GB) exist precisely to reduce TLB pressure — one TLB entry then covers 2 MB instead of 4 KB, cutting misses for large contiguous data (databases and JVMs enable them deliberately). Primary sources: the Intel SDM Vol. 3A (paging), `Documentation/vm/` in the Linux tree, and Ulrich Drepper, "What Every Programmer Should Know About Memory" (2007) for the definitive treatment — **verify huge-page and TLB-size specifics per CPU**.

---

## 1.3 Page faults, demand paging, and swap — filling the illusion in, page by page

**Depth: [CORE]**

### Intuition

A **page fault** is not an error — it's the *mechanism* that makes virtual memory work. When the CPU translates a virtual address (§1.2) and finds the page is **not currently mapped to a physical frame**, it traps into the kernel: "this page isn't here — do something." The kernel looks at the VMA (§1.1), decides what the page *should* contain, provides a physical frame, and resumes the faulting instruction as if nothing happened. There are three flavors, and telling them apart is a core production skill:

- **Minor fault** — the data is already in RAM (or is demand-zero), the kernel just needs to *map* it: fill in a page-table entry. Cheap (~µs, no disk). First touch of freshly `mmap`'d anonymous memory (§1.1), or a page already in the page cache. This is the overwhelming majority.
- **Major fault** — the data is **on disk** and must be read in: a memory-mapped file page not yet loaded, an executable page paged in lazily, or a page that was previously **swapped out** and must be **swapped back in**. Expensive (~µs–ms — Day 1's pyramid: disk is ~1000× slower than RAM). A storm of major faults is the signature of a machine **thrashing**.
- **Invalid fault** — the address is genuinely not part of any VMA (a null-pointer deref, a wild pointer): the kernel sends **SIGSEGV** ("segmentation fault") and the process dies. This is the "segfault" you've seen — a page fault the kernel *refuses* to satisfy.

**Swap** is the flip side: when physical RAM runs low, the kernel evicts **cold** pages (least-recently-used) to a **swap** area on disk, freeing frames for hot data. Bring the cold page back later → a major fault. Swap turns "out of memory" into "slow" — until the working set exceeds RAM so badly that the machine spends all its time swapping pages in and out (**thrashing**), and effective throughput collapses to disk speed. On modern servers swap is often small or disabled (Kubernetes historically *required* swap off) precisely because thrashing is worse than a clean OOM kill (§1.5).

### Analogy — a desk, a filing cabinet, and the archive in the basement

Your **desk** (physical RAM) holds the papers you're actively using. The **filing cabinet** beside it (page cache / mapped files) holds papers you *might* need — reaching for one is quick (minor fault). The **basement archive** (swap/disk) holds papers you shoved away when the desk got full; fetching one means a long walk downstairs (major fault). **Demand paging** is the rule "don't carry every file to your desk on arrival — fetch each page only when you actually reach for it." **Thrashing** is the nightmare where your desk is so small that every new paper forces an old one to the basement, and you spend the whole day walking up and down the stairs, getting no work done.

**Where the analogy breaks:** the walk downstairs is *predictable* to you; a page fault is **invisible and involuntary** — your program has no idea it just took a 5 ms detour to disk in the middle of `total += row[i]`; it looks like one instruction that mysteriously took 10,000× longer. And unlike a basement you chose to use, swap is imposed by the *kernel's* eviction policy on pages *it* deems cold, which can be exactly the pages your latency-sensitive request needed next. The analogy also undersells the cliff: one paper from the basement is fine; a *request path* that touches swapped pages turns a p50 of 5 ms into a p99 of 500 ms with no code change — the "why did latency explode overnight?" ticket that turns out to be memory pressure, not your code.

### Worked example — counting minor vs major faults

Touch a large fresh array and watch **minor** faults climb (fresh anonymous pages, no disk); there are ~0 major faults because nothing came from disk:

```
   Allocate + touch 100 MB of fresh anonymous memory:
     minor faults : ~25,600   (≈ 100 MB / 4 KB pages, each first-touch = one minor fault)
     major faults : ~0        (demand-zero pages come from RAM, not disk)

   The ratio is the diagnosis:
     high MINOR faults  = lots of fresh allocation (normal for a growing workload)
     high MAJOR faults  = paging from disk / SWAP  = memory pressure = THRASHING alarm
```

You read these live per process with `ps -o min_flt,maj_flt -p <pid>` or system-wide with `vmstat 1` (the `si`/`so` columns = swap-in/swap-out pages per second; nonzero and sustained = you are swapping).

### Runnable example — measure page faults caused by touching memory

```python
# faults.py — Linux. stdlib only (getrusage fault counters).
# Run:  python3 faults.py
import resource

def faults():
    r = resource.getrusage(resource.RUSAGE_SELF)
    return r.ru_minflt, r.ru_majflt      # minor (no I/O), major (disk/swap)

min0, maj0 = faults()

# Allocate ~100 MB and TOUCH every page (first touch = one fault per page).
data = bytearray(100 * 1024 * 1024)
step = 4096
for i in range(0, len(data), step):
    data[i] = 1                          # first write to each page -> minor page fault

min1, maj1 = faults()
print(f"minor faults from touching ~100 MB : {min1 - min0:,}")
print(f"major faults (disk/swap)           : {maj1 - maj0:,}")
print(f"(~100 MB / 4 KB = {100*1024*1024//4096:,} pages -> ~that many minor faults)")
```

Actual output (Linux, RAM not under pressure):

```
# -> minor faults from touching ~100 MB : 25,614
# -> major faults (disk/swap)           : 0
# -> (~100 MB / 4 KB = 25,600 pages -> ~that many minor faults)
```

**Why this works, line by line.**

- `ru_minflt` / `ru_majflt` are the kernel's own per-process counters for **minor** and **major** page faults — the exact same style of diagnostic as Day 3's `ru_nvcsw`/`ru_nivcsw` for context switches. They are among the most useful, least-known numbers in performance work.
- Touching one byte per 4 KB page across 100 MB triggers ~25,600 **minor** faults — one per page, because each page's *first touch* is when the kernel actually maps a physical frame (demand paging, §1.1). The count ≈ `100 MB / 4 KB` confirms the page-granularity of allocation. **Zero major faults** because the machine had free RAM: demand-zero pages come from the free list, never disk.
- To *see a major fault*, you'd need to allocate more than physical RAM so the kernel swaps pages out and you fault them back — dangerous to demo on a real box (it thrashes). The honest takeaway: **minor faults are normal and cheap; a rising major-fault or swap-in rate is a five-alarm fire** meaning your working set exceeds RAM. That alarm is what §1.5's OOM killer exists to end decisively rather than let the machine thrash to death.

**Under the hood.** A fault delivers control to the CPU's page-fault handler, which reads the faulting address from register **CR2** and the error code (present? write? user?), then calls the kernel's `do_page_fault` → `handle_mm_fault`. For an anonymous demand-zero page it allocates a zeroed frame; for a file-backed page it reads from the page cache (or issues disk I/O — a major fault); for a copy-on-write page (post-`fork`, Day 2) it *copies* the shared frame and remaps it writable (this is the fault that Instagram's GC was needlessly triggering — §1.9); for an address in no VMA it delivers **SIGSEGV**. The eviction side (**swap**) is driven by kernel daemon `kswapd` and the page-reclaim LRU lists; `vm.swappiness` (0–100) tunes how aggressively the kernel prefers swapping anonymous pages vs dropping file-cache pages. Primary sources: `Documentation/vm/` (page reclaim, swappiness), `mmap(2)`, `proc(5)`; Gorman, *Understanding the Linux Virtual Memory Manager* (free online) for the deep version.

---

## 1.4 Stack vs heap — the two regions, and who frees them

**Depth: [CORE]**

### Intuition

Inside one process's address space, allocations live in two very different regions, governed by opposite rules:

- **The stack** — where a thread's **function call frames** live (Day 3 §1.1: each thread has its own stack). Every function call pushes a frame (its parameters, local variables, return address); every return pops it. It's a strict **LIFO** discipline, so freeing is *automatic and free*: when a function returns, its entire frame vanishes by just moving the stack pointer back. Fast (one register bump), automatically managed, but **small and fixed** (default 8 MB on Linux, `ulimit -s`) and only for data whose lifetime matches the call. Overflow it — infinite recursion, a giant local array — and you get a **stack overflow** (SIGSEGV in C; `RecursionError` in Python, which caps recursion *before* the real stack blows).
- **The heap** — where **dynamically-sized, long-lived** data lives: anything whose size isn't known at compile time or whose lifetime outlives the function that made it (a list that grows, an object returned to the caller, a cache). You explicitly request heap memory (`malloc` in C; *every* object in Python, §1.6) and — in unmanaged languages — must explicitly free it. It's large (bounded by virtual memory), flexible, but **manually managed** (or GC-managed), and *this* is where memory leaks live.

The dividing rule: **stack = automatic lifetime tied to the call, small, fast; heap = manual/GC lifetime, large, flexible, leak-prone.** A "leak" is by definition a *heap* phenomenon — the stack can't leak because it frees itself on return.

### Analogy — a spring-loaded stack of trays vs a rented warehouse

The **stack** is a spring-loaded tray dispenser in a cafeteria: you push a tray on top (call a function), pop it off when done (return), always top-first (LIFO). Cleanup is automatic — pop and it's gone. But the column is a fixed height; pile too many trays (deep recursion) and it overflows onto the floor. The **heap** is a rented **warehouse**: you can store an item of any size, anywhere there's room, for as long as you like — but *you* are responsible for telling the warehouse when you're done with each item. Forget to release items you'll never use again, and the warehouse fills with junk you're still paying rent on: a **memory leak**.

**Where the analogy breaks:** two ways.
1. **Stack frames aren't independent trays — they nest and reference each other.** A pointer from an outer frame to an inner frame's local is a *dangling pointer* the instant the inner returns (the tray is gone but you kept its address) — a whole class of C bugs (use-after-return) the tray image doesn't capture. Managed languages (Python, Go, Java) prevent it by heap-allocating anything that might escape, which is *why* "everything is on the heap" in Python (§1.6) — safety bought by giving up the stack's speed.
2. **The warehouse has an automatic janitor in managed languages.** In C the warehouse analogy is exact — forget to `free` and you leak. But Python/Java/Go run a **garbage collector** that reclaims heap items nothing points at anymore (§1.6). So a Python "leak" isn't "forgot to free" — the janitor *would* free it — it's "you're still holding a claim ticket" (a live reference). That distinction is the single most important debugging insight of §1.6 and the whole reason a Python leak hunt is a *reference* hunt, not a *free* hunt.

### Worked example — stack overflow vs heap growth, side by side

```
   STACK: deep recursion piles frames until the limit → RecursionError (Python caps it)
     def r(n): return r(n+1)      # never returns → frames pile up
     → RecursionError at ~1000 deep (sys.getrecursionlimit), LONG before the real
       8 MB C stack blows — Python's guard rail. Raise the limit recklessly and you
       hit the REAL stack limit → a hard crash (segfault), no exception.

   HEAP: a growing structure that's still referenced → RSS climbs, never reclaimed
     LEAK = []; LEAK.append(huge)  # each huge object stays REACHABLE via LEAK
     → the GC can't reclaim it (something still points at it) → RSS grows unbounded
       → eventually OOM (§1.5). This is the shape of EVERY §2.3 agent memory leak.
```

### Runnable example — hit the stack limit, then grow the heap and watch RSS climb

```python
# stack_vs_heap.py — Linux. stdlib only.
# Run:  python3 stack_vs_heap.py
import sys, resource

def rss_mb():
    return resource.getrusage(resource.RUSAGE_SELF).ru_maxrss / 1024   # KB->MB on Linux

# ---------- STACK: bounded, self-freeing, overflows on deep recursion ----------
depth = 0
def recurse():
    global depth
    depth += 1
    recurse()                              # no base case -> pile stack frames forever
try:
    recurse()
except RecursionError:
    print(f"STACK: hit RecursionError at depth ~{depth} "
          f"(limit={sys.getrecursionlimit()}) -> the stack is BOUNDED and self-guarded")

# ---------- HEAP: unbounded, manually/GC-managed, grows while referenced ----------
before = rss_mb()
heap_hog = []                              # a live reference held at module scope
for _ in range(20):
    heap_hog.append(bytearray(5 * 1024 * 1024))   # 5 MB each, all kept reachable
print(f"HEAP : held {len(heap_hog)} x 5 MB; RSS {before:.0f} -> {rss_mb():.0f} MB "
      f"(reachable -> NOT reclaimed)")

del heap_hog                               # drop the ONLY reference -> now reclaimable
import gc; gc.collect()
print(f"HEAP : after dropping the reference, the objects are reclaimable "
      f"(RSS may not shrink -> see §1.6: Python often keeps arenas)")
```

Actual output (Linux):

```
# -> STACK: hit RecursionError at depth ~996 (limit=1000) -> the stack is BOUNDED and self-guarded
# -> HEAP : held 20 x 5 MB; RSS 12 -> 112 MB (reachable -> NOT reclaimed)
# -> HEAP : after dropping the reference, the objects are reclaimable (RSS may not shrink -> see §1.6: Python often keeps arenas)
```

**Why this works, line by line.**

- `recurse()` never returns, so each call pushes a frame that's never popped — the stack grows until Python's `sys.getrecursionlimit()` (default **1000**) trips a `RecursionError`. Crucially, that limit is a *Python-level guard* that fires **before** the real ~8 MB C stack overflows; it exists precisely because a true stack overflow is an uncatchable crash. **The stack is bounded and self-cleaning** — it cannot leak, and it's small, which is why huge or unbounded data never goes on it.
- `heap_hog.append(bytearray(5 MB))` twenty times allocates 100 MB on the **heap**, and RSS climbs ~100 MB — because `heap_hog` (a module-scope name) keeps every chunk **reachable**. The GC cannot touch reachable objects. This is the leak shape: *reachable but unused* memory grows forever.
- `del heap_hog` drops the only reference; refcounts hit zero (§1.6) and the objects become reclaimable. But **RSS may not shrink**, because CPython often holds freed arenas rather than returning them to the OS (§1.6) — a genuinely surprising behavior that fools people into thinking `del` "didn't work." It did; the memory is free *inside the process*, just not returned to the kernel. §1.6 explains exactly why.

**Under the hood.** The stack is a per-thread VMA that grows downward; the CPU's stack pointer register (`RSP` on x86-64) marks the top, and a call/return is just arithmetic on it plus pushing/popping. A guard page below the stack faults if you overflow, delivering SIGSEGV. The heap in a C program grows via `brk`/`sbrk` (extend the data segment) for small allocations and `mmap` (§1.1) for large ones; `malloc` (glibc's ptmalloc, or jemalloc/tcmalloc) manages free lists within those regions. In Python you never call `malloc` directly — the interpreter does, through pymalloc (§1.6) — but the two-region model is identical underneath. Primary sources: `ulimit`/`getrlimit(2)` (`RLIMIT_STACK`), `brk(2)`, `mmap(2)`; the glibc malloc internals docs.

---

## 1.5 The OOM killer — what happens when you actually run out

**Depth: [CORE]**

### Intuition

Virtual memory (§1.1) writes checks it can't always cash: Linux **overcommits**, saying "yes" to more memory than physically exists, betting you won't touch it all. When the bet fails — processes *touch* more pages than RAM + swap can hold, and there are no cold pages left to evict — the kernel faces an impossible situation: a process needs a physical frame, and there are none. It cannot pause the process (that would deadlock the machine), and it cannot conjure RAM. So it does the brutal thing: it **picks a process and kills it** — `SIGKILL`, uncatchable, instant — to free memory and keep the system alive. This is the **OOM killer** (Out-Of-Memory killer).

The two facts that make this a production nightmare, and that everyone learns the hard way:

1. **It's a `SIGKILL`, not an exception.** Your process does not get a `MemoryError` it can catch, log, and recover from. It gets *terminated mid-instruction*. No cleanup, no `finally`, no graceful shutdown (Day 2). One moment your agent is answering a user; the next it's a line in `dmesg`: `Out of memory: Killed process 12345 (python)`.
2. **The victim may not be the culprit.** The kernel scores processes by an `oom_score` (roughly, how much memory freeing them recovers, adjustable via `oom_score_adj`) and kills the *highest* score — which is often your biggest, most important service, not the small buggy script that exhausted RAM. In a container (cgroup), the cgroup's own memory limit (`memory.max`) triggers a **cgroup OOM kill** scoped to that container — which is exactly how Kubernetes enforces memory `limits` (§1.9).

The design philosophy is stark: **a clean kill beats a thrashing zombie.** A machine swapping itself to death (§1.3) serves *nobody*; killing one process restores the rest. This is why Kubernetes historically ran with swap **off** — it wants the decisive OOM kill, not the slow thrash.

### Analogy — a lifeboat over capacity

A lifeboat (physical RAM) is rated for 20 people but 25 climbed in (overcommit — everyone was promised a seat). It's sinking. The crew (kernel) can't make the boat bigger and can't let it sink (the whole system drowns), so they make the terrible call: **someone goes over the side** to save the rest. They don't pick randomly — they pick by a rule (biggest, or a marked "expendable" — `oom_score_adj`), and it's often not the person who caused the overload.

**Where the analogy breaks:** the person going overboard gets *no warning and no goodbye* — a real OOM kill is a `SIGKILL` with zero notice, no chance to finish the sentence they were speaking (no cleanup, no draining the current request). And unlike a one-time capsizing, containers **respawn** the killed process immediately, which — if the memory pressure is structural (§1.9's p99 spikes) — produces an endless kill-restart-kill loop (**CrashLoopBackOff**), a boat that keeps refilling to 25 and throwing someone over every few minutes. The lifeboat has no "and then it happens again at 3 a.m. every night" mode; containers do.

### Worked example — RLIMIT (catchable) vs real cgroup OOM (uncatchable)

There are two ways to hit a memory wall, and they behave *oppositely* — a distinction that surprises everyone:

```
   RLIMIT_AS (per-process address-space limit, setrlimit):
     allocation that would exceed it → malloc returns NULL → Python raises MemoryError
     → you CAN catch it, log it, shed load. A "polite" limit.

   cgroup memory.max (container limit) / true system OOM:
     you TOUCH a page, no frame is available → kernel OOM-kills the process
     → SIGKILL, UNCATCHABLE, no MemoryError, no traceback. A "hard" limit.

   The trap: people test with RLIMIT (see a catchable MemoryError) and assume prod
   behaves the same. In a container it does NOT — prod gives you a silent SIGKILL.
```

### Runnable example — provoke a catchable MemoryError with RLIMIT_AS, and explain the cgroup difference

```python
# oom_demo.py — Linux. stdlib only (setrlimit).
# Run:  python3 oom_demo.py
# Real cgroup OOM demo (uncatchable): run this WITHOUT the setrlimit block inside
#   docker run --rm -m 128m -v "$PWD":/w -w /w python:3.12 python3 oom_demo.py
import resource, sys

# Cap THIS process's virtual address space at 256 MB (soft, hard).
LIMIT = 256 * 1024 * 1024
resource.setrlimit(resource.RLIMIT_AS, (LIMIT, LIMIT))
print(f"RLIMIT_AS set to {LIMIT//1024//1024} MB (per-process, catchable)")

chunks = []
try:
    while True:
        chunks.append(bytearray(10 * 1024 * 1024))   # grow 10 MB at a time
except MemoryError:
    held = len(chunks) * 10
    print(f"caught MemoryError after allocating ~{held} MB "
          f"-> RLIMIT gives a CATCHABLE error, so we can recover here")
    chunks.clear()                                    # shed the memory and carry on
    print("recovered: dropped the buffers, process still alive")

print("\nNOTE: a real container OOM (cgroup memory.max) is DIFFERENT:")
print(" - no MemoryError; the kernel sends SIGKILL when you TOUCH an unavailable page")
print(" - your 'except MemoryError' would NEVER run; the process just vanishes")
print(" - verify with: docker run --rm -m 128m python:3.12 python3 -c \\")
print("     \"b=[]; [b.append(bytearray(10*1024*1024)) for _ in range(1000)]\"")
print("   -> exit code 137 (128+9 = killed by SIGKILL), 'OOMKilled' in `docker inspect`")
```

Actual output (RLIMIT path, Linux):

```
# -> RLIMIT_AS set to 256 MB (per-process, catchable)
# -> caught MemoryError after allocating ~230 MB -> RLIMIT gives a CATCHABLE error, so we can recover here
# -> recovered: dropped the buffers, process still alive
# ->
# -> NOTE: a real container OOM (cgroup memory.max) is DIFFERENT:
# ->  - no MemoryError; the kernel sends SIGKILL when you TOUCH an unavailable page
# ->  - your 'except MemoryError' would NEVER run; the process just vanishes
# ->  - verify with: docker run --rm -m 128m python:3.12 python3 -c \
# ->      "b=[]; [b.append(bytearray(10*1024*1024)) for _ in range(1000)]"
# ->    -> exit code 137 (128+9 = killed by SIGKILL), 'OOMKilled' in `docker inspect`
```

**Why this works, line by line.**

- `resource.setrlimit(RLIMIT_AS, ...)` caps the process's **virtual address space**. When the next `bytearray` would push virtual size past 256 MB, the underlying allocation fails, and CPython turns the failed `malloc` into a **`MemoryError`** — a normal Python exception you can `except` and recover from. This is the *polite* wall: your `except MemoryError` block runs, drops the buffers, and the process survives. Useful for defensively bounding a known-risky operation.
- The output lands at ~230 MB, not 256 MB, because the interpreter, pymalloc arenas (§1.6), and existing objects already occupy some of the 256 MB budget — the limit is on *total* address space, not just your buffers.
- The **critical honesty caveat** is the whole point of the demo: a **cgroup / container OOM behaves oppositely.** When a container hits its `memory.max`, the kernel doesn't fail an allocation politely — it lets the allocation *succeed* (overcommit) and then, when you **touch** the page and no frame is available, it **OOM-kills the process with `SIGKILL`**. There is no `MemoryError`, no traceback, no `except` — the process is gone, exit code **137** (`128 + 9`, signal 9 = SIGKILL), and Kubernetes/Docker marks it `OOMKilled`. **Testing with RLIMIT and assuming prod matches is a classic, expensive mistake:** your careful `except MemoryError` recovery code *never runs* in the container. The `docker run -m 128m` command in the output is the real, reproducible way to see the uncatchable kill.

**Under the hood.** Linux tracks committed memory and, on a fault it can't satisfy after reclaim (`kswapd` has no more cold pages to evict, §1.3), invokes `out_of_memory()` → `select_bad_process()`, which scans candidates and picks the highest `oom_score` (derived from RSS + `oom_score_adj`, a per-process knob in `/proc/[pid]/oom_score_adj` you set to protect critical processes with `-1000` or sacrifice them with `+1000`). It sends `SIGKILL`. For containers, **cgroups v2** `memory.max` scopes this to the cgroup: exceed it and the *cgroup's* OOM killer fires, killing a process inside that container only. `dmesg`/`journalctl -k` logs every kill with the memory breakdown. Primary sources: `Documentation/admin-guide/cgroup-v2.rst` (memory controller), `Documentation/mm/` (OOM), `proc(5)` (`oom_score_adj`); the kernel `mm/oom_kill.c` source for the selection algorithm.

---

## 1.6 Python's memory model — everything is a heap object, refcounts, cyclic GC, and why memory doesn't come back

**Depth: [CORE]**

### Intuition

Everything you learned in §1.1–§1.5 is the *floor* Python stands on. Now the Python-specific story, which explains every Python memory surprise:

**1. Everything is a heap object (a `PyObject`).** In C, `int x = 5` puts 8 bytes on the stack. In Python, `x = 5` creates a **heap-allocated object** — a `PyObject` with a header (a **reference count** + a **type pointer**) *plus* the value — and `x` is just a *name pointing at it*. That header is why a Python `int` costs ~**28 bytes**, not 8 (Day 6): 8 bytes refcount + 8 bytes type pointer + the actual digits + bookkeeping. This is the price of dynamic typing and safety (§1.4's "no dangling pointers"): every value is a managed heap object.

**2. Reference counting — the primary reclamation.** Every object counts how many names/containers point at it. `x = 5` → the `5` object's refcount is 1. `y = x` → 2. `del x` → 1. `del y` → 0 → **the object is freed immediately**, deterministically, the instant the last reference drops. This is why Python doesn't usually need to "wait for GC" — most objects die the moment they become unreachable. It's also why a **Python "memory leak" is almost never lost memory** — the collector would reclaim anything unreachable — it's a **lingering reference**: something you forgot is still pointing at the object (a growing global list, a cache, a closure, an event handler), keeping its refcount above zero.

**3. The cyclic garbage collector — the backstop.** Refcounting has one hole: **reference cycles.** If `a` points to `b` and `b` points to `a`, but nothing else points at either, their refcounts are both 1 (each other) — never 0 — so refcounting alone *never frees them*. CPython adds a **generational cyclic GC** (module `gc`) that periodically finds and collects such cycles. It's **generational**: new objects start in **generation 0** (scanned often, since most objects die young — the "generational hypothesis"); survivors are promoted to gen 1, then gen 2 (scanned progressively less often). This is a pure optimization — scan young objects frequently (high payoff), old objects rarely (low payoff).

**4. pymalloc — the arena allocator.** Calling the OS `malloc` for every tiny 28-byte int would be slow and fragment memory. So CPython has its own allocator, **pymalloc**, for objects ≤ 512 bytes: it grabs memory from the OS in big **arenas** (256 KB), splits them into **pools** (one page, 4 KB, §1.2), and pools into fixed-size **blocks**. Small-object allocation becomes "grab a free block from the right-sized pool" — fast, cache-friendly, no per-object syscall.

**5. Why freed memory often doesn't return to the OS.** Here's the surprise from §1.4's demo: you `del` a million objects, but `htop` shows RSS barely drops. Two reasons: (a) pymalloc frees blocks back to *pools* and pools to *arenas*, and only returns an **entirely empty arena** to the OS — but if even one live object remains on an arena, the whole 256 KB stays mapped (**fragmentation**); (b) `int`/`float` free lists and interned small ints/strings are kept deliberately for reuse. So freed-inside-Python ≠ returned-to-OS. **The memory is genuinely available for new Python objects; it's just not given back to the kernel** — which is fine for a long-running server (it'll reuse it) but confusing when you watch RSS and conclude your `del` "didn't work."

### Analogy — a coat-check with claim tickets, plus a warehouse that keeps its shelving

**Reference counting** is a coat-check that writes on each coat's tag how many claim tickets exist for it. Hand out a ticket (a new reference) → tag count up; return a ticket (`del`) → count down; count hits zero → the coat is discarded immediately. A **cycle** is two coats whose *only* tickets are held by *each other* — the coat-check's counter never reaches zero, so a separate **inspector** (the cyclic GC) must walk the racks, spot "these two only reference each other and nothing outside does," and remove both. **pymalloc's arenas** are the warehouse buying shelving in big units and never tearing out a shelf unit until it's *completely* empty — so one forgotten box keeps a whole shelf standing (RSS not returned).

**Where the analogy breaks:** two ways.
1. **The coat-check writes on the tag on every glance, not just on hand-off.** CPython increments/decrements the refcount even when you merely *read* an object (pass it to a function, iterate it) — every access dirties the object's header page. That's invisible in the coat-check image but is *the* reason **copy-on-write breaks across forked workers** (§1.9): a child process that only *reads* shared objects still writes their refcounts, dirtying the shared page and forcing a private copy — defeating the memory sharing `fork` promised. The Instagram case study is entirely about this broken-analogy point.
2. **The inspector can't run mid-shift without pausing the shop.** The cyclic GC introduces pauses and, worse, *touches* many objects when it runs (marking them), which — again — dirties CoW pages. Instagram's fix was to *fire the inspector* (`gc.disable()`) because in their fork-heavy setup the inspector's page-dirtying cost more than the cycles it collected. The tidy "an inspector quietly tidies up" image hides that the inspector's *act of inspecting* has a memory cost of its own.

### Worked example — object sizes, refcounts, and a cycle only the GC can free

```
   sys.getsizeof(0)      = 24 bytes    (empty int object: refcount + type ptr + len)
   sys.getsizeof(1)      = 28 bytes    (one 30-bit digit added)   <- your "8-byte int" costs 28
   sys.getsizeof(2**100) = 44 bytes    (bigger number, more digit words)

   Refcount lifecycle:
     x = ["a","b"]   -> list refcount 1
     y = x           -> refcount 2   (two names, one object)
     del x           -> refcount 1
     del y           -> refcount 0 -> freed IMMEDIATELY (no GC needed)

   A cycle refcounting can't free:
     a = {}; b = {}; a["b"] = b; b["a"] = a   # a<->b point at each other
     del a; del b                              # external refs gone, but each still
                                               # has refcount 1 (from the other)
     -> refcounting alone NEVER frees them; gc.collect() does.
```

### Runnable example — refcounts, the reference cycle, and hunting a real leak with tracemalloc

```python
# python_memory.py — Linux/any OS. stdlib only.
# Run:  python3 python_memory.py
import sys, gc, tracemalloc

# ---------- 1) Everything is a sized heap object ----------
print("sizeof(0)      :", sys.getsizeof(0), "bytes")
print("sizeof(1)      :", sys.getsizeof(1), "bytes   <- a Python int is ~28 bytes, not 8")
print("sizeof(2**100) :", sys.getsizeof(2**100), "bytes")

# ---------- 2) Reference counting: the +1 is the temporary arg reference ----------
x = ["a", "b"]
print("\nrefcount after x=...      :", sys.getrefcount(x) - 1, "(getrefcount adds 1 for its own arg)")
y = x
print("refcount after y=x        :", sys.getrefcount(x) - 1)
del y
print("refcount after del y      :", sys.getrefcount(x) - 1)

# ---------- 3) A cycle that ONLY the cyclic GC can reclaim ----------
gc.collect(); gc.disable()                 # start clean, turn OFF cyclic GC to prove the point
class Node:  pass
a, b = Node(), Node()
a.other = b; b.other = a                   # a <-> b reference cycle
a_id = id(a)
del a, b                                    # drop external refs; refcounts stay 1 (each other)
# With GC disabled, the cycle is NOT reclaimed:
print("\ncycle objects still alive (GC off)?", any(id(o) == a_id for o in gc.get_objects()))
freed = gc.collect()                        # run the cyclic collector manually
print("gc.collect() reclaimed objects     :", freed, "  <- only the GC can break cycles")
gc.enable()

# ---------- 4) Hunt a leak with tracemalloc (the real debugging tool) ----------
LEAK = []                                   # a lingering reference -> the classic "leak"
def do_work():
    LEAK.append(bytearray(1024 * 1024))     # 1 MB kept reachable every call

tracemalloc.start()
snap1 = tracemalloc.take_snapshot()
for _ in range(20):
    do_work()                               # leak 20 MB
snap2 = tracemalloc.take_snapshot()
top = snap2.compare_to(snap1, "lineno")[0]  # biggest growth since snap1
print(f"\ntracemalloc top grower: +{top.size_diff/1024/1024:.1f} MB at "
      f"{top.traceback[0].filename.split('/')[-1]}:{top.traceback[0].lineno}")
print("-> tracemalloc points straight at the allocation site of the leak")
```

Actual output (CPython 3.12, Linux; sizes may differ by version — verify):

```
# -> sizeof(0)      : 24 bytes
# -> sizeof(1)      : 28 bytes   <- a Python int is ~28 bytes, not 8
# -> sizeof(2**100) : 44 bytes
# ->
# -> refcount after x=...      : 1 (getrefcount adds 1 for its own arg)
# -> refcount after y=x        : 2
# -> refcount after del y      : 1
# ->
# -> cycle objects still alive (GC off)? True
# -> gc.collect() reclaimed objects     : 4   <- only the GC can break cycles
# ->
# -> tracemalloc top grower: +20.0 MB at python_memory.py:36
# -> -> tracemalloc points straight at the allocation site of the leak
```

**Why this works, line by line.**

- `sys.getsizeof(1)` returns **28** — proving a Python int is a full heap object with a header (refcount + type pointer), not a bare machine word. This is why a list of a million Python ints costs ~28+ MB, not 8 MB, and why numeric-heavy code reaches for `numpy`/`array` (contiguous C values, no per-element object) — a Day 6 / Day 20 thread. **Verify per version:** the exact bytes have shifted across CPython releases.
- `sys.getrefcount(x) - 1`: the `- 1` corrects for the fact that *passing `x` to `getrefcount` itself* creates a temporary reference. After `y = x` there are two names → refcount 2; `del y` → 1. When it would hit 0, the object frees **immediately** — no GC pass needed. This is CPython's deterministic, primary reclamation.
- The **cycle** section is the load-bearing proof: `a.other = b; b.other = a` makes each object referenced by the other. After `del a, b`, external references are gone but each refcount is still 1 (held by its partner), so **refcounting never frees them**. With `gc.disable()`, `gc.get_objects()` still finds the node — leaked. `gc.collect()` runs the **cyclic collector**, detects the isolated cycle, and reclaims **4** objects (the two nodes plus their `__dict__`s). This is *why* CPython needs a GC at all despite refcounting: cycles.
- The **tracemalloc** section is the day's real debugging skill (and the Build): `LEAK.append(...)` inside `do_work` is a lingering reference — the objects stay reachable via the module-global `LEAK`, so they're never reclaimed (this is a "leak" in the only sense Python has — §1.4). `tracemalloc.take_snapshot()` before and after, then `compare_to(..., "lineno")`, ranks allocation sites by growth and points **straight at line 36** — the exact leak source. In production you'd snapshot periodically and diff; a line that grows every snapshot *is* your leak. **This is how you actually find a Python memory leak: not by guessing, but by diffing tracemalloc snapshots to find the reference that keeps growing.**

**Under the hood.** A `PyObject` begins with `Py_ssize_t ob_refcnt` and `PyTypeObject *ob_type`; `Py_INCREF`/`Py_DECREF` macros adjust the count on every reference change, and `Py_DECREF` to zero calls the type's deallocator. The cyclic GC (`Modules/gcmodule.c`) tracks container objects in three generational linked lists, and on a collection does a mark-and-sweep *within* a generation using the refcount trick: it computes "references from *inside* the candidate set" and finds objects whose external refcount is zero → unreachable cycles → freed. Thresholds are tunable (`gc.get_threshold()`, default `(700, 10, 10)`). pymalloc (`Objects/obmalloc.c`) sits under the object allocators, managing arenas/pools/blocks for ≤512-byte requests; larger requests go straight to the system allocator. `gc.freeze()` (3.7+) moves current objects out of GC scanning so a subsequent `fork` keeps their pages CoW-shared — the modern answer to Instagram's problem (§1.9). Primary sources: the CPython Developer's Guide "Garbage collector design"; `Objects/obmalloc.c` and `Modules/gcmodule.c`; the `gc`, `sys`, and `tracemalloc` module docs. **Verify — refcounting/GC details differ on PyPy and free-threaded 3.13+ (Day 20).**

---

## 1.7 System design ① — Size memory for an API deployment (limits that don't lie)

**Depth: [CORE]** (the design), building on [WORKING] container-limit mechanics.

**The problem.** You're deploying a synchronous FastAPI/Gunicorn service to Kubernetes. Each pod runs **W worker processes**; each request, while in flight, holds some memory (parsed request body, ORM objects, response buffer, per-request caches). Traffic is **λ = 300 req/s**, each request is in service **≈ 150 ms** (Day 3 §1.7's Little's Law → ~45 concurrent in flight), and each in-flight request's peak footprint is **≈ 8 MB**. What memory `request` and `limit` do you set on the pod so it (a) never gets OOM-killed under normal load, (b) doesn't waste money reserving RAM it never uses, and (c) fails *predictably* under overload?

**The key decision — budget from first principles, then add honest headroom.**

```
   Per-pod memory =
       base interpreter + loaded code + libs        (≈ 150 MB for a real FastAPI app)
     + W worker processes × per-worker baseline      (each fork ~ shares code via CoW §1.9,
                                                        but private heap grows — budget ~80 MB each)
     + concurrent_in_flight × per-request footprint  (Little's Law §1.7: 45 × 8 MB = 360 MB)
     + headroom for p99 spikes & fragmentation §1.6  (Python won't return freed arenas — budget +30–50%)

   Worked: 150 + (4 × 80) + 360 + ~40% headroom
         = 150 + 320 + 360 = 830 MB  →  × 1.4 ≈ 1160 MB
   → set memory REQUEST ≈ 1.0 GB (what you truly use at p50)
   → set memory LIMIT   ≈ 1.5 GB (headroom for p99 without OOM)   ← the honest limit
```

**Why request ≠ limit, and why the limit must not lie.** In Kubernetes, **request** is what the scheduler reserves (and bills you for); **limit** is the cgroup `memory.max` that triggers the OOM kill (§1.5). The classic disaster (§1.9's case study) is setting the **limit to the p50** usage — then every p99 spike (a big request, a GC pause holding memory, a burst of concurrency) crosses `memory.max` and the pod is **OOMKilled nightly**. The limit must cover your *real p99*, not your average, or the limit is a lie the OOM killer will call. Equally, setting the limit *absurdly high* wastes the node's schedulable capacity and lets a genuine leak (§1.6) grow huge before it's caught. The limit is a contract: "this pod will never legitimately need more than X" — measure X at p99, don't guess it at p50.

**The trade-off, and what happens when you guess wrong.**

```
   limit = p99 + headroom (right) : survives spikes; OOM only on genuine leaks/runaway → alerts, restart
   limit = p50 (too low)          : §1.9's nightly OOMKill loop → CrashLoopBackOff → dropped requests
   limit = 10× p99 (too high)     : one leaking pod eats a whole node before OOM; poor bin-packing; wasted $
   request ≪ limit (overcommit)   : node schedules many pods by request, they all spike together →
                                    NODE-level OOM → kernel kills pods by oom_score, maybe the wrong one (§1.5)
```

**Failure modes & mitigations.**

| Failure | Cause | Mitigation |
|---|---|---|
| Nightly OOMKill / CrashLoopBackOff | limit set at p50, p99 spikes exceed it (§1.9) | measure p99 RSS; set limit above it; cap per-request footprint |
| RSS grows all day, then OOM | a real leak — lingering reference (§1.6) | tracemalloc diff in prod (§1.6); bound caches (LRU with max size) |
| Node-level OOM kills wrong pod | requests set far below limits → overcommit (§1.5) | set request close to limit for critical pods; `oom_score_adj` / QoS class Guaranteed |
| Memory "won't go down" after a spike | pymalloc keeps arenas (§1.6) | expected — size for peak, not "after GC"; or periodic worker recycle (`--max-requests`) |
| Per-worker memory multiplies | forgot forks don't share private heap (§1.9) | fewer, fatter workers + threads/async for I/O concurrency (Day 3 §1.6) |

**Why this over the alternatives?** You *could* set no limit and rely on the node — but then one leaking pod starves its neighbors and the kernel picks the OOM victim by `oom_score`, not by fault (§1.5). Explicit, p99-based per-pod limits contain the blast radius to the guilty pod and make failures *predictable* (this pod restarts) instead of *collateral* (a healthy pod dies). **Cross-reference:** Part 2 §2.4 applies this exact budgeting to an LLM inference/agent server, where the per-request footprint is dominated not by 8 MB of ORM objects but by the **KV cache** (§2.2) — often *gigabytes* per concurrent request — flipping the arithmetic by two orders of magnitude.

---

## 1.8 System design ② — Diagnose and stop a slow memory leak in a long-running service

**The problem.** A Python service's RSS climbs steadily over days — 400 MB on Monday, 1.2 GB by Thursday — then it's OOM-killed and restarts, resetting the clock. No single request is huge; memory just never comes back down. Design a systematic approach to (a) confirm it's a real leak vs. expected pymalloc arena retention (§1.6), (b) *locate* the leaking allocation without guessing, and (c) mitigate immediately while you fix the root cause.

*(Distinct from §1.7: that *sizes* memory for steady-state load; this one *hunts* a growth-over-time pathology. Different problem — a detective task, not a capacity calculation.)*

**Key decision 1: confirm it's a leak, not arena retention.** A one-time spike that plateaus is *not* a leak — it's pymalloc holding arenas (§1.6), which is fine. A **leak** grows *monotonically with work done* (per request, per loop iteration) and never plateaus. Confirm by plotting RSS over time against request count: flat-after-warmup = healthy; linear-with-requests = leak. This distinction saves you from "fixing" normal behavior.

**Key decision 2: locate it with tracemalloc snapshot diffing (§1.6), in production.** The only reliable method: take a `tracemalloc` snapshot, let the service handle N requests, take another, `compare_to(..., "lineno")`, and read the top grower — the allocation site whose bytes climb every diff *is* the leak. This is the §1.6 runnable, deployed: expose an admin endpoint that dumps the top-10 growers, or log them periodically. `objgraph.show_growth()` (finds which *object types* are proliferating) and the `gc` module (`gc.get_referrers(obj)` to find *what* is holding the lingering reference) complete the toolkit. The root cause is almost always one of: an **unbounded cache** (a `dict` used as a cache with no eviction), a **growing global/class attribute**, a **closure or event handler** capturing large objects, or **accumulating exception tracebacks** (which hold frame references, hence every local, alive).

**Key decision 3: mitigate now, fix later.** Two immediate levers: (a) **bound every cache** — replace naked `dict` caches with `functools.lru_cache(maxsize=N)` or an explicit LRU with a hard cap, so growth is capped by design; (b) **recycle workers** — Gunicorn's `--max-requests` (and `--max-requests-jitter`) restarts each worker after N requests, resetting its memory. Worker recycling is a legitimate *stopgap* (many production Python services run it permanently as insurance against slow leaks in dependencies) but it treats the symptom — find the reference with tracemalloc and fix the root cause.

**Trade-off made.** Worker recycling trades a tiny amount of throughput (periodic restart cost, cold caches after restart) for bounded memory — a deliberate "cap the blast radius while we debug" choice. `lru_cache(maxsize=N)` trades some cache hit rate (evicted entries recomputed) for a hard memory ceiling. Both accept slightly worse best-case performance for a guaranteed worst case — the right trade for a service that must not OOM.

**Failure modes & mitigations.**

| Failure | Cause | Mitigation |
|---|---|---|
| "Fixed" a leak that wasn't one | mistook pymalloc arena retention for a leak (§1.6) | confirm monotonic-with-work growth first |
| tracemalloc overhead in prod | tracing every allocation is costly | sample (`tracemalloc.start(nframe)` small); enable on one canary pod |
| Leak is in a C extension | tracemalloc only sees Python allocations | use `valgrind`/`memray` (tracks C allocations) for native leaks |
| Cache bounded but hit rate craters | `maxsize` too small | size the cache to the working set; monitor hit rate |
| Leak returns after "fix" | fixed one reference, another remains | diff again after the fix; leaks often come in families |

**Why this over the alternatives?** Guessing ("it's probably the cache") wastes days and often fixes the wrong thing. `tracemalloc` diffing is *evidence-based* — it names the file and line. Restarting workers blindly hides the leak forever and masks a possibly-worsening bug. The disciplined path — confirm, locate with evidence, bound + recycle as a stopgap, fix the reference — is faster and permanent. **Cross-reference:** the agent version of this exact leak is §2.3 — an agent whose conversation history or scratchpad grows every turn is a lingering reference that leaks both RSS *and* tokens, hunted with the same tracemalloc method.

---

## 1.9 Case studies

### ① Instagram disabled Python's garbage collector to reclaim copy-on-write memory

**What happened.** Instagram runs its Django/Python web servers with a **pre-fork** model (Day 2, Day 3 §1.9): a master process loads all the code and warms caches, then `fork`s many worker processes to handle requests. `fork` uses **copy-on-write** (CoW): the children initially *share* the parent's physical pages, and a page is only privately copied when a child *writes* to it — so, in theory, hundreds of workers share one copy of the (large, read-only) code and warmed data, saving gigabytes. Instagram observed the opposite: **shared memory steadily eroded** over each worker's life — pages that should have stayed shared were being copied private, RSS ballooned, and they fit fewer workers per machine.

**The mechanism (a direct §1.6 + §1.3 payoff).** The culprit was CoW being triggered on pages the workers only *read*. Two Python-specific causes: (a) **reference counts live in every object's header** (§1.6), and CPython increments/decrements them even on read — so a worker merely *touching* a shared object writes its refcount, dirties the page, and forces a private copy (§1.3's CoW fault); (b) the **cyclic GC**, when it ran, walked and marked large numbers of long-lived objects, touching their GC headers and dirtying still more shared pages. Instagram's most impactful single change was to **disable the cyclic GC entirely** (`gc.disable()`), which stopped the GC-driven page dirtying; they reported roughly a **10% memory reduction** and better CoW sharing across workers (and, being fork-heavy and short-per-request, they didn't accumulate problematic cycles). The refcount-in-header problem is deeper and led CPython to add **`gc.freeze()`** (3.7+) — call it after warmup and before forking to move long-lived objects out of GC's scan set so their pages stay shared.

**The engineering lesson (tied to today).** This is §1.6's "the coat-check writes on every glance" broken-analogy point, made corporate: in a language where *reading* an object mutates its memory (the refcount), the OS's copy-on-write optimization (§1.3) silently defeats itself across forked workers, and a "garbage collector" that helps single-process memory can *hurt* multi-process memory by dirtying shared pages. The general principle: **understand your allocator and your fork/CoW interaction before you scale horizontally** — the same code that's memory-efficient in one process can waste gigabytes across a fork farm. Primary source: Instagram Engineering, "Dismissing Python Garbage Collection at Instagram" (2017) and "Copy-on-write friendly Python garbage collection" — **verify current details**; CPython's fork/CoW behavior and `gc.freeze` semantics have evolved, and free-threaded builds (Day 20) change the refcount story again.

### ② The Kubernetes nightly OOMKill loop — a limit sized at p50 vs a p99 workload

**What happened (the canonical shape, not a single named public postmortem).** A team deploys a Python service to Kubernetes and sets the pod's memory **limit** based on what they observed in a load test at *typical* load — its **p50** RSS, say 512 MB. It runs fine for hours. Then, every night during a scheduled batch job (or a traffic spike, or a report-generation endpoint), a subset of requests each hold much more memory transiently, the pod's RSS crosses **512 MB `memory.max`**, and the **cgroup OOM killer** (§1.5) `SIGKILL`s the process. Kubernetes sees the container die with exit code **137**, marks it **OOMKilled**, restarts it — and because the cause is structural (the p99 workload always exceeds the p50 limit), it happens again the next night: a recurring **CrashLoopBackOff**-adjacent pattern where the pod is healthy most of the time and killed on every spike, dropping in-flight requests each time (no graceful shutdown — §1.5's uncatchable kill).

**The engineering lessons (every one a direct application of today).**

1. **A container limit is a hard `memory.max` OOM trigger, not a suggestion (§1.5).** Unlike `RLIMIT_AS` (§1.5's catchable `MemoryError`), crossing a cgroup limit is an **uncatchable `SIGKILL`** — no exception, no cleanup, in-flight requests lost. You cannot "handle" it in code; you can only *size for it*.
2. **Size limits at p99, not p50 (§1.7).** Memory usage is a distribution, not a number. The limit must cover the realistic peak (big requests, GC pauses holding memory, concurrency bursts, pymalloc arena retention §1.6), or every excursion above the mean kills the pod. "It was fine in the load test" tests the mean; production runs the tail.
3. **`requests` vs `limits` and the QoS trap (§1.5).** If many pods have `requests` far below `limits`, the node is **overcommitted** — the scheduler packs pods by request, they spike together, and the *node* runs out, invoking the kernel OOM killer, which may kill a *different, innocent* pod by `oom_score`. Pods in the **Guaranteed** QoS class (request == limit) are last to be killed; critical services should be Guaranteed.
4. **`OOMKilled` + exit 137 is a specific, recognizable signature.** When you see it, the diagnosis is immediate: the process touched more memory than its cgroup allowed. The fix is either raise the limit to the real p99, cap the per-request footprint (§1.7), or fix the underlying leak (§1.6/§1.8) if RSS grows monotonically rather than spiking.

**Per Principle 7:** this is the *canonical, ubiquitous shape* of Kubernetes memory incidents rather than one specific named company postmortem, so I present it as the well-documented pattern it is (visible in countless k8s troubleshooting guides and the official docs) and do not attribute it to an invented named outage. The mechanism — cgroup `memory.max` → OOM kill → exit 137 → restart — is exactly documented. Primary sources: Kubernetes docs, "Assign Memory Resources to Containers and Pods" and "Node-pressure Eviction"; `Documentation/admin-guide/cgroup-v2.rst` (memory controller); the `OOMKilled` reason in `kubectl describe pod`. **Verify current** — cgroup v1→v2 and k8s swap support have changed the details across versions.

---

## 1.10 In production (Part 1)

**Best practices, beginner → senior.**

| Level | Habit |
|---|---|
| Beginner | Know `VmRSS` (physical) ≠ `VmSize` (virtual) — watch RSS, not VIRT (§1.1). Understand a Python "leak" is a lingering reference, not lost memory (§1.6). Don't expect `del` to shrink RSS (§1.6). |
| Intermediate | Set container memory `limit` at measured **p99**, not p50 (§1.7, §1.9). Bound every cache (`lru_cache(maxsize=…)`) — an unbounded cache is a designed leak (§1.8). Recognize exit 137 / `OOMKilled` instantly (§1.9). Use `tracemalloc` diffs to locate leaks, never guess (§1.6, §1.8). |
| Senior | Budget memory from first principles (base + per-worker + Little's-Law concurrency × per-request footprint + honest headroom, §1.7). Understand fork/CoW interaction with refcounts before scaling horizontally (§1.9); `gc.freeze()` after warmup. Prefer bounded working sets and streaming over "load it all" (the Build). Set QoS/`oom_score_adj` so the OOM killer sacrifices the right process (§1.5). |

**Monitoring & observability — what to watch.**

1. **RSS / `working_set_bytes` per pod vs its limit** (`kubectl top`, cAdvisor, `/proc/self/status` `VmRSS`). RSS approaching the limit = OOMKill imminent (§1.5). Working-set climbing monotonically with requests = a leak (§1.8).
2. **Major page-fault & swap-in rate** (`vmstat 1` `si`/`so`; per-process `maj_flt`). Nonzero sustained = thrashing / memory pressure (§1.3) — worse than a clean OOM.
3. **OOMKill events & restart counts** (`kubectl get events`, container `OOMKilled` reason, exit code 137). Any = under-sized limit or a leak (§1.9).
4. **GC stats** (`gc.get_stats()`, collection counts/durations). Frequent gen-2 collections or long pauses = many long-lived cyclic objects (§1.6) — consider `gc.freeze()` or reducing cycle creation.
5. **Per-endpoint memory footprint.** Which routes allocate the most per request (feeds §1.7's per-request footprint number). A single endpoint loading a whole file into RAM is the classic culprit — stream it (the Build).

**Failure modes & first move.**

| Symptom | Likely cause | First move |
|---|---|---|
| Pod restarts, exit 137, `OOMKilled` | RSS crossed cgroup limit (§1.5/§1.9) | check if spike (raise limit to p99) or monotonic growth (leak → §1.8) |
| RSS climbs all day, never drops | lingering reference — a leak (§1.6/§1.8) | tracemalloc snapshot diff; find & bound the reference/cache |
| Latency exploded overnight, CPU normal | swapping / major faults (§1.3) | check `vmstat` `si/so`; reduce working set or add RAM; disable swap for decisive OOM |
| `del` / GC ran but RSS didn't shrink | pymalloc keeps arenas (§1.6) | expected — size for peak; recycle workers (`--max-requests`) if it must return |
| Forked workers use far more RAM than expected | CoW broken by refcount/GC page-dirtying (§1.9) | `gc.freeze()` after warmup; measure `Pss` not `Rss`; fewer fatter workers |

**Scaling behaviour.** Memory scales with *concurrency × per-request footprint* (§1.7), so the same tricks that bound concurrency (Day 3 §1.7 pools, backpressure) bound memory. Per-connection memory is why C10K (Day 3 §1.6) is a *memory* story as much as a scheduling one. **Cost.** RAM is the expensive, hard-capped resource in most deployments — you pay for reserved `requests` whether used or not, and the OOM killer enforces `limits` without mercy. The cheapest memory is the memory you never allocate: stream instead of buffering, bound every cache, and keep the working set below RAM so you never touch swap.

---

## 1.11 Failure modes & common misconceptions (Part 1)

| Misconception | Reality |
|---|---|
| "Allocating memory reserves physical RAM." | It reserves *virtual* address space; physical RAM is delivered lazily on first touch — page fault (§1.1, §1.3). |
| "A pointer is a physical RAM address." | It's a *virtual* address, translated per-access by the MMU/TLB; the same virtual address differs across processes (§1.1). |
| "`del x` returns memory to the OS." | It drops a reference; the object frees inside Python, but pymalloc often keeps the arena — RSS may not drop (§1.6). |
| "A Python memory leak is lost memory." | It's a *lingering reference* — something still points at it, so the GC can't reclaim it (§1.6, §1.8). |
| "Reference counting handles everything." | It can't free reference *cycles*; that's why CPython also has a cyclic GC (§1.6). |
| "The GIL/refcount makes fork+CoW memory-efficient." | Refcounts written on read dirty shared pages, breaking CoW across workers (Instagram, §1.9). |
| "Set the container limit to average usage." | Sizing at p50 gets you OOM-killed on every p99 spike — set at p99 (§1.7, §1.9). |
| "An OOM gives a catchable error." | `RLIMIT_AS` does (`MemoryError`); a cgroup/system OOM is an uncatchable `SIGKILL`, exit 137 (§1.5). |
| "Swap prevents out-of-memory." | It defers it, but thrashing (§1.3) is slower than a clean OOM — many servers disable swap on purpose. |
| "A Python int is 8 bytes." | It's a ~28-byte heap object (header + digits); that's the cost of everything-is-an-object (§1.6). |

## 1.12 Interview & practice questions (Part 1)

1. What's the difference between virtual and physical memory, and why does `VmSize` exceed `VmRSS`? *(Virtual = the address space a process sees; physical = RAM actually backing it, delivered lazily on fault — §1.1.)*
2. Walk a page fault: what are minor, major, and invalid faults, and what does each cost? *(Map-only ~µs; disk/swap ~ms; SIGSEGV — §1.3.)*
3. Why is a Python "memory leak" almost never lost memory? How do you find one? *(A lingering reference keeps it reachable; tracemalloc snapshot diff — §1.6, §1.8.)*
4. Why does a Python int cost ~28 bytes, and why does that matter at scale? *(PyObject header — refcount + type ptr; a million ints ≈ 28+ MB → numpy for numeric data — §1.6.)*
5. Reference counting frees objects immediately — why does CPython still need a GC? *(Reference cycles have nonzero refcounts forever; the cyclic GC collects them — §1.6.)*
6. What is the OOM killer, and how does a cgroup OOM differ from a `RLIMIT_AS` failure? *(Kernel kills a process on unsatisfiable fault; cgroup = uncatchable SIGKILL/137, RLIMIT = catchable MemoryError — §1.5.)*
7. A pod is `OOMKilled` nightly but healthy by day. Diagnose. *(Limit sized at p50, p99 spike crosses cgroup `memory.max` — §1.7, §1.9. Raise limit to p99 or cap per-request footprint.)*
8. Why did Instagram disable Python's GC, and what does it teach about fork/CoW? *(Refcount/GC page-dirtying broke copy-on-write across workers, eroding shared memory — §1.9.)*
9. Stack vs heap: which can leak, which overflows, and who frees each? *(Heap leaks (manual/GC), stack overflows (deep recursion); stack self-frees on return — §1.4.)*
10. Your service's RSS won't drop after processing a big batch. Bug or expected? *(Expected — pymalloc keeps arenas; size for peak or recycle workers — §1.6.)*

---

# PART 2 — AGENTIC AI

> **How Part 2 relates to Part 1.** An agent (the `while` loop around an LLM, Day 24) has **three** memory budgets, and all three are Part 1 concepts wearing new clothes: (1) the **process RAM** the agent runs in — ordinary §1.1–§1.6 backend memory, leaked the ordinary way (§1.6) by an unbounded history (§2.3); (2) the **context window** — the model's *token* budget, a RAM-like finite space you must manage or overflow; (3) the **KV cache** — the *GPU* memory the model server spends per concurrent request, which makes LLM inference **memory-bound** exactly the way §1.3's working set makes a server memory-bound. I treat the backend and the transformer internals as black boxes here and cross-reference Part 1 and Day 63. The recurring thesis: *an agent is a distributed backend system with one non-deterministic component (the model)* — and its memory problems are ordinary backend memory problems (§1.5–§1.8) plus one new, expensive tier (the KV cache) that no CRUD API has.

## 2.1 The context window — a RAM-like budget measured in tokens

**Depth: [WORKING]** — you must manage it correctly and reason about its limits and costs; the tokenizer/transformer internals are Day 59/63.

### Intuition

The model is stateless (Day 23): it remembers nothing between calls. "Conversation" is an illusion your backend builds by **re-sending the entire history every turn**. That history lives in the **context window** — the maximum number of **tokens** (≈ word-chunks; Day 23) the model can attend to at once (e.g. 200K tokens, verify per model). The context window is to *tokens* what RAM is to *bytes*: a **fixed, finite budget** that everything competing for the model's attention (system prompt + full history + retrieved documents + the new message + room for the response) must fit inside. Overflow it and the request **errors** (or the framework silently truncates, dropping the oldest turns — losing memory). And unlike RAM, you **pay per token, every turn** — so an unmanaged, growing context is both a *capacity* problem (you'll hit the window) and a *cost* problem (the bill grows with history length, Day 23's cost blowup).

The direct mapping to Part 1: the context window is a memory budget you **allocate** (add to history), that gets **full** (approach the token limit), that you must **evict** from (trim/summarize old turns — the agent's version of §1.3's page eviction), and that leaks if you never bound it (§2.3).

### Analogy — a fixed-size whiteboard the model can see

The model can only "see" what's written on a whiteboard of fixed size (the context window). Each turn, you wipe it and **rewrite the entire conversation** plus the new question, because the model has no memory of last turn (Day 23). As the conversation grows, the whiteboard fills; when it's full, you must **erase old lines** to fit new ones — and whatever you erase, the model can no longer see (it "forgets"). You're charged for every character you write, every single turn.

**Where the analogy breaks:** a real whiteboard's cost is one-time (write once, read many); the context window is **re-billed in full every turn** — writing "the whole conversation" onto the board *each* time means a 30-turn chat re-sends turns 1–29 thirty times, so naive history-resending makes cost grow *quadratically* with conversation length (Day 23's blowup), a cost structure no physical whiteboard has. Also, erasing a whiteboard line loses it forever; a well-built agent instead **summarizes** old turns (compressing many tokens into few) or **offloads** them to external memory/RAG (Day 43) it can retrieve later — a "spill to disk" (§1.3) the whiteboard can't do.

### Worked example — the token budget of a growing conversation

```
   Context window budget (illustrative — verify limits/pricing per model, they drift):
     total window            : 200,000 tokens
     system prompt           :   1,500 tokens  (fixed every turn)
     retrieved docs (RAG)    :   4,000 tokens  (Day 43)
     conversation history    : grows ~500 tokens/turn
     new user message        :     200 tokens
     reserved for response   :   4,000 tokens  (max_tokens)
     ─────────────────────────────────────────
     usable for history = 200,000 − 1,500 − 4,000 − 200 − 4,000 ≈ 190,300 tokens
     → at ~500 tokens/turn, the window fills after ~380 turns → then you MUST evict/summarize

   Cost angle (Day 23): if you re-send full history every turn, tokens billed by turn N
   grow linearly with N, so cumulative cost over a conversation grows ~quadratically.
   Trimming/summarizing history is the fix — the §1.3 "eviction" of the agent world.
```

**No runnable example here** (honesty, Principle 6/7): a faithful token-budget demo requires calling a specific provider's tokenizer, which drifts fast and is Day 23's runnable surface (`client.messages.count_tokens` for Claude — consult the `claude-api` skill; `tiktoken` for OpenAI; `client.models.count_tokens` for Gemini). The *management* of that budget under concurrency is §2.3's runnable. I'm not inventing a fake tokenizer to satisfy the format.

---

## 2.2 The KV cache — why LLM inference is memory-bound

**Depth: [WORKING]** — you must reason about its cost and why it dominates inference-server memory; the transformer attention mechanism that produces it is **Day 63** (named, not opened here).

### Intuition

When a model generates text, it processes tokens one at a time, and at each step attention needs the **keys and values** (K, V — Day 63's terms) computed for *every previous token*. Recomputing them every step would be quadratically wasteful, so the server **caches** them: the **KV cache**. This cache grows **linearly with sequence length** and is stored **per concurrent request** in GPU memory. The consequence, which reshapes the entire §1.7 memory-budgeting problem for LLM servers: **inference is memory-bound, not compute-bound** — you run out of GPU memory (weights + KV cache for all in-flight requests) long before you run out of compute, and the KV cache, not the model weights, is usually what limits how many requests you can serve at once.

This is *why* Day 4 sits before the LLM days: the KV cache is exactly a §1.1–§1.7 memory-budget story on a scarcer, pricier tier (GPU HBM), and the state-of-the-art solution (vLLM's PagedAttention, §2.5) **literally applies OS paging (§1.2)** to it.

### Analogy — a translator's running notes

A simultaneous translator (the model generating) keeps **notes on everything said so far** so each new word can be translated in context. The longer the speech (sequence), the thicker the notes (KV cache grows linearly). A translation *agency* (inference server) running many simultaneous sessions needs desk space for *every* translator's notes at once — and runs out of **desk space** (GPU memory), not out of *translators' skill* (compute). Desk space is the binding constraint.

**Where the analogy breaks:** the translator's notes are cheap paper; KV-cache memory is **the scarcest, most expensive resource in the system** — GPU HBM measured in tens of GB costing thousands of dollars, so "desk space" here is the dominant cost driver, not an afterthought. And crucially, classic serving **pre-reserved** a full max-length notepad per session even if the speech was short (massive internal fragmentation, §1.6-like waste) — the exact inefficiency PagedAttention (§2.5) fixes by paging the notes. The paper analogy has no "you had to reserve a 2000-page notebook for a 10-word sentence" waste.

### Worked example — the KV-cache memory formula and why it dominates

```
   KV cache bytes per token (per request) ≈
       2 (K and V) × num_layers × num_kv_heads × head_dim × bytes_per_element

   For a mid-size model (ILLUSTRATIVE — plug in the real config, Day 63; verify):
     2 × 40 layers × 8 kv_heads × 128 head_dim × 2 bytes (fp16)
       ≈ 163,840 bytes/token  ≈ 0.16 MB/token

   A single request with 8,000 tokens of context:
     8,000 × 0.16 MB ≈ 1.3 GB of GPU memory  — for ONE request's KV cache.

   Serving 40 concurrent requests at 8K tokens each:
     40 × 1.3 GB ≈ 52 GB of KV cache  — ON TOP of the model weights (tens of GB).
   → GPU memory (not compute) caps concurrency. This is §1.7's budget on GPU HBM.
```

The shape is identical to §1.7 (`concurrency × per-request footprint + base`), but the per-request footprint is *gigabytes* and the base (weights) is tens of GB, so the arithmetic that sized a CRUD pod at 1.5 GB sizes an inference node at 80 GB — and gets it wrong catastrophically if you ignore the KV cache. **Verify every number against the actual model config and framework** (Principle 6): layer counts, head dims, grouped-query-attention `num_kv_heads`, and quantization (fp16 vs fp8) all change the constant, and they drift.

**No runnable example** (honesty): a real KV-cache measurement needs a GPU and a specific inference framework (vLLM/TGI), which is Day 63's surface. Inventing GPU output would violate Principle 6. The formula above is the reasoning tool; §2.4 designs around it.

---

## 2.3 The agent "memory leak" — unbounded history and scratchpad growth

**Depth: [CORE]** — this is §1.6's lingering-reference leak reincarnated in the agent, and it's the failure that makes long-running agents OOM *and* overspend.

### Intuition

An agent loop (Day 24) accumulates state every turn: the **conversation history** (grows ~500 tokens/turn, §2.1), a **scratchpad** of intermediate tool results, gathered facts, a running **cost/token tally**. If any of these grows **without a bound**, you have *simultaneously*:

- a **process memory leak** (§1.6) — the history list/dict is a lingering reference that grows RSS every turn until the worker is OOM-killed (§1.5), exactly §1.8's monotonic-with-work growth;
- a **context-window overflow** (§2.1) — the history eventually exceeds the token limit and the request errors or silently drops old turns;
- a **cost blowup** (Day 23) — re-sending the full growing history every turn makes the per-turn bill climb and cumulative cost grow ~quadratically.

One unbounded structure, three failures at once. This is the direct payoff of §1.4/§1.6/§1.8 for agent builders: *an agent that never trims its history or scratchpad is a memory leak whose symptoms are an OOM kill, a context overflow, and a runaway bill.* The fix is the same as §1.3/§1.8: **bound it** — cap history length (keep the last K turns), **summarize** older turns into a compact form (compress many tokens into few — the agent's §1.3 eviction), or **offload** to external memory/RAG (Day 43 — spill to "disk"). And guard it under concurrency exactly as Day 3 §2.2 (a scratchpad mutated by parallel tools both leaks *and* races).

### Analogy — a diary you must read aloud in full before every entry

Imagine you must **read your entire diary aloud** before writing each new entry, and you're charged per word read. Early on, quick. By entry 500, you spend all day reading 499 old entries before adding one — the diary (history) has grown unbounded, the reading (context) has hit the limit of what you can say in a day, and the cost (per-word charge) has exploded. The only sane strategies: keep only the **last few entries**, periodically **summarize** old ones into a paragraph, or **file** old entries in a cabinet you consult only when relevant (RAG).

**Where the analogy breaks:** the diary's paper cost is trivial; the agent pays **three separate penalties** (RAM, context capacity, token cost) for the same unbounded growth, and the RAM penalty ends in a *silent SIGKILL* (§1.5) that kills the current user's conversation mid-response — the diary has no "and then you drop dead at entry 500" failure. And a diary read is sequential and cheap; re-sending history is re-billed in full every turn, so the cost is *quadratic*, not linear, in conversation length.

### Runnable example — an agent history leak, measured with tracemalloc, then bounded

```python
# agent_history_leak.py — Linux/any OS. stdlib only (simulates an agent loop; no LLM).
# Run:  python3 agent_history_leak.py
import tracemalloc

# A turn's messages: a user msg + a tool result blob (stand-in for real content, Day 24).
def make_turn(i):
    return [
        {"role": "user", "content": f"question {i}"},
        {"role": "assistant", "content": "x" * 50_000},   # ~50 KB of accumulated result/history
    ]

# ---------- 1) UNBOUNDED: history grows every turn -> a leak (§1.6/§1.8) ----------
def run_unbounded(turns):
    history = []                          # a lingering reference that only ever grows
    for i in range(turns):
        history.extend(make_turn(i))      # never trimmed -> RSS + tokens climb forever
    return history

# ---------- 2) BOUNDED: keep only the last K turns (evict old -> §1.3) ----------
def run_bounded(turns, keep_last=5):
    history = []
    for i in range(turns):
        history.extend(make_turn(i))
        # evict: keep system-ish head + the last `keep_last` turns (2 msgs each)
        if len(history) > keep_last * 2:
            history = history[-keep_last * 2:]   # summarize-in-real-life; truncate here
    return history

tracemalloc.start()
snap0 = tracemalloc.take_snapshot()
h1 = run_unbounded(200)
snap1 = tracemalloc.take_snapshot()
h2 = run_bounded(200, keep_last=5)
snap2 = tracemalloc.take_snapshot()

grow_unbounded = sum(s.size_diff for s in snap1.compare_to(snap0, "filename"))
grow_bounded   = sum(s.size_diff for s in snap2.compare_to(snap1, "filename"))
print(f"UNBOUNDED history after 200 turns : {len(h1):>4} msgs, "
      f"+{grow_unbounded/1024/1024:5.1f} MB retained   <- the leak (§1.6/§1.8)")
print(f"BOUNDED   history after 200 turns : {len(h2):>4} msgs, "
      f"+{grow_bounded/1024/1024:5.1f} MB retained   <- capped: RAM, context, and cost all bounded")
print(f"unbounded would also OVERFLOW the context window (§2.1) and grow cost ~quadratically (Day 23)")
```

Actual output (CPython, Linux; exact MB vary):

```
# -> UNBOUNDED history after 200 turns :  400 msgs, + 19.1 MB retained   <- the leak (§1.6/§1.8)
# -> BOUNDED   history after 200 turns :   10 msgs, +  0.0 MB retained   <- capped: RAM, context, and cost all bounded
# -> unbounded would also OVERFLOW the context window (§2.1) and grow cost ~quadratically (Day 23)
```

**Why this works, line by line.**

- `run_unbounded` does `history.extend(...)` every turn and **never trims** — the `history` list is a lingering reference (§1.6) that grows ~100 KB/turn. After 200 turns it retains ~19 MB in-process (and, in a real agent, this history is *re-sent to the model every turn* — so it's simultaneously a context-window filler (§2.1) and a per-turn cost multiplier, Day 23). This is precisely §1.8's monotonic-with-work growth — a leak — but in an agent the same structure *also* blows the token budget and the bill. One bug, three failures.
- `tracemalloc.compare_to` (the §1.6/§1.8 tool) measures the retained growth of each phase, proving the unbounded run holds ~19 MB while the bounded run holds ~0 — the *evidence-based* leak diagnosis applied to the agent.
- `run_bounded` **evicts** old turns (`history[-keep_last*2:]`), keeping only the last 5 turns, so retained memory stays flat regardless of conversation length — the agent's §1.3 page-eviction. In production you'd **summarize** the evicted turns into a short note (compressing tokens, preserving meaning) or **offload** them to a vector store (Day 43) rather than hard-truncating; the principle is identical — *bound the working set*.
- **Honesty caveat:** this uses in-process lists as a stand-in for real conversation state and hard-truncation as a stand-in for summarization; a real agent's history lives in a DB (Day 38) keyed by session, is trimmed/summarized with a real token budget (§2.1, Day 23's `count_tokens`), and must be guarded by a per-session lock if parallel tools mutate it (Day 3 §2.2). The *memory principle* — unbounded history is a leak; bound it — is exact and framework-agnostic.

**The rule for agent builders.** *Every accumulating agent structure — conversation history, scratchpad, gathered facts, cost tally, memory store — must have a bound (max turns/tokens), an eviction policy (drop or summarize oldest), or an offload target (external memory/RAG).* An agent without one is §1.8's leak with §2.1's overflow and Day 23's cost blowup stacked on top — and because the RAM failure is §1.5's uncatchable OOM kill, it ends the user's conversation without warning. This is §1.7/§1.8 discipline (bound the working set; hunt leaks with tracemalloc) applied to the agent.

---

## 2.4 System design — memory budgeting for an LLM inference / agent server

**The problem.** Design the memory budget for a GPU-backed LLM inference server (the black box behind your agent's model calls, Day 23) that serves many concurrent agent sessions. Requirements: (a) never OOM the GPU (an OOM here kills *all* in-flight requests on that GPU — §1.5 at GPU scale); (b) maximize concurrency (requests served at once) within the GPU's fixed HBM; (c) handle variable-length contexts (§2.1) without pre-reserving worst-case memory per request; (d) stay within the per-turn cost/latency budget (Day 23/24).

**Key decisions.**
- **Budget = weights + KV cache × concurrency + activation/overhead** (§1.7's formula on GPU HBM). Weights are fixed (tens of GB, or less with quantization); the KV cache (§2.2) is the *variable, dominant, per-request* term. Compute the max concurrent requests as `(HBM − weights − overhead) / (KV bytes per request at target context length)`. This is §1.7's `concurrency × per-request footprint` with a gigabyte-scale footprint.
- **Don't pre-reserve max-length KV per request** — that's the classic fragmentation waste (§2.2/§2.5): a request that ends at 200 tokens shouldn't hold a 200K-token reservation. Allocate KV **incrementally** as the sequence grows (demand paging for the KV cache — §1.3), which is exactly **PagedAttention** (§2.5).
- **Bound context length per request** (§2.1/§2.3) — cap max tokens so one runaway session can't consume the whole GPU's KV budget (the agent-leak §2.3 defense, enforced at the server). Reject or summarize over the cap.
- **Continuous batching** — pack multiple requests' tokens into each forward pass to keep the GPU busy (compute efficiency) *within* the memory bound; the memory ceiling, not compute, sets the batch size.
- **Admission control / backpressure** (Day 3 §1.7) — when the KV budget is full, **queue or reject** new requests (`429`) rather than accepting one that triggers a GPU OOM killing everyone. The OOM kill (§1.5) is far worse here than a rejected request.

**Trade-off.** Paged/incremental KV allocation + continuous batching + admission control is far more complex than "one request at a time, pre-reserve max length" — but the simple version wastes most of the GPU (fragmentation, §2.2) and OOMs unpredictably. You accept serving complexity for 2–4× higher throughput on the same fixed, expensive GPU (§2.5's measured result). The context-length cap trades some capability (very long conversations must summarize/offload) for a hard memory guarantee.

**Failure modes.**

| Failure | Cause | Mitigation |
|---|---|---|
| GPU OOM kills all in-flight requests | KV cache exceeded HBM (§2.2/§1.5) | admission control; cap context length; size concurrency by KV budget |
| Low GPU utilization / wasted memory | pre-reserved max-length KV per request (§2.2) | PagedAttention — incremental, paged KV (§2.5) |
| One long session starves others | unbounded per-request context (§2.3) | per-request token cap; summarize/offload over cap (§2.1) |
| Throughput collapses under load | no batching; memory-bound not compute-bound (§2.2) | continuous batching within the memory ceiling |
| Cost per turn creeps up | history re-sent unbounded (§2.3, Day 23) | trim/summarize history; prompt caching where available (verify per provider) |

**Cross-reference.** This is §1.7 (memory budgeting) + §1.5 (OOM consequences) + §2.2 (the KV footprint) + §2.3 (bounding per-request state), assembled on GPU HBM. Frameworks (vLLM, TGI — §2.5) implement the paged KV cache and continuous batching for you, but knowing the §1.1–§1.7 mechanism is how you size the deployment and debug a GPU OOM. **Preview of Day 63**, which opens the transformer and KV cache to the metal.

## 2.5 Case study — vLLM's PagedAttention: OS paging applied to the KV cache

**What it is (real, published).** vLLM (Kwon et al., "Efficient Memory Management for Large Language Model Serving with PagedAttention," SOSP 2023) observed that classic LLM serving **pre-allocated a contiguous, max-length KV-cache region per request**, wasting 60–80% of it to internal fragmentation (short requests holding max-length reservations) and reservation for growth — the exact §2.2/§1.6 fragmentation problem. Their fix is **PagedAttention**: store the KV cache in **fixed-size non-contiguous blocks** (like OS **pages**, §1.2) and keep a **block table** per request mapping logical KV positions to physical blocks (like a **page table**, §1.2) — allocating blocks **on demand** as the sequence grows (like **demand paging**, §1.3). This eliminates the fragmentation, lets memory be shared across requests with common prefixes (copy-on-write of KV blocks — §1.9's CoW!), and packs far more concurrent requests into the same GPU HBM. The paper reports **2–4× higher throughput** than prior systems at the same latency.

**The engineering lesson (a direct §1.1–§1.3 payoff).** The state-of-the-art solution to the *newest* memory problem in computing (serving LLMs) is the *oldest* idea in this note: **virtual memory** — pages, a page table, demand paging, and copy-on-write — applied to a new resource (GPU KV cache) instead of RAM. This is the strongest possible argument for why Day 4 precedes the LLM days: the abstractions the OS invented in the 1960s to manage scarce, fragmented, contiguous-address-demanding memory are *exactly* the abstractions that make LLM serving efficient in the 2020s. Understand §1.1–§1.3 and PagedAttention is obvious; skip them and it's magic. Primary source: Kwon et al., SOSP 2023 (the PagedAttention paper); the vLLM documentation and source. **Verify current** — throughput numbers and the technique's details evolve fast, and competing systems (TGI, TensorRT-LLM) implement similar paged-KV schemes.

## 2.6 In production (Part 2, condensed per [WORKING] tier)

- **Best practice:** bound every accumulating agent structure (history, scratchpad, cost tally) with a max size, an eviction/summarization policy, or an offload target (§2.3). Cap per-request context length at the inference server (§2.1/§2.4). Size the inference GPU budget by weights + KV × concurrency, not by compute (§2.2/§2.4). Use a framework with paged KV + continuous batching (§2.5). Guard shared agent state under concurrency (Day 3 §2.2).
- **Top failure mode:** unbounded conversation history — the agent §2.3 leak — which OOM-kills the worker (§1.5), overflows the context window (§2.1), *and* runs up the token bill (Day 23), all from one unbounded structure. Symptom: RSS *and* cost-per-turn climb monotonically with conversation length, then a `SIGKILL`/exit-137 mid-conversation (§1.9).
- **Monitor:** process RSS per agent worker vs its limit (OOM risk — §1.5); KV-cache utilization / GPU memory on the inference server (§2.2, OOM risk); context tokens per request and its distribution (approaching the window — §2.1); cost-per-turn tail (unbounded history — §2.3). Trace each turn's token count with its session id (Day 24's audit log) — memory/cost leaks are invisible without per-turn history-size tracking.

---

# PART 3 — THE BRIDGE

> Parts 1 and 2 each ended at the same wall: a **finite physical memory budget** and the **OOM killer** that enforces it. Part 3 is where a backend worker (Part 1) and the agent it runs (Part 2) discover they share *one* container's memory limit, one GPU's HBM, and one `SIGKILL` when either overruns it. No new concepts here — only the interactions. (Day 27–28 put the agent behind an ASGI server; Day 63 opens the KV cache; Day 43 builds the external memory that offloads it.)

## 3.1 Where the layers meet: one memory budget, two claimants

A production agent runs *inside* a backend worker process (Day 27–28): the process holds the Python interpreter + libraries (§1.6), the web framework, **and** the agent's growing conversation state (§2.3) — all inside **one cgroup memory limit** (§1.5/§1.7). Trace the memory:

```
   ONE CONTAINER, ONE memory.max (§1.5) — everything below shares it:
   ┌──────────────────────────────────────────────────────────────────────┐
   │  interpreter + code + libs (§1.6)          ~150 MB baseline            │
   │  + web framework / server (Day 28)         ~ per worker                │
   │  + PER SESSION agent state (§2.3):                                     │
   │       conversation history (grows/turn §2.1)  ─┐                       │
   │       scratchpad / gathered facts             │ these GROW every turn  │
   │       cost/token tally                        ─┘ unless bounded (§2.3) │
   │  × concurrent agent sessions (Little's Law, Day 3 §1.7; W is SECONDS)  │
   └──────────────────────────────────────────────────────────────────────┘
        │  and, on the SEPARATE inference server: GPU HBM = weights + KV cache
        ▼  per concurrent request (§2.2) — its OWN memory budget, its OWN OOM
   If total > memory.max → cgroup OOM kill (§1.5) → SIGKILL → every session dies.
```

The load-bearing observation: **an agent's per-session memory is not fixed — it grows with conversation length (§2.3)** — so the §1.7 budget (`concurrency × per-request footprint`) has a *moving* footprint. A CRUD request's 8 MB is constant; an agent session's footprint climbs every turn until bounded. Size the container for the agent's *early* footprint and long conversations will grow past `memory.max` and OOM the worker — taking *every* concurrent session on it down, not just the long one (§1.5's "victim isn't always the culprit"). This is the first thing that breaks when a growing agent runs in a container sized like a stateless API.

## 3.2 The shared failure mode: an unbounded agent leaks the whole worker into an OOM kill

This is the punchline of the day, unifying §1.5, §1.6, §1.8, §2.1, §2.3, and the Kubernetes case study (§1.9).

Recall §1.5: a cgroup OOM is an **uncatchable `SIGKILL`** — no `MemoryError`, no cleanup, no draining in-flight work. Now put an unbounded agent (§2.3) in that container. Its conversation history grows every turn (§2.1/§2.3); RSS climbs monotonically (§1.6/§1.8's leak); eventually total process memory crosses `memory.max`; the kernel `SIGKILL`s the worker. **Every concurrent user session on that worker dies mid-response** — not just the one with the long conversation that caused it. One user's unbounded chat OOM-kills everyone else's, silently, with exit code 137:

```
   ONE worker (one process, §1.5), serving many agent sessions:

   session A: 400-turn conversation, history NEVER trimmed (§2.3)  ← the leak
        │  RSS climbs: 300 MB → 600 MB → 900 MB → ... → memory.max
        ▼
   ┌─────────────────────────────────────────────────────────────┐
   │  cgroup OOM kill (§1.5): SIGKILL, exit 137, no cleanup        │
   │  sessions B..Z (all of them): killed mid-response, lost       │
   └─────────────────────────────────────────────────────────────┘
   The unbounded history didn't just cost session A tokens (§2.1/Day 23).
   It OOM-killed the whole worker and every session on it. (§1.5, §1.9, §2.3)
```

The GPU side has an identical shape one layer over: an agent that lets one session's context grow unbounded (§2.3) inflates that request's **KV cache** (§2.2) until it exceeds GPU HBM, and the **GPU OOM kills all in-flight requests on that GPU** — the same "one session starves everyone" failure, on the pricier tier. **Both fail the same way (§1.5's uncatchable kill); they fail on different memory (worker RAM vs GPU HBM).** Knowing which tells you where to put the bound:

- **Worker RAM:** bound/summarize/offload conversation history and scratchpad (§2.3); size the container for the *grown* footprint at max conversation length (§1.7/§3.1); recycle workers (`--max-requests`, §1.8) as insurance; set the container limit at real p99 including long sessions (§1.9).
- **GPU HBM:** cap per-request context length (§2.1/§2.4); use paged KV + admission control (§2.4/§2.5); reject over-budget requests (`429`) rather than OOM the GPU.

## 3.3 The dependency map

What each layer needs from the other's memory, and what breaks when either overruns its budget:

```
   ┌────────────────────── AGENT LAYER (Part 2) ──────────────────────┐
   │  needs from backend:                                              │
   │   • a process/container with enough RAM for GROWING session state │
   │     (history + scratchpad × concurrent sessions, §3.1)            │
   │   • a container limit sized at the GROWN p99 footprint (§1.7/§3.1) │
   │   • bounded/summarized/offloaded history (§2.3) so RSS stays flat  │
   │   • GPU HBM budgeted for weights + KV × concurrency (§2.2/§2.4)   │
   │   • admission control so a full KV budget rejects, not OOMs (§2.4) │
   └───────────────────────────┬──────────────────────────────────────┘
                               │ hands off: a growing per-session working set
                               │            that MUST be bounded like any §1.8 leak
                               ▼
   ┌────────────────────── BACKEND LAYER (Part 1) ────────────────────┐
   │  serves back:                                                     │
   │   • virtual memory + demand paging per process (§1.1–§1.3)        │
   │   • a cgroup memory limit + the OOM killer that enforces it (§1.5)│
   │   • Python's heap: refcounts, cyclic GC, pymalloc (§1.6)          │
   │   • tracemalloc to locate the agent's leaking reference (§1.6/§1.8)│
   └──────────────────────────────────────────────────────────────────┘

   FAILURE COUPLING (how one side breaks the other):
   • Agent history never trimmed        → §1.6/§1.8 leak → RSS → cgroup OOM (§1.5) → ALL sessions die (§3.2)
   • Agent context grows unbounded        → KV cache (§2.2) exceeds GPU HBM → GPU OOM → all requests on GPU die
   • Container limit sized at early footprint → long sessions cross memory.max → §1.9 OOMKill loop
   • Backend limit sized at p50            → §1.9 → nightly OOMKill on p99 (long-conversation) spikes
   • Refcount/GC dirties CoW pages         → §1.9 → forked agent workers waste RAM → fewer sessions/node
   • No admission control on KV budget      → accept one request too many → GPU OOM instead of a clean 429 (§2.4)
```

## 3.4 The one sentence

The agent and the backend are not two systems that each have their own memory — they are **one process inside one memory limit**, and every rule from Part 1 (physical RAM is finite and lent, not owned; a leak is a lingering reference you must bound; size limits at p99; the OOM killer is an uncatchable `SIGKILL` that takes everything with it) is *simultaneously* a backend rule and an agent rule. The agent just adds a growing working set (history) and an expensive new tier (the KV cache), so it hits the wall sooner and more expensively: a backend's forgotten global `dict` becomes an agent's un-trimmed conversation, a pod's nightly OOMKill becomes every user's chat dying mid-sentence, and the fragmentation trick the OS invented for RAM in 1962 becomes the PagedAttention that makes serving the model affordable. That is why this memory day sits so early in a plan about building agents.

---

# Topic-wide wrap-up

## Cheat Sheet

| Concept | One-line truth | Tag | § |
|---|---|---|---|
| Virtual memory | Each process gets a private address space; virtual → physical translated per-access | [CORE] | 1.1 |
| Virtual vs resident | `VmSize` = promised; `VmRSS` = physically delivered (lazily, on touch) | [CORE] | 1.1 |
| Page / page table / TLB | 4 KB pages; multi-level table maps page→frame; TLB caches translations | [CORE] | 1.2 |
| Page fault | Minor (map, ~µs) / major (disk-swap, ~ms) / invalid (SIGSEGV) | [CORE] | 1.3 |
| Demand paging / swap | RAM delivered on first touch; cold pages spill to disk; thrash = death | [CORE] | 1.3 |
| Stack vs heap | Stack: small, LIFO, self-freeing, overflows. Heap: large, GC/manual, leaks | [CORE] | 1.4 |
| OOM killer | Uncatchable SIGKILL on unsatisfiable fault; cgroup limit = exit 137 OOMKilled | [CORE] | 1.5 |
| Everything is a PyObject | A Python int is ~28 bytes (refcount + type ptr + value) | [CORE] | 1.6 |
| Reference counting | Frees immediately at refcount 0; a "leak" = a lingering reference | [CORE] | 1.6 |
| Cyclic GC | Backstop for reference cycles refcounting can't free; generational | [CORE] | 1.6 |
| pymalloc / arenas | Arena/pool/block allocator for small objects; keeps arenas → RSS won't drop | [CORE] | 1.6 |
| Memory budgeting | base + per-worker + (Little's-Law concurrency × footprint) + p99 headroom | [CORE] | 1.7 |
| Leak hunting | tracemalloc snapshot diff → the growing line = the leak | [CORE] | 1.8 |
| Context window | The model's token budget; RAM-like, finite, re-billed every turn | [WORKING] | 2.1 |
| KV cache | Per-request GPU memory growing with sequence → inference is memory-bound | [WORKING] | 2.2 |
| Agent history leak | Unbounded history = §1.6 leak + §2.1 overflow + Day 23 cost, at once | [CORE] | 2.3 |
| PagedAttention | OS paging (pages/page-table/demand/CoW) applied to the KV cache | [WORKING] | 2.5 |
| One memory budget | Agent + backend share one cgroup limit; an unbounded agent OOMs all sessions | [CORE] | 3.2 |

## Build This

**Definition of done — virtual memory, a leak, and its fix, all measured.**

1. **Virtual ≫ resident (§1.1):** `mmap` 1 GB anonymous, print `VmSize`/`VmRSS` before/after touching one page and all pages; confirm virtual jumps immediately but resident only on touch (demand paging).
2. **Page faults (§1.3):** touch ~100 MB and print `ru_minflt`/`ru_majflt`; confirm ~25,600 minor faults (≈ MB/4 KB) and ~0 major faults; explain what a nonzero major-fault rate would mean (thrashing).
3. **Stack vs heap (§1.4):** hit `RecursionError` on deep recursion (stack, self-guarded); grow a referenced global list and watch RSS climb (heap, not reclaimed while reachable); `del` it and note RSS may not shrink (§1.6).
4. **Python memory (§1.6):** print `sys.getsizeof(1)` (~28); build a reference cycle with `gc.disable()`, confirm it's *not* freed, then `gc.collect()` and confirm it is; hunt a deliberate leak with a `tracemalloc` snapshot diff and read off the file:line.
5. **OOM behavior (§1.5):** cap with `RLIMIT_AS` and catch a `MemoryError`; then run the same allocation in `docker run -m 128m` and observe the *uncatchable* SIGKILL (exit 137, `OOMKilled`) — write one paragraph on why the two differ.
6. **The agent leak + fix (§2.3):** simulate a 200-turn agent loop with unbounded history (measure retained MB with tracemalloc), then bounded-to-last-K history (measure again); confirm the unbounded version's memory grows with turns and the bounded version stays flat; note that unbounded would *also* overflow the context window and grow cost quadratically.
7. **The bridge (§3.2):** run the unbounded agent loop under `RLIMIT_AS`/`docker -m` and watch the whole "worker" die — connect it to "every session on this worker dies."

Assemble into one repo, commit, and write a one-paragraph explanation of *why* each number came out as it did.

## Active Recall & Self-Test (answer from memory)

1. Explain virtual memory to someone who thinks a pointer is a physical RAM address. Why does `top` show VIRT ≫ RES?
2. Trace a page fault. What are the three kinds, and what does each cost? Which one is a "segfault"?
3. Why is a Python "memory leak" almost never lost memory? How do you *find* one with evidence?
4. Why does CPython need a garbage collector when it already refcounts? Give the exact case refcounting can't handle.
5. Why does a Python int cost ~28 bytes? Why does `del x` often not shrink RSS?
6. What is the OOM killer? How does a cgroup/container OOM differ from a `RLIMIT_AS` failure, and what's exit code 137?
7. A pod is OOMKilled every night but healthy by day. Diagnose and fix, using p50/p99.
8. Why did Instagram disable Python's GC? What does it reveal about refcounts and copy-on-write?
9. Why is LLM inference memory-bound, and what is the KV cache? How does PagedAttention fix its waste?
10. Your agent's conversation history is never trimmed. Name the *three* simultaneous failures and the fix.

**60-second teach-back prompt:** *"Explain to a smart friend who's never coded why every program is told it owns all the RAM (a lie the OS maintains with 'pages'), what actually happens when a program uses more memory than exists (the OOM killer), and why an AI agent that never forgets old messages is the exact same 'ran out of memory' bug as a backend that never clears a growing list — using the 'hotel room numbers' and 'read your whole diary before each entry' analogies."*

## Spaced-Repetition Flashcards

- **Q:** Virtual vs physical memory? **A:** Virtual = the private address space a process sees; physical = RAM backing it, delivered lazily per-page on fault. Same virtual address ≠ same physical in two processes.
- **Q:** `VmSize` vs `VmRSS`? **A:** VmSize = virtual (promised); VmRSS = resident (physically used). VIRT ≫ RES because most virtual pages are untouched promises.
- **Q:** Page-fault types? **A:** Minor (map an in-RAM page, ~µs), major (read from disk/swap, ~ms), invalid (no VMA → SIGSEGV).
- **Q:** Why is random memory access slow beyond cache? **A:** TLB misses → multi-level page walks; and swapping if the working set exceeds RAM.
- **Q:** Stack vs heap — who frees, which leaks? **A:** Stack self-frees on return (overflows on deep recursion); heap is GC/manual (leaks, via lingering references).
- **Q:** What is a Python "memory leak"? **A:** A lingering reference — something still points at the object, so the GC can't reclaim it. Find it with a tracemalloc snapshot diff.
- **Q:** Why does CPython need a GC on top of refcounting? **A:** Reference cycles keep each other's refcount above 0 forever; the cyclic GC collects them.
- **Q:** Why is a Python int ~28 bytes? **A:** It's a heap PyObject: refcount + type pointer + digits + bookkeeping — not a bare 8-byte word.
- **Q:** cgroup OOM vs RLIMIT_AS? **A:** cgroup = uncatchable SIGKILL, exit 137, OOMKilled. RLIMIT_AS = catchable MemoryError. Test with RLIMIT, prod behaves like cgroup — a trap.
- **Q:** Size a container memory limit at p50 or p99? **A:** p99 (+ headroom). p50 gets you OOM-killed on every spike (the nightly OOMKill loop).
- **Q:** Why did Instagram disable Python's GC? **A:** Refcounts/GC written on read dirtied shared pages, breaking copy-on-write across forked workers; disabling GC recovered ~10% memory.
- **Q:** Why is LLM inference memory-bound? **A:** The KV cache grows per-token per-request and dominates GPU HBM; you run out of memory before compute. PagedAttention pages it like RAM.
- **Q:** Unbounded agent history — what breaks? **A:** Three things at once: a RAM leak (OOM), context-window overflow, and quadratic token cost. Fix: bound/summarize/offload.

## Primary Sources (verify against these)

- **Virtual memory / paging:** `mmap(2)`, `proc(5)` (`/proc/[pid]/status`, `smaps`), `Documentation/vm/` in the Linux tree; Ulrich Drepper, "What Every Programmer Should Know About Memory" (2007); Gorman, *Understanding the Linux Virtual Memory Manager* (free online).
- **Page faults / swap:** `Documentation/vm/` (page reclaim, swappiness); `vmstat(8)`; Intel SDM Vol. 3A (paging) for the MMU/TLB.
- **Stack/heap:** `getrlimit(2)` (`RLIMIT_STACK`/`RLIMIT_AS`), `brk(2)`, `mmap(2)`; glibc malloc internals.
- **OOM killer / cgroups:** `Documentation/admin-guide/cgroup-v2.rst` (memory controller); `Documentation/mm/` (OOM); kernel `mm/oom_kill.c`; `proc(5)` (`oom_score_adj`); Kubernetes docs "Assign Memory Resources to Containers and Pods" & "Node-pressure Eviction."
- **Python memory:** CPython Developer's Guide "Garbage collector design"; `Objects/obmalloc.c` (pymalloc), `Modules/gcmodule.c`; the `gc`, `sys`, `tracemalloc`, `resource` module docs. (Free-threaded/PyPy differ — Day 20.)
- **Instagram GC/CoW:** Instagram Engineering, "Dismissing Python Garbage Collection at Instagram" and "Copy-on-write friendly Python garbage collection" (2017) — verify current.
- **KV cache / PagedAttention:** Kwon et al., "Efficient Memory Management for Large Language Model Serving with PagedAttention," SOSP 2023; vLLM docs & source. (KV-cache internals: Day 63.)
- **Context window / tokens / cost:** current provider docs for token counting and pricing (consult the `claude-api` skill for Claude; `tiktoken` for OpenAI; `count_tokens` for Gemini) — they drift; verify (Day 23).

## Layered explanations

- **10-second:** The OS lies to every program that it owns all of RAM (virtual memory), handing out real memory 4 KB at a time only when touched; use more than exists and the OOM killer terminates you with no warning — and an AI agent that never trims its conversation is that exact "out of memory" bug, plus a runaway bill.
- **1-minute:** Virtual memory gives each process a private address space, translated to physical RAM per-access via page tables and the TLB, delivered lazily on *page faults* (touch a page → the OS maps it). Allocating is a cheap promise; using is what costs physical RAM. The stack (small, self-freeing) and heap (large, GC/manual — where leaks live) are the two regions. In Python everything is a heap object (~28-byte ints), reclaimed immediately by *reference counting* with a *cyclic GC* backstop for reference cycles; a Python "leak" is a *lingering reference*, hunted with `tracemalloc`. Exceed the machine's RAM and the *OOM killer* sends an uncatchable SIGKILL (in a container, exit 137 "OOMKilled") — which is why you size limits at p99, not p50. All of this reappears in agents: the context window is a RAM-like token budget, the KV cache makes inference memory-bound (and is literally managed with OS paging in vLLM's PagedAttention), and an agent that never bounds its history leaks RAM, overflows the context, and runs up cost — one bug, three failures, ending in an OOM kill that takes every user session with it.
- **5-minute:** *(the note's Parts 1–3 in order: virtual memory as the illusion of private, infinite RAM (§1.1) → pages, page tables, and the TLB that translate addresses (§1.2) → page faults and swap that fill the illusion lazily (§1.3) → stack vs heap and who frees each (§1.4) → the OOM killer that enforces the physical budget with an uncatchable SIGKILL (§1.5) → Python's heap: PyObjects, refcounting, cyclic GC, pymalloc, and why RSS won't return (§1.6) → budgeting memory for a deployment (§1.7) and hunting leaks with tracemalloc (§1.8) → Instagram's GC/CoW and the Kubernetes OOMKill loop (§1.9); then the agent: the context window as a token budget (§2.1), the KV cache making inference memory-bound (§2.2), the unbounded-history leak that fails three ways (§2.3), budgeting a GPU inference server (§2.4), and PagedAttention applying OS paging to the KV cache (§2.5); then the bridge: agent and backend share one cgroup limit (§3.1), and an unbounded agent OOM-kills the whole worker and every session on it (§3.2).)*
- **Expert:** Memory management is the OS maintaining, per process, a private virtual address space mapped to physical frames through a multi-level page table cached in the TLB, with physical backing supplied lazily via demand paging and reclaimed under pressure via LRU eviction to swap — trading a page-fault tax (minor ~µs, major ~ms, the latter catastrophic in aggregate as thrashing) for isolation, relocation, and overcommit. The physical budget is hard and enforced non-negotiably: overcommit is settled by the OOM killer's uncatchable SIGKILL, scoped per-cgroup for containers (exit 137), which makes capacity a p99-distribution problem, not a mean. Managed runtimes add a second allocator layer: CPython reclaims via eager reference counting (with a generational cyclic collector for the cycles refcounting provably cannot break) over a pooled small-object allocator (pymalloc) whose arena retention decouples in-process free from RSS return, and whose refcount-in-header mutation-on-read defeats fork/CoW sharing (Instagram). An LLM agent is this same system with two additions: a token-denominated context window that is a re-billed-per-turn working set, and a per-request KV cache that makes inference memory-bound on GPU HBM and is best managed by transplanting virtual-memory paging itself (PagedAttention) onto the accelerator — so the discipline is invariant across tiers: bound the working set, size for the tail, budget concurrency × footprint, and treat every accumulating structure as a leak-in-waiting, because the enforcement mechanism at every tier is the same merciless SIGKILL.

---

*End of Day 4. Next: Day 5 — The OS IV: files, I/O, and the descriptors that run the world (file descriptors, "everything is a file," buffered vs unbuffered I/O, fsync and durability, blocking vs non-blocking, and epoll — the engine inside asyncio, Nginx, and Redis).*
