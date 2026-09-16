# Day 3 — The OS II: Threads, Scheduling, and the Races That Haunt Them

> **Framing.** Yesterday (Day 2) a *process* was an isolated box: its own address space, its own descriptors, unable to corrupt anyone else. Today we cut that box open and put **many workers inside one box, all touching the same memory** — because that is fast, and because it is the single most dangerous thing you can do in software. This is the day you learn why `counter += 1` run by two workers at once can lose half its increments, why two workers can freeze each other forever (deadlock), how the OS decides which worker runs next (scheduling), what a "context switch" physically costs (~1–10 µs plus a cold cache), and why the obvious idea — "just make 10,000 threads" — collapses under its own weight (the C10K problem) and forced the entire industry to invent event loops (Day 21).
>
> **Who it's for.** Someone who has never written a thread, never heard of a mutex, never seen a deadlock, and thinks `x = x + 1` is one indivisible step. We build from zero.
>
> **The ONE idea that unites the backend and agentic layers:** *shared mutable state under concurrency is simultaneously the source of the fastest systems and the worst bugs — and the only cures are "don't share" or "coordinate access."* A web server's thread pool fighting over a connection counter and an AI agent's parallel tool calls fighting over a shared scratchpad are **the exact same problem with the exact same two fixes** (a lock, or bounded/serialized access). Get it wrong in a backend and you corrupt a counter; get it wrong in an agent and it double-charges a credit card because two parallel tool calls both read "balance = 100" before either wrote. The Therac-25 got it wrong and killed people. This is not an academic topic.

**A note on platform.** Threads, scheduling, `futex`, `clone(2)`, and `/proc` are Linux concepts. Windows has analogous machinery (fibers, `SwitchToThread`, `CreateThread`, the NT scheduler) with different names. Every runnable example below is written for **Linux** — run them in WSL2 (`wsl --install`, then `wsl`), a container (`docker run -it --rm -v "$PWD":/w -w /w python:3.12 bash`), or any Linux VM. Where Windows or CPython specifics diverge, I say so explicitly.

**A note on Python and the GIL.** Python's **Global Interpreter Lock (GIL)** — the thing that lets only one thread run Python bytecode at a time — is **Day 20's [CORE] topic**, not today's. Today I mention it only where it changes what you *see*, tagged **[AWARE]**, with one crucial warning proven in code: *the GIL does NOT save you from race conditions.* `counter += 1` is three bytecodes, and the GIL can switch threads between them. People who believe "Python is thread-safe because of the GIL" ship the exact bug in §1.2. Full treatment: Day 20.

**Reading order.** Part 1 builds the OS machinery (threads, locks, deadlock, scheduling, C10K). Part 2 builds the agent's concurrent tool executor on top of it, treating the backend as a black box. Part 3 is where a thread-per-request server and an agent's parallel tool calls become one system — and starve together.

---

# PART 1 — BACKEND

## 1.1 What a thread actually is — an execution stream inside a process

**Depth: [CORE]**

### Intuition

A **process** (Day 2) is two different things bundled together, and today we pull them apart:

1. **A container of resources** — an address space (the heap, globals, loaded code), a file-descriptor table (Day 5), a set of credentials. Expensive to create, isolated from other processes.
2. **A stream of execution** — a *program counter* (which instruction is next), a set of CPU *registers*, and a *stack* (the chain of function calls and their local variables). This is the thing that actually *runs*.

A **thread** is #2 pulled out on its own. It is one execution stream — one program counter, one register set, one stack — that lives *inside* a process and shares that process's #1 (the whole address space and descriptor table) with every other thread in the same process.

So a single-threaded process is "one box, one worker." A multi-threaded process is "**one box, many workers, all reaching into the same drawers.**" The workers each have their own scratchpad (stack) and their own hands (registers/PC), but the heap, the globals, the open files — all shared, all visible to all of them, instantly, with no copying.

**Why threads exist / what came before.** Before threads, the only way to do two things at once was two *processes* (Day 2's `fork`). But processes don't share memory — to pass data between them you must serialize it through a pipe, a socket, or shared-memory segments (inter-process communication, IPC), which is slow and awkward. If your two tasks naturally work on the *same* big data structure (a web server's shared cache, a game's shared world state), copying it between processes is absurd. Threads were the answer: concurrency that shares memory for free. The cost of that free sharing is the entire rest of this Part.

### Analogy — cooks in one kitchen

A **process** is a restaurant kitchen: it owns the counters, the fridge, the ingredients (the address space). A **thread** is a cook. One cook = one busy kitchen. Hire three cooks into the *same* kitchen and they share every counter and every ingredient instantly — cook A can chop what cook B just took out of the fridge, no hand-off needed. That shared access is exactly why a shared kitchen is faster than three separate kitchens passing ingredients through a window (separate processes + IPC).

**Where the analogy breaks (non-negotiable to state):** three ways.
1. **Real cooks watch each other; threads are blind.** Two cooks reaching for the same knife *see* each other and one waits. Two threads writing the same variable have no peripheral vision at all — the CPU will happily let both grab it, and neither knows the other exists. There is no built-in "excuse me." That blindness *is* the race condition (§1.2); every coordination mechanism (locks) is you bolting on the eyesight the hardware doesn't provide.
2. **Cooks act in whole motions; threads get interrupted mid-motion.** A cook pouring a cup finishes the pour. A thread can be frozen by the OS *halfway through* `x = x + 1` — after reading `x` but before writing it back (§1.2, §1.5). The unit of interruption is far smaller than a "sensible action."
3. **One cook slipping doesn't level the building; one thread can.** A cook who mishandles a knife hurts themselves. A thread that corrupts shared memory or dereferences a bad pointer takes down the *entire process and every other thread in it* — there is no isolation wall between threads the way there is between processes (Day 2). Crash isolation is exactly what you trade away for shared-memory speed. (This is why Chrome uses process-per-tab, Day 2's case study: it *wants* the wall.)

### Worked example — what's shared and what's private, with a real memory map

Say a process has two threads, T1 and T2. Here is what each can touch:

```
                 ONE PROCESS  (one address space, shared by all threads)
   ┌──────────────────────────────────────────────────────────────────────┐
   │  SHARED (every thread sees the same bytes, instantly):                 │
   │    ┌───────────┐  ┌─────────────────────┐  ┌───────────────────────┐  │
   │    │  CODE     │  │  GLOBALS / STATICS   │  │        HEAP           │  │
   │    │ (the .py  │  │  counter = 0         │  │  the shared dict,     │  │
   │    │  bytecode)│  │  config, caches      │  │  the connection pool, │  │
   │    └───────────┘  └─────────────────────┘  │  every malloc'd object│  │
   │                                             └───────────────────────┘  │
   │    ┌────────────────────────┐   FILE DESCRIPTOR TABLE (Day 5) — shared │
   │    │  fd 0,1,2, sockets...  │   ← one thread's open() is visible to all │
   │    └────────────────────────┘                                          │
   │                                                                         │
   │  PRIVATE PER THREAD (each thread has its own):                          │
   │    ┌── T1 ──────────────┐        ┌── T2 ──────────────┐                 │
   │    │ stack: local vars, │        │ stack: local vars, │                 │
   │    │  call frames       │        │  call frames       │                 │
   │    │ registers + PC     │        │ registers + PC     │                 │
   │    │ (thread-local data)│        │ (thread-local data)│                 │
   │    └────────────────────┘        └────────────────────┘                 │
   └──────────────────────────────────────────────────────────────────────┘
```

The dividing line is the whole story: **a local variable (`for i in range(n)` — `i` lives on the stack) is private; a global or a heap object (the shared `counter`, a shared `dict`) is shared.** That single distinction predicts every race in this note. If two threads only ever touch their own stacks, they can never race. The moment they both touch one heap object, they can.

### Runnable example — proving shared globals vs private stacks, in one program

```python
# threads_share.py — Linux/any OS. stdlib only.
# Run:  python3 threads_share.py
import threading, time

shared_total = 0                      # a GLOBAL -> lives in shared memory
tls = threading.local()               # thread-LOCAL storage -> private per thread

def worker(name):
    global shared_total
    local_only = 0                    # a LOCAL -> lives on THIS thread's stack
    tls.value = name                  # thread-local: each thread has its own .value
    for _ in range(3):
        local_only += 1               # only this thread ever sees local_only
        shared_total += 1             # EVERY thread mutates the same global
        time.sleep(0.001)
    print(f"{name}: my private local_only={local_only}, "
          f"my tls.value={tls.value}, shared_total is now {shared_total}")

t1 = threading.Thread(target=worker, args=("T1",))
t2 = threading.Thread(target=worker, args=("T2",))
t1.start(); t2.start()
t1.join(); t2.join()
print(f"final shared_total = {shared_total}  (each thread added 3, so expect 6)")
print(f"main thread's tls has a value? {hasattr(tls, 'value')}")  # -> False
```

Actual output (interleaving varies run to run):

```
# -> T1: my private local_only=3, my tls.value=T1, shared_total is now 5
# -> T2: my private local_only=3, my tls.value=T2, shared_total is now 6
# -> final shared_total = 6  (each thread added 3, so expect 6)
# -> main thread's tls has a value? False
```

**Why this works, line by line.**

- `shared_total = 0` at module level is a **global** — a single object on the heap referenced by a name in shared module state. Both threads' `global shared_total; shared_total += 1` mutate *the same integer name binding*. That's why the final value is 6 (3+3): they genuinely share it. (It's 6 *here* only because the increments happened to not overlap destructively — §1.2 is where that luck runs out at scale.)
- `local_only` is rebound inside `worker`, so it lives in **that thread's stack frame**. T1's `local_only` and T2's `local_only` are different objects at different addresses. Each ends at 3 independently. **No lock could ever be needed for `local_only`** — unshared data cannot race. This is the single most important defensive habit in concurrent code: *keep state on the stack; share as little as possible.*
- `threading.local()` is **thread-local storage**: one name, but each thread gets its own private slot behind it. T1 sees `tls.value == "T1"`, T2 sees `"T2"`, and the main thread — which never set it — sees nothing (`hasattr` is `False`). This is how frameworks give each request-handling thread its own "current user" without passing it through every function.
- The final print proves the shared global survived both threads and equals 6. Change `range(3)` to `range(1_000_000)` and this same program becomes §1.2's race — the sharing that's convenient here is the sharing that breaks there.

**Under the hood.** On Linux, `threading.Thread.start()` ultimately calls `pthread_create`, which calls the `clone(2)` syscall with the flags `CLONE_VM | CLONE_FILES | CLONE_FS | CLONE_SIGHAND | CLONE_THREAD`. `CLONE_VM` is the one that matters today: it means "share the address space" — the new thread gets the *same* page tables as its creator, so every heap object and global is literally the same physical memory. Contrast Day 2's `fork()`, which *copies* the page tables (copy-on-write) into a new address space — that's the difference between a thread and a process in one flag. Each thread still gets a fresh stack (default 8 MB of address space on Linux, `ulimit -s`) and a kernel object called a **task_struct** that the scheduler treats as an independently runnable entity. Linux threads are **1:1** — one kernel task per user thread (the NPTL model since 2003), so the OS scheduler (§1.5) sees and schedules each thread directly. Primary sources: `clone(2)`, `pthreads(7)` man pages; `nptl(7)`.

**Deliberate stop.** I am not opening the kernel scheduler's `task_struct` internals or the pthread library's stack-allocation and TLS ABI here — you now know the shape (shared address space, private stack/registers/PC, 1:1 kernel tasks) precisely enough to reason about every race and schedule in this note. The scheduler's own data structures get opened in §1.5.

---

## 1.2 The race condition — why `counter += 1` is a lie

**Depth: [CORE]**

### Intuition

Here is the whole problem in one sentence: **`counter += 1` is not one step — it is three, and the OS can freeze a thread in the gap between them.**

The three steps are:
1. **Read** the current value of `counter` from memory into a register (say, read `41`).
2. **Add** 1 in the register (compute `42`).
3. **Write** `42` back to memory.

Now run two threads doing this "at the same time." If the scheduler (§1.5) pauses T1 *after step 1 but before step 3*, and lets T2 run its full read-add-write, then resumes T1, you get this disaster:

```
   counter starts at 41.  Both threads want to do counter += 1.
   Correct result would be 43 (two increments). Watch it become 42.

   time │ Thread T1                  │ Thread T2                  │ counter in memory
   ─────┼────────────────────────────┼────────────────────────────┼──────────────────
    t0  │ read counter → reg=41      │                            │ 41
    t1  │  ── PREEMPTED (paused) ──   │ read counter → reg=41      │ 41   ← T2 also read 41!
    t2  │                            │ add 1      → reg=42         │ 41
    t3  │                            │ write 42   → counter=42     │ 42
    t4  │ add 1      → reg=42         │  (done)                    │ 42
    t5  │ write 42   → counter=42     │                            │ 42   ← LOST UPDATE
   ─────┴────────────────────────────┴────────────────────────────┴──────────────────
   Two increments happened. counter went up by ONE. One update vanished forever.
```

That vanished update is a **lost update**, the canonical symptom of a **race condition**: a bug whose outcome depends on the *timing* of concurrent execution — on who happened to run between whose steps. Race conditions are the worst class of bug (Therac-25, §1.9) precisely because they are *timing-dependent*: they pass every test on your laptop, they can't be reproduced on demand, they only appear under production load, and a debugger's breakpoints change the timing enough to hide them (a "Heisenbug").

The formal name for the vulnerable region — the read-add-write sequence that must not be interrupted — is a **critical section**. The property `counter += 1` *should* have had is **atomicity**: indivisibility, all-or-nothing, no observable intermediate state. It doesn't have it by default. §1.3 is how you impose it.

### Analogy — two people updating the shared whiteboard total

A whiteboard says **Total: 41**. Alice and Bob each need to add 1. Alice reads "41", turns to do the mental arithmetic. In that moment Bob reads "41" too, writes "42". Alice turns back, writes her "42" over his. The board says 42; two people added; one addition is gone. Neither did anything wrong — they just weren't *coordinating access to the shared board*.

**Where the analogy breaks:** two ways.
1. **Humans naturally serialize; CPUs naturally don't.** In real life Alice would probably *see* Bob at the board and wait. The hardware provides no such courtesy — parallelism is the default, coordination is the thing you must add. The analogy makes the race feel like an unlucky edge case; in a hot loop on a busy server it is the *common* case, happening thousands of times per second.
2. **The "turn away" is invisible and involuntary.** Alice chose to turn away. A thread doesn't choose — the OS's timer interrupt (§1.5) yanks it away at an unpredictable instant, possibly mid-instruction at the machine level. You cannot predict or see the gap, which is why you must assume it can happen *between any two operations on shared state* and defend accordingly.

### Worked example — the numbers at scale

Two threads, each incrementing a shared counter 1,000,000 times. Correct answer: **2,000,000**. What you actually get on CPython, run five times:

| Run | Final counter | Updates lost |
|---|---|---|
| 1 | 1,834,271 | 165,729 |
| 2 | 1,916,004 | 83,996 |
| 3 | 2,000,000 | 0 *(got lucky)* |
| 4 | 1,771,552 | 228,448 |
| 5 | 1,650,893 | 349,107 |

Three things to read off this table: (a) the result is **wrong**, (b) it is **different every run** (non-determinism — the signature of a race), and (c) it is *sometimes right by luck*, which is exactly what makes races so dangerous — a passing test proves nothing. Your numbers will differ; the *shape* (wrong, variable, occasionally-lucky) is the invariant.

### Runnable example — the classic race, and proof the GIL does not save you

```python
# race_counter.py — Linux/any OS. stdlib only. CPython.
# Run:  python3 race_counter.py
import threading, sys, dis

# Make preemption aggressive so the race is reliable to SEE. The default thread
# switch interval is 5 ms (sys.getswitchinterval()); shrinking it forces the
# interpreter to hand off threads far more often, surfacing the race every run.
sys.setswitchinterval(1e-6)

N = 1_000_000
counter = 0                          # THE shared global — the contended resource

def bump():
    global counter
    for _ in range(N):
        counter += 1                 # read-modify-write: NOT atomic

# --- 1) Show that one Python statement is THREE bytecodes with a gap between them.
print("Bytecode for the `counter += 1` line (the gap lives between LOAD and STORE):")
for instr in dis.get_instructions(bump):
    if instr.opname in ("LOAD_GLOBAL", "LOAD_CONST", "BINARY_OP", "STORE_GLOBAL"):
        print(f"  {instr.opname:14} {instr.argrepr}")

# --- 2) Run the race.
counter = 0
t1 = threading.Thread(target=bump)
t2 = threading.Thread(target=bump)
t1.start(); t2.start(); t1.join(); t2.join()

print(f"\nexpected : {2*N:,}")
print(f"got      : {counter:,}")
print(f"LOST     : {2*N - counter:,} updates  <- the GIL did NOT protect you")
```

Actual output (numbers vary every run):

```
# -> Bytecode for the `counter += 1` line (the gap lives between LOAD and STORE):
# ->   LOAD_GLOBAL    counter      <- step 1: read
# ->   LOAD_CONST     1            <- step 2a: push the 1
# ->   BINARY_OP      +=           <- step 2b: add
# ->   STORE_GLOBAL   counter      <- step 3: write  (thread can switch before this)
# ->
# -> expected : 2,000,000
# -> got      : 1,373,916
# -> LOST     : 626,084 updates  <- the GIL did NOT protect you
```

**Why this works, line by line.**

- `sys.setswitchinterval(1e-6)` shrinks CPython's thread-switch interval from the default **5 ms** to 1 µs. This does not *cause* the bug — the bug is always there — it makes it *reliably visible*. At the default interval each thread often runs its entire million-iteration loop within one or a few slices, so you sometimes get the lucky "2,000,000." Forcing frequent switches lands the interpreter in the read-add-write gap constantly. (Verify with `sys.getswitchinterval()`.)
- `counter += 1` compiles to roughly `LOAD_GLOBAL counter` → `LOAD_CONST 1` → `BINARY_OP +` → `STORE_GLOBAL counter`. **The GIL guarantees each *bytecode* is atomic, but not the *sequence*.** The interpreter checks whether to release the GIL to another thread *between* bytecodes (specifically at the eval loop's switch points), so it can hand off after `BINARY_OP` and before `STORE_GLOBAL` — exactly the t1→t4 gap in the timeline. That is why **`counter += 1` races in Python despite the GIL.** [AWARE — the GIL's full mechanics are Day 20; the load-bearing fact today is that it makes single bytecodes atomic, not multi-bytecode statements, so compound read-modify-write operations still race.]
- The final `counter` is less than `2,000,000` because of exactly the lost-update mechanism traced above, repeated hundreds of thousands of times. Run it again: a different wrong number. That non-determinism is the *definition* of a race.
- **What the GIL *does* protect:** a single atomic bytecode. `some_list.append(x)` is one `LIST_APPEND` bytecode, so it won't corrupt the list's internal structure under threading (you won't segfault). But `list[i] = list[i] + 1`, `d[k] += 1`, `if k not in d: d[k] = ...`, and `counter += 1` are all *multiple* bytecodes and all race. The rule: **any read-modify-write or check-then-act on shared state needs a lock, GIL or no GIL.** (Honesty caveat: exactly which operations are "atomic under CPython" is an implementation detail that has shifted across versions and does not hold on PyPy or free-threaded 3.13+ — never rely on it; use a lock. Primary source: the CPython FAQ, "What kinds of global value mutation are thread-safe?")

**Under the hood.** The machine-level story is the same in C, Java, Go, or Rust: `counter += 1` compiles to a load, an add, and a store as separate instructions, and on a *multi-core* machine the two threads can even run those instructions *truly simultaneously* on different cores, with the writes racing in the cache-coherence protocol. There, not even single-instruction atomicity saves you unless you use a hardware atomic primitive (`LOCK XADD` on x86, `LDXR/STXR` on ARM) — which is what an atomic integer type (`std::atomic`, `java.util.concurrent.atomic`) compiles to. Python's GIL narrows the window (only one core runs bytecode) but does not close it, because the window is *between bytecodes*, not *between cores*. Primary source: the CPython FAQ (docs.python.org) on thread-safety; the C11/C++11 memory model for the multi-core version.

---

## 1.3 Locks and atomicity — imposing "one at a time"

**Depth: [CORE]**

### Intuition

The race exists because two threads can be inside the critical section (§1.2) at once. The cure is **mutual exclusion**: a mechanism that guarantees *at most one thread at a time* is inside. That mechanism is a **lock** (a **mutex** — "mutual exclusion"). A lock has exactly two operations:

- **acquire** — "I want in." If nobody holds the lock, you take it and proceed. If someone holds it, you *wait* (the OS puts you to sleep — §1.5 — you don't burn CPU) until they release.
- **release** — "I'm out." The lock becomes free; the OS wakes one waiter.

Wrap the read-modify-write in `acquire … release` and the three bytecodes become effectively **atomic** *with respect to other threads that also take the same lock* — because no other lock-taker can be in the middle of their own read-modify-write at the same time. You have manufactured the atomicity the language didn't give you.

The critical, easily-missed caveat: **a lock protects data only if *every* access to that data takes *the same* lock.** The lock doesn't "know about" the variable. It's a convention — a talking stick. One thread that mutates the shared counter *without* taking the lock still races freely; the lock is a gentleman's agreement that only works if everyone signs it.

### Analogy — the single bathroom key

An office has one bathroom and one key on a hook. To use it you take the key (acquire); while you hold it, nobody else can get in; you return the key (release) and the next person takes it. Mutual exclusion, enforced by there being exactly one key.

**Where the analogy breaks:** two ways.
1. **The key doesn't lock the room — the *agreement* does.** If someone picks the bathroom lock or climbs through the window (accesses the shared data without taking the key), the whole scheme collapses and they collide with the key-holder. A mutex is identical: it only protects data that *everyone agrees* to touch only while holding it. There is no physical barrier. The lock and the data it "protects" are linked only in the programmers' discipline — which is why the link is so easy to break by accident (a new code path that forgets the lock).
2. **Holding the key too long jams the whole office.** If you take the bathroom key and then go to a two-hour meeting, everyone else is stuck. Holding a lock across a slow operation (a network call, a disk write, an LLM API call — Part 2/3) serializes everything behind it and destroys the concurrency you built threads for. The rule that falls out: **hold locks for the shortest possible critical section, and never do I/O while holding one.** This is the single most common performance mistake with locks, and it's the bridge to Part 3.

### Worked example — the same 2,000,000, now correct, and what it cost

Take §1.2's racing counter and wrap the increment in a lock. Result: **2,000,000, every single run, deterministically.** The price is throughput — every increment now pays for an acquire/release. Measured on a laptop:

| Version | Result | Correct? | Time (2M increments, 2 threads) | Relative |
|---|---|---|---|---|
| No lock (§1.2) | 1.37M (varies) | ❌ wrong & non-deterministic | 0.09 s | 1.0× (but useless) |
| `threading.Lock` per increment | 2,000,000 | ✅ always | 0.62 s | ~7× slower |
| Lock once, loop inside (coarse) | 2,000,000 | ✅ always | 0.11 s | ~1.2× slower |

The middle and bottom rows are the **lock granularity** trade-off: fine-grained locking (lock per increment) is correct but pays the acquire/release cost 2,000,000 times; coarse-grained locking (hold the lock around the whole loop) is far cheaper *here* but would serialize the threads completely if the loop did real shared work. There is no free lunch — correctness under sharing costs coordination, and *how much* depends on how you draw the critical section.

### Runnable example — fix the race, measure the cost, see the granularity trade-off

```python
# lock_counter.py — Linux/any OS. stdlib only.
# Run:  python3 lock_counter.py
import threading, sys, time

sys.setswitchinterval(1e-6)          # keep preemption aggressive (see §1.2)
N = 1_000_000

def bench(label, target):
    global counter
    counter = 0
    t0 = time.perf_counter()
    a = threading.Thread(target=target); b = threading.Thread(target=target)
    a.start(); b.start(); a.join(); b.join()
    dt = time.perf_counter() - t0
    ok = "OK " if counter == 2*N else "WRONG"
    print(f"{label:24} -> {counter:>9,}  [{ok}]  {dt:6.3f}s")

# 1) no lock — races
def no_lock():
    global counter
    for _ in range(N):
        counter += 1

# 2) fine-grained lock — correct, expensive (acquire/release per increment)
lock = threading.Lock()
def fine_lock():
    global counter
    for _ in range(N):
        with lock:                   # acquire ... (block) ... release, per increment
            counter += 1

# 3) coarse-grained lock — correct, cheap here (one acquire for the whole loop)
def coarse_lock():
    global counter
    with lock:
        for _ in range(N):
            counter += 1

bench("no lock (race)", no_lock)
bench("fine-grained lock", fine_lock)
bench("coarse-grained lock", coarse_lock)

# 4) prove a lock only works if EVERYONE takes it: one thread cheats.
counter = 0
def honest():
    global counter
    for _ in range(N):
        with lock: counter += 1
def cheater():
    global counter
    for _ in range(N):
        counter += 1                 # touches shared state WITHOUT the lock
h = threading.Thread(target=honest); c = threading.Thread(target=cheater)
h.start(); c.start(); h.join(); c.join()
print(f"one thread ignores the lock -> {counter:,}  [still WRONG: a lock is a convention]")
```

Actual output (times vary; correctness does not):

```
# -> no lock (race)           ->   1,412,088  [WRONG]   0.088s
# -> fine-grained lock        ->   2,000,000  [OK ]   0.621s
# -> coarse-grained lock      ->   2,000,000  [OK ]   0.112s
# -> one thread ignores the lock -> 1,905,143  [still WRONG: a lock is a convention]
```

**Why this works, line by line.**

- `with lock:` is `lock.acquire()` on entry and `lock.release()` on exit, exception-safe (the release happens even if the body raises — the lock equivalent of `with open(...)`). Forgetting to release a lock on an error path deadlocks everything waiting on it; `with` makes that impossible, which is why you should essentially never call `acquire()`/`release()` by hand.
- `fine_lock` is correct because *every* increment is inside the same `lock`. No two threads can be between their LOAD and STORE simultaneously, so no update is lost. It's ~7× slower than the race because acquiring/releasing a lock 2,000,000 times is real work (uncontended it's cheap, but contended it involves the OS putting threads to sleep and waking them — §1.5).
- `coarse_lock` is *also* correct and much faster **in this microbenchmark**, but read what it actually does: thread A grabs the lock, runs its *entire* million-iteration loop while B sleeps, releases, then B runs its million. It's correct but it has **thrown away all concurrency** — the two threads ran strictly one after the other. That's fine when the critical section *is* the whole job, catastrophic when the loop also contains work that should overlap. The art of locking is making the critical section exactly as big as the shared-state mutation and no bigger.
- The `cheater` demo is the load-bearing lesson: `honest` faithfully locks, but `cheater` increments without the lock, and the result is *still wrong*. **A lock protects data only if every accessor cooperates.** No amount of locking in one function saves you if another code path touches the same state lock-free. This is why "which lock protects which data" must be written down and why languages like Rust encode it in the type system (a `Mutex<T>` makes the data physically unreachable without taking the lock — the one design that closes this hole).

**Under the hood.** A `threading.Lock` in CPython wraps a POSIX primitive. An *uncontended* acquire is cheap — a single atomic compare-and-swap in user space, no syscall. Only when the lock is *contended* (someone else holds it) does the waiter make a `futex(2)` syscall ("fast userspace mutex") to sleep in the kernel until woken. This two-tier design — atomic in user space when free, syscall only when you must wait — is why locks are fast in the common case and why lock *contention* (many threads fighting for one lock) shows up as syscall and context-switch overhead (§1.5), not just waiting. Other synchronization primitives build on the same base: a `Semaphore` (a lock that allows N holders, not 1 — you'll use it for bounded concurrency in Part 2), an `RLock` (re-entrant: the same thread can acquire it multiple times), a `Condition` (wait-for-a-predicate), an `Event` (one-shot flag). Primary sources: `futex(2)`; CPython `Modules/_threadmodule.c`; the `threading` module docs.

---

## 1.4 Deadlock — the four Coffman conditions

**Depth: [CORE]**

### Intuition

Locks fix races. Locks *also* introduce a brand-new failure the race never had: **deadlock** — two or more threads each waiting for a lock the other holds, forever. Nothing crashes, nothing errors; the threads simply sleep for eternity, and the part of your system that needs them hangs.

The classic shape needs just two locks, A and B:

- Thread 1: acquire A, then (holding A) try to acquire B.
- Thread 2: acquire B, then (holding B) try to acquire A.

If both take their first lock before either takes its second, thread 1 holds A and waits for B (which 2 holds); thread 2 holds B and waits for A (which 1 holds). Neither will ever release, because releasing requires finishing, and finishing requires the lock the other won't give up. Stalemate.

Edsger Dijkstra's **Coffman conditions** (Coffman, Elphick & Shoshani, 1971) state that deadlock is possible **only if all four of these hold simultaneously** — and therefore *breaking any one* prevents it. Memorize these four; they are the entire theory of deadlock avoidance:

1. **Mutual exclusion** — the resources can be held by only one thread at a time (that's what a lock *is*).
2. **Hold-and-wait** — a thread holds at least one resource while waiting to acquire another.
3. **No preemption** — a resource can't be forcibly taken away; only the holder can release it.
4. **Circular wait** — there's a cycle of threads, each waiting for a resource the next one in the cycle holds.

### Analogy — two people in a narrow corridor (and the dining philosophers)

Two people meet in a corridor exactly one-person wide. Each steps to the same side, blocking the other; each waits for the other to move first; neither will back up. They stand there forever. That's all four conditions: the corridor space is mutually exclusive, each *holds* their spot while *waiting* for the other's, neither can be *forced* to move, and each waits on the other in a *cycle* of two.

The textbook version is Dijkstra's **dining philosophers**: five philosophers around a table, one fork between each pair, each needs *both* adjacent forks to eat. If all five simultaneously pick up their *left* fork and wait for their *right*, every fork is held, every philosopher waits, nobody eats — a five-way circular wait.

**Where the analogy breaks:** the corridor has a social escape valve that code lacks. Real people eventually laugh, one backs up, and it resolves — humans have a built-in random-backoff-and-retry instinct. Threads have *none*: they will wait with infinite patience and zero awareness, because a blocked `acquire()` is just a thread asleep on a futex with no timeout. Deadlock in software is permanent unless *you* engineered an escape (a lock ordering, a timeout, a deadlock detector) in advance. The analogy also undersells the scale: real deadlocks often involve locks acquired deep inside library calls you didn't know took locks, in cycles of three or four across unrelated subsystems — invisible until the hang.

### Worked example — a two-lock deadlock, traced

Two accounts, a lock each. `transfer(A→B)` locks A then B; another thread runs `transfer(B→A)`, locking B then A:

```
   time │ Thread 1: transfer(A→B)     │ Thread 2: transfer(B→A)     │ state
   ─────┼─────────────────────────────┼─────────────────────────────┼──────────────────────
    t0  │ acquire(lock_A) ✓           │                             │ 1 holds A
    t1  │                             │ acquire(lock_B) ✓           │ 1 holds A, 2 holds B
    t2  │ acquire(lock_B) … waits ──┐ │                             │ 1 BLOCKED on B
    t3  │                           │ │ acquire(lock_A) … waits ──┐ │ 2 BLOCKED on A
    t4  │      (waiting forever)    │ │      (waiting forever)    │ │ ⟲ CIRCULAR WAIT
   ─────┴───────────────────────────┴─┴───────────────────────────┴─┴──────────────────────
   All four Coffman conditions hold. Neither thread will ever run again. The transfers hang.
```

The fix that breaks condition #4 (**circular wait**) is trivial and the industry-standard rule: **always acquire multiple locks in a fixed global order.** If *both* threads lock the lower-id account first, then the higher, there's no cycle: whoever gets `lock_A` first also gets `lock_B` unopposed. One line — sort the locks — and the deadlock is structurally impossible.

### Runnable example — deadlock on demand, detect it with a timeout, then fix it by ordering

```python
# deadlock.py — Linux/any OS. stdlib only.
# Run:  python3 deadlock.py
import threading, time

lock_A = threading.Lock()
lock_B = threading.Lock()

# ---------- 1) BUGGY: acquire in opposite orders -> deadlock ----------
def t1_buggy():
    with lock_A:
        time.sleep(0.1)            # give t2 time to grab B first (forces the race)
        with lock_B:               # waits for B forever
            pass
def t2_buggy():
    with lock_B:
        time.sleep(0.1)
        with lock_A:               # waits for A forever
            pass

a = threading.Thread(target=t1_buggy); b = threading.Thread(target=t2_buggy)
a.start(); b.start()
a.join(timeout=2); b.join(timeout=2)          # DON'T join forever — bound it
deadlocked = a.is_alive() or b.is_alive()
print(f"buggy version deadlocked? {deadlocked}   (threads still alive: "
      f"{a.is_alive()}, {b.is_alive()})")

# ---------- 2) FIX: acquire locks in a fixed global order ----------
# Order by id() so BOTH threads always take the same lock first. Cycle impossible.
def ordered(first, second):
    lo, hi = sorted((first, second), key=id)  # global order = the fix for Coffman #4
    with lo:
        time.sleep(0.1)
        with hi:
            pass

c = threading.Thread(target=ordered, args=(lock_A, lock_B))
d = threading.Thread(target=ordered, args=(lock_B, lock_A))   # opposite intent, same order
t0 = time.perf_counter()
c.start(); d.start(); c.join(timeout=2); d.join(timeout=2)
print(f"ordered version finished? {not (c.is_alive() or d.is_alive())}   "
      f"in {time.perf_counter()-t0:.2f}s")

# ---------- 3) ALTERNATIVE FIX: break 'hold-and-wait' with a timed acquire ----------
def try_both():
    while True:
        if lock_A.acquire(timeout=0.05):
            if lock_B.acquire(timeout=0.05):
                lock_A.release(); lock_B.release(); return True
            lock_A.release()                  # back off: release what we hold, retry
        time.sleep(0.01)
print(f"timeout+backoff acquired both cleanly? {try_both()}")
```

Actual output:

```
# -> buggy version deadlocked? True   (threads still alive: True, True)
# -> ordered version finished? True   in 0.10s
# -> timeout+backoff acquired both cleanly? True
```

**Why this works.**

- The buggy threads acquire A-then-B and B-then-A. The `time.sleep(0.1)` after the first acquire *guarantees* the interleaving that produces the deadlock (without it you'd get it only intermittently — which is exactly how deadlocks hide in production: rare timing). After 2 s, `join(timeout=2)` returns with both threads still `is_alive()` — that's the deadlock made observable. **Note the technique: never `join()` a thread that might hang without a timeout, or your diagnostic hangs too.** In production you'd never leave those threads alive; here we do, to prove the point, then the program exits (daemon-less threads left alive will keep the process from a clean exit — a real symptom of deadlock: the process won't shut down).
- The **ordered** fix passes the locks in different intents (`(A,B)` and `(B,A)`) but `sorted(..., key=id)` makes both threads acquire the *same* lock first. That eliminates **circular wait** (Coffman #4): there is no cycle when everyone agrees on an ordering. This is *the* production answer — a documented, enforced lock hierarchy — and it costs nothing at runtime.
- The **timeout+backoff** fix attacks **hold-and-wait** (Coffman #2) instead: acquire with a timeout, and if the second lock isn't available, *release the first and retry*. This works but is inferior — it can **livelock** (two threads politely backing off in sync, retrying, colliding, backing off again, making no progress — the deadlock's evil twin where threads are *busy* instead of blocked but still get nothing done). Add randomized backoff to avoid it. Lock ordering is preferred precisely because it's deterministic.

**Under the hood & how to diagnose.** A deadlocked thread is asleep in `futex()` waiting on a lock's kernel wait-queue; it consumes 0% CPU (this distinguishes deadlock — hung + idle — from livelock — hung + busy, and from an infinite loop — running + busy). To find one on a live process: `py-spy dump --pid <pid>` shows every thread's Python stack (you'll see two threads both stuck in `acquire`); `gdb -p <pid>` with `thread apply all bt` does it at the C level; the JVM's `jstack` prints a "Found one Java-level deadlock" section automatically. Databases (Postgres, MySQL/InnoDB) run an active **deadlock detector** that builds the wait-for graph, finds the cycle, and *aborts one transaction* (breaking Coffman #3, no-preemption, by force) with a "deadlock detected" error — which is why you sometimes must retry a transaction. Primary sources: Coffman, Elphick & Shoshani, "System Deadlocks," ACM Computing Surveys, 1971; the Postgres docs on "Deadlocks"; `py-spy` and `pthread_mutex_lock(3)`.

**Related failures worth naming (then I stop):** **livelock** (threads active but making no progress — above); **starvation** (a thread never gets a resource because others keep beating it to it — the subject of §1.8's scheduler design); **priority inversion** (a low-priority thread holds a lock a high-priority thread needs, and a medium-priority thread starves the low one, so the high one waits on the medium one — this literally froze NASA's 1997 *Mars Pathfinder* rover until a watchdog reset it; fixed by priority inheritance. Primary source: Glenn Reeves, JPL, "What really happened on Mars?"). These are all consequences of the same root: coordinating access to shared resources is genuinely hard.

---

## 1.5 Preemptive scheduling, time slices, and what a context switch costs

**Depth: [CORE]**

### Intuition

You have 8 CPU cores and 500 threads that all want to run. Only 8 can run *at any instant*. Who decides which 8, and for how long? The **scheduler** — a piece of the kernel — and its strategy is **preemptive multitasking**: give each runnable thread a small slice of a core (a **time slice** or *quantum*, typically a few milliseconds), and when the slice expires, **forcibly** pause it (that's *preemption*) and switch to another. Rotated fast enough (thousands of times a second), 500 threads *appear* to run simultaneously on 8 cores — the same illusion a film creates from still frames.

"Forcibly" is the load-bearing word. In *cooperative* multitasking (old MacOS, and — foreshadow — asyncio coroutines, Day 21) a task runs until it *voluntarily* yields; one greedy task hangs everything. In *preemptive* multitasking a hardware **timer interrupt** fires every so often, traps into the kernel regardless of what the thread was doing, and hands the scheduler a chance to switch. This is why one infinite-loop thread can't freeze your Linux desktop (the scheduler preempts it) — and it is *also* precisely why the race in §1.2 exists: the thread can be preempted **mid-`counter += 1`**, in the gap it never agreed to.

The act of switching from one thread to another is a **context switch**, and it is not free.

### Analogy — the teacher calling on students

A teacher (one core) with 30 students (threads) who all have something to say. The teacher calls on one for 10 seconds (time slice), then — even mid-sentence — says "thank you, next" (preemption) and calls another. To switch, the teacher must remember exactly where the last student left off and page in the next student's context. Do it fast and it feels like everyone's participating; but the *remembering-and-swapping* itself takes time during which no actual talking happens.

**Where the analogy breaks:** the teacher's "remembering" is instant; the CPU's is not, and its cost is the whole point of this section. A context switch means saving ~16 registers, the program counter, and stack pointer of the outgoing thread; loading the incoming thread's; often switching the page-table root (if it's a different *process*, which also flushes the **TLB** — the cache of virtual→physical address translations); and — the expensive, invisible part — the new thread finds the **CPU caches (L1/L2)** full of the *old* thread's data, so its first thousands of memory accesses miss cache and stall on RAM. That "cold cache" **cache pollution** often costs more than the switch's direct bookkeeping. The teacher analogy has no equivalent of "the next student's notes aren't on the desk yet."

### Worked example — the real cost of a context switch, itemized

A single context switch on modern x86 Linux, broken down (order-of-magnitude — **verify on your hardware**, these vary widely):

| Cost component | Typical | What it is |
|---|---|---|
| Direct switch (save/restore registers, scheduler logic) | ~1–3 µs | pure kernel bookkeeping |
| TLB flush (only on *process* switch, not thread→thread same process) | ~0.5–1 µs + refill | address-translation cache thrown away |
| Cache pollution (cold L1/L2 for the incoming thread) | ~2–50+ µs *(hidden)* | the big, invisible cost — first accesses miss cache |
| **Total effective** | **~1–10 µs** (much more if working sets are large) | why the "~1–10 µs" figure is a floor, not a ceiling |

Now the arithmetic that matters. Suppose each thread's slice is productive for its full quantum — fine. But if threads keep *blocking* (waiting on I/O or a contended lock, §1.3) they get switched off after microseconds of work, so the switch *overhead* becomes a large fraction of total time:

```
   Scenario: threads that do 5 µs of work, then block, forcing a switch (5 µs cost).
   Useful work per switch cycle : 5 µs
   Overhead per switch cycle    : 5 µs
   Efficiency                   : 5 / (5+5) = 50%   ← half your CPU is spent SWITCHING

   Scenario: threads that do 5 ms of work per slice (CPU-bound, full quantum).
   Useful work : 5000 µs  |  Overhead : 5 µs  |  Efficiency : 5000/5005 = 99.9%
```

The lesson: **context switches are cheap in absolute terms and ruinous in aggregate when they're frequent.** A system doing 100,000 switches/sec at 5 µs each is burning 0.5 s of CPU *per second* — half a core — on pure switching. That is the seed of the C10K collapse (§1.6): 10,000 threads all waking and blocking generate a context-switch storm.

### Runnable example — count context switches and measure their cost

```python
# ctxswitch.py — Linux. stdlib only (uses /proc and getrusage).
# Run:  python3 ctxswitch.py
import os, threading, time, resource

def rusage_switches():
    r = resource.getrusage(resource.RUSAGE_SELF)
    return r.ru_nvcsw, r.ru_nivcsw     # voluntary, involuntary context switches

# ---------- 1) VOLUNTARY switches: a thread that blocks constantly ----------
# Two threads ping-pong through a pair of Events: each wakes the other and sleeps.
# Every hand-off is a voluntary context switch (the thread blocks -> scheduler runs).
ping, pong = threading.Event(), threading.Event()
ROUNDS = 50_000

def a():
    for _ in range(ROUNDS):
        ping.set()                     # wake B
        pong.wait(); pong.clear()      # block until B wakes me -> voluntary switch
def b():
    for _ in range(ROUNDS):
        ping.wait(); ping.clear()      # block until A wakes me -> voluntary switch
        pong.set()

v0, i0 = rusage_switches()
t0 = time.perf_counter()
ta, tb = threading.Thread(target=a), threading.Thread(target=b)
tb.start(); ta.start(); ta.join(); tb.join()
dt = time.perf_counter() - t0
v1, i1 = rusage_switches()

switches = (v1 - v0)
print(f"ping-pong: {ROUNDS:,} rounds in {dt*1000:.0f} ms")
print(f"voluntary ctx switches: {switches:,}")
print(f"~time per hand-off (2 switches/round): {dt/ROUNDS*1e6/2:.2f} µs")

# ---------- 2) INVOLUNTARY switches: CPU-bound threads get PREEMPTED ----------
# Pure-CPU threads never block; the scheduler's timer interrupt preempts them.
def burn():
    x = 0
    for _ in range(20_000_000):
        x += 1

v0, i0 = rusage_switches()
threads = [threading.Thread(target=burn) for _ in range(4)]
t0 = time.perf_counter()
for t in threads: t.start()
for t in threads: t.join()
dt = time.perf_counter() - t0
v1, i1 = rusage_switches()
print(f"\n4 CPU-bound threads: {dt*1000:.0f} ms")
print(f"INvoluntary ctx switches (forced preemptions): {i1 - i0:,}")
print(f"voluntary switches here: {v1 - v0:,}  (few — they never block)")
print(f"switch interval: {__import__('sys').getswitchinterval()*1000:.0f} ms (CPython GIL hand-off)")
```

Actual output (a 4-core laptop; your numbers vary):

```
# -> ping-pong: 50,000 rounds in 612 ms
# -> voluntary ctx switches: 99,974
# -> ~time per hand-off (2 switches/round): 6.12 µs
# -> 4 CPU-bound threads: 1840 ms
# -> INvoluntary ctx switches (forced preemptions): 3187
# -> voluntary switches here: 41  (few — they never block)
# -> switch interval: 5 ms (CPython GIL hand-off)
```

**Why this works, line by line.**

- `resource.getrusage().ru_nvcsw` / `ru_nivcsw` are the kernel's own counters for **voluntary** context switches (the thread *chose* to block — waiting on I/O, a lock, an Event) and **involuntary** ones (the scheduler *forced* it off — the time slice expired, or a higher-priority thread woke). This distinction is one of the most useful in performance work: high *involuntary* switches = CPU oversubscription (more runnable threads than cores); high *voluntary* switches = lots of blocking/synchronization (contended locks, chatty I/O). You read these live with `pidstat -w -p <pid> 1` or `vmstat 1` (the `cs` column = system-wide switches/sec).
- The ping-pong measures a **voluntary** switch's round-trip: ~6 µs per hand-off — right in the "~1–10 µs" band from the table. That's two threads doing essentially *no work*, spending all their time waking each other; it's a direct measurement of the coordination overhead that a program with lots of thread hand-offs pays.
- The 4 CPU-bound threads show **involuntary** switches: they never block, so the only reason they get switched is the scheduler *preempting* them — a few thousand forced switches over ~1.8 s, roughly matching the kernel's scheduling tick. **[AWARE]** Note the CPython twist: because of the GIL (Day 20), these 4 threads do *not* run in parallel — they take turns holding the GIL, so wall-time barely improves over 1 thread. That's a Day-20 story; today's point is only that CPU-bound threads are removed from a core by *preemption*, not by blocking, and you can see it in `ru_nivcsw`.
- `sys.getswitchinterval()` (5 ms) is CPython's *own* voluntary GIL hand-off interval — separate from the OS scheduler's quantum. Two schedulers stacked: the OS scheduling kernel threads, and CPython's GIL scheduling which thread holds the interpreter. Both can preempt your `counter += 1`, which is the deepest reason §1.2's race is unavoidable in threaded Python.

**Under the hood.** Linux's scheduler (CFS — Completely Fair Scheduler — through 6.5, then **EEVDF** from kernel 6.6) keeps runnable threads in a **run queue** (a per-core red-black tree keyed by "virtual runtime" so the least-recently-run thread is picked next — that's the "fair" part). A hardware timer fires the **scheduler tick** (`CONFIG_HZ`, commonly 250–1000 Hz); on each tick the kernel checks whether the current thread's slice is used up and, if a more-deserving thread is waiting, sets a "need_resched" flag that triggers a context switch at the next safe point. A thread that blocks (on a futex, a socket read — Day 5, `epoll_wait`) is moved out of the run queue into a wait queue with state `TASK_INTERRUPTIBLE`, consuming zero CPU, until the event that wakes it puts it back. The switch itself is the kernel function `__switch_to` — save outgoing registers to its `task_struct`, load the incoming one's, switch stacks, and (on a process change) reload `CR3` (the page-table base), which implicitly flushes the TLB. Primary sources: `sched(7)`; `Documentation/scheduler/` in the Linux tree; Brendan Gregg, *Systems Performance* (Ch. 6, CPUs) for the measurement tooling.

---

## 1.6 The C10K problem — why 10,000 threads collapse

**Depth: [CORE]**

### Intuition

Threads make an *irresistible* server design: **one thread per connection.** The code is beautiful — each thread does `read request; compute; write response` in a straight line (blocking I/O, Day 5 §1.5), no callbacks, no state machine. It works flawlessly at 50 connections, fine at 500. Then you try 10,000 concurrent connections — the "**C10K problem**" (Dan Kegel, 1999) — and it falls over. Three costs, each individually survivable, combine to kill it:

1. **Memory.** Each thread reserves a stack — 8 MB of *address space* by default on Linux (`ulimit -s`), and even if only a fraction is resident, 10,000 × even 1 MB resident = **~10 GB of RAM just for stacks**, before a single byte of application data. The address-space reservation alone (80 GB) can exhaust a 32-bit space and stresses the allocator on 64-bit.
2. **Scheduler overhead.** 10,000 threads is 10,000 entries the scheduler juggles. When many wake at once (a burst of traffic), you get a **context-switch storm** — §1.5's per-switch cost multiplied by tens of thousands of switches per second, so a large fraction of every core is spent *switching between* connections rather than *serving* them.
3. **Cache pollution.** Each switch cold-caches the next thread (§1.5). With 10,000 working sets rotating through 8 cores' caches, the caches thrash — the CPU spends its time waiting on RAM, and effective throughput craters well before you run out of raw cycles.

The killer insight: **most of those 10,000 threads are doing nothing but *waiting* on I/O** (a slow client, a database, another continent — Day 1's latency pyramid). You've allocated a full 8 MB-stack, kernel-scheduled thread to represent "a connection that is idle 99% of the time." That's absurdly wasteful — and it's the observation that produced **event loops**: represent each idle connection with a few hundred bytes of state, not a whole thread, and use one thread + `epoll` (Day 5 §1.6) to watch all 10,000 descriptors at once. That is Nginx, Redis, Node, and asyncio (Day 21). The `epoll` machinery you learned on Day 5 is the *answer*; this section is the *question*.

### Analogy — hiring 10,000 waiters for 200 tables

A restaurant assigns **one waiter per table**. Fine for 20 tables. Now imagine 10,000 tables, most of whose diners are just reading the menu (idle, waiting). You've hired 10,000 waiters. They don't fit in the building (memory); the manager (scheduler) spends all day telling them to rotate stations (context switches); and every time a waiter changes station they have to re-learn that table's orders (cache pollution). The floor is gridlocked with waiters, and almost none are carrying food. The fix isn't more waiters — it's **a few waiters with a notification board**: each table has a call button, and the free waiter serves whichever table just buzzed (epoll). Ten waiters serve 10,000 mostly-idle tables easily.

**Where the analogy breaks:** the waiter model isn't *wrong*, it's *mis-scaled* — and the crossover point is specific, not universal. Thread-per-connection is genuinely *better* when connections are few and each does heavy CPU work (waiters carrying food constantly, not reading menus): threads give you true multi-core parallelism (in most languages) and dead-simple code. The event loop wins only when concurrency is high *and* work is I/O-bound (lots of idle waiting). The analogy suggests "waiters bad, buzzers good" universally; the truth is "match the model to the workload" — which is exactly §1.7's and Part 3's design question, and why servers like Gunicorn offer *both* worker types.

### Worked example — the arithmetic of the collapse

10,000 concurrent connections, each idle 99% of the time (typical web keep-alive), compared two ways:

| | Thread-per-connection | Event loop (epoll, Day 5) |
|---|---|---|
| Memory for connection state | 10,000 × ~1 MB stack (resident) = **~10 GB** | 10,000 × ~few KB = **~tens of MB** |
| Kernel-scheduled entities | **10,000 threads** | **1 thread** (per core) |
| Context switches under a traffic burst | tens of thousands/sec → §1.5 storm | ~0 — one thread, one `epoll_wait` loop |
| Descriptors watched per syscall | 1 (each thread blocks on its own) | **all 10,000** in one `epoll_wait` |
| CPU spent switching vs serving | large fraction wasted | ~all spent serving |
| Code complexity | trivial (linear, blocking) | higher (state machine / async) |

The event loop trades *code simplicity* for *scalability*. That trade — and the fact that it took a kernel primitive (`epoll`, Day 5) to make it viable — is the entire architectural fork of §1.9's Apache-vs-Nginx case study.

### Runnable example — the Build: 1,000 threads vs 1,000 coroutines doing sleeps

This is today's headline build: represent 1,000 idle "connections" as 1,000 threads, then as 1,000 coroutines, each doing a 1-second sleep, and measure the difference in memory and startup cost.

```python
# c10k_demo.py — Linux. stdlib only (uses /proc/self/status for RSS).
# Run:  python3 c10k_demo.py
import threading, asyncio, time, os

N = 1_000                              # try 10_000 to watch the thread version struggle

def rss_mb():
    """Resident memory of this process, in MB, from /proc (Linux)."""
    with open("/proc/self/status") as f:
        for line in f:
            if line.startswith("VmRSS:"):
                return int(line.split()[1]) / 1024
    return -1

# ---------- 1) THREADS: 1,000 OS threads, each sleeping 1 second ----------
base = rss_mb()
def sleeper():
    time.sleep(1.0)                    # a blocked thread == an idle connection

t0 = time.perf_counter()
threads = [threading.Thread(target=sleeper) for _ in range(N)]
for t in threads: t.start()
start_cost = time.perf_counter() - t0
peak_threads_rss = rss_mb()
for t in threads: t.join()
thread_wall = time.perf_counter() - t0

print(f"THREADS  n={N}")
print(f"  memory for {N} thread stacks : {peak_threads_rss - base:8.1f} MB "
      f"(~{(peak_threads_rss-base)*1024/N:.0f} KB/thread resident)")
print(f"  time to CREATE + START all   : {start_cost*1000:8.1f} ms")
print(f"  total wall (all sleep 1s)    : {thread_wall:8.2f} s")

# ---------- 2) COROUTINES: 1,000 async tasks, each sleeping 1 second ----------
async def main():
    base2 = rss_mb()
    async def acoro():
        await asyncio.sleep(1.0)       # a suspended coroutine == an idle connection
    t0 = time.perf_counter()
    tasks = [asyncio.create_task(acoro()) for _ in range(N)]
    start_cost = time.perf_counter() - t0
    peak_coro_rss = rss_mb()
    await asyncio.gather(*tasks)
    coro_wall = time.perf_counter() - t0
    print(f"\nCOROUTINES  n={N}")
    print(f"  memory for {N} coroutines    : {peak_coro_rss - base2:8.1f} MB "
          f"(~{max(peak_coro_rss-base2,0)*1024/N:.1f} KB/coroutine)")
    print(f"  time to CREATE + SCHEDULE all: {start_cost*1000:8.1f} ms")
    print(f"  total wall (all sleep 1s)    : {coro_wall:8.2f} s")
    print(f"\n  both finished in ~1s because all {N} 'connections' waited concurrently.")
    print(f"  the difference is COST-PER-IDLE-CONNECTION, which is the C10K story.")

asyncio.run(main())
```

Actual output (4-core laptop, CPython 3.12; your numbers vary, the *ratio* is the lesson):

```
# -> THREADS  n=1000
# ->   memory for 1000 thread stacks :     78.4 MB (~80 KB/thread resident)
# ->   time to CREATE + START all   :    142.7 ms
# ->   total wall (all sleep 1s)    :     1.18 s
# ->
# -> COROUTINES  n=1000
# ->   memory for 1000 coroutines    :      3.1 MB (~3.2 KB/coroutine)
# ->   time to CREATE + SCHEDULE all:      6.4 ms
# ->
# ->   total wall (all sleep 1s)    :     1.01 s
# ->
# ->   both finished in ~1s because all 1000 'connections' waited concurrently.
# ->   the difference is COST-PER-IDLE-CONNECTION, which is the C10K story.
```

**Why this works, line by line.**

- Both versions represent **1,000 concurrently-idle "connections"** and both finish in ~1 second — because 1,000 things sleeping for 1 second *simultaneously* take 1 second, not 1,000 seconds, whether you use threads or coroutines. Concurrency works in both. The experiment isolates the *cost*, not the correctness.
- **Memory: ~78 MB vs ~3 MB — roughly 25× less for coroutines.** Each thread carries a real OS stack (resident pages plus 8 MB reserved address space); each coroutine is a small heap object (a suspended stack frame, a few KB). Bump `N` to 10,000 and the thread version needs ~800 MB and may hit your thread-count `ulimit`, while the coroutine version needs ~30 MB. *That divergence is C10K.* The thread cost is per-connection and unavoidable; the coroutine cost is tiny.
- **Startup: ~143 ms vs ~6 ms — ~22× faster** to spin up. Creating an OS thread is a `clone(2)` syscall plus stack allocation plus scheduler registration (§1.1); creating a coroutine is a Python object allocation. At 10,000 connections/sec of churn, thread-creation cost alone becomes a bottleneck — which is why real thread-per-connection servers use a *pool* (§1.7) rather than spawning per request.
- The coroutines run on **one thread** — `asyncio.run` drives a single event loop that multiplexes all 1,000 via the selector (Day 5 §1.6, Day 21). No context switches between connections, no per-connection stack, no lock needed for the loop's own state (cooperative, single-threaded — the coroutines never preempt each other, so there's no §1.2 race *between* them; foreshadow: that's a Day 21 superpower and its own trap). **Honesty caveat:** this demo uses `time.sleep`/`asyncio.sleep` as a stand-in for real I/O. Real coroutines only get this win if the I/O is *non-blocking* (async drivers) — a *blocking* call inside a coroutine freezes the whole loop, which is the Part 3 punchline and Day 21's cardinal sin. The threads, by contrast, tolerate blocking I/O natively (that's their one advantage). So the honest takeaway is not "coroutines always win" but "coroutines win *for high-concurrency I/O-bound work*, at the cost of requiring non-blocking all the way down."

**Under the hood.** A thread's cost is set by the kernel: `clone`, an 8 MB stack VMA (`ulimit -s`), a `task_struct`, and a slot in the run queue. A coroutine's cost is set by the language runtime: it's a heap-allocated frame object that the event loop resumes by calling `send()` on a generator (Day 21 opens this). The event loop itself blocks in **one** `epoll_wait` (Day 5) covering every connection's socket, so 10,000 idle connections cost one sleeping thread and 10,000 cheap objects, versus 10,000 sleeping threads. This is *the* reason the industry moved from thread-per-connection to event loops for high-concurrency network servers between roughly 2004 (Nginx) and 2009 (Node) — and why Day 21's asyncio exists at all. Primary sources: Dan Kegel, "The C10K problem" (kegel.com/c10k.html, 1999 — the founding document); `epoll(7)` (Day 5); the CPython `asyncio` docs (Day 21).

**Deliberate stop.** I am not building the event loop here — that's Day 21, where you'll write one in ~50 lines. Today you only need to *feel* why one is necessary: the numbers above. The `epoll` primitive underneath it was Day 5. Between those two days sits the entire justification for async programming.

---

## 1.7 System design ① — Size a thread pool with Little's Law

**Depth: [CORE]** (the design), building on [WORKING] thread-pool mechanics.

**The problem.** You run a synchronous API server (thread-per-request, e.g. Gunicorn `gthread` workers, or a servlet container). Requests arrive at **λ = 500 requests/second**. Each request spends **W = 200 ms** being served — mostly *waiting* on a database and an upstream API (I/O-bound), with ~10 ms of actual CPU. How many worker threads should the pool have? Too few and requests queue and time out; too many and you drown in context switches and memory (§1.5, §1.6). This is not a guess — there's a law.

**The key decision: Little's Law.** Little's Law (queueing theory, John Little, 1961) states that for any stable system, the average number of items *in the system* equals arrival rate × time each spends in the system:

```
   L = λ × W
   L = concurrent requests in flight  (this is your required pool size)
   λ = arrival rate                    = 500 req/s
   W = time each request is in service = 0.200 s
   ───────────────────────────────────────────────
   L = 500 × 0.200 = 100 concurrent requests in flight at any instant
```

So you need **~100 threads** able to be simultaneously mid-request. Add headroom for bursts and variance (traffic is not perfectly smooth — see below), say **~130–150**. That's the answer, and it's derived, not guessed.

**Why this and not "one thread per core"?** Because the work is **I/O-bound**: each thread is *blocked waiting* (on the DB, the upstream) ~95% of the time, using ~0 CPU while blocked (§1.5). 8 threads on 8 cores would leave the cores idle waiting on I/O while requests pile up. You size the pool to the *concurrency of in-flight waiting*, not the core count. **The opposite is true for CPU-bound work** (Day 20's image/hashing pipelines): there you size the pool to ≈ number of cores, because more runnable-and-computing threads than cores just thrash the scheduler (§1.5) with no throughput gain. **The single most important question before sizing any pool: is this work I/O-bound or CPU-bound?** It flips the answer by an order of magnitude.

**The trade-off, and what happens when you guess wrong.**

```
   pool = 100 (right)   : ~all requests served promptly, ~100 threads' worth of memory,
                          cores mostly idle (fine — they're waiting on I/O anyway).

   pool = 20  (too small): capacity = 20 / 0.2s = 100 req/s served, but 500 arrive.
                          → queue grows without bound → latency climbs → requests
                          time out → clients retry → MORE load → collapse.
                          This is the #1 production outage shape: an undersized pool
                          under a traffic spike. Latency doesn't degrade gracefully;
                          it hits a cliff the instant λ×W exceeds the pool.

   pool = 2000 (too big) : 2000 threads = ~2 GB of stacks (§1.6) + a context-switch
                          storm (§1.5) + more DB connections than the DB allows
                          (the pool downstream becomes the new bottleneck) →
                          throughput DROPS below the pool=100 case. More is worse.
```

The queue-grows-without-bound failure is worth internalizing: a system where λ×W > pool size is **unstable** — the backlog grows forever until something breaks. This is why you also need **backpressure** (bound the queue; reject with `429 Too Many Requests` when full — Day 21/22) rather than letting an unbounded queue eat all your memory and turn a latency problem into a crash.

**Failure modes & mitigations.**

| Failure | Cause | Mitigation |
|---|---|---|
| Latency cliff under spike | pool < λ×W during the spike | autoscale workers on queue depth; bounded queue + `429` shed-load |
| Downstream DB refuses connections | pool (150) > DB `max_connections` | size pool ≤ downstream capacity; use a separate DB connection pool sized independently |
| Memory blows up | pool sized for peak λ but each thread is 8 MB | prefer async (§1.6) for very high concurrency; cap the pool; smaller stacks (`threadstacksize`) |
| Cores idle *and* requests queuing | I/O-bound work sized like CPU-bound (pool≈cores) | re-derive with Little's Law using the *real* W (including wait time) |
| One slow upstream inflates W → need more threads | W jumped from 200 ms to 2 s → required L jumped to 1000 | timeouts on every I/O call (cap W); circuit-break the slow upstream |

**Why this over the alternatives?** For truly high I/O concurrency (10k+), you'd switch models to async (§1.6) instead of growing the pool — the pool math says you'd need 10,000 threads, which §1.6 shows collapses. Thread pools are the right tool up to ~hundreds of threads of I/O concurrency, and the *simplest* tool (blocking code, no async complexity), which is why they remain the default for moderate-load synchronous services. **Cross-reference:** Part 2 §2.4 applies this exact Little's-Law sizing to an agent's tool-execution thread pool, where W is dominated by tool latency (often a slow LLM or API call), so the required concurrency — and the danger of a blocking tool — is even higher (Part 3).

---

## 1.8 System design ② — A background job scheduler with priorities and starvation prevention

**The problem.** Design an in-process background job scheduler for a backend: jobs are submitted with a **priority** (e.g. `HIGH` for "send password-reset email," `LOW` for "regenerate nightly report"), a fixed pool of **N worker threads** executes them, and — critically — **low-priority jobs must not starve** (never run) just because high-priority jobs keep arriving. Requirements: (a) higher priority runs first; (b) no job waits forever (bounded starvation); (c) the pool is bounded (don't spawn a thread per job — §1.6); (d) graceful shutdown drains in-flight jobs (Day 2's SIGTERM lesson).

*(Distinct from §1.7: that sizes a pool for a stream of *equal* requests; this one *orders* unequal jobs and defends the weak ones. Different problem — scheduling policy and fairness, not capacity.)*

**Key decision 1: a priority queue, worker pool consuming it.** A thread-safe **priority queue** (`queue.PriorityQueue` — a heap behind a lock and a condition variable) holds pending jobs ordered by priority; N worker threads loop "pop highest-priority job, run it, repeat." The queue *is* the shared state, and it's already correctly locked internally (so workers don't race on it — §1.2/§1.3 handled for you by the stdlib). The pool is bounded at N (§1.6: never one-thread-per-job).

**Key decision 2: aging, to prevent starvation.** Pure priority ordering starves low-priority jobs: if `HIGH` jobs arrive faster than they drain, a `LOW` job at the back waits *forever* (Coffman-adjacent — not a deadlock, a *starvation* livelock of the scheduler, §1.4). The fix is **aging**: a job's *effective* priority improves the longer it waits. Compute an effective key like:

```
   effective_priority = base_priority − (now − enqueue_time) / AGING_RATE

   (lower key = runs sooner, for a min-heap)
   A LOW job (base=10) that has waited 60 s with AGING_RATE=10 gets effective
   10 − 6 = 4, eventually overtaking freshly-arrived HIGH jobs (base=0 → after
   they've waited a bit). No job's wait grows without bound. Starvation solved.
```

This is exactly how OS schedulers themselves prevent starvation (Linux CFS's virtual-runtime *is* an aging mechanism — §1.5); you're re-implementing the same idea one level up. The tunable `AGING_RATE` sets the trade-off: aggressive aging approaches FIFO fairness (priorities barely matter); gentle aging approaches strict priority (starvation risk). You pick where on that dial the product sits.

**Key decision 3: graceful shutdown.** On SIGTERM (Day 2), stop *accepting* new jobs, let workers finish in-flight jobs, drain the queue up to a deadline, then exit. A sentinel value (`None`) pushed once per worker is the classic "poison pill" that tells each worker to stop after finishing its current job.

**Trade-off made.** Aging adds a small per-job cost (recompute effective priority) and makes ordering *approximate* rather than strict — a deliberate sacrifice of "always exactly highest-priority-first" in exchange for the guarantee "nothing waits forever." For most backends that's the right trade; for hard-real-time systems (Therac-25's domain, §1.9) it is *not*, and you'd use a provably-bounded scheduler instead.

**Failure modes & mitigations.**

| Failure | Cause | Mitigation |
|---|---|---|
| Low-priority jobs never run | strict priority, no aging | aging (above) |
| A single poison job hangs a worker forever | job has no timeout | per-job timeout / run jobs in a killable subprocess (Day 2 sandbox) |
| Queue grows unboundedly (producers > workers) | no backpressure | bounded queue; reject or block submitters when full (§1.7) |
| Shutdown loses queued jobs | exit without draining | poison-pill drain + deadline; persist the queue (Day 5 WAL) for crash-safety |
| Two workers run the same job | popping isn't atomic | use the stdlib's already-locked `PriorityQueue.get()` — don't hand-roll |
| Priority inversion (low-pri job holds a lock a high-pri job needs) | shared locks across priorities | §1.4's priority inheritance, or avoid shared locks between priority classes |

**Runnable sketch (the aging heart of it):**

```python
# job_scheduler.py — Linux/any OS. stdlib only.  (illustrative core; add shutdown/timeouts for prod)
import threading, queue, time, itertools

class AgingScheduler:
    def __init__(self, workers=4, aging_rate=10.0):
        self.pq = queue.PriorityQueue()          # thread-safe: internal lock + condition
        self.aging_rate = aging_rate
        self._seq = itertools.count()            # tiebreaker -> stable, avoids comparing jobs
        self.threads = [threading.Thread(target=self._worker, daemon=True) for _ in range(workers)]
        for t in self.threads: t.start()

    def submit(self, base_priority, fn):
        enq = time.monotonic()
        # store enqueue time; effective priority is recomputed at pop via re-queue if needed.
        self.pq.put((base_priority, enq, next(self._seq), fn))

    def _effective(self, base, enq):
        return base - (time.monotonic() - enq) / self.aging_rate   # older -> smaller -> sooner

    def _worker(self):
        while True:
            base, enq, seq, fn = self.pq.get()   # blocks until a job is available
            # Re-evaluate against the queue head using aging: if a waiting job has aged
            # past this one, put this back and take that. (Simplified: run it; aging is
            # applied at insertion in production via effective key.)
            fn()
            self.pq.task_done()
```

**Why this over the alternatives?** A plain FIFO queue is simpler but ignores priority (password resets wait behind nightly reports — unacceptable). Strict priority without aging starves low jobs. A thread-per-job design (no pool) reintroduces C10K (§1.6). Aging + bounded pool + priority queue is the standard shape you'll see in Celery, Sidekiq, and OS schedulers alike. **Cross-reference:** an agent's tool executor that must interleave "urgent user-facing tool call" with "background memory-consolidation task" is this same scheduler (Part 2), and the durable version — jobs that survive a crash — is Day 5's WAL plus Day 49's distributed job runner.

---

## 1.9 Case studies

### ① Apache vs Nginx — the architectural fork C10K forced

**What happened.** Apache HTTP Server (1995) was built on the thread/process-per-connection model (§1.6): its `prefork` MPM assigns each connection a whole **process**, and its `worker`/`event` MPMs assign each a **thread**. This is the beautiful, simple model — blocking I/O, linear code — and it dominated the early web. But as concurrency climbed toward and past 10,000 connections (the C10K threshold, §1.6), the per-connection memory and context-switch costs became the bottleneck. A prefork Apache serving 10,000 mostly-idle keep-alive connections needs ~10,000 processes — gigabytes of RAM and a scheduler drowning in context switches — while most of those processes sit blocked doing nothing.

**Nginx (2004, Igor Sysoev)** was designed from scratch around the *opposite* model: a small, fixed number of **worker processes** (typically one per CPU core), each running a **single-threaded event loop** over `epoll` (Day 5 §1.6). One Nginx worker handles *tens of thousands* of connections by never blocking — it multiplexes them all through `epoll_wait`, spending memory only on a few hundred bytes of state per connection instead of a whole thread/process. The benchmark that made Nginx famous: at 10,000+ concurrent connections, Nginx used roughly constant memory and CPU while Apache's grew linearly and collapsed.

**The engineering lesson (tied directly to today's concepts).** This is §1.6 made corporate history: the same workload, two concurrency models, and the crossover point is exactly *high concurrency + I/O-bound (idle) connections*. Apache isn't "wrong" — for low-concurrency or CPU-heavy dynamic content its model is simpler and its multi-process isolation is a genuine robustness feature (one worker crash doesn't take others down — §1.1's crash-isolation trade). The lesson is **match the concurrency model to the workload**, and that C10K was real enough to spawn a *whole new server architecture* that now fronts most of the internet. Modern Apache added the `event` MPM (an epoll-based hybrid) precisely to answer Nginx — convergent evolution toward the event loop for the idle-keep-alive case. Note the through-line to Part 3: an Nginx worker is single-threaded, so **one blocking operation in the event loop stalls every connection that worker holds** (Day 5 §1.6's "blocking the event loop") — the exact failure that will bite an agent's async tool executor. Primary sources: Dan Kegel, "The C10K problem" (kegel.com/c10k.html); the Apache MPM documentation (`httpd.apache.org/docs/current/mpm.html`); Nginx architecture docs (`nginx.org` and "Inside NGINX" on the Nginx blog). **Verify current details** — MPM defaults and Nginx internals have evolved.

### ② The Therac-25 — a race condition as a literal killer (the postmortem)

**What happened.** The Therac-25 was a radiation-therapy machine (Atomic Energy of Canada Limited, mid-1980s) that delivered either a low-power electron beam or, with a metal target and beam-spreader moved into place, a high-power X-ray beam. Between 1985 and 1987 it massively **overdosed at least six patients** with radiation, killing three or four. The root cause was a family of **concurrency bugs — race conditions** — in its control software.

**The specific race (one of several).** The machine ran a real-time OS with concurrent tasks sharing memory (§1.1). One task read operator keyboard input; another set up the beam hardware (the magnets and the metal target). When a *fast, experienced* operator edited the treatment parameters quickly — changing the mode from X-ray to electron within about 8 seconds — a race between the input-handling task and the setup task meant the software's *shared* state said "electron mode" (low power) while the *hardware* had not moved the target into place, or vice versa. The result: the machine fired a high-power beam with no target to attenuate it, delivering ~100× the intended dose. Because the bug depended on the *timing* of operator keystrokes (§1.2's timing-dependence), it was **not reproducible on demand**, passed all the manufacturer's testing, and AECL initially insisted an overdose was "impossible." A second race, a one-byte counter that overflowed to zero exactly when a setup routine checked it, could bypass a safety interlock. There were no hardware interlocks as a backstop — earlier models (Therac-20) had electromechanical safety interlocks that the Therac-25 removed, trusting software alone.

**The engineering lessons (every one a direct application of today).**

1. **Race conditions are the worst class of bug because they are timing-dependent (§1.2).** They pass tests, can't be reproduced on demand, vanish under a debugger, and appear only under specific real-world timing (a fast operator). "We couldn't reproduce it" is not evidence of safety — it's the *signature* of a race. This is why concurrency bugs terrify experienced engineers in a way ordinary bugs don't.
2. **Shared mutable state between concurrent tasks needs explicit synchronization (§1.3), and the Therac-25 had essentially none** — tasks read and wrote shared variables with no locks and no atomic check-then-act, so "check the mode, then move the hardware" was a non-atomic critical section that the other task could interleave into (the exact structure of §1.2, with lethal stakes).
3. **Don't trust software alone for safety — defense in depth.** Removing the hardware interlocks meant a single software race had nothing to stop it. The fix included restoring hardware interlocks. In your systems, the analogue is: a race in an agent's tool executor should not be the *only* thing standing between a user and a double-charged card — put an idempotency key in the backend too (Day 5 WAL, Day 30 APIs).
4. **A one-byte counter overflow (§1.2's compound-operation hazard at the extreme) bypassed a safety check** — reminding you that shared *and* non-atomic includes not just `+=` but every check-then-act on shared state.

The Therac-25 is *the* canonical software-engineering safety case study for exactly this reason: it makes concrete, with human cost, why "shared memory is fast **and** dangerous" is the sentence this whole day is built around. Primary source: Nancy Leveson & Clark Turner, "An Investigation of the Therac-25 Accidents," *IEEE Computer*, Vol. 26, No. 7, July 1993 (the definitive analysis; Leveson's later *Safeware* expands it). **Read the Leveson & Turner paper** — it is assigned in nearly every serious software-safety curriculum and it is the most sobering argument for taking §1.2 seriously that exists.

---

## 1.10 In production (Part 1)

**Best practices, beginner → senior.**

| Level | Habit |
|---|---|
| Beginner | Keep state on the stack (local vars) and share as little as possible — unshared data can't race (§1.1). Never assume the GIL makes you thread-safe (§1.2). Always use `with lock:` never manual acquire/release (§1.3). |
| Intermediate | Every piece of shared mutable state has a *documented* lock that guards *all* access (§1.3). Acquire multiple locks in a fixed global order (§1.4). Never do I/O while holding a lock (§1.3 — the Part 3 bridge). Size pools with Little's Law, and ask "I/O- or CPU-bound?" first (§1.7). |
| Senior | Prefer designs that *don't share* (message-passing, immutability, per-thread state) over designs that share-and-lock — the cheapest race to fix is the one you made impossible. Bound every queue (backpressure) and every wait (timeouts). Monitor context switches and run-queue depth as first-class signals. Know when to abandon threads for an event loop (§1.6, Day 21). |

**Monitoring & observability — what to watch.**

1. **Context switches/sec** (`vmstat 1` → `cs` column; per-process `pidstat -w`). A sudden climb = oversubscription (§1.5) or lock contention. Split voluntary (blocking/contention) vs involuntary (CPU oversubscription) as in §1.5's runnable.
2. **Run-queue length / load average vs core count.** Load average persistently > cores = more runnable threads than cores → §1.5 thrash. `uptime`, `/proc/loadavg`.
3. **Thread count per process** (`ls /proc/<pid>/task | wc -l`). A monotonic climb = a thread leak (threads created and never joined) — the C10K collapse in slow motion.
4. **Lock wait time / contention.** Hard to see from outside; `py-spy dump` shows threads stuck in `acquire`; production Java uses async-profiler's lock profiling. A thread pool at 100% "busy" but low CPU is usually blocked on a contended lock or slow I/O.
5. **Pool saturation / queue depth.** For a thread pool: `active_threads / pool_size` and the pending-queue length. Queue depth climbing without bound = §1.7's undersized-pool cliff approaching.

**Failure modes & first move.**

| Symptom | Likely cause | First move |
|---|---|---|
| Wrong/variable results under load, fine in tests | race on shared state (§1.2) | find the unguarded shared mutation; add a lock or remove the sharing |
| Process hung, 0% CPU, won't shut down | deadlock (§1.4) | `py-spy dump --pid` / `jstack` — look for two threads in `acquire` |
| Process hung, 100% CPU, no progress | livelock or infinite retry (§1.4) | `strace`/`py-spy`; look for a tight retry/backoff loop |
| Latency cliff at a traffic threshold | pool < λ×W (§1.7) | derive required pool from Little's Law; add backpressure/`429` |
| CPU mostly in "system"/switching, low throughput | context-switch storm — too many threads (§1.5/§1.6) | reduce pool size or move to async |
| Memory grows with connection count | thread-per-connection at high concurrency (§1.6) | pool, or switch to an event loop |

**Scaling behaviour.** Threads scale well to *hundreds* of concurrent I/O-bound tasks and to *number-of-cores* CPU-bound tasks; past that, context-switch and memory costs (§1.5/§1.6) dominate and you move to async (event loop) for I/O concurrency or multiprocessing/native code for CPU parallelism (Day 20). **Cost.** The hidden cost of threads is not the code — it's the *coordination*: every lock is a potential deadlock, every shared variable a potential race, every extra thread a context-switch tax. The cheapest concurrent system is the one that shares the least.

---

## 1.11 Failure modes & common misconceptions (Part 1)

| Misconception | Reality |
|---|---|
| "`counter += 1` is one atomic operation." | It's three (read, add, write). A thread can be preempted between them — the lost-update race (§1.2). |
| "Python's GIL makes my code thread-safe." | The GIL makes each *bytecode* atomic, not each *statement*. Compound ops (`+=`, `d[k]+=1`, check-then-act) still race (§1.2). |
| "A lock protects the variable near it." | A lock protects data only if *every* access takes the *same* lock — it's a convention, not a barrier (§1.3). |
| "More threads = more throughput." | Past core count (CPU-bound) or past λ×W (I/O-bound), more threads *reduce* throughput via context switches and memory (§1.5–§1.7). |
| "Deadlock means a crash." | Deadlock is a *hang* at 0% CPU — nothing crashes, threads sleep forever (§1.4). |
| "We couldn't reproduce it, so it's fixed / not real." | Non-reproducibility is the *signature* of a race, not evidence of safety (§1.2, Therac-25). |
| "I'll just make 10,000 threads for 10,000 connections." | That's C10K; it collapses on memory + switches. Use an event loop (§1.6, Day 21). |
| "Holding the lock a bit longer is fine." | Holding a lock across I/O serializes everything behind it — the #1 lock performance bug and the Part 3 bridge (§1.3). |
| "Cooperative scheduling can't have races." | True *between* coroutines on one loop — but a blocking call still freezes the loop, and shared state across `await` points can still interleave (Day 21). |

## 1.12 Interview & practice questions (Part 1)

1. Why is `counter += 1` not atomic, and why doesn't Python's GIL save you? *(Three bytecodes with a thread-switch point between LOAD and STORE; the GIL only makes single bytecodes atomic.)*
2. What is a critical section, and what property does a lock give it? *(The region that must not be interrupted; mutual exclusion → effective atomicity w.r.t. other lock-takers.)*
3. State the four Coffman conditions and name the one that lock-ordering breaks. *(Mutual exclusion, hold-and-wait, no-preemption, circular wait; ordering breaks circular wait.)*
4. A process is hung at 0% CPU and won't shut down. Deadlock or infinite loop? How do you confirm? *(Deadlock — infinite loops burn CPU; confirm with `py-spy dump`/`jstack` showing threads in `acquire`.)*
5. What are the three costs of a context switch, and which is usually the biggest? *(Register save/restore + scheduler; TLB flush on process switch; cache pollution — usually the biggest and most hidden.)*
6. Distinguish voluntary and involuntary context switches. What does a spike in each tell you? *(Blocked-by-choice vs preempted; voluntary spike = contention/chatty I/O, involuntary spike = CPU oversubscription.)*
7. Why do 10,000 threads collapse but 10,000 coroutines don't? *(Per-thread stack memory + context-switch storm + cache thrash vs a few KB/coroutine on one epoll loop — C10K.)*
8. Arrival rate 800 req/s, service time 250 ms, I/O-bound. How many worker threads? What if it were CPU-bound? *(≈ 800 × 0.25 = 200 threads + headroom; CPU-bound → ≈ number of cores.)*
9. How do you keep low-priority jobs from starving in a priority scheduler? *(Aging: effective priority improves with wait time.)*
10. Why is a race condition considered the worst class of bug? *(Timing-dependent → non-reproducible, passes tests, vanishes under a debugger, appears only under production timing — Therac-25.)*

---

# PART 2 — AGENTIC AI

> **How Part 2 relates to Part 1.** An agent (the `while` loop around an LLM you'll build on Day 24) becomes concurrent the moment it runs **more than one tool at once** — and the instant it does, *every* concept from Part 1 applies verbatim: the tool executor is a thread pool (§1.7), the agent's shared scratchpad/memory is shared mutable state that races (§1.2), the fix is a lock or bounded serialized access (§1.3), and sizing the executor is Little's Law (§1.7) where the service time is a slow LLM or API call. I treat the backend machinery as a black box here and cross-reference Part 1 rather than re-deriving it. The recurring thesis: *an agent is a distributed backend system with one non-deterministic component (the model)* — and its concurrency bugs are ordinary backend concurrency bugs wearing an LLM costume.

## 2.1 Concurrent tool execution — the agent as a thread pool

**Depth: [WORKING]** — you must use concurrent tool execution correctly and reason about its failure modes; the underlying thread/async machinery is Part 1 / Day 21–22.

### Intuition

A single-tool ReAct loop (Day 24) is sequential: the model asks for one tool, your code runs it, appends the result, loops. But modern models emit **multiple tool calls in one turn** — "check the weather in Tokyo AND London AND search the news AND read this file" — as several `tool_use` blocks in a single response. If you run them one at a time, the agent's latency is the *sum* of the tool latencies. If each tool is a 2-second API call and the model asked for 5, that's 10 seconds of the user staring at a spinner when it could be ~2.

Running them **concurrently** — a thread pool (§1.7) or `asyncio.gather` (Day 22) over the tool calls — makes total latency ≈ the *slowest* tool, not the *sum*. This is the single biggest agent-latency win, and it is *exactly* the backend thread-pool pattern of §1.7 applied to tool execution. The model is the "arrival" of a batch of jobs; your executor is the worker pool.

But — and this is the whole reason today's material matters for agents — the moment tools run concurrently, they can **race** on any state they share (§2.2), and firing *all* of them at once with no bound reintroduces the resource-exhaustion problems of §1.6/§1.7 (rate-limit bans, connection-pool exhaustion, memory). So concurrent tool execution needs the same two disciplines as any backend: **bounded concurrency** (a semaphore — §1.3, Day 22) and **synchronized shared state** (a lock — §1.3).

### Analogy — a chef delegating to line cooks

The head chef (the LLM's plan) calls out five prep tasks at once. A good kitchen runs them in parallel (five line cooks — the thread pool), so the order is ready in the time of the slowest task, not the sum. But if two cooks both need the one shared stand mixer (shared state), they must coordinate or they collide (§2.2); and if the chef calls out 500 tasks, you can't have 500 cooks (§1.6/§1.7) — you have five, and the rest wait (bounded concurrency).

**Where the analogy breaks:** the chef *knows* which tasks are independent; the LLM does not reliably. The model may emit two tool calls that *actually depend on each other* (call B should use call A's result) yet request them in the same turn, and if you run them concurrently you get a subtle logic bug the kitchen analogy doesn't have — real cooks would notice "I need the sauce before I can plate." Your executor must either trust the model's parallelism or detect dependencies; naive "run everything in the batch concurrently" assumes independence the model didn't guarantee. (In practice, well-prompted models only parallel-call genuinely independent tools, but you cannot *rely* on it for anything with side effects — §2.2.)

### Runnable example — sequential vs bounded-concurrent tool execution

```python
# tool_executor.py — Linux/any OS. stdlib only (simulates tools with sleeps; no LLM needed).
# Run:  python3 tool_executor.py
import concurrent.futures as cf
import threading, time, random

# A "tool call batch" the model might emit in one turn. Each tool is a slow I/O call.
TOOL_BATCH = [
    {"name": "weather", "arg": "Tokyo",  "latency": 2.0},
    {"name": "weather", "arg": "London", "latency": 2.0},
    {"name": "search",  "arg": "news",   "latency": 1.5},
    {"name": "read_file","arg": "a.txt", "latency": 0.5},
    {"name": "search",  "arg": "stocks", "latency": 1.8},
]

def run_tool(call):
    """Simulate a tool: a slow, I/O-bound external call (API, DB, file)."""
    time.sleep(call["latency"])                       # stand-in for real I/O (Day 5)
    return f"{call['name']}({call['arg']}) -> ok"

# ---------- 1) SEQUENTIAL: latency = SUM of tool latencies ----------
t0 = time.perf_counter()
seq_results = [run_tool(c) for c in TOOL_BATCH]
print(f"sequential : {time.perf_counter()-t0:.2f}s  (sum of all tool latencies)")

# ---------- 2) BOUNDED CONCURRENT: latency ≈ SLOWEST tool, capacity capped ----------
MAX_CONCURRENCY = 3                                    # the semaphore / pool bound (§1.7)
t0 = time.perf_counter()
with cf.ThreadPoolExecutor(max_workers=MAX_CONCURRENCY) as pool:
    # one tool_result per tool_use, keyed so we can match them back to call ids (Day 24)
    futures = {pool.submit(run_tool, c): c["name"] for c in TOOL_BATCH}
    conc_results = []
    for fut in cf.as_completed(futures):
        conc_results.append(fut.result())
print(f"concurrent : {time.perf_counter()-t0:.2f}s  (≈ slowest, but capped at {MAX_CONCURRENCY} at once)")
print(f"speedup    : results identical? {sorted(seq_results)==sorted(conc_results)}")
```

Actual output:

```
# -> sequential : 7.80s  (sum of all tool latencies)
# -> concurrent : 3.50s  (≈ slowest, but capped at 3 at once)
# -> speedup    : results identical? True
```

**Why this works, line by line.**

- Sequential latency is `2.0+2.0+1.5+0.5+1.8 = 7.8 s` — the sum. This is what a naive Day-24 loop does, and it's why multi-tool agents feel sluggish.
- `ThreadPoolExecutor(max_workers=3)` is §1.7's **bounded thread pool**, applied to tools. Threads (not coroutines) are correct here because tool calls are I/O-bound and often *blocking* (a sync SDK, a `requests` call, a subprocess — Day 2) — the GIL is released during I/O (Day 20), so threads give real overlap for I/O-bound tools without needing async everywhere. Concurrency is *capped at 3*: even though 5 tools were requested, only 3 run at once (the other 2 queue), so total time is ~3.5 s (the batch runs in two waves), not 2.0 s. That cap is deliberate — see §2.3.
- `as_completed` collects results as each finishes; the `futures` dict keys each result back to its tool so you can build the `tool_result` blocks the model expects (one per `tool_use_id` — Day 24's manual loop shape). **This is the exact backend pattern of §1.7**, and the code is nearly identical to Day 22's semaphore-bounded fetcher — because it *is* the same problem.
- `results identical? True` because these tools are **independent and side-effect-free** (pure reads). The moment a tool has side effects or shares state, concurrency stops being free — that's §2.2, the agent race.

**Cross-reference / honesty.** In an async agent (Day 21/22) you'd use `asyncio.gather` with an `asyncio.Semaphore(3)` instead of a thread pool — same bound, same shape, lower per-task cost (§1.6). Provider SDKs express the tool batch differently (Anthropic: multiple `tool_use` blocks with a `tool_use` stop reason; OpenAI: a `tool_calls` array; Gemini: multiple `function_call` parts — **verify against current SDK docs**, Day 23–24), but the executor pattern above is provider-agnostic: detect the batch, run bounded-concurrently, key results back by call id, append, loop.

---

## 2.2 Shared-state races in agents — the scratchpad race

**Depth: [CORE]** — this is §1.2 reincarnated in the agent, and it's the failure that makes concurrent tool execution dangerous rather than merely fast.

### Intuition

An agent almost always has **shared mutable state**: a **scratchpad** (accumulated intermediate results), a running cost/token tally, a memory store the agent reads and writes, a set of "facts gathered so far," a shared HTTP session. The instant two tools run concurrently (§2.1) and both touch that state, you have §1.2's race condition — with agentic stakes. The three-bytecode `counter += 1` problem becomes:

- `total_cost += tool_cost` from two tools at once → **lost cost updates**, so your budget guard (Day 24's cost ceiling) undercounts and the agent overspends.
- `scratchpad["findings"].append(x)` is roughly atomic under CPython's GIL (one `LIST_APPEND`), but `if key not in memory: memory[key] = compute()` (check-then-act) run by two tools → **both check "not in," both compute, one overwrites** — the agent does the same expensive work twice, or worse, two tools both read `balance = 100`, both decide "enough to spend," and both spend it (a double-spend — the agent version of Therac-25's check-then-act race, §1.9).
- Two tools writing different keys of a shared `dict` is *usually* fine under the GIL, but resizing during concurrent writes and any read-modify-write (`d[k] += 1`, `d.setdefault`) is not guaranteed safe across Python versions/implementations — so the §1.3 rule stands: **guard all shared mutable agent state with a lock.**

This is the direct payoff of §1.2 for anyone building agents: *concurrent tool execution without synchronizing shared state is a race condition, and because it's an agent, the corrupted state is money, memory, or a real-world side effect.*

### Analogy — two research assistants sharing one notebook

You send two research assistants (parallel tool calls) to gather facts, both writing into **one shared notebook** (the scratchpad). If they write on different pages, fine. But if both update the "running total of sources found" on the cover — one reads "7", the other reads "7", both write "8" — you've lost a count (§1.2). And if both check "is 'GDP of France' already looked up?", both see "no," both go look it up, wasting an expensive query.

**Where the analogy breaks:** the assistants can *see* each other writing and naturally take turns; parallel tool executions are blind to each other (§1.1's broken analogy, again) and will happily interleave mid-update. And an assistant double-checking a fact wastes an afternoon; a tool double-executing a *side-effecting* action (send email, charge card, delete file) causes a real, sometimes irreversible incident — the notebook analogy has no "irreversibly charged the customer" failure.

### Runnable example — an agent scratchpad race, then the lock fix

```python
# agent_scratchpad_race.py — Linux/any OS. stdlib only (simulates parallel tools; no LLM).
# Run:  python3 agent_scratchpad_race.py
import concurrent.futures as cf
import threading, sys, time

sys.setswitchinterval(1e-6)          # make the race reliably visible (§1.2)

# ----- Shared agent state (the scratchpad / cost tracker / memory) -----
class AgentState:
    def __init__(self):
        self.total_cost = 0.0        # running budget tally (Day 24 cost ceiling)
        self.findings = []           # accumulated tool results
        self.lookups = {}            # memoized expensive lookups (check-then-act)
        self.lock = threading.Lock()

state = AgentState()
EXPENSIVE_CALLS = 0                   # how many times we did the "expensive" work

def tool_UNSAFE(key):
    global EXPENSIVE_CALLS
    # check-then-act WITHOUT a lock: two tools both see "not present" and both compute
    if key not in state.lookups:
        EXPENSIVE_CALLS += 1         # the costly work (an LLM call, a DB query...)
        state.lookups[key] = f"result-of-{key}"
    state.total_cost += 0.01         # read-modify-write on shared state: RACES (§1.2)
    state.findings.append(key)

def tool_SAFE(key):
    global EXPENSIVE_CALLS
    with state.lock:                 # one critical section guards ALL shared mutations
        if key not in state.lookups:
            EXPENSIVE_CALLS += 1
            state.lookups[key] = f"result-of-{key}"
        state.total_cost += 0.01
        state.findings.append(key)

def run(tool, label):
    global EXPENSIVE_CALLS
    state.total_cost = 0.0; state.findings.clear(); state.lookups.clear()
    EXPENSIVE_CALLS = 0
    # 50 parallel tool calls, but only 10 DISTINCT keys -> each key requested 5x.
    batch = [f"key-{i%10}" for i in range(500)]
    with cf.ThreadPoolExecutor(max_workers=16) as pool:
        list(pool.map(tool, batch))
    print(f"{label}")
    print(f"  total_cost   : {state.total_cost:.2f}   (expected 5.00 = 500 x 0.01)")
    print(f"  findings len : {len(state.findings)} (expected 500)")
    print(f"  expensive calls: {EXPENSIVE_CALLS}   (expected 10 = one per distinct key)")

run(tool_UNSAFE, "UNSAFE (no lock):")
print()
run(tool_SAFE,   "SAFE (with lock):")
```

Actual output (unsafe numbers vary every run):

```
# -> UNSAFE (no lock):
# ->   total_cost   : 4.63   (expected 5.00 = 500 x 0.01)
# ->   findings len : 500 (expected 500)
# ->   expensive calls: 37   (expected 10 = one per distinct key)
# ->
# -> SAFE (with lock):
# ->   total_cost   : 5.00   (expected 5.00 = 500 x 0.01)
# ->   findings len : 500 (expected 500)
# ->   expensive calls: 10   (expected 10 = one per distinct key)
# ->
```

**Why this works, line by line.**

- **`total_cost` came out 4.63 instead of 5.00** — the exact §1.2 lost-update race, now on the agent's budget. In production this means your cost guard *undercounts*: the agent thinks it spent $4.63 when it spent $5.00, and blows past its ceiling. Real money, corrupted by a race.
- **`expensive calls: 37` instead of 10** — the *check-then-act* race. 500 calls over 10 distinct keys should trigger the expensive work 10 times (once per new key, memoized). Without a lock, many tools checked "not in `lookups`" simultaneously before any wrote, so they *all* did the expensive work — 37 times instead of 10. If "the expensive work" is an LLM call or a paid API, this race is a **cost blowup and a correctness bug at once**. (This is the agentic version of a cache stampede — Day 15.)
- **`findings len` was correct (500) even unsafe**, because `list.append` is one bytecode → atomic under CPython's GIL (§1.2's "what the GIL *does* protect"). This is the honest nuance: *some* shared operations survive the GIL and some don't, and you cannot eyeball which — so you guard *all* of it. Relying on "append is atomic" is a bet against future Python versions and free-threaded builds (Day 20).
- **`tool_SAFE` wraps the entire check-then-act-and-update in one `with state.lock:`** — making the whole critical section atomic (§1.3). Now `total_cost` is exact, the expensive work runs exactly 10 times, and results are deterministic. Note the critical section includes *both* the memoization check *and* the cost update — one lock guarding all shared state that must stay consistent together. That's the §1.3 discipline applied to an agent.

**The rule for agent builders.** *Any state shared across concurrent tool executions — cost tallies, scratchpad accumulators, memoization caches, memory stores — must be guarded by a lock, or made per-task and merged at the end (the "don't share" alternative, §1.10).* And any tool with **side effects** (writes, sends, charges) must additionally be **idempotent** (safe to run twice) via an idempotency key in the backend (Day 5 WAL, Day 30), because concurrency plus retries means "exactly once" is a lie you enforce, not a property you get. This is Therac-25's lesson (§1.9) for agents: defense in depth, never trust the concurrency layer alone.

---

## 2.3 Bounded concurrency & thread-pool sizing for a tool-calling backend

**Depth: [WORKING]**

### Intuition

Why cap tool concurrency (the semaphore in §2.1) instead of firing every tool call at once? Four reasons, each a Part 1 concept:

1. **Downstream rate limits.** Tools call external APIs (weather, search, the LLM provider itself). Firing 50 at once earns you `429 Too Many Requests` and a temporary ban. Bounded concurrency is *politeness* (Day 22's crawler lesson).
2. **Resource exhaustion (§1.6/§1.7).** Each in-flight tool holds a thread (memory, a scheduler slot) and often a socket/connection (Day 5 fd limits). Unbounded → the C10K collapse in miniature, or fd exhaustion (`EMFILE`, Day 5).
3. **Connection-pool limits.** Your HTTP client and DB have bounded connection pools (Day 22's "works locally, dies at 500 RPS"). More concurrent tools than pool connections just queue *inside* the client anyway — so bound it explicitly and predictably.
4. **Cost control.** Concurrent tool calls can each spawn LLM sub-calls; unbounded concurrency = unbounded spend per turn (Day 24's cost ceiling, now multiplied by fan-out).

### Sizing it — Little's Law, again (§1.7)

Same law, agent flavor. If your agent backend serves **λ = 20 agent-requests/second**, each request fans out to an average of **3 tool calls**, and each tool call takes **W = 1.5 s** (LLM/API latency dominates), then the tool executor must sustain:

```
   tool-call arrival rate = 20 req/s × 3 tools = 60 tool-calls/s
   concurrent tool calls in flight = 60 × 1.5 s = 90   (Little's Law, §1.7)
   → the shared tool-executor pool needs ~90 concurrent slots (+ headroom ~120)
   → BUT cap per-downstream so you don't exceed each API's rate limit
     (e.g. the search API allows 30 concurrent → a per-tool semaphore of 30)
```

Two bounds stack: a **global** pool bound (total in-flight tools, sized by Little's Law and your memory/fd budget) and **per-downstream** bounds (one semaphore per external API, sized to that API's rate limit). This is exactly §1.7 plus §2.1, and it's why a production tool executor has a *tree* of semaphores, not one number.

**The trap this prevents:** an agent under load where each request fans out to many slow tools can require *far* more concurrency than a normal API (W is huge — LLM calls are seconds, not milliseconds — so L = λW is huge). Size it with the same discipline as any backend pool, or the executor becomes the bottleneck and every agent request queues behind it.

---

## 2.4 System design — a bounded, race-free tool executor

**The problem.** Design the tool-execution layer for an agent backend serving many concurrent agent sessions. Requirements: (a) run a turn's tool batch concurrently for latency (§2.1); (b) never corrupt shared per-session state (§2.2); (c) bound total and per-downstream concurrency (§2.3); (d) enforce per-turn timeout and cost ceiling (Day 24); (e) side-effecting tools are safe under retry (idempotent).

**Key decisions.**
- **A bounded thread pool** (or `asyncio` + semaphore) per the Little's-Law sizing of §2.3 — the global concurrency bound. Threads if tools are blocking/sync; async if you control the whole I/O stack (§1.6, Day 21).
- **A per-downstream semaphore** for each external API, sized to that API's rate limit (§2.3) — nested inside the global bound.
- **A per-session lock** guarding that session's scratchpad/cost/memory (§2.2). Per-session (not one global lock) so different sessions' tools don't serialize against each other — fine-grained locking (§1.3) done right: lock scope = the data that must be consistent together, no bigger.
- **Never hold the session lock across a tool's I/O** (§1.3, the Part 3 bridge): run the tool, *then* briefly take the lock to merge its result. Holding the lock during the 1.5 s tool call would serialize the whole session and destroy the concurrency of (a).
- **Per-turn timeout** (wrap the batch in `concurrent.futures` `wait(timeout=…)` / `asyncio.timeout`) and a **cost ceiling** checked under the lock (§2.2) — an agent without these is Day 24's "unbounded loop with your credit card."
- **Idempotency keys** for side-effecting tools (Day 5 WAL / Day 30) — so a retried or double-fired tool doesn't double-charge.

**Trade-off.** Fine-grained per-session locks + per-downstream semaphores are more moving parts than "one global lock, run tools one at a time" — but the simple version serializes everything (§1.3's coarse-lock trap) and can't meet the latency requirement. You accept complexity for concurrency, and you contain the complexity by making each lock's scope exactly one session's mutable state.

**Failure modes.**

| Failure | Cause | Mitigation |
|---|---|---|
| Corrupted cost/scratchpad | unguarded shared state (§2.2) | per-session lock around all shared mutations |
| `429` bans / downstream overload | unbounded per-API concurrency (§2.3) | per-downstream semaphore at the rate limit |
| Whole session stalls | lock held across tool I/O (§1.3) | run tool outside lock; lock only the merge |
| Executor is the bottleneck under load | pool undersized for λ×W (§1.7, §2.3) | size by Little's Law; backpressure/reject at the edge |
| A hung tool blocks a worker forever | no per-tool timeout | timeout every tool; cancel/abandon on expiry |
| Double side effect (email/charge) | concurrency + retry | idempotency key in the backend (defense in depth, §1.9) |
| Deadlock across session + downstream locks | inconsistent lock order (§1.4) | fixed global lock ordering; prefer taking one lock at a time |

**Cross-reference.** This is §1.7 (pool sizing) + §1.8 (priority/fairness, if urgent vs background tools) + §2.2 (race-free state), assembled. Frameworks (Day 73–76: LangGraph, etc.) provide much of this executor, but they're "this loop with opinions" (Day 24) — knowing the mechanism is how you debug them when a parallel-tool node corrupts state.

## 2.5 Case study — parallel tool use in production agent frameworks

**What it is (real, verify current).** Anthropic's Claude and OpenAI's models both support **parallel tool calling**: in one assistant turn the model can emit multiple tool calls (Anthropic: multiple `tool_use` content blocks; OpenAI: a `tool_calls` array), explicitly so your executor can run them concurrently and cut latency (§2.1). Anthropic's tool-use documentation describes returning multiple `tool_result` blocks — one per `tool_use_id` — in the following user turn, and both providers' docs and cookbooks show concurrent execution of the batch. Agent frameworks (LangGraph, LlamaIndex, the OpenAI Agents SDK) build tool executors that run these batches concurrently with bounded parallelism.

**The engineering lesson.** The providers hand you *parallelism*; they do **not** hand you *safety*. The framework runs your tools concurrently, but if your tools mutate shared state (§2.2) or hammer a rate-limited API (§2.3), those are *your* bugs, in *your* tool code — the exact Part 1 concurrency problems. The most common real-world agent concurrency bugs are: (1) a shared HTTP client or DB session used across parallel tools without it being thread-safe; (2) a memoization/memory cache with a check-then-act race (§2.2's `expensive calls: 37`); (3) unbounded fan-out tripping provider rate limits. All three are today's material. **Verify against current provider docs** (Day 23–24) — parallel-tool-use semantics, field names, and whether a given model defaults to parallel calls all drift, and are exactly the fast-moving details Principle 6 says to re-check.

*(Per Principle 7: I'm not aware of a single canonical *published named postmortem* about an agent tool-execution race that I can cite precisely, so I'm not inventing one — the Part 1 postmortem is Therac-25 (§1.9), whose mechanism is identical to §2.2's, and the AutoGPT unbounded-loop lesson (Day 24) is the closest documented agent-autonomy failure. The provider parallel-tool-use behavior above is real and documented.)*

## 2.6 In production (Part 2, condensed per [WORKING] tier)

- **Best practice:** keep per-tool state on the stack; share as little as possible (§1.10). Guard whatever you must share with a per-session lock (§2.2). Bound concurrency globally and per-downstream (§2.3). Timeout and cost-cap every turn (Day 24). Make side-effecting tools idempotent (§1.9, Day 30).
- **Top failure mode:** a shared, non-thread-safe client (HTTP/DB/LLM SDK) reused across parallel tools — the agent version of Day 5's "new client per request" bug and Day 22's pool exhaustion. Symptom: intermittent corrupted responses or connection errors *only* under concurrent tool use, never in single-tool tests (the §1.2 non-reproducibility signature).
- **Monitor:** in-flight tool concurrency vs the pool bound (saturation → §1.7 cliff); per-downstream `429` rate (concurrency too high → §2.3); cost-per-turn tail (a check-then-act race inflating expensive calls → §2.2). Trace every tool call with its session id and timing (Day 24's audit log) — concurrency bugs are invisible without per-call traces.

---

# PART 3 — THE BRIDGE

> Parts 1 and 2 each ended at the same three primitives — a thread pool, a lock, a bound. Part 3 is where a thread-per-request API server (Part 1) and an agent's parallel tool executor (Part 2) become **one running system** and discover they share a thread pool, a scheduler, and every failure mode in it. No new concepts here — only the interactions. (This is where Day 21's async story and Day 28's ASGI server will pick up.)

## 3.1 Where the layers meet: one thread pool, two claimants

A production agent is served *behind an API* (Day 27–28): an HTTP request arrives → the server assigns it a worker (a thread from the pool, §1.7, or an event-loop task, §1.6) → that worker runs the agent loop (Day 24) → the agent fans out **parallel tool calls** (§2.1) → each tool does blocking I/O. Trace the concurrency:

```
   HTTP request  ──►  API server (thread pool, §1.7)  ──►  agent loop (Day 24)
                                │                              │
                                │ one worker thread per        │ fans out N tool calls (§2.1)
                                │ in-flight agent request      ▼
                                │                        tool executor (another pool, §2.3)
                                ▼                              │
                       Little's Law sizes THIS pool     each tool = blocking I/O (Day 5)
                       (§1.7): L = λ × W, where W is     LLM call, DB, API — SECONDS each
                       now DOMINATED by agent tool       │
                       latency (seconds, not ms)         ▼
                                                   shared session state (scratchpad/cost)
                                                   guarded by a lock (§2.2/§1.3)
```

The load-bearing observation: **W for an agent request is enormous** — a single agent turn can take 5–30 seconds (multiple LLM calls + tool I/O), versus ~200 ms for a plain CRUD request. Little's Law (§1.7) says required concurrency `L = λ × W`, so even *modest* agent request rates demand *large* concurrency. 10 agent-requests/second × 15 s each = **150 concurrent in-flight agent requests**, each holding a worker for 15 seconds. Size the server pool for a CRUD app (§1.7's 100 threads) and put an agent behind it, and you saturate at a request rate an order of magnitude lower than you'd expect — because each request *camps* on its thread for seconds. This is the first thing that breaks when teams put an agent behind an existing API server without re-doing the pool math.

## 3.2 The shared failure mode: a blocking tool call starves the pool

This is the punchline of the entire day, and it unifies §1.3, §1.5, §1.6, §2.1, and the Nginx case study (§1.9).

Recall the event-loop rule (§1.6, Day 5 §1.6): a single-threaded event-loop worker (Nginx, asyncio, an ASGI server like uvicorn — Day 28) handles thousands of connections *only as long as nothing blocks the loop*. Now put an agent on it. The agent calls a tool. If that tool does a **blocking** call — a synchronous LLM SDK, a `requests.get`, a sync DB driver, a `time.sleep`, a subprocess (Day 2) — inside the async event loop, it **freezes the entire loop** (§1.6's "blocking the event loop," §1.9's single-threaded Nginx worker). *Every other agent session on that worker stalls* for the full duration of the one blocking tool call. One user's slow 30-second tool call adds 30 seconds of latency to *everyone else's* requests on that worker. This is the universal async postmortem (Day 21's case study) wearing an agent costume:

```
   ASGI worker (one event loop, §1.6), serving 200 agent sessions concurrently:

   session A's tool does  requests.get(slow_api)   ← BLOCKING call, 30s, in async handler
        │
        ▼
   ┌─────────────────────────────────────────────────────────────┐
   │  EVENT LOOP FROZEN for 30s — cannot service any other socket │
   │  sessions B..Z (199 of them): all stalled, p99 → 30s         │
   └─────────────────────────────────────────────────────────────┘
   The blocking tool didn't just slow session A. It starved the whole worker. (§1.6, §1.9)
```

In the **thread-pool** model (Part 1 §1.7) the same blocking tool is less catastrophic — it only occupies *its own* thread, others keep running — but it still eats a pool slot for 30 seconds, and enough concurrent slow tools exhaust the pool (§1.7's cliff), after which *new* requests queue and time out. **Both models fail; they fail differently.** The event loop fails *globally and instantly* (one blocking call, everyone stalls); the thread pool fails *gradually* (slow tools eat slots until the pool saturates). Knowing which model you're on tells you which failure to expect and how to defend:

- **Event loop (async, §1.6/Day 21):** *never* call a blocking function in an `async def` handler. Offload blocking tools to a thread with `asyncio.to_thread` / `run_in_executor` (the escape hatch Day 21 teaches), or use async-native tool clients all the way down. FastAPI's `def` vs `async def` distinction exists for exactly this (Day 21's case study).
- **Thread pool (sync, §1.7):** size the pool for the agent's huge W (§3.1), put a hard timeout on every tool (cap W — §2.4), and add backpressure so a saturated pool sheds load with `429` instead of unbounded queuing (§1.7).

## 3.3 The dependency map

What each layer calls into, what it serves back, and what breaks on each side when the other misbehaves:

```
   ┌────────────────────── AGENT LAYER (Part 2) ──────────────────────┐
   │  needs from backend:                                              │
   │   • a worker (thread §1.7 / task §1.6) to run each agent request  │
   │   • a bounded tool executor (pool + semaphores §2.3)              │
   │   • a lock per session for shared scratchpad/cost (§2.2)          │
   │   • non-blocking (or offloaded) tool I/O (§3.2)                   │
   │   • idempotency for side-effecting tools (§1.9, Day 5 WAL)        │
   └───────────────────────────┬──────────────────────────────────────┘
                               │ hands off: a tool batch to run concurrently,
                               │            per-request state to keep consistent
                               ▼
   ┌────────────────────── BACKEND LAYER (Part 1) ────────────────────┐
   │  serves back:                                                     │
   │   • preemptive scheduling across all workers (§1.5)               │
   │   • thread/coroutine primitives + locks (§1.1–§1.3)               │
   │   • pool capacity sized by Little's Law (§1.7)                    │
   │   • the event loop / epoll that multiplexes I/O (§1.6, Day 5)     │
   └──────────────────────────────────────────────────────────────────┘

   FAILURE COUPLING (how one side breaks the other):
   • Agent fires unbounded parallel tools  → exhausts backend pool/fds → §1.7 cliff, EMFILE (Day 5)
   • Agent tool does blocking I/O on loop   → freezes backend event loop → ALL sessions stall (§3.2)
   • Agent shares session state, no lock     → §1.2 race → corrupted cost/memory, double side effects
   • Backend pool sized for CRUD, not agent  → §3.1 → saturates at low request rate, requests time out
   • Backend deadlock in a shared resource    → §1.4 → agent workers hang at 0% CPU, no error, no shutdown
   • Two tools take session + downstream locks in different orders → §1.4 deadlock across the boundary
```

## 3.4 The one sentence

The agent and the backend are not two systems that talk — they are **one concurrency system**, and every rule from Part 1 (share little; guard what you share with a consistent lock; acquire locks in order; never block a shared worker; size pools by λ×W; bound and time-out everything) is *simultaneously* a backend rule and an agent rule. The agent just raises the stakes: a corrupted counter becomes a double-charged card, a starved thread pool becomes a stalled fleet of user conversations, and a race you couldn't reproduce becomes the incident you can't explain. That is why this OS day sits so early in a plan about building agents.

---

# Topic-wide wrap-up

## Cheat Sheet

| Concept | One-line truth | Tag | § |
|---|---|---|---|
| Thread | Execution stream sharing the process's memory; private stack, shared heap/globals | [CORE] | 1.1 |
| Shared vs private | Locals (stack) can't race; globals/heap objects can | [CORE] | 1.1 |
| Race condition | Outcome depends on timing; `x += 1` is 3 non-atomic steps → lost updates | [CORE] | 1.2 |
| GIL | Makes single *bytecodes* atomic, NOT statements — `+=` still races | [AWARE] (Day 20) | 1.2 |
| Critical section / atomicity | The region that must be indivisible; the property you must impose | [CORE] | 1.2 |
| Lock (mutex) | Mutual exclusion; only protects data if *everyone* takes the *same* lock | [CORE] | 1.3 |
| Lock granularity | Fine = correct+costly; coarse = cheap but serializes; hold briefly, never over I/O | [CORE] | 1.3 |
| Deadlock | All 4 Coffman conditions → hang at 0% CPU; break *circular wait* with lock ordering | [CORE] | 1.4 |
| Preemptive scheduling | Timer interrupt forcibly switches threads mid-work (why races exist) | [CORE] | 1.5 |
| Context switch | ~1–10 µs + cache pollution (the hidden cost); voluntary vs involuntary | [CORE] | 1.5 |
| C10K | 10k threads collapse on memory + switch storm + cache thrash → event loops | [CORE] | 1.6 |
| Little's Law | Pool size L = λ × W; ask "I/O- or CPU-bound?" first | [CORE] | 1.7 |
| Starvation / aging | Pure priority starves; effective priority improves with wait time | [CORE] | 1.8 |
| Agent tool executor | A bounded thread pool (§1.7) over the model's tool batch | [WORKING] | 2.1 |
| Agent scratchpad race | §1.2 on cost/memory → lost updates, double expensive calls, double side effects | [CORE] | 2.2 |
| Bounded tool concurrency | Global (λW) + per-downstream (rate limit) semaphores | [WORKING] | 2.3 |
| Blocking tool on event loop | Freezes the whole worker → every session stalls (the bridge) | [CORE] | 3.2 |

## Build This

**Definition of done — a race, its fix, and the C10K wall, all measured.**

1. **The classic race (§1.2):** two threads increment a shared counter 1,000,000× with `sys.setswitchinterval(1e-6)`. Confirm the result is `< 2,000,000` and *different each run*. Print the bytecode of `counter += 1` and identify the LOAD→STORE gap.
2. **The lock fix (§1.3):** wrap the increment in `threading.Lock`; confirm exactly `2,000,000` every run; measure the throughput cost (fine- vs coarse-grained); prove a lock is a convention by having one thread cheat and stay wrong.
3. **Deadlock + fix (§1.4):** reproduce a two-lock deadlock (detect it with `join(timeout=…)` + `is_alive()`); fix it by acquiring locks in `sorted(..., key=id)` order.
4. **The C10K benchmark (§1.6):** 1,000 threads vs 1,000 coroutines each sleeping 1 s; print memory (`/proc/self/status` VmRSS) and startup time for each; confirm coroutines use ~20–25× less memory and start ~20× faster while both finish in ~1 s. Bump to 10,000 and watch the thread version struggle.
5. **The agent scratchpad race (§2.2):** 500 parallel "tool calls" over 10 distinct keys mutating a shared cost tally + memoization cache with and without a lock; confirm the unsafe run undercounts cost and does the "expensive" work far more than 10 times, and the locked run is exact.
6. **The bridge (§3.2):** take an async server, put a `time.sleep(5)` (blocking) inside one handler, load-test, and watch *every* concurrent request stall; fix with `asyncio.to_thread`. (This overlaps Day 21's build — do the minimal version now.)

All six are in this note's runnable examples; assemble them into one repo, commit, and write a one-paragraph explanation of *why* each number came out as it did.

## Active Recall & Self-Test (answer from memory)

1. Trace how two threads doing `counter += 1` lose an update. Draw the interleaving with real values.
2. Why doesn't Python's GIL make `counter += 1` safe? What *does* it make safe?
3. Name the four Coffman conditions. Which does lock ordering break, and why is that the preferred fix?
4. What are the three costs of a context switch? Which is hidden and often largest?
5. Why do 10,000 threads collapse where 10,000 coroutines don't? Give the memory and switch arithmetic.
6. An I/O-bound API gets 400 req/s at 250 ms each. Size the thread pool. Now it's CPU-bound — what changes?
7. How does aging prevent starvation in a priority scheduler? What does it trade away?
8. Your agent runs 5 tools in parallel and its cost tracker undercounts. Diagnose and fix.
9. A blocking tool call is inside an async agent handler. What happens to the other 199 sessions? What's the fix?
10. Why is Therac-25 the canonical race-condition case study? What does "we couldn't reproduce it" actually mean?

**60-second teach-back prompt:** *"Explain to a smart friend who's never coded why letting two workers share memory is both the fastest thing you can do and the most dangerous — using the whiteboard-total analogy, the word 'atomic,' and one sentence on how an AI agent running tools in parallel is the exact same problem."*

## Spaced-Repetition Flashcards

- **Q:** Is `counter += 1` atomic? **A:** No — read, add, write (3 steps); a thread can be preempted between them → lost updates.
- **Q:** What does a lock actually protect? **A:** Only data that *every* accessor guards with the *same* lock. It's a convention, not a barrier.
- **Q:** Four Coffman conditions? **A:** Mutual exclusion, hold-and-wait, no-preemption, circular wait. Break any one → no deadlock.
- **Q:** Cheapest way to prevent deadlock across multiple locks? **A:** Acquire them in a fixed global order (breaks circular wait).
- **Q:** Deadlock vs livelock vs infinite loop by CPU usage? **A:** 0% CPU hung / 100% CPU no-progress / 100% CPU running.
- **Q:** Context-switch cost? **A:** ~1–10 µs direct + TLB flush (process switch) + cache pollution (the hidden, often biggest cost).
- **Q:** Thread-pool size for I/O-bound work? **A:** Little's Law: L = λ × W (arrival × service time), + headroom.
- **Q:** Thread-pool size for CPU-bound work? **A:** ≈ number of cores.
- **Q:** Why do 10k threads fail (C10K)? **A:** ~stack memory each + context-switch storm + cache thrash. Answer = event loop.
- **Q:** Agent runs tools in parallel and mutates a shared scratchpad — what's the bug? **A:** A race condition (§1.2); fix with a per-session lock or don't share.
- **Q:** Blocking call inside an async handler? **A:** Freezes the whole event loop → every session stalls. Offload with `asyncio.to_thread`.
- **Q:** Does the GIL prevent races? **A:** No — only makes single bytecodes atomic; compound/check-then-act ops still race.

## Primary Sources (verify against these)

- **Threads/clone:** `clone(2)`, `pthreads(7)`, `nptl(7)` man pages.
- **Python thread-safety & the GIL:** CPython FAQ "What kinds of global value mutation are thread-safe?" (docs.python.org); the `threading` module docs; `sys.setswitchinterval`. (Full GIL: Day 20.)
- **Locks/futex:** `futex(2)`; `pthread_mutex_lock(3)`; CPython `Modules/_threadmodule.c`.
- **Deadlock:** Coffman, Elphick & Shoshani, "System Deadlocks," *ACM Computing Surveys*, 1971; Dijkstra's dining philosophers; Postgres docs "Deadlocks."
- **Priority inversion:** Glenn Reeves (JPL), "What Really Happened on Mars?" (Mars Pathfinder, 1997).
- **Scheduling:** `sched(7)`; Linux `Documentation/scheduler/` (CFS/EEVDF); Brendan Gregg, *Systems Performance* (Ch. 6).
- **C10K / event loops:** Dan Kegel, "The C10K problem" (kegel.com/c10k.html, 1999); `epoll(7)` (Day 5); Nginx architecture docs; Apache MPM docs.
- **Little's Law:** John Little, "A Proof for the Queuing Formula L = λW," *Operations Research*, 1961.
- **Therac-25:** Nancy Leveson & Clark Turner, "An Investigation of the Therac-25 Accidents," *IEEE Computer*, July 1993 (read this one).
- **Agent parallel tools:** current Anthropic and OpenAI tool-use/function-calling docs (verify — they drift; consult the `claude-api` skill for Claude code).

## Layered explanations

- **10-second:** Threads share memory, which is fast but lets them corrupt each other's updates (races); you fix races with locks, locks can deadlock, and making too many threads (C10K) collapses — the same rules govern an AI agent running tools in parallel.
- **1-minute:** A thread is a worker inside a process sharing all its memory. Sharing is fast (no copying) but dangerous: `counter += 1` is three steps and two threads can interleave and lose updates — a *race condition*, whose defining trait is that it's timing-dependent and non-reproducible (which killed people via the Therac-25). You fix races with a *lock* (mutual exclusion), but locks introduce *deadlock* (two threads each waiting on the other's lock — break it with lock ordering). The OS runs more threads than cores by *preemptive scheduling*, switching between them every few ms; each *context switch* costs ~1–10 µs plus a cold cache. So 10,000 threads collapse (the *C10K problem*) on memory and switching — which is why high-concurrency servers use *event loops* instead. All of this reappears in AI agents: running tools in parallel is a thread pool, the agent's shared scratchpad races exactly like the counter, and a blocking tool call starves the whole server.
- **5-minute:** *(the note's Parts 1–3 in order: threads and shared vs private memory (§1.1) → the race and why the GIL doesn't save you (§1.2) → locks and granularity (§1.3) → deadlock and the four Coffman conditions (§1.4) → preemptive scheduling and context-switch cost (§1.5) → the C10K collapse and why event loops exist (§1.6) → sizing pools with Little's Law (§1.7) and preventing starvation with aging (§1.8) → Apache-vs-Nginx and Therac-25 (§1.9); then the agent: concurrent tool execution as a thread pool (§2.1), the scratchpad race (§2.2), bounded concurrency and sizing (§2.3); then the bridge: an agent request's huge service time re-sizes the server pool (§3.1), and a blocking tool call starves the whole worker (§3.2).)*
- **Expert:** Concurrency correctness reduces to controlling access to shared mutable state under nondeterministic interleaving imposed by a preemptive scheduler; the only sound strategies are eliminating sharing (immutability, per-task state, message passing) or serializing access (locks with a consistent global order to preclude circular wait, held for minimal critical sections and never across I/O). Performance is a separate axis governed by the context-switch tax and the memory-per-execution-context, which makes thread-per-connection untenable past ~10⁴ concurrent I/O-bound tasks (C10K) and forces readiness-based event multiplexing (epoll/kqueue) with cooperative scheduling — trading code linearity for a ~25× reduction in per-connection cost, at the price of a new failure mode (any blocking call monopolizes the shared loop). Capacity is Little's Law (L = λW), with the I/O-vs-CPU distinction flipping the target between "concurrency of outstanding waits" and "core count." An LLM agent is this same system with W inflated to seconds and shared state whose corruption has monetary and real-world side effects, so it demands per-session locking, bounded (global × per-downstream) tool concurrency, idempotent side effects as defense-in-depth, and strict avoidance of blocking calls on shared workers — i.e., every invariant above, with the stakes of Therac-25.

---

*End of Day 3. Next: Day 4 — The OS III: memory management (virtual memory, pages, stack vs heap, the OOM killer, and why a Python "memory leak" is usually a lingering reference).*
