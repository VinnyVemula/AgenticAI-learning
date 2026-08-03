# Day 8 — Consolidation: The Machine Model

> **Framing.** Days 1–7 built seven separate views of one machine: the CPU/memory/latency pyramid, the kernel and syscalls, threads and the scheduler, virtual memory and Python's allocator, file descriptors and I/O, CPython's internals, and the toolchain that assembles it all. Today they become **one model**. This note does not re-teach those days — it welds them together, gives you the apparatus to prove you own the model (teach-backs, recall bank, one synthesis trace), and works two synthesis designs in depth: a **code-execution sandbox for an AI agent** and a **single-machine job runner**.
>
> **The ONE idea that unites the backend and agentic layers:** *a running program does nothing but cross boundaries, and every boundary crossing is priced.* A register-to-register add crosses nothing (~0.3 ns). A DRAM load crosses the cache boundary (~80 ns). A syscall crosses into the kernel (~hundreds of ns). A disk flush crosses into hardware (~100 µs–10 ms). A network call crosses machines (~0.5 ms–150 ms). **A model call crosses into a datacenter full of GPUs and comes back a token at a time (~10 ms *per token*).** Performance, correctness, and safety are all the same question asked at different boundaries: *what crosses, how often, and who is allowed to cross?* Part 1 is the machine's boundaries. Part 2 is the agent's boundaries. Part 3 is what happens when the agent's boundary lands on top of the machine's.

---

## How to use this note (the consolidation protocol)

This is a *working* day, not a reading day. The order matters:

1. **Teach back first, read second.** Go to the wrap-up's **7 × 60-second teach-back cards**, cover the model answers, and speak each one aloud with a timer. Do this *before* reading Parts 1–3. Whatever you fumble marks the section you actually need.
2. **Write the synthesis trace from memory.** One page: "the life of `python app.py`". Then read §1.3 and diff. The diff *is* the study plan.
3. **Re-answer the recall bank** (wrap-up) cold. Answers are inline; do not peek until you have committed to an answer out loud or on paper.
4. **Redo the weakest build.** §1.7 (job runner) and §3.1 (sandbox) are the two builds designed to force every day back into play; if you don't know which build is weakest, do the sandbox — it touches Days 2, 3, 4, and 5 simultaneously.
5. **Run the runnable examples on the machine in front of you.** Every number in this note is an order of magnitude with a "verify locally" flag, not a promise. The point of measuring is that *your* numbers become the ones you remember.

> **Platform honesty, stated once so it isn't repeated.** Days 2–5 are POSIX-shaped: `fork`, `exec`, `rlimit`, `epoll`, and namespaces are Linux concepts. On Windows the *concepts* survive but the *interfaces* differ (`CreateProcess` instead of fork/exec, Job Objects instead of rlimits, IOCP instead of epoll). Every example below is marked **[portable]**, **[Linux/WSL only]**, or **[Windows note]**. If you are on Windows, install WSL2 and run the Linux-only ones there — the alternative is learning a mental model you can't verify.

---

# PART 1 — BACKEND

## 1.1 The seven days as one machine

**Depth: [CORE]** — this section is the spine of the whole note; every black box named here is opened either below or on the day it came from.

### Intuition

Seven days felt like seven subjects because each had its own vocabulary. They are actually **six layers of one stack plus the tooling that builds it**, and each layer exists to sell you the same lie: *"you have the machine to yourself."* The CPU pretends memory is fast (caches). The kernel pretends you own the CPU (scheduler) and all of memory (virtual memory) and that devices are files (file descriptors). CPython pretends objects live forever and that arithmetic is free (bytecode + refcounts). The toolchain pretends your dependency set is the only one on disk (venvs).

Consolidation means knowing **which lie you are standing on** when something goes wrong.

### The map

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ D7  TOOLCHAIN         uv / venv / pyproject / wheels / ruff                   │
│     "which interpreter + which site-packages"      → resolves sys.path        │
├──────────────────────────────────────────────────────────────────────────────┤
│ D6  LANGUAGE RUNTIME  CPython: objects, refcount+GC, bytecode, eval loop, GIL │
│     "your code is data being interpreted"          → makes syscalls for you   │
├──────────────────────────────────────────────────────────────────────────────┤
│ D5  I/O + FDs         fd table, read/write, buffering, fsync, select/epoll    │
│     "everything is a file"                         → the only way out         │
├──────────────────────────────────────────────────────────────────────────────┤
│ D4  MEMORY            virtual addrs, page tables, TLB, page faults, mmap,     │
│                       pymalloc arenas/pools/blocks → "you own all of RAM"     │
├──────────────────────────────────────────────────────────────────────────────┤
│ D3  EXECUTION         threads, run queue, preemption, context switch, races,  │
│                       locks, memory visibility     → "you own a CPU"          │
├──────────────────────────────────────────────────────────────────────────────┤
│ D2  KERNEL            processes, fork/exec/wait, syscall trap, signals,       │
│                       privilege rings              → the one trusted layer    │
├──────────────────────────────────────────────────────────────────────────────┤
│ D1  HARDWARE          cores, registers, L1/L2/L3, DRAM, NVMe, NIC, latency    │
│                       pyramid                      → the prices are set here  │
└──────────────────────────────────────────────────────────────────────────────┘
        Every arrow ↑ is an abstraction. Every arrow ↓ is a cost.
```

### Analogy

**The stack is a chain of translators at a border crossing.** Your Python source speaks Python; the interpreter translates to bytecode; the eval loop translates bytecode into C function calls; the C library translates those into syscall numbers in registers; the kernel translates syscalls into device commands; the device translates commands into electrons. Each translator adds latency and each one can *refuse* (a permission error, an OOM, a closed fd).

**Where the analogy breaks:** real translators preserve meaning, but these layers deliberately *change* it. Virtual memory hands you an address that does not exist in DRAM; the fd `4` is not a file, it's an index into a per-process table; `write()` returning success does not mean bytes reached the disk. The layers are not faithful translators — they are **caching liars**, and the lies are the performance.

### The five invariants (the compressed form of Days 1–7)

Learn these five sentences and you can regenerate most of the week.

| # | Invariant | One-line mechanism | Day |
|---|---|---|---|
| **I1** | **The kernel is a library you call by trapping, not by jumping.** | `syscall` instruction switches privilege ring + stack; args in registers; the kernel validates *everything* because your pointers are untrusted. | D2 |
| **I2** | **Every address your code sees is a lie the MMU maintains.** | virtual page → page table walk → physical frame, cached in the TLB; a miss is a page fault, and a *major* fault is a disk read wearing a memory access's clothing. | D4 |
| **I3** | **Every external thing is a small integer.** | fd = index into the process's fd table → kernel `struct file` → inode/socket/pipe. Files, sockets, pipes, epoll instances, even processes (pidfd) are fds. | D5 |
| **I4** | **Concurrency is scheduling; parallelism is cores; the GIL decides which one you got.** | The scheduler multiplexes runnable threads onto cores; CPython's GIL serializes *bytecode execution* but is released around blocking I/O and in C extensions that opt out. | D3 + D6 |
| **I5** | **Cost is boundary crossings and cache misses, not instructions.** | 10⁹ arithmetic ops/s vs ~10⁵–10⁶ syscalls/s vs ~10³ fsyncs/s vs ~10 model tokens/s. Optimization = *cross less*, or *cross in batches*. | D1 (+ everything) |

**Worked example that uses all five at once.** A Flask handler reads a 4 KB JSON body, parses it, appends a line to a log, and returns:

```
I1  recv()  → trap into kernel, copy 4 KB from socket buffer into your page
I2  the destination page is fresh from pymalloc's arena → first touch = minor page fault (~1–2 µs)
I3  the socket is fd 7; the log is fd 9; both are ints in this process's fd table
I4  the GIL is held during json.loads (pure bytecode), released during recv() and write()
I5  the handler's ~50 µs of CPU is dwarfed by 1 fsync (~200 µs–5 ms) if you added one per request
```

That last line is the whole week in one sentence: **the code you wrote is almost never the cost.**

### Under the hood: the one stack diagram to memorize

```
  your_function()                       ← D6 CPython frame, on the C stack + heap
    └─ CALL bytecode → ceval loop       ← D6 specializing adaptive interpreter (PEP 659, 3.11+)
        └─ C function (e.g. os.write)   ← D6 CPython object layer → libc
            └─ libc write() wrapper     ← D2 puts syscall nr in rax, args in rdi/rsi/rdx
                └─ SYSCALL instruction  ← D2 ring 3 → ring 0, kernel stack swap
                    └─ vfs_write()      ← D5 fd table lookup → struct file → f_op->write
                        └─ page cache   ← D4 dirty pages; writeback later, or now on fsync
                            └─ block layer → NVMe queue → device  ← D1 the real price
```

**Where I deliberately stop:** below the NVMe submission queue (flash translation layers, wear levelling, DRAM refresh, cache coherence protocol details such as MESI transitions). Treat the device as a black box with a *latency distribution*, not a mechanism — until you are writing a database storage engine, in which case Day 1's numbers plus the device's own datasheet are where you resume.

---

## 1.2 The unified cost model

**Depth: [CORE]** — this is the single most reusable artifact of Phase 0, and it is the thing you will still be using on Day 90.

### Intuition

Day 1 gave you a latency pyramid for *memory*. Days 2–5 added tiers that don't fit on that pyramid: syscalls, page faults, context switches, flushes. Consolidated, they form **one ladder spanning 8 orders of magnitude** — and every engineering decision you make is "which rung does this operation land on, and how many times do I climb it?"

### Analogy

**The ladder is a price list for leaving your desk.** Thinking at your desk is free (registers/L1). Reaching for a book on the shelf costs a second (L2/L3). Walking to the filing cabinet costs a minute (DRAM). Asking the front desk to fetch something costs a coffee break (syscall). Driving to the archive costs an afternoon (disk). Mailing another city costs days (network). Commissioning an expert report costs a week (an LLM call).

**Where the analogy breaks:** desk work and errands are *serial* for a human; a CPU overlaps them aggressively (out-of-order execution, prefetch, multiple outstanding misses, async I/O). A single ~80 ns DRAM access does not cost 80 ns of *throughput* if 8 of them are in flight. The ladder gives you **latency**, and latency only becomes cost when you can't overlap it — which is exactly why blocking calls (Day 5) and lock contention (Day 3) hurt so much: they destroy overlap.

### The ladder (order of magnitude; verify locally with §1.2's runnable)

| Rung | Operation | Typical latency | Ops/sec budget | Day |
|---|---|---|---|---|
| 0 | register / L1 hit | ~0.3–1 ns | 10⁹ | D1 |
| 1 | L2 hit | ~3–5 ns | 2×10⁸ | D1 |
| 2 | L3 hit | ~15–40 ns | 5×10⁷ | D1 |
| 3 | DRAM access (L3 miss) | ~80–100 ns | 10⁷ | D1 |
| 4 | one CPython bytecode op | ~10–30 ns | 3×10⁷ | D6 |
| 5 | dict lookup / attribute access | ~30–60 ns | 2×10⁷ | D6 |
| 6 | **minor page fault** (first touch) | ~1–3 µs | 3×10⁵ | D4 |
| 7 | **trivial syscall** (`write` to /dev/null) | ~0.2–1 µs (higher with KPTI/Spectre mitigations) | 10⁶ | D2 |
| 8 | **thread context switch** | ~1–5 µs direct, +10–50 µs of cache/TLB damage | 2×10⁵ | D3 |
| 9 | GIL handoff (`sys.setswitchinterval` default **5 ms**) | 5 ms ceiling on hand-off latency | — | D3+D6 |
| 10 | NVMe random read (**major page fault**) | ~20–100 µs | 10⁴–10⁵ | D1+D4 |
| 11 | `fsync()` on NVMe / SSD | ~100 µs–5 ms | 10²–10³ | D5 |
| 12 | same-datacenter RTT | ~0.2–0.5 ms | — | D1 |
| 13 | HDD seek | ~5–10 ms | ~100 | D1 |
| 14 | cross-continent RTT | ~50–150 ms | — | D1 |
| 15 | **one LLM output token** | ~10–50 ms | 20–100 tok/s | Part 2 |

Read the two ends: **rung 0 to rung 15 is roughly 10⁸.** One output token costs what ~30 million L1 hits cost. That single fact is why Part 2's engineering (caching, batching, concurrency limits) looks nothing like Part 1's.

### Runnable example — measure four rungs of the ladder on your own machine

```python
# cost_ladder.py — measure userspace vs syscall vs page-fault vs fsync costs.
# pip install: nothing (stdlib only). Python 3.11+.
# [portable] — fsync section works everywhere; page-fault numbers are most
# meaningful on Linux/WSL where you can cross-check with /usr/bin/time -v.
import os, sys, time, tempfile, platform

N = 200_000

def bench(label, fn, n=N):
    fn()                                  # warm up: pay one-time faults/imports first
    t0 = time.perf_counter_ns()
    for _ in range(n):
        fn()
    t1 = time.perf_counter_ns()
    print(f"{label:<38} {(t1 - t0) / n:9.1f} ns/op")

# Rung 4/5: pure userspace work — no boundary crossed.
d = {"k": 1}
bench("dict lookup (userspace)", lambda: d["k"])

# Rung 7: a real syscall on every call. os.write is a thin wrapper; the cost
# you see is dominated by the ring transition, not by moving 1 byte.
devnull = os.open(os.devnull, os.O_WRONLY)
buf = b"x"
bench("os.write() to /dev/null (syscall)", lambda: os.write(devnull, buf))

# Contrast: buffered writes batch many logical writes into few syscalls.
f = open(os.devnull, "wb", buffering=8192)
bench("buffered f.write() (amortized)", lambda: f.write(buf))

# Rung 6: first touch of fresh pages. Allocating 64 MiB and touching each 4 KiB
# page once forces minor faults; dividing by page count gives a per-fault cost.
PAGE, MB = 4096, 64
def touch_fresh_pages():
    ba = bytearray(MB * 1024 * 1024)      # lazily backed by the OS
    for off in range(0, len(ba), PAGE):
        ba[off] = 1                        # first write to each page → minor fault
t0 = time.perf_counter_ns(); touch_fresh_pages(); t1 = time.perf_counter_ns()
pages = MB * 1024 * 1024 // PAGE
print(f"{'first-touch per 4KiB page (fault+zero)':<38} {(t1 - t0) / pages:9.1f} ns/op")

# Rung 11: durability. Each fsync waits for the device to acknowledge.
with tempfile.TemporaryDirectory() as tmp:
    path = os.path.join(tmp, "d.log")
    fd = os.open(path, os.O_WRONLY | os.O_CREAT, 0o600)
    n = 200
    t0 = time.perf_counter_ns()
    for _ in range(n):
        os.write(fd, b"record\n")
        os.fsync(fd)                       # the expensive part
    t1 = time.perf_counter_ns()
    os.close(fd)
    print(f"{'write+fsync (durable)':<38} {(t1 - t0) / n / 1000:9.1f} us/op")

print(f"\npython {sys.version.split()[0]} on {platform.platform()}")
```

Invocation and a real shape of output (numbers are from a mid-range laptop under WSL2 — **yours will differ; that is the point**):

```console
$ python cost_ladder.py
dict lookup (userspace)                       31.4 ns/op
os.write() to /dev/null (syscall)            612.0 ns/op
buffered f.write() (amortized)                69.8 ns/op
first-touch per 4KiB page (fault+zero)      1180.0 ns/op
write+fsync (durable)                        903.2 us/op

python 3.12.3 on Linux-5.15.167.4-microsoft-standard-WSL2-x86_64-with-glibc2.35
```

**Why this works, line by line.**

- `bench()` calls `fn()` once before timing: that pays the one-time costs (import, first page touches, branch predictor warm-up) so the loop measures steady state. Skipping this is the single most common microbenchmark bug.
- `perf_counter_ns()` is the monotonic high-resolution clock; `time.time()` is wall-clock and can jump (NTP), which would silently corrupt results.
- The `dict lookup` line is your **rung 4/5 baseline** — the cost of Python doing nothing interesting. Every other number should be read as a multiple of it.
- `os.write(devnull, b"x")` crosses the ring boundary every iteration: the ~20× jump over the dict lookup *is invariant I1 made visible*. Note it writes to `/dev/null`, so no device is involved — this is pure syscall overhead.
- The buffered `f.write()` line is the counter-move: Python's `BufferedWriter` accumulates in a userspace buffer and issues one syscall per 8 KiB. Same logical operation, ~10× cheaper, because you crossed the boundary 1/8192 as often. **This is why Day 5's buffering matters and why "just add a log line per request" is not free.**
- `touch_fresh_pages` allocates 64 MiB but only writes one byte per 4 KiB page. The allocation itself is cheap (the kernel hands out *virtual* address space); the cost lands on **first touch**, when the fault handler finds no physical frame, allocates and zeroes one, and updates the page table. That per-page number is invariant I2 with a price tag, and it's why RSS grows lazily while VSZ jumps immediately.
- The `fsync` loop is the durability boundary. Notice it's reported in **microseconds**, not nanoseconds: ~1000× a syscall and ~30,000× a dict lookup. Any design that fsyncs per item has capped itself at rung 11 — which is exactly the constraint that shapes the job runner in §1.7.

**Honesty caveat.** `fsync()` returning success is a *weaker* promise than most people assume: with write-back caching in the drive, and historically with Linux's error handling (see the fsyncgate case study in §1.9), a successful `fsync` has not always meant "durable". Treat this measurement as "the cost of asking", not "proof of durability".

---

## 1.3 The synthesis build — the life of `python app.py`

**Depth: [CORE]** — this is the day's centrepiece. Write your version from memory *first*, then read.

### Intuition

You have typed `python app.py` thousands of times. If the machine model is real, you should be able to narrate, without notes, every boundary crossed between pressing Enter and your first line of code running — and say which day each crossing came from. If there is a gap in the narration, that gap is a gap in the model, not a detail.

### Analogy

**It is a relay race where each runner hands off a baton and then dies or waits.** The shell runs, clones itself, the clone throws away its own body and becomes `python`, the dynamic loader wires up its limbs, the interpreter builds a world, and only then does your code get the baton.

**Where the analogy breaks:** the handoffs are not symmetric. `fork` produces a *copy* that shares physical pages copy-on-write (so nothing is really duplicated until written); `exec` keeps the *same* process — same pid, same fds, same parent — and replaces only the address space. A relay runner who keeps their number and their kit but swaps their entire body is a strange runner. Hold onto the strangeness: it is exactly why fd inheritance (§3.1) is a security problem and why `fork()` in a threaded program is dangerous.

### The trace (14 stages)

```
① SHELL PARSE                                                    [D7 · D2]
   bash splits the line, resolves `python` by walking $PATH (a series of
   stat/execve attempts). In a venv, $PATH puts .venv/bin first — this is
   the *entire* mechanism by which "which python" is answered.

② fork()                                                          [D2 · D4]
   The shell clones itself: new pid, copied fd table, page tables marked
   copy-on-write. No memory is actually copied yet (I2).

③ execve("/path/.venv/bin/python", ["python","app.py"], envp)      [D2 · D4]
   Same process, brand-new address space. The kernel tears down the old
   mappings, reads the ELF header, maps segments (.text read-only+shared,
   .data/.bss private), sets up the stack with argv/envp/auxv, and jumps
   to the ELF entry point. FDs 0/1/2 survive — that is why your prints
   land in the same terminal (I3).

④ DYNAMIC LINKING                                                 [D4 · D5]
   ld.so mmaps libpython3.x.so, libc, libssl…, resolves symbols (lazily
   via the PLT unless BIND_NOW). Each mmap is address space, not RAM;
   pages arrive on demand as major/minor faults.

⑤ INTERPRETER INIT (Py_InitializeFromConfig)                       [D6]
   Builds the runtime: allocators (pymalloc arenas), the small-int and
   interned-string caches, type objects, the builtins module, the GIL
   (created here, held by the main thread), the import machinery.

⑥ PATH CONFIGURATION                                              [D7]
   Locates sys.prefix by finding `pyvenv.cfg` next to the executable →
   sets sys.path (stdlib zip/dir, then site-packages). `site.py` runs and
   processes .pth files. This is 100% of what "activating a venv" means.

⑦ IMPORTS                                                         [D5 · D6 · D7]
   For each import: sys.modules check → finders walk sys.path → many
   stat() syscalls → read the .pyc from __pycache__ if its header matches
   the .py's mtime+size, else compile and write one → unmarshal code
   objects → execute the module body. Every module body is *code that
   runs at import time*.

⑧ COMPILE app.py                                                  [D6]
   tokenize → PEG parse → AST → symbol table → bytecode; wrapped in a code
   object (co_code, co_consts, co_names, co_varnames). __main__ has no
   .pyc cache — the top-level script is recompiled every run.

⑨ EXECUTE MODULE BODY                                             [D6 · D4]
   A frame is pushed; the eval loop dispatches bytecode. Every literal,
   class, and def becomes a heap object with a refcount. Function defs
   create function objects; they do not run.

⑩ YOUR WORK BEGINS: e.g. socket(), bind(), listen()               [D5 · D2]
   Each returns the lowest free fd. bind() may fail EADDRINUSE — a kernel
   state check, not a Python one.

⑪ THE EVENT LOOP / WORKER MODEL                                   [D3 · D5]
   Either: threads blocking in read() (scheduler parks them; GIL released),
   or: one thread in epoll_wait() multiplexing thousands of fds (I3+I4),
   or: processes, each with its own GIL and address space.

⑫ STEADY STATE                                                    [D1–D5]
   Per request: epoll returns → read() → parse (GIL held) → maybe disk or
   DB (fd, blocking or async) → write() → back to epoll. Page cache absorbs
   file reads; pymalloc absorbs object churn; RSS plateaus.

⑬ SIGNAL / SHUTDOWN                                               [D2 · D3]
   Ctrl-C → SIGINT to the foreground process group → CPython sets a flag
   and raises KeyboardInterrupt in the main thread at the next bytecode
   boundary (this is why a C-level blocking call can delay it).

⑭ EXIT                                                            [D2 · D5]
   atexit handlers, module teardown, buffered files flushed (unflushed
   buffers are LOST if you _exit or crash), fds closed by the kernel,
   exit status reaped by the shell's wait().
```

### Runnable example — make the trace observable from inside the process

```python
# life_of_python.py — print the boundaries your own process is standing on.
# pip install: nothing (stdlib only). Python 3.11+.
# [portable] — every section degrades gracefully off-Linux and says so.
import os, sys, time, platform, resource if os.name == "posix" else None  # noqa
```

That import line is deliberately illegal Python — a reminder that `resource` is POSIX-only. Here is the version that actually runs:

```python
# life_of_python.py — print the boundaries your own process is standing on.
# pip install: nothing (stdlib only). Python 3.11+.
# [portable] — POSIX-only sections are guarded and announce themselves.
import os, sys, time, platform, sysconfig

print("=" * 70)
print("① ② ③  IDENTITY — one process, survived fork+exec")
print(f"  pid={os.getpid()}  ppid={os.getppid()}  argv={sys.argv}")
print(f"  executable={sys.executable}")
print(f"  interpreter start→now: {time.time() - psutil_free_start:.3f}s"
      if False else f"  clock now: {time.strftime('%H:%M:%S')}")

print("\n④ ⑥  ADDRESS SPACE + PATH CONFIG (venv resolution)")
print(f"  sys.prefix       = {sys.prefix}")
print(f"  sys.base_prefix  = {sys.base_prefix}")
print(f"  in a venv?       = {sys.prefix != sys.base_prefix}")
print(f"  site-packages    = {sysconfig.get_paths()['purelib']}")
print(f"  sys.path[0]      = {sys.path[0]!r}   # '' or script dir")

print("\n⑦  IMPORTS ALREADY DONE BEFORE YOUR FIRST LINE")
print(f"  len(sys.modules) = {len(sys.modules)}")
print(f"  sample           = {sorted(sys.modules)[:8]}")

print("\n⑧ ⑨  YOUR CODE IS DATA (bytecode)")
def add(a, b):
    return a + b
import dis
dis.dis(add)
print(f"  co_consts={add.__code__.co_consts} co_varnames={add.__code__.co_varnames}")

print("\n⑩ I3  FILE DESCRIPTORS — the small integers")
if hasattr(os, "listdir") and os.path.isdir("/proc/self/fd"):     # Linux/WSL
    for name in sorted(os.listdir("/proc/self/fd"), key=lambda s: int(s)):
        try:
            target = os.readlink(f"/proc/self/fd/{name}")
        except OSError:
            target = "<gone>"
        print(f"  fd {name:>3} -> {target}")
else:
    print("  /proc not available (not Linux). stdin/out/err are still fds 0,1,2:")
    for fd, label in ((0, "stdin"), (1, "stdout"), (2, "stderr")):
        print(f"  fd {fd} = {label}  isatty={os.isatty(fd)}")

print("\nI2 I4  MEMORY + THREADS")
import threading
print(f"  threads alive       = {threading.active_count()} "
      f"({[t.name for t in threading.enumerate()]})")
print(f"  GIL switch interval = {sys.getswitchinterval()} s")
if os.name == "posix":
    import resource
    ru = resource.getrusage(resource.RUSAGE_SELF)
    print(f"  max RSS             = {ru.ru_maxrss / 1024:.1f} MiB")
    print(f"  minor page faults   = {ru.ru_minflt}")
    print(f"  major page faults   = {ru.ru_majflt}   # >0 means real disk reads")
    print(f"  voluntary ctx sw    = {ru.ru_nvcsw}    # blocked on I/O")
    print(f"  involuntary ctx sw  = {ru.ru_nivcsw}   # preempted by scheduler")
else:
    print("  resource module is POSIX-only; on Windows use "
          "psutil.Process().memory_info() / Performance Monitor.")

print("\n⑭  EXIT — buffered data is flushed by the interpreter, not the kernel")
print(f"  platform: {platform.platform()}")
```

Run it, and then verify the *outside* view of the same trace with these commands:

```console
$ python life_of_python.py            # the inside view (above)

# ⑦ Which imports cost what? (cumulative µs per module, deepest first)
$ python -X importtime life_of_python.py 2>&1 | tail -12
import time: self [us] | cumulative | imported package
import time:      1234 |       1234 |   sysconfig
...

# ⑤⑥⑦ How much of startup is *not* your code?
$ python -c "pass"        ;# baseline interpreter start
$ python -S -c "pass"     ;# skip site.py → no site-packages, no .pth files
# time both; the delta is stage ⑥.

# ①–⑭ [Linux/WSL only] Every syscall, counted. This is the ground truth.
$ strace -f -c -o /tmp/trace.txt python life_of_python.py
$ head -15 /tmp/trace.txt
# Expect execve=1, mmap/mprotect in the dozens (④), openat+newfstatat in the
# hundreds (⑦ walking sys.path), write for your prints (⑩), and exit_group=1.
# [Windows note] the equivalent tool is Sysinternals Process Monitor; filter
# on the python.exe PID and watch CreateFile/QueryDirectory instead of openat.
```

**Why this works, line by line.**

- `sys.prefix != sys.base_prefix` is *the* venv test. A venv is not a sandbox and not a container: it is a directory with a `pyvenv.cfg` that redirects `sys.prefix`, which redirects `sys.path`, which redirects imports. That's Day 7 in one boolean.
- `len(sys.modules)` before you import anything shows that stage ⑦ already ran ~30–70 imports on your behalf. Every one of those was dozens of `stat()` syscalls. This is why interpreter startup is milliseconds, not microseconds, and why CLI tools obsess over lazy imports.
- `dis.dis(add)` prints the actual bytecode. Seeing `LOAD_FAST a / LOAD_FAST b / BINARY_OP + / RETURN_VALUE` is the moment Day 6 stops being abstract: your function is a `bytes` object plus tuples of constants and names.
- The `/proc/self/fd` walk is invariant I3 made literal — symlinks from small integers to terminals, files, and sockets. On Windows the concept holds (HANDLEs), but the numbering and the `/proc` view do not; the code says so rather than pretending.
- `getrusage` is the cheapest window into Day 3 and Day 4 you will ever get: `ru_minflt` vs `ru_majflt` distinguishes "I touched new memory" from "I waited on the disk", and `ru_nvcsw` vs `ru_nivcsw` distinguishes "I blocked" from "I was preempted". Those four counters are the first thing to look at when a service is mysteriously slow.
- `strace -c` is the authoritative external check. If your written-from-memory trace claimed something the syscall counts contradict, believe the counts.

**Honesty caveat.** The exact syscall counts and even the *sequence* differ across Python versions, libc versions, and distributions (e.g. `openat` vs `open`, `newfstatat` vs `stat`, whether `.pyc` files already exist). Do not memorize counts. Memorize the *stages* and the *kinds* of syscalls each stage produces — that generalizes, the counts do not.

---

## 1.4 Runtime and toolchain, consolidated

**Depth: [WORKING]** — you must reason about these correctly and know their failure modes; you do not need to reimplement CPython or a package resolver.

### Intuition

Days 6 and 7 answer two questions the other days assume: *what is actually executing your source*, and *which files were on `sys.path` when it started*. Consolidated, they form the "identity" of a run: **interpreter + path config + compiled bytecode**. Almost every "works on my machine" bug is a mismatch in one of those three.

### The mental model, stated once

| Thing | What it really is | Failure mode it causes |
|---|---|---|
| Object | `PyObject` header (refcount + type ptr) + payload | refcount cycles → collected by the generational GC, not by refcounting |
| Refcount | `Py_INCREF/DECREF` on every reference change | why pure-Python object churn is expensive and why the GIL exists (unsynchronized refcounts would race — I4) |
| Bytecode | code object; cached as `.pyc` keyed on source mtime+size | stale `.pyc` in a copied tree; `__pycache__` written on first import |
| Eval loop | specializing adaptive interpreter (PEP 659, 3.11+) — quickens hot bytecodes into type-specialized forms | why microbenchmarks warm up; why a polymorphic hot loop is slower than a monomorphic one |
| GIL | one lock around bytecode execution; released around blocking I/O and by opt-in C extensions; hand-off bounded by `sys.setswitchinterval` (5 ms) | CPU-bound threads don't scale; free-threaded builds (PEP 703, 3.13 experimental → 3.14) change this — **verify against your interpreter, this is moving** |
| venv | directory + `pyvenv.cfg` that moves `sys.prefix` | "pip installed it but import fails" = wrong interpreter, not a broken package |
| wheel | zip with a compatibility-tagged filename; install = unpack | `sdist` fallback → surprise C compiler requirement |
| lockfile (uv/poetry) | pinned, hashed resolution of the dependency graph | unlocked deps → irreproducible builds across machines and time |

**Where the analogy for venvs usually breaks (worth stating explicitly):** people describe a venv as "an isolated environment", which imports a containment intuition it does not deserve. A venv isolates *`sys.path`* and nothing else — same kernel, same filesystem, same network, same uid, same rlimits. That distinction is exactly what §3.1 is about: if you need real isolation for untrusted code, a venv contributes zero.

### Runnable example — where does startup time actually go?

```python
# startup_cost.py — quantify stages ⑤⑥⑦ of the life-of-python trace.
# pip install: nothing (stdlib only). Python 3.11+. [portable]
import subprocess, sys, time, statistics

def timed(args, reps=7):
    best = []
    for _ in range(reps):
        t0 = time.perf_counter()
        subprocess.run(args, check=True,
                       stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
        best.append(time.perf_counter() - t0)
    return statistics.median(best) * 1000  # ms

py = sys.executable
scenarios = {
    "interpreter + site (baseline)": [py, "-c", "pass"],
    "interpreter, -S (no site.py)":  [py, "-S", "-c", "pass"],
    "import json":                   [py, "-c", "import json"],
    "import asyncio":                [py, "-c", "import asyncio"],
    "import unittest":               [py, "-c", "import unittest"],
}
base = None
for label, args in scenarios.items():
    ms = timed(args)
    if base is None:
        base = ms
    print(f"{label:<32} {ms:6.1f} ms   (+{ms - base:5.1f} ms over baseline)")
```

```console
$ python startup_cost.py
interpreter + site (baseline)       28.4 ms   (+  0.0 ms over baseline)
interpreter, -S (no site.py)       19.7 ms   (+ -8.7 ms over baseline)
import json                         31.9 ms   (+  3.5 ms over baseline)
import asyncio                      48.6 ms   (+ 20.2 ms over baseline)
import unittest                     44.1 ms   (+ 15.7 ms over baseline)

$ python -X importtime -c "import asyncio" 2>&1 | sort -k2 -t'|' -rn | head -5
# shows which submodules dominate the 20 ms
```

**Why this works.** `subprocess.run` measures a *whole process lifecycle* — fork/exec (stage ②③), dynamic linking (④), interpreter init (⑤), path config (⑥), imports (⑦), teardown (⑭). Taking the **median of 7** rather than the mean suppresses scheduler noise (Day 3: your benchmark competes for cores with everything else). The `-S` row isolates stage ⑥: the delta is the cost of `site.py` walking site-packages and processing `.pth` files. The import rows isolate stage ⑦ per package, and `-X importtime` decomposes that further. The practical payoff: a CLI that imports `asyncio` at module scope has bought itself ~20 ms of latency on *every invocation* before doing any work — which is why lazy imports are a real technique and not premature optimization.

---

## 1.5 Concurrency, consolidated: what the GIL actually costs

**Depth: [WORKING]** — you need to pick the right execution model and predict its failure, not rewrite the scheduler.

### Intuition

Day 3 gave you scheduling and races; Day 6 gave you the GIL. Consolidated, they yield **one decision procedure**: classify the work as CPU-bound or I/O-bound, then choose threads, processes, or async — and know exactly which lie you are relying on.

### Analogy

**The GIL is a single microphone in a room of speakers.** Everyone can be *in* the conversation (concurrency), but only the person holding the mic can *speak* (execute bytecode). If a speaker steps out to make a phone call (blocking I/O), they hand over the mic — so a room of phone-callers works beautifully.

**Where the analogy breaks:** the microphone is not handed over politely on request. Hand-off happens at bytecode boundaries and is bounded by the 5 ms switch interval, so a thread doing a long C-level operation that *doesn't* release the GIL can hold the mic through a scheduler quantum — and, worse, some C extensions (NumPy on large arrays, compression, hashing) *do* release it, meaning true parallelism sneaks in exactly where the analogy says it can't. "One mic" predicts the common case, not all cases.

### The decision table

| Work shape | Right tool | Why | Dominant failure mode |
|---|---|---|---|
| CPU-bound pure Python | `ProcessPoolExecutor` / `multiprocessing` | separate interpreters → separate GILs → real cores | memory ×N; pickling cost at the boundary; fork+threads hazards |
| CPU-bound in C (numpy/blas) | threads can work | those libraries release the GIL | oversubscription (N threads × BLAS threads) → thrash |
| Blocking I/O, few hundred conns | threads | GIL released in `read`/`write`; scheduler parks them | ~8 MiB stack VSZ each; context-switch churn at high counts |
| Blocking I/O, 10k+ conns | async (`epoll` under the hood) | one thread, one fd table, no per-conn stack | **any** blocking call stalls *all* connections |
| Mixed | async loop + process pool for the CPU part | keeps the loop free | forgetting `run_in_executor` → §3.2's coupled failure |

### Runnable example — prove the GIL with four measurements

```python
# gil_lab.py — threads vs processes for CPU-bound and I/O-bound work.
# pip install: nothing (stdlib only). Python 3.11+.
# [portable] — on Windows, processes use spawn (slower start); numbers shift,
# the *conclusion* does not.
import time, os
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

def cpu_task(n=6_000_000):
    """Pure-Python CPU burn: every iteration executes bytecode → needs the GIL."""
    x = 0
    for i in range(n):
        x += i * i
    return x

def io_task(seconds=0.25):
    """Stand-in for a blocking read(): time.sleep releases the GIL."""
    time.sleep(seconds)
    return seconds

def run(label, executor_cls, fn, n=4, **kw):
    t0 = time.perf_counter()
    with executor_cls(max_workers=n, **kw) as ex:
        list(ex.map(fn, [None] * n) if False else [f.result() for f in
             [ex.submit(fn) for _ in range(n)]])
    dt = time.perf_counter() - t0
    print(f"{label:<44} {dt:6.2f}s")
    return dt

if __name__ == "__main__":          # REQUIRED on Windows/macOS spawn start method
    print(f"cores={os.cpu_count()}  switchinterval={__import__('sys').getswitchinterval()}s\n")

    t0 = time.perf_counter(); [cpu_task() for _ in range(4)]
    serial_cpu = time.perf_counter() - t0
    print(f"{'CPU x4, serial':<44} {serial_cpu:6.2f}s")
    t_cpu_threads = run("CPU x4, 4 THREADS  (GIL-serialized)", ThreadPoolExecutor, cpu_task)
    t_cpu_procs   = run("CPU x4, 4 PROCESSES (real parallelism)", ProcessPoolExecutor, cpu_task)

    t0 = time.perf_counter(); [io_task() for _ in range(4)]
    serial_io = time.perf_counter() - t0
    print(f"\n{'IO  x4, serial':<44} {serial_io:6.2f}s")
    t_io_threads = run("IO  x4, 4 THREADS  (GIL released)", ThreadPoolExecutor, io_task)

    print(f"\nspeedups vs serial:")
    print(f"  CPU threads   {serial_cpu / t_cpu_threads:4.2f}x   <- ~1.0 is the GIL")
    print(f"  CPU processes {serial_cpu / t_cpu_procs:4.2f}x   <- scales with cores")
    print(f"  IO  threads   {serial_io / t_io_threads:4.2f}x   <- ~N, I/O needs no GIL")
```

```console
$ python gil_lab.py
cores=8  switchinterval=0.005s

CPU x4, serial                                 2.51s
CPU x4, 4 THREADS  (GIL-serialized)            2.62s
CPU x4, 4 PROCESSES (real parallelism)         0.79s

IO  x4, serial                                 1.00s
IO  x4, 4 THREADS  (GIL released)              0.25s

speedups vs serial:
  CPU threads   0.96x   <- ~1.0 is the GIL
  CPU processes 3.18x   <- scales with cores
  IO  threads   4.00x   <- ~N, I/O needs no GIL
```

**Why this works, line by line.**

- `cpu_task` is deliberately *pure Python* (`x += i * i` in a loop). Every iteration is bytecode, so the GIL must be held; four threads take turns and you measure ~1× — often slightly *worse* than serial, because you also paid for 5 ms-bounded hand-offs and cache disruption.
- `ProcessPoolExecutor` gives each worker its own interpreter, own GIL, own address space. Speedup approaches core count and stops there (Day 1: you cannot exceed the hardware). The gap between 4.0× and the observed ~3.2× is process start-up, pickling of the return value, and the fact that your cores are also running everything else on the machine.
- `io_task` uses `time.sleep`, which releases the GIL — the same thing a real `read()` does. Four sleepers overlap almost perfectly, so wall time collapses to one sleep. That's invariant I4: *blocking is where Python threads shine, because blocking is when they let go of the mic.*
- `if __name__ == "__main__":` is not decoration. On Windows and modern macOS the default start method is **spawn**, which re-imports your module in the child; without the guard, the child re-runs the benchmark and you get infinite process recursion. This is the single most common cross-platform multiprocessing bug.

**Honesty caveat.** On a free-threaded build (PEP 703; `python3.13t`/3.14+ with the GIL disabled) the "CPU threads" row can show real speedup. If your interpreter is free-threaded, `sys._is_gil_enabled()` reports it. This area is actively changing — **verify against your interpreter's docs rather than trusting any number here.**

---

## 1.6 The cross-day failure catalogue

**Depth: [WORKING]** — the point of a machine model is diagnosis speed. This table is the payoff: symptom → mechanism → which day → first move.

| Symptom you actually see | Mechanism | Day | First move |
|---|---|---|---|
| Process vanishes, exit code 137, no traceback | OOM-killer / cgroup memory limit sent SIGKILL — uncatchable | D2, D4 | `dmesg`/`kubectl describe`; cap memory *inside* the process too (§3.1) |
| Latency fine at p50, terrible at p99 | GC pauses, GIL hand-off (5 ms), page-cache misses, or a lock convoy | D3, D4, D6 | measure, don't guess: `ru_majflt`, run-queue length, GC stats |
| RSS grows forever, no leak in your code | fragmentation of pymalloc arenas, or caches you never bound | D4, D6 | bound the cache; check `tracemalloc`; consider per-request processes |
| "Too many open files" at ~1024 or ~65k | fd leak — I3: fds are a finite per-process table | D5 | find the un-closed fd (`/proc/PID/fd`), use context managers |
| Data present in logs, gone after power loss | write ≠ durable; no `fsync`, or fsync of file but not directory | D5 | fsync the file *and* the parent dir after rename |
| Two workers processed the same job | no atomic claim; a rename/lock race (TOCTOU) | D3, D5 | make the claim a single atomic `rename`/`link` (§1.7) |
| Counter is short by a few thousand | non-atomic read-modify-write across threads | D3 | a lock, or move the state into one owner |
| Async service freezes under load | a blocking call inside the event loop | D3, D5 | `run_in_executor` / async client; see §2.4 |
| CPU pegged at 100% with 8 threads and no progress | GIL contention, or a spin-wait, or catastrophic backtracking (§1.9) | D3, D6 | profile; move CPU work to processes; add a CPU rlimit |
| Child process becomes a zombie | parent never `wait()`ed; the exit status is still held | D2 | always reap; `subprocess` handles it if you `wait`/context-manage |
| Everything slows down, disk light solid | thrashing: working set > RAM, pages evicted and re-faulted | D4 | reduce working set or add RAM; swap is not a capacity plan |
| First request after deploy is 10× slower | cold page cache, cold `.pyc`, cold specialization, cold JIT-ish warm-up | D1, D4, D6 | warm on startup; measure with the cold case included |

---

## 1.7 System design ① — a single-machine job runner

**Problem.** Accept up to 50 jobs/second, each a small JSON payload; execute each **exactly once** (or, honestly, *at-least-once with idempotent execution*); survive `kill -9` of any worker and a machine power-loss without losing an acknowledged job; run on one box with no Redis, no Postgres, no cloud queue. You will meet the distributed twin of this design on Day 49 — build the single-machine version now so that the distributed one is a *change of failure model*, not a new subject.

### Requirements → the four hard parts

1. **Durability of enqueue** — once `enqueue()` returns, the job survives power loss. (Rung 11: this is an fsync, ~100 µs–5 ms. It caps enqueue throughput; batching or group-commit is the escape hatch.)
2. **Atomic claim** — exactly one worker may take a given job. (Day 3's race, solved with a filesystem primitive rather than a lock.)
3. **Crash recovery** — a worker killed mid-job must not strand the job forever. (Day 2's process lifecycle: leases + a reaper.)
4. **Graceful shutdown** — SIGTERM should finish the current job, not abandon it. (Day 2's signals.)

### Key decision: the queue *is* the directory

```
queue/
  ready/       job-000123.json                 ← enqueued, unclaimed
  inflight/    job-000123.json.wrk-4711.lease  ← claimed by pid 4711 at time T
  done/        job-000123.json
  failed/      job-000123.json + .err
```

The claim operation is `os.rename(ready/J, inflight/J.wrk-<pid>.lease)`. **POSIX `rename(2)` is atomic**: exactly one of N racing workers succeeds; the losers get `FileNotFoundError` and move on. No lock file, no lock server, no lease negotiation — the kernel's directory-entry update *is* the mutual exclusion (I1 + I3).

### The trade-off, stated honestly

| Choice | Buys | Costs |
|---|---|---|
| Directory-as-queue | zero dependencies; crash state is visible with `ls`; atomic claim for free | one syscall-heavy dir op per state change; ~10³–10⁴ jobs/s ceiling; no ordering guarantee beyond filename sort; degrades with millions of files in one directory |
| fsync on enqueue | true durability | caps enqueue at rung 11 (~200–1000/s per single fsync); mitigate with group commit |
| Lease + reaper | recovery from `kill -9` | at-least-once delivery → **handlers must be idempotent**; a slow job can have its lease stolen |
| Process-per-worker | GIL-free parallelism, fault isolation (a segfaulting job kills only its worker) | ×N memory; IPC cost; must reap children |
| SQLite/WAL instead | transactions, indexes, one file, better at high counts | a real dependency, and now you own its locking model |

### Failure modes and the answer to each

- **Crash after write, before rename into `ready/`** → a partial file exists in `tmp/` and is never visible. The writer writes to `tmp/`, fsyncs, then renames — so `ready/` only ever contains complete files. This "write-temp, fsync, atomic-rename, fsync-dir" sequence is the single most valuable durability idiom in this note.
- **Crash while holding a lease** → lease file's mtime stops advancing; the reaper renames it back to `ready/` after `LEASE_TTL`. Cost: at-least-once.
- **Slow job exceeds TTL** → duplicate execution. Fix: the worker *heartbeats* by touching the lease file, and the reaper uses mtime, not claim time.
- **Poison job kills every worker that touches it** → attempt counter in the filename or a sidecar; after N attempts, move to `failed/`. Without this, one bad job takes down the whole pool in a loop (the local version of a poison-queue, exactly like the dead-letter queue in a cloud pipeline).
- **Disk full** → `enqueue` fails loudly (good). Handle `ENOSPC` at the boundary; never silently drop.

### Why this over the alternatives

- **In-memory `queue.Queue`**: loses everything on restart. Fine for intra-process fan-out, disqualified by requirement 1.
- **SQLite**: genuinely better past ~10⁵ jobs or when you need queries; the reason to start with directories is *pedagogical and operational* — every state transition is a filesystem operation you can see, and there is no second concurrency model to learn.
- **Redis/RabbitMQ/SQS**: correct at scale and the right answer on Day 49; here they would hide the exact mechanisms (atomicity, durability, leases) this build exists to teach.

### Runnable example — the whole job runner, ~90 lines

```python
# jobq.py — a durable, crash-recoverable, single-machine job runner.
# pip install: nothing (stdlib only). Python 3.11+.
# [portable with a caveat] POSIX rename(2) is atomic. On Windows, os.replace
# maps to MoveFileExW(MOVEFILE_REPLACE_EXISTING), which replaces silently but
# does NOT carry POSIX's cross-process atomicity guarantee in all cases, and
# renaming an OPEN file fails. Run this on Linux/WSL for the real guarantees.
import json, os, sys, signal, time, uuid, pathlib

ROOT = pathlib.Path(sys.argv[1] if len(sys.argv) > 1 else "./queue")
TMP, READY, INFLIGHT, DONE, FAILED = (ROOT / d for d in
    ("tmp", "ready", "inflight", "done", "failed"))
LEASE_TTL, MAX_ATTEMPTS = 30.0, 3

def setup():
    for d in (TMP, READY, INFLIGHT, DONE, FAILED):
        d.mkdir(parents=True, exist_ok=True)

def _fsync_dir(path: pathlib.Path) -> None:
    """A rename is only durable once the DIRECTORY entry is flushed."""
    fd = os.open(path, os.O_RDONLY)          # a directory is just an fd (I3)
    try:
        os.fsync(fd)
    finally:
        os.close(fd)

def enqueue(payload: dict) -> str:
    """Durable enqueue: write to tmp, fsync file, atomic rename, fsync dir."""
    job_id = f"{time.time_ns()}-{uuid.uuid4().hex[:8]}"
    tmp_path = TMP / f"{job_id}.json"
    data = json.dumps({"id": job_id, "attempts": 0, "payload": payload}).encode()
    fd = os.open(tmp_path, os.O_WRONLY | os.O_CREAT | os.O_EXCL, 0o600)
    try:
        os.write(fd, data)
        os.fsync(fd)                          # rung 11: the durability boundary
    finally:
        os.close(fd)
    os.replace(tmp_path, READY / f"{job_id}.json")   # atomic publish
    _fsync_dir(READY)
    return job_id

def claim() -> tuple[pathlib.Path, dict] | None:
    """Atomically take one job. Losers of the race simply see FileNotFoundError."""
    for entry in sorted(os.listdir(READY)):            # FIFO-ish by name
        src, dst = READY / entry, INFLIGHT / f"{entry}.wrk{os.getpid()}.lease"
        try:
            os.rename(src, dst)                        # <-- the mutual exclusion
        except (FileNotFoundError, PermissionError):
            continue                                   # another worker won
        return dst, json.loads(dst.read_bytes())
    return None

def heartbeat(lease: pathlib.Path) -> None:
    os.utime(lease, None)                              # push mtime forward

def complete(lease: pathlib.Path, job: dict) -> None:
    os.replace(lease, DONE / f"{job['id']}.json")

def fail(lease: pathlib.Path, job: dict, err: str) -> None:
    job["attempts"] += 1
    if job["attempts"] >= MAX_ATTEMPTS:
        (FAILED / f"{job['id']}.err").write_text(err)
        os.replace(lease, FAILED / f"{job['id']}.json")
    else:
        tmp = TMP / f"{job['id']}.retry"
        tmp.write_text(json.dumps(job))
        os.replace(tmp, READY / f"{job['id']}.json")   # back to ready
        os.unlink(lease)

def reap() -> int:
    """Return jobs whose lease has gone stale (worker died) to ready/."""
    n, now = 0, time.time()
    for entry in os.listdir(INFLIGHT):
        lease = INFLIGHT / entry
        try:
            if now - lease.stat().st_mtime < LEASE_TTL:
                continue
            job = json.loads(lease.read_bytes()); job["attempts"] += 1
            tmp = TMP / f"{job['id']}.reap"; tmp.write_text(json.dumps(job))
            os.replace(tmp, READY / f"{job['id']}.json"); os.unlink(lease); n += 1
        except FileNotFoundError:
            continue
    return n

_stop = False
def _on_term(signum, frame):
    global _stop
    _stop = True                    # set a flag; NEVER do real work in a handler
    print(f"[{os.getpid()}] SIGTERM: finishing current job, then exiting", flush=True)

def handle(job: dict) -> None:
    """Your idempotent business logic. At-least-once delivery is the contract."""
    time.sleep(0.05)
    if job["payload"].get("boom"):
        raise RuntimeError("poison job")

def worker(max_idle=3.0) -> None:
    signal.signal(signal.SIGTERM, _on_term)
    signal.signal(signal.SIGINT, _on_term)
    idle_since = time.time()
    while not _stop:
        reap()
        got = claim()
        if got is None:
            if time.time() - idle_since > max_idle:
                break
            time.sleep(0.05); continue
        idle_since = time.time()
        lease, job = got
        try:
            handle(job); heartbeat(lease); complete(lease, job)
            print(f"[{os.getpid()}] done {job['id']}", flush=True)
        except Exception as exc:                       # never let one job kill the loop
            fail(lease, job, repr(exc))
            print(f"[{os.getpid()}] failed {job['id']}: {exc!r}", flush=True)
    print(f"[{os.getpid()}] worker exit", flush=True)

if __name__ == "__main__":
    setup()
    mode = os.environ.get("MODE", "demo")
    if mode == "worker":
        worker()
    else:
        ids = [enqueue({"n": i}) for i in range(20)] + [enqueue({"boom": True})]
        print(f"enqueued {len(ids)} jobs into {READY}")
        pids = []
        for _ in range(3):                             # 3 worker PROCESSES → 3 GILs
            pid = os.fork() if hasattr(os, "fork") else None
            if pid == 0:
                worker(); os._exit(0)
            pids.append(pid)
        for pid in pids:
            os.waitpid(pid, 0)                         # reap children (no zombies)
        print("counts:",
              {d.name: len(os.listdir(d)) for d in (READY, INFLIGHT, DONE, FAILED)})
```

```console
$ python jobq.py /tmp/q
enqueued 21 jobs into /tmp/q/ready
[41823] done 1754003912345678901-1a2b3c4d
[41824] done 1754003912345690012-9f8e7d6c
...
[41825] failed 1754003912346001234-bbccddee: RuntimeError('poison job')
[41823] worker exit
[41824] worker exit
[41825] worker exit
counts: {'ready': 0, 'inflight': 0, 'done': 20, 'failed': 0}

# Prove crash recovery: start a worker, kill -9 it mid-job, watch the reaper.
$ MODE=worker python jobq.py /tmp/q &  ; sleep 0.2 ; kill -9 %1
$ ls /tmp/q/inflight            # a stranded lease
job-...json.wrk41999.lease
$ python -c "import jobq; jobq.LEASE_TTL=0; print('reaped', jobq.reap())"
reaped 1
```

**Why this works, line by line.**

- **`enqueue` is four operations in a fixed order**: write to `tmp/`, `fsync` the *file* (data reaches the device), `os.replace` into `ready/` (atomic publish — a reader never sees a half-written job), `fsync` the *directory* (the new directory entry itself reaches the device). Skip the last step and a power loss can leave you with durable file contents and no name pointing at them. This is the idiom that databases, editors, and package managers all use, and it is pure Day 5.
- **`claim()` races on `os.rename`.** Three workers list the same `ready/` contents; all three attempt the same rename; the kernel serializes the directory-entry update so exactly one succeeds and the others raise `FileNotFoundError`. This replaces a lock with an *atomic state transition*, which is the same trick as a compare-and-swap in Day 3 — and it is why the workers need no shared memory at all.
- **The lease filename encodes the owner (`wrk<pid>`) and its mtime encodes liveness.** `heartbeat()` is just `os.utime`. The reaper's rule — "stale mtime ⇒ the owner is gone" — is a *failure detector*, and like every failure detector it can be wrong (a paused worker looks dead). That inevitability is why the delivery guarantee is at-least-once, and why `handle()` carries the comment demanding idempotency. On Day 49 the same reasoning reappears as visibility timeouts and fencing tokens.
- **`_on_term` sets a flag and returns.** Signal handlers run at unpredictable points between bytecodes (Day 2 + Day 6); doing I/O or taking locks inside one risks reentrancy deadlock. The loop checks `_stop` at a safe boundary — the canonical graceful-shutdown shape.
- **`os.fork()` gives each worker its own interpreter and GIL** (§1.5), and `os.waitpid` reaps them so no zombies remain (Day 2). On Windows `os.fork` doesn't exist — hence the `hasattr` guard; use `multiprocessing` with `if __name__ == "__main__"` there.
- **`except Exception` around `handle()`** implements the failure philosophy: one bad job produces a `failed/` entry, never a dead worker. Bounding `attempts` prevents a poison job from cycling forever.

**Definition of done for this build:** (a) `kill -9` a worker mid-job and see the job complete after the reaper runs; (b) `kill -TERM` a worker and see it finish the current job before exiting; (c) run two workers over 1000 jobs and assert `len(done) == 1000` with no duplicates in a side-effect log; (d) measure your enqueue rate with and without the `fsync` calls and explain the ratio using rung 11.

---

## 1.8 System design ② — sizing a mixed CPU + I/O service on one box

**Problem.** One 8-core box, 16 GiB RAM. An HTTP API where each request does ~5 ms of blocking I/O (a database round trip) and ~15 ms of pure-Python CPU (validation, serialization, a small transform). Target: 400 req/s at p99 < 200 ms. How many processes, how many threads each, and what breaks first?

### Requirements → arithmetic before architecture

- CPU demand: 400 req/s × 15 ms = **6.0 core-seconds per second** → you need ≥6 cores of *pure-Python* throughput. With 8 cores, headroom is thin but real.
- I/O demand: 400 × 5 ms = 2.0 "connection-seconds" per second → ~2 concurrent in-flight DB calls on average; trivial.
- **Therefore this is a CPU-bound service wearing an I/O costume**, and invariant I4 decides the shape: threads cannot deliver 6 cores of pure-Python work, so the top-level unit *must* be processes.

### Key decision and the trade-off

**Decision: `workers = cores + 1 = 8–9` processes, 2 threads each (or async with a small executor), not 1 process × 64 threads.**

| Option | Predicted outcome | Why |
|---|---|---|
| 1 proc × 64 threads | ~1 core of CPU throughput → ~65 req/s, p99 collapses | GIL serializes the 15 ms CPU segment; 64 threads add 5 ms hand-off jitter and context-switch cost |
| 8 procs × 1 thread | ~7.5 cores usable → ~450 req/s, but each process idles during its 5 ms DB wait | no intra-process overlap; ~25% of each worker's time is blocked |
| **8 procs × 2 threads** | one thread computes while the other waits on I/O; ~460–500 req/s | GIL is *released* during the DB call, so the second thread makes progress — threads used exactly where they work |
| 8 procs × 16 threads | no better than 2, worse p99 | CPU is saturated; extra threads only add queueing and switch overhead — latency grows, throughput doesn't |
| async, 1 process | worst | the 15 ms CPU segment blocks the loop for every connection (see §2.4/§3.2) |

**Memory check (Day 4).** 9 processes × ~150 MiB RSS ≈ 1.35 GiB — fine, *provided* copy-on-write is not being destroyed. If you pre-fork after loading a 500 MiB model or cache, the pages start shared, but Python's refcount updates *write* to the object headers, dirtying pages and un-sharing them over time. This is why pre-fork servers with big in-memory data see RSS creep — a Day 4 + Day 6 interaction worth naming out loud.

### Failure modes at the edges

- **Queue collapse above ~470 req/s.** CPU saturates; by Little's Law the queue grows without bound and p99 goes to infinity. Fix: bound the accept queue and shed load (fast 503) rather than accepting work you can't finish.
- **One slow query stalls a worker's threads.** With 2 threads, a 2 s query blocks half that worker's capacity. Fix: client-side timeouts, always.
- **Thread stacks inflate VSZ**, alarming dashboards that watch virtual instead of resident memory. Day 4 vocabulary prevents a false alarm.
- **BLAS/NumPy oversubscription.** If the "transform" uses NumPy, each of 9 processes may spawn 8 BLAS threads → 72 runnable threads on 8 cores → thrash. Fix: `OMP_NUM_THREADS=1` per worker. This is the single most common production mis-sizing in Python services.

### Why this over the alternatives

Vertical splitting (one CPU-heavy service + one I/O-heavy service) is the *right* next step if the CPU segment grows, because it lets each part scale on its own axis — but it introduces a network hop (rung 12, ~0.3 ms) and a deployment. On one box, at this ratio, process-level parallelism plus a couple of threads extracts nearly all the available hardware with no new failure domains.

---

## 1.9 Case studies

Real, named, primary-sourced. Two of the three are postmortems, because failure exposes mechanism better than success.

### 1. PostgreSQL "fsyncgate" (2018) — *the durability boundary is not what you think*

**What happened.** PostgreSQL discovered that on Linux, if a write-back error occurred *after* dirty pages had been handed to the kernel, the error could be reported to a *single* `fsync()` caller and the pages then marked clean — so a later `fsync()` returned success even though the data was never written. Postgres, which relied on retrying `fsync` after a failure, could therefore acknowledge a checkpoint that had silently lost data.

**Mechanism, in this note's vocabulary.** Rung 11 is not a synchronous wire to the platter. `write()` puts bytes in the page cache (Day 4/5); `fsync()` asks for them to be pushed; error *state* lived on the page, and once consumed it was gone. Two processes, one fd table each (I3), sharing one page cache — and error delivery was per-fd, not per-page.

**Engineering lesson.** A durability claim is only as strong as its **error path**, and error paths are the least-tested code in any system. PostgreSQL's fix was blunt and correct: on `fsync` failure, `PANIC` and recover from WAL rather than pretend. Apply the same rule to §1.7: an `fsync` that returns an error must fail the enqueue loudly, never be retried into apparent success.

**Primary sources.** Craig Ringer's pgsql-hackers thread "PostgreSQL's handling of fsync() errors is unsafe and risks data loss at least on XFS" (2018-03-28); LWN.net, "PostgreSQL's fsync() surprise" (2018); PostgreSQL release notes for the data-loss-on-fsync-failure change (PANIC on fsync failure, PG 12). *Verify current kernel behaviour before relying on any specific claim — Linux's error reporting changed after this (errseq_t work).*

### 2. Cloudflare global outage, 2 July 2019 — *CPU is a shared resource and regexes are code*

**What happened.** A single regular expression deployed to Cloudflare's WAF contained nested quantifiers that caused catastrophic backtracking. It consumed 100% of CPU on the request-processing cores across the fleet; HTTP traffic served 502s worldwide for roughly 27 minutes until the rule was rolled back via a global kill switch.

**Mechanism.** Day 1 + Day 3: the regex engine's backtracking made an *input-dependent* exponential number of steps, on the same cores that served requests. No memory limit, no fd limit, no network limit was hit — only the CPU budget, and there was no per-request CPU ceiling to hit *first*. Everything else on those cores starved.

**Engineering lesson, and the direct link to Part 3.** Untrusted-input-driven work needs a **CPU bound**, not just a memory bound. In §3.1 this becomes `RLIMIT_CPU` plus a wall-clock kill: the agent sandbox's most likely failure is not an exploit, it is an accidental infinite loop, and only a CPU/time cap turns that from an outage into an error message. Also: a global kill switch is a feature.

**Primary source.** Cloudflare Blog, "Details of the Cloudflare outage on July 2, 2019" (John Graham-Cumming, 12 July 2019).

### 3. GitLab.com database incident, 31 January 2017 — *untested recovery is not recovery*

**What happened.** During an incident response, an engineer ran a destructive directory removal against the wrong database host. Recovery then revealed that five separate backup/replication mechanisms were each broken or ineffective; ~6 hours of issue/MR data was permanently lost. GitLab published a detailed public postmortem and live-streamed the recovery.

**Mechanism.** Days 2 and 5 in their least glamorous form: an unrecoverable syscall (`unlink` on live data files) executed by a privileged human on the wrong process's data, and durability mechanisms that had *never been exercised end-to-end*.

**Engineering lesson.** Every durability guarantee is a claim about a **restore you have actually performed**. In §1.7 terms: the `done/` directory is not a backup, and a test that only checks the happy path proves nothing about crash recovery. Write the `kill -9` test.

**Primary source.** GitLab, "Postmortem of database outage of January 31" (2017), on about.gitlab.com/blog.

*(Honesty per Principle 7: I have deliberately not invented a fourth case study for the "worker sizing" design in §1.8 — the BLAS-oversubscription pattern is folklore-common but I know of no single canonical public postmortem for it, so it is presented as a mechanism, not as an incident.)*

---

## 1.10 In production — operating the machine model

**Best practices, beginner → senior.**

| Level | Habit |
|---|---|
| Beginner | Use context managers for every fd. Set a timeout on every network call. Never `except: pass`. |
| Intermediate | Bound every queue, cache, and pool. Log with structured fields, buffered, never fsynced per line. Distinguish RSS from VSZ when reading dashboards. |
| Senior | Put a limit on every resource *before* the kernel does it for you (memory, CPU, fds, wall clock). Make retries idempotent by construction. Load-shed rather than queue without bound. Measure the cold path, not just the warm one. |

**Monitoring — the ten signals that map to Days 1–6.**

1. run-queue length / load average (D3: more runnable than cores?)
2. involuntary context switches per second (D3: preemption churn)
3. RSS *and* VSZ per process (D4: real vs reserved)
4. major page faults per second (D4: you are reading disk through memory)
5. swap in/out (D4: thrashing = the end of predictable latency)
6. open fd count vs the rlimit (D5: leak detector)
7. disk `fsync` latency p99 (D5: your durability tax)
8. GC collection counts/durations by generation (D6)
9. per-worker CPU time vs wall time (D3/D6: are you GIL-bound or I/O-bound?)
10. exit codes of children — especially 137 (SIGKILL/OOM) and 139 (SIGSEGV) (D2)

**Recovery playbook.** OOM-killed → shrink working set or raise the limit *and* add an in-process cap so you fail with an exception instead of SIGKILL. fd exhaustion → find the leak with `/proc/PID/fd`; raising the rlimit only buys time. Stuck worker → check whether it is blocked (voluntary switches climbing, CPU flat) or spinning (CPU pegged); those need opposite fixes. Corrupt queue state → the directory layout in §1.7 is designed so recovery is `mv inflight/* ready/`.

**Scaling behaviour.** Vertical scaling ends where a single kernel's contention ends (locks, one page cache, one NIC). The machine model tells you *which* rung you hit first: if it's CPU, add processes then boxes; if it's fsync, batch (group commit) or relax the guarantee; if it's context switches, reduce concurrency; if it's memory bandwidth, restructure data layout — adding cores will not help.

**Cost.** On rented hardware, cost tracks the *rungs you climb most*. A service doing one fsync per request pays for IOPS-provisioned storage; one doing per-request CPU work pays for cores; one holding a large cache pays for RAM. Because the ladder spans 10⁸, moving an operation down one rung (e.g. buffering 8 KiB instead of writing per line) is routinely a 10× cost change — far more than any code-level micro-optimization.

---

## 1.11 Part 1 — interview and practice questions

1. Walk me through `python app.py` from Enter to your first executed line. *(If you cannot name 10+ stages, redo §1.3.)*
2. A process shows VSZ 12 GiB, RSS 300 MiB. Is anything wrong? *(No — address space ≠ memory; check major faults instead.)*
3. Why is `write()` fast and `fsync()` slow, and what exactly does each promise?
4. You have 8 cores and a pure-Python CPU-bound job. Threads give 1× speedup. Explain precisely, then give two ways to get real parallelism.
5. Two workers processed the same job. Give three mechanisms that could cause it and the one filesystem primitive that prevents the simplest one.
6. What is a minor page fault, a major page fault, and why does the distinction dominate p99 latency?
7. Your service dies with exit code 137 and no traceback. What happened and why is there no traceback?
8. Why can't a signal handler safely take a lock?
9. `pip install requests` succeeded but `import requests` fails. Explain using `sys.prefix`.
10. Rank by latency: L3 hit, minor fault, syscall, context switch, NVMe read, fsync, same-DC RTT. Give an order of magnitude for each.
11. **Practice build:** add group-commit to §1.7's `enqueue` (batch up to 50 ms of enqueues behind one fsync) and measure the throughput change. Explain the durability semantics you just changed.
12. **Practice build:** instrument `jobq.py` to log `ru_minflt`, `ru_majflt`, and voluntary/involuntary context switches per job, then explain the pattern you see.

<!-- APPEND HERE -->
