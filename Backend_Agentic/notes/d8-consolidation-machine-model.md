# Day 8 — Consolidation: The Machine Model

> **Framing.** Days 1–7 taught you five separate machines: the latency pyramid, the kernel's process model, threads and scheduling, virtual memory, and files/descriptors — plus how CPython turns your source into running bytecode and how the toolchain keeps that code trustworthy. Today adds no new machine. It welds the five together into one mental model — "what actually happens when a computer runs your code" — and then uses that welded model as raw material for two decisions a real backend engineer makes constantly: how do you let untrusted code run at all, and how do you make work durable on a single box before you reach for a distributed queue. **Agentic connection (real, not stretched):** an agent's "run this code" tool call *is* today's trace, just triggered by a model instead of a human typing at a shell, and today's sandbox design is the concrete shape that tool takes in production. The job-runner design is deliberately backend-only until its final paragraph, where it previews Day 49's distributed twin — durable execution is exactly how modern agent frameworks (LangGraph's checkpointing, Temporal-backed agent loops) survive a crash mid-tool-call.
>
> **A note on evidence.** Every transcript in this note that is *not* explicitly attributed to Day 2/4/5/6 was executed for real, on the machine this note was written on (native Windows 11, Python 3.12.7, Git Bash), while writing this note — not hand-derived from memory. Where that matters (numbers that depend on your OS, your Python build, or what's installed in your `site-packages`), it's called out. Where a mechanism is POSIX-only (namespaces, `prlimit`, process groups) it's flagged, because — as you're about to see firsthand — it matters: this specific machine has no WSL2 distribution installed and no Docker, so several of Day 4/5's own `resource`-module builds cannot run here at all. That is not a hypothetical warning; it's what happened when this note's Linux-only code was checked.

---

## Before you read the rest of this note: do the consolidation, don't just read about it

The study plan's instruction for today is "teach-backs for Days 1–7 (60 s each, aloud); re-answer all active-recall questions from memory; redo the weakest build" — and that instruction is the actual point of the day, not a preamble to skip. Notes can't do this part for you: recall is a retrieval act, and retrieval only strengthens memory when it's effortful and unaided. So the table below is deliberately *not* a restatement of Days 1–7's content (that already exists, in full, in each day's own note — restating it here would violate the same "don't repeat yourself" discipline those notes were written under). Instead it's three things per day: the one sentence you should be able to defend aloud, a pointer to where that day's own recall questions live, and a *new* question that only makes sense once you've done today's synthesis — because it forces you to connect that day to at least one other.

| Day | The one sentence (teach back aloud, 60s) | Re-answer from memory | New synthesis question (do this after reading the rest of Day 8) |
|---|---|---|---|
| 1 — Latency pyramid | "Fast" isn't a property of a language or a server; it's where in the register→cache→RAM→SSD→network hierarchy your program's time actually goes, and a program is CPU-bound or I/O-bound depending on which tier dominates. | `d1-cpu-memory-latency-pyramid.md` → *Active Recall and Self-Test* (8 Qs) | Day 2's traced `python -c "print('hi')"` costs roughly 1,900 syscalls. Even at a pessimistic 1 µs each, that's under 2 ms. Using Day 1's own measured numbers, is that closer to "one dict lookup" or "one cross-continent HTTP request" — and what does that imply about optimizing interpreter startup versus optimizing a model call? |
| 2 — Kernel, syscalls, processes | A process is the unit of isolation, resource control, and killability; fork+exec, signals, and containers are all just arithmetic on those three properties. | `d2-os-kernel-syscalls-processes.md` → *Active Recall & Self-Test* (15 Qs) | Day 2 already combined its own sandbox and graceful-shutdown designs (§3.2). What does today's sandbox design (below) add that Day 2 explicitly deferred to "Day 3 opens… Day 4 previews… Day 5's [CORE] topic"? |
| 3 — Threads, scheduling, races | Shared mutable state under concurrency is simultaneously the fastest thing you can build and the worst bug you can ship; the only cures are "don't share" or "coordinate access." | `d3-os-threads-scheduling-races.md` → *Active Recall & Self-Test* (10 Qs) | Today's job runner uses worker **processes**, not threads, and coordinates them without a single `threading.Lock`. Which of Day 3's four Coffman conditions, or which race-condition mechanics, simply don't apply once "shared state" means "shared address space" and there isn't one? |
| 4 — Memory management | Memory is a finite physical budget you're only lent the illusion of owning; the discipline of living inside it is the same whether you're sizing a container, hunting a reference leak, or capping a sandboxed subprocess. | `d4-os-memory-management.md` → *Active Recall & Self-Test* (10 Qs) | Day 4 gave you two memory caps: `RLIMIT_AS` (catchable `MemoryError`) and the cgroup/`docker -m` limit (uncatchable `SIGKILL`). Today's sandbox has to pick one as its *enforced* boundary. Which one, and why does the answer change if you can't guarantee a container runtime under the sandbox? |
| 5 — Files, I/O, descriptors | A file descriptor is a lease on a kernel object, and I/O is deciding when to wait for it, when to buffer it, and when to pay for durability. | `d5-os-files-io-descriptors.md` → *Active Recall & Self-Test* (15 Qs) | Day 5's write-ahead-log design (§1.7) proved "append + fsync before ack" is the durability primitive for a database *record*. What has to change when the thing being made durable is a *job*, and the process consuming it can crash mid-job rather than mid-write? |
| 6 — Python under the hood | Interpreter overhead only matters where there's no bigger wait to hide behind. | `d6-python-under-the-hood.md` → *Active Recall & Self-Test* (8 Qs) | In Day 2's syscall trace, exactly one call — `write(1, "hi\n", 3)` — corresponds to code you wrote. Using Day 6's own pipeline diagram, which stage produced the bytecode behind that call, and why does none of the *compiling* work show up as a syscall at all? |
| 7 — The modern toolchain | A locked, typed, linted, tested skeleton enforced by CI (not memory) makes the deterministic 95% of a system trustworthy, so the model's non-determinism is the only remaining unknown. | `d7-modern-python-toolchain.md` → *Active Recall & Self-Test* (8 Qs) | If your Day 8 sandbox runs *agent-generated* code in a subprocess, does Day 7's `mypy --strict` / `ruff` pipeline protect you from anything that code does at runtime? What's the actual boundary that does the protecting? |

**Redo the weakest build.** Don't guess which one that is — check: for each of Days 1–7, can you currently run that day's `Build` from a blank terminal, with no notes open, and get the same qualitative result (the race loses updates, the OOM kill fires, the FD limit trips)? The first one you can't is the one to redo before continuing. If more than one fails, redo the earliest — later days' builds usually assume the earlier mechanism is already solid muscle memory (Day 5's `fsync`-durability build, for instance, is much easier to reason about once Day 4's page-cache-vs-disk distinction is automatic).

---

## The life of `python app.py`

**Depth: [CORE].** This is the day's central synthesis, and its two required system-design scenarios (below) don't stand apart from it — they're this same trace, aimed at two different real problems. Nothing here is opened further than Days 2–6 already opened it; the job of this section is to show the order those pieces fire in, which none of Days 2–6 needed to do individually.

### Intuition

Here is the trap Days 2–6 quietly set up, one you should notice and correct right now: each of those days handed you one layer — a process, a page table, a file descriptor, a bytecode instruction — in isolation, with everything else held fixed or waved away as "elsewhere." That's the right way to *learn* five machines, but it leaves you with a wrong intuition about how they actually run: as five separate systems that occasionally talk to each other. They aren't. The instant you type `python app.py` and hit Enter, all five fire in one specific, mechanical sequence, in the same address space, on the same CPU, and the order matters — you cannot explain why a Python CLI tool "feels slow to start" (a real, googleable, engineer-hours-consuming complaint) without knowing which of the five layers the slowness is actually in, and the only way to know that is to have walked the whole sequence at least once, end to end, instead of five times, once per layer.

This also matters for a very concrete reason that will recur for the rest of this plan: every "worker" you will ever build — a FastAPI process (Day 27+), a Celery/RQ task worker, an agent's sandboxed tool-execution subprocess (this day's own System design ①) — *is* one more instance of this exact trace, just with a different `app.py`. Understanding this trace once means you never have to re-derive "why did that process take 200ms just to say hello" from scratch again; you'll recognize which of six stages ate the time.

### Analogy — a new employee's first day, every single morning

Picture a new hire starting a job. Day one has an orientation (get a badge, learn where the fire exits are, get handed the employee handbook) before they do any actual work, and once oriented, they read their specific task list and start executing it, checking off items one at a time, occasionally looking things up in a reference binder they haven't needed yet. At the end of the shift, they finish or hand off whatever's in progress, return anything borrowed, clock out, and leave.

That maps cleanly: getting hired and badged is `fork`+`exec` (Day 2) creating the process; orientation is the interpreter's own startup (below); the handbook and any department-specific memos are `sys.path` and the `site` module; the task list is your script's source code; checking off items one at a time is the `ceval` loop executing bytecode (Day 6); looking things up in the reference binder is an `import` triggering nested loads; clocking out is `Py_FinalizeEx` and process exit (Day 2's exit codes, Day 4's refcount teardown, Day 5's descriptor closing).

**Where this analogy breaks — and why the break is the important part.** A human employee is oriented *once*, ever; the cost is amortized across years of subsequent workdays. An interpreter's "orientation" (core + main initialization, encodings, `sys.path`, `site`) happens **every single invocation**, with zero memory of the previous run. That's not a minor mismatch — it's the entire reason "Python startup time" is a real, named, recurring engineering cost (this section's case studies and In Production notes are both about exactly that), rather than a one-time fee you pay once per machine. If the analogy tempts you to think "well, the orientation cost is a rounding error over a career," correct that immediately: for a CLI tool invoked thousands of times a day, or a serverless function whose container is thrown away between invocations (this section's AWS Lambda case study), that "orientation" is paid, in full, on every single call.

### Visual — the six stages at a glance

```
 SHELL (a process)                                                          Day 2
     │  you type: python app.py <Enter>
     ▼
 ┌─────────────────────────── STAGE 0 — fork + exec ────────────────────────┐
 │ shell fork()s a child (COW copy of shell's address space, fd table)      │
 │ child execve("/usr/bin/python3", ["python3","app.py"], envp)             │
 │ same PID, same fds (0/1/2 inherited) — shell's own code is now gone      │
 └────────────────────────────────────────────────────────────────────────┘
     ▼
 ┌──────────────────── STAGE 1 — interpreter startup (NEW today) ───────────┐
 │ Uninitialized → Runtime Pre-Init → Main Interpreter Init → Initialized   │
 │ builtin types + singletons + GIL created; `encodings` imported (must     │
 │ succeed); sys.path computed; `site` imported (.pth files, site-packages) │
 └────────────────────────────────────────────────────────────────────────┘
     ▼
 ┌──────────────────── STAGE 2 — locate + read the script ──────────────────┐
 │ openat("app.py") → a fresh fd (Day 5's descriptor table; by now Stage 1 │
 │ has already opened and closed many others, so it's not a small number)  │
 │ read() the bytes, close() the fd                                        │
 │ NEVER cached to __pycache__ — the __main__ script is recompiled fresh    │
 └────────────────────────────────────────────────────────────────────────┘
     ▼
 ┌────────────────── STAGE 3 — tokenize → parse → compile ──────────────────┐
 │ Day 6's pipeline: tokenizer → PEG parser → AST → compiler → code object  │
 │ (a *nested* copy of this whole diagram runs for every `import`, except   │
 │  imported modules DO get cached to __pycache__/*.pyc — Stage 2/3 skip    │
 │  straight to Stage 4 on a cache hit)                                     │
 └────────────────────────────────────────────────────────────────────────┘
     ▼
 ┌───────────────────── STAGE 4 — the ceval loop runs ───────────────────────┐
 │ bytecode executes top to bottom in __main__'s namespace                  │
 │ print(...) → TextIOWrapper → BufferedWriter → write(1, b"...", n)        │
 │ (Day 2's own traced example — that write() IS this stage, for real)      │
 └────────────────────────────────────────────────────────────────────────┘
     ▼
 ┌───────────────────── STAGE 5 — shutdown + exit ───────────────────────────┐
 │ Py_FinalizeEx(): atexit callbacks, flush stdio, refcounts unwind (Day 4) │
 │ kernel closes remaining fds (Day 5), exit_group(status) (Day 2)          │
 │ parent shell's waitpid() returns, reaps status, prints next prompt       │
 └────────────────────────────────────────────────────────────────────────┘
```

### Worked example — tracing `app.py` from Enter to prompt

Take a concrete, minimal, real script:

```python
# app.py
import json


def summarize(counts: dict[str, int]) -> str:
    return json.dumps(counts, sort_keys=True)


if __name__ == "__main__":
    print(summarize({"b": 2, "a": 1}))
```

**Stage 0 — fork + exec (Day 2, cross-referenced, not re-derived).** Your shell is itself a running process. It parses `python app.py`, resolves `python` against `$PATH`, then does exactly what Day 2's §1.5 traced with real PIDs: `fork()` gives a child that is a copy-on-write duplicate of the shell (same fd table — it inherits fds 0/1/2, which is *why* `print()` later lands in your terminal at all — same cwd, uid, environment, rlimits), and in the gap before `exec`, nothing changes yet. Then `execve("/usr/bin/python3", argv, envp)` replaces the child's program: same PID, same fds, entirely new code — "the shell's code is simply gone," in Day 2's own words. The shell's own thread now blocks in `waitpid()`, which is why your terminal shows no new prompt until the whole trace below finishes.

**Stage 1 — interpreter startup.** This is the one stage Days 1–7 genuinely never opened (confirmed: Day 6's pipeline diagram starts from "your `.py` text," already in hand — it does not describe how the interpreter got itself ready to read that text). The canonical description is [PEP 432 — Restructuring the CPython Startup Sequence](https://peps.python.org/pep-0432/), implemented as [PEP 587 — Python Initialization Configuration](https://peps.python.org/pep-0587/) starting in Python 3.8. PEP 432 names four states the interpreter passes through:

1. **Uninitialized** — nothing exists yet; the embedding `main()` decides the memory allocator and OS-interface encoding.
2. **Runtime Pre-Initialization** — "no interpreter is available"; only builtin/frozen modules could be imported even if you tried. `PYTHONHASHSEED` is resolved here.
3. **Main Interpreter Initialization** — the interpreter state and main thread state now exist: built-in types, the singletons (`None`/`True`/`False`, the small-int cache underlying Day 4's `sys.getsizeof` numbers), and the GIL are created. The `encodings` package is imported — and *must* succeed, or the interpreter fatally aborts, because without a codec nothing else can even be decoded from disk, including the source file you're about to read. `sys.path` is computed from `PYTHONPATH`, the install layout, and the script's own directory. This phase concludes by importing the `site` module (unless you pass `-S`), which walks `site-packages`, executes any `.pth` files it finds there, and is the thing that turns "an installed package" into "an importable name" — this is exactly the layer a virtual environment or `uv`-managed project (Day 7) manipulates.
4. **Initialized** — full interpreter, fully operational; only `sys.argv[0]` and `__main__`-specific metadata are still incomplete, because your script hasn't run yet.

**Honesty caveat, verified live on this note's own machine:** this phase is not free, and it is not fixed. Skipping it with `python -S app.py` (which skips only the `site` import, not the rest of core init) measured **62–73 ms** across three runs here, versus **204–214 ms** for a normal `python app.py` — roughly **140 ms**, on *this specific machine*, attributable to `site` alone. `python -X importtime app.py` shows exactly why: this machine's `site-packages` includes a `sitecustomize.py` that pulls in `wrapt`, `certifi_win32`, and a `usercustomize.py` — packages this script never asked for and never imports — and `site`'s own line in the importtime trace reports a **152,569 µs (≈152.6 ms) cumulative** cost, almost entirely from those. This is a corporate-Windows-laptop artifact (very likely a TLS-interception/certificate-patching tool installed by IT), not a Python defect — but it is *exactly* the kind of invisible cost that "the interpreter does some setup before your code runs" hides if you never look. Your own numbers will differ; that variability is the point, not a discrepancy to worry about.

```
$ python -X importtime app.py
import time: self [us] | cumulative | imported package
import time:       125 |        125 | winreg
import time:       243 |        243 |   _io
...
import time:       718 |     144690 |     wrapt
import time:       498 |     146427 |   certifi_win32.bootstrap
import time:       209 |        209 |   sitecustomize
import time:      1890 |     152569 | site
...
import time:       883 |       3302 | json          # <- our own `import json`, AFTER site finishes
{"a": 1, "b": 2}
```
Notice `json` (our script's own import) appears *after* `site` finishes — direct, observable confirmation that Main Interpreter Initialization (which concludes with `site`) really does complete before your script's own Stage 3/4 begin, exactly as PEP 432 specifies.

**Stage 2 — locating and reading the script (Day 5).** `Py_RunMain` sees `argv[1] = "app.py"` and opens it: an `openat()` syscall returns a fresh entry in the process's descriptor table (Day 5 §1.1) — not a small number like 3, because by this point Stage 1 has already `openat`'d and `close`'d dozens of shared libraries and encoding/`site` files (Day 2's own trace of ~480 `openat` calls for a bare `print('hi')` gives the right order of magnitude) — followed by `read()` calls pulling in the source bytes, then `close()`. **Deliberately non-obvious fact, worth stating plainly because it surprises people who assume "Python caches everything":** the `__main__` script — the file you named on the command line — is **never** written to or read from `__pycache__`. CPython always recompiles it from source, every single run. Only *imported* modules (our `json` import, below) get the `__pycache__/*.pyc` treatment. This is documented import-system behavior, not a bug: the main script is normally run at most once per process lifetime, so caching its bytecode buys nothing, whereas `json` (and everything it imports) might be re-imported across thousands of process starts.

**Stage 3 — tokenize → parse → compile (Day 6, applied to real source).** Compiling `app.py` walks Day 6's exact four-stage pipeline (tokenizer → PEG parser → AST → compiler), producing a code object. Compiling `import json`, by contrast, is where Stage 2+3 can *short-circuit*: CPython checks `__pycache__/json.cpython-312.pyc` (the exact filename pattern from [PEP 3147](https://peps.python.org/pep-3147/)) against the source file's recorded modification time and size (the default **timestamp-based** invalidation) or, opt-in, against a stored content hash (**hash-based** invalidation, [PEP 552](https://peps.python.org/pep-0552/)) — if it matches, CPython loads the already-compiled bytecode straight from disk and skips tokenizing/parsing/compiling entirely. This is the mechanism-level reason Day 6's cost model ("interpreter overhead only matters where there's no bigger wait to hide behind") doesn't even apply to most of your imports on the second and subsequent runs — the compiler never touches them again.

**Verified, real `dis` output for this exact `app.py`, on this machine (CPython 3.12.7, Windows):**
```
  1           0 RESUME                   0

              2 LOAD_CONST               0 (0)
              4 LOAD_CONST               1 (None)
              6 IMPORT_NAME              0 (json)
              8 STORE_NAME               0 (json)

  4          10 LOAD_CONST               2 ('counts')
             ...
             30 LOAD_CONST               4 (<code object summarize ...>)
             32 MAKE_FUNCTION            4 (annotations)
             34 STORE_NAME               4 (summarize)

  8          36 LOAD_NAME                5 (__name__)
             38 LOAD_CONST               5 ('__main__')
             40 COMPARE_OP              40 (==)
             44 POP_JUMP_IF_FALSE       18 (to 82)

  9          46 PUSH_NULL
             48 LOAD_NAME                6 (print)
             ...
             62 CALL                     1
             70 CALL                     1
             78 POP_TOP
             80 RETURN_CONST             1 (None)

Disassembly of <code object summarize ...>:
  4           0 RESUME                   0
  5           2 LOAD_GLOBAL              1 (NULL + json)
             12 LOAD_ATTR                2 (dumps)
             32 LOAD_FAST                0 (counts)
             34 LOAD_CONST               1 (True)
             36 KW_NAMES                 2 (('sort_keys',))
             38 CALL                     2
             46 RETURN_VALUE
```
This is Day 6's exact `LOAD_NAME`/`STORE_NAME` (module scope) vs `LOAD_FAST` (function scope) distinction, on live code. One detail Day 6 didn't have a name for and this note won't open further: `LOAD_ATTR 2 (dumps)` prints as `NULL + json` internally in 3.11+'s specializing adaptive interpreter (part of PEP 659) — a runtime optimization that rewrites hot bytecode into faster specialized forms after the first few executions. **Deliberate stop:** that machinery is real and load-bearing for why "Python is fast enough" more often than Day 6's naive cost model predicts, but opening it fully belongs with Day 6's own [WORKING]-tier "moving frontier" note, not here.

**Stage 4 — the `ceval` loop runs (Day 6 mechanism, Day 2 syscall trace, now literally the same event).** Execution proceeds top to bottom in `__main__`'s namespace: `IMPORT_NAME` triggers `json`'s own miniature Stage 2/3/4 (with its submodules `json.scanner`/`json.decoder`/`json.encoder` each independently cache-checked — this machine's importtime trace showed `json`'s own cumulative cost at **3,302 µs**, entirely separate from and *after* `site`'s 152 ms); `MAKE_FUNCTION` builds the `summarize` function object; the `if __name__ == "__main__":` guard runs (this is *why* that guard exists at all — Day 2's model of "the script always executes top to bottom" means without the guard, importing `app.py` from elsewhere would also print output); finally `print(summarize(...))` calls down through `sys.stdout` (a `TextIOWrapper` wrapping a `BufferedWriter`, Day 5's buffering stack) to a single `write(1, b'{"a": 1, "b": 2}\n', 18)` syscall — **this is Day 2's own traced `write(1, "hi\n", 3)` call, verbatim mechanism, different payload.** Everything before this one syscall — all ~1,900 of Day 2's traced syscalls for `openat`/`newfstatat`/`mmap`/`read`/`close`/`rt_sigaction` — happened in service of *reaching* this single line of output.

**Stage 5 — shutdown and exit (Day 2 + Day 4 + Day 5, converging).** Once the module body finishes (or an exception propagates unhandled to the top), `Py_RunMain` calls `Py_FinalizeEx()`: registered `atexit` callbacks run, `sys.stdout`/`sys.stderr` are flushed (more `write()` calls if anything was buffered and not yet flushed — Day 5's "your `write()` usually lands in the page cache, not on disk" applies here too, one layer up, at the Python buffer level), then every object still alive gets its reference count walked down (Day 4's refcounting) as the interpreter tears itself down. The kernel then reclaims the process: every remaining open file descriptor is dropped (Day 5 — the underlying open-file descriptions are closed, page-cache-dirty writes are *not* automatically `fsync`'d just because the process exited, which is precisely Day 5's fsync-vs-buffering distinction resurfacing), the process calls `exit_group(status)` (Day 2's exit-code table: `0` here, since nothing raised), and the parent shell's blocked `waitpid()` returns with that status, reaping the child before it can even become a zombie (Day 2 §1.6/§1.8), and prints your next prompt.

### Under the hood

The one sentence that makes all six stages cohere: **every stage exists to satisfy a precondition the next stage silently assumes.** `exec` can't run without `fork` giving it a process to become; the interpreter can't execute bytecode without `encodings` being importable (Stage 1) so it can even *decode* the source file it's about to open (Stage 2); the compiler can't run without a token stream (Stage 3, Day 6); the `ceval` loop can't produce output without descriptors 0/1/2 having survived, untouched, from Stage 0's `fork` all the way to Stage 4; and none of the memory allocated across all four prior stages is reclaimed until Stage 5's refcount teardown runs. This is also why the trap named in the Intuition section is real and not pedantic: a bug or a slow path in *any* stage is invisible from inside the stages after it — a `site.py` that takes 150 ms adds no bytecode, emits no exception, and shows up nowhere in `dis` output; you only find it by measuring wall-clock time and asking, empirically, which stage it went to (`-X importtime`, or Day 2's `strace`/ProcMon method applied to the whole run rather than one line).

### Case studies

**① Mercurial's `demandimport` — engineering directly against Stage 1/3's cost.** Mercurial (`hg`), a real, widely-used, Python-based version control system, is invoked constantly and briefly — the opposite usage pattern from a long-running server, and exactly the pattern where Stage 1–3 cost (not Stage 4 execution cost) dominates wall-clock time, since most invocations do a small amount of real work. Mercurial's core developers built `demandimport`, which rewrites Python's own import machinery at startup so that `import foo` returns a placeholder object immediately and only actually loads `foo` on first attribute access — deferring Stage 2/3's cost for every module the current command doesn't end up needing. This idea predates today's standardized approach: [PEP 690 (Lazy Imports)](https://peps.python.org/pep-0690/) and its successor [PEP 810 (Explicit Lazy Imports)](https://peps.python.org/pep-0810/) both cite Mercurial's implementation as prior art. The extracted, installable version of the same technique ([`demandimport` on PyPI](https://pypi.org/project/demandimport/), derived from Mercurial's code) reports startup-time reductions as high as 70% and memory reductions as high as 40% on real-world CLI tools that import far more than any single invocation uses. **The engineering lesson:** if your program's *dominant* cost is Stage 1–3, not Stage 4, the fix isn't "make the interpreter faster" (Day 6's escape-hatch ladder, which is about hot *execution* paths) — it's making imports lazy, because most of what a CLI imports on a given run, it never touches.

**② The Python 3.12 `os.fork()`-in-a-threaded-process hazard — an ongoing, not a single dated, failure.** Since Python 3.12, `os.fork()` on POSIX raises a `DeprecationWarning` — *"This process (pid=…) is multi-threaded, use of fork() may lead to deadlocks in the child"* — when CPython detects other threads are running at fork time. The mechanism is a direct, live instance of Day 3's own material: `fork()` copies only the calling thread into the child; any lock another thread happened to hold at that instant (including CPython's own internal locks — the import lock, the logging module's lock, memory-allocator locks) is copied *in its held state*, with no thread left alive in the child to ever release it. Deadlock, or a subtler silent corruption, follows the first time the child's single surviving thread needs that lock. This isn't a historical postmortem with one date — it's an ongoing engineering hazard that real, named, currently-maintained projects have independently hit and filed issues about after upgrading to 3.12: [`gunicorn`](https://github.com/benoitc/gunicorn/issues/3289), [`gevent`](https://github.com/gevent/gevent/issues/2052), [`buildbot`](https://github.com/buildbot/buildbot/issues/7276), and [`fail2ban`](https://github.com/fail2ban/fail2ban/issues/3766) all surface the same warning for the same underlying reason. Python's own `multiprocessing` module defaults to the `fork` start method on Linux today; **verify this before relying on it** — CPython's own roadmap has discussed moving the Linux default to `forkserver` in a future release specifically because of this class of bug. **The engineering lesson, applied directly to this note's own job-runner design below:** any service that is even incidentally multi-threaded (an async web framework using a thread pool for blocking calls, a logging handler with a background thread) must spawn worker subprocesses via `subprocess.Popen` (which uses `fork`+immediate `execve`, or `posix_spawn`, with no Python-level code — and therefore no Python-level lock — running in the gap) rather than `multiprocessing.Process(...).start()` with the default fork behavior, or must explicitly pass `mp.get_context("spawn")`.

### In production

- **Measure startup as a real, monitored number, not folklore.** `python -X importtime yourapp.py` (redirect stderr to a file — the trace prints there) is the single fastest way to find which import is eating your Stage 1–3 budget; treat "time from process start to first request served" as a p50/p99 metric on the same dashboard as request latency (Day 1's own lens — a latency budget doesn't stop mattering just because the "request" is your own process booting).
- **Cold start is a first-class production cost in serverless.** [AWS's own Lambda documentation and engineering blog](https://aws.amazon.com/blogs/compute/under-the-hood-how-aws-lambda-snapstart-optimizes-function-startup-latency/) describe exactly this trace happening on a fresh execution environment before your handler runs a single line: interpreter startup, then your module-level imports and client construction, then your handler. For interpreted runtimes this commonly adds on the order of 100 ms to low seconds depending on how much you import at module scope — *verify current figures, this area and its pricing evolve*. The standard mitigations are direct applications of this section: defer expensive imports/client construction out of module scope where safe, keep the import graph small, and (where available) use a snapshot/provisioned-concurrency feature that skips re-running Stages 0–3 entirely on a "warm" invocation.
- **CPython 3.11+ shrank Stage 1's floor for everyone.** [Python 3.11's "What's New"](https://docs.python.org/3/whatsnew/3.11.html) documents that core startup modules are now **frozen** — their compiled code objects are statically embedded in the interpreter binary itself, skipping the `__pycache__` read → unmarshal → heap-allocate steps this section's Stage 3 described, for a documented **10–15% faster interpreter startup**. Controllable via `-X frozen_modules=on|off` if you ever need to rule it out while debugging. *Verify against your installed version* — this is exactly the kind of version-specific fact that drifts.
- **Common mistake, junior → senior:** a junior engineer profiles a slow CLI tool with `cProfile` wrapping the whole run and is confused that "everything is slow, nothing stands out" — because `cProfile` measures Stage 4 execution, and the cost is sitting in Stage 1–3 (imports), which shows up as flat, undifferentiated time at the very top of the call stack. A senior engineer reaches for `-X importtime` *first*, specifically because it's the tool built for the layer `cProfile` isn't designed to break down.
- **Failure mode:** a `sitecustomize.py` or `.pth` file dropped into `site-packages` by some other tool (a corporate security agent, a monitoring vendor's "auto-instrumentation" install, a stale package leftover from a previous environment) silently adds Stage 1 cost to *every* Python process on the machine, including ones that never asked for it — as this very note's own measurements showed. Diagnose with `-X importtime`; the fix is almost always "delete or gate the `sitecustomize.py`/`.pth` file," not "optimize your own code," and the diagnosis is easy to skip past because nothing about it appears in your own source.

**Deliberate stops, named on purpose:** this section does not open the dynamic linker's symbol-resolution work that happens before `main()` even runs (loading `libpython3.x`, resolving its symbols) — that's OS-loader territory, one more layer below Day 2's fork/exec, and it's a black box until a project specifically forces you into it (usually via a linking error, not a Python one). It also does not re-open the GIL, threads, or `asyncio`'s event loop — Day 3 named the GIL and deferred it to Day 20; this note keeps that promise.

---

## System design ① — a code-execution sandbox for an agent's tool call

**The problem.** Your agent service exposes a `run_python(code: str) -> str` tool: the model can emit arbitrary Python as part of a tool call, and your backend must execute it and return stdout/stderr to the model, inside a fixed 10-second tool-timeout budget, without that code being able to (a) survive past the call, (b) exhaust the host's RAM, (c) exhaust the host's file descriptors or hang other requests, (d) reach the network, or (e) leave orphaned processes behind if it times out. Concrete scale: an API-worker fleet where each worker handles ~50 concurrent agent sessions, on hosts with 8 GB RAM and a default 1024 FD `ulimit`.

**Realistic alternatives, and the one already ruled out.** Day 2's own §1.10 already worked this exact problem down to a tier table and ruled out Tier 0 (`eval()`/`exec()` in-process — "never, not a sandbox") and named Tier 3 (a Firecracker/gVisor microVM per execution) as the answer *only* once the code is hostile, multi-tenant, product-facing input. This design doesn't re-derive either of those; it takes Day 2's own chosen middle answer — **Tier 2: a separate process, an unprivileged uid, a read-only root filesystem, no network, seccomp-filtered, with rlimits and a timeout** — and fills in the four mechanisms Day 2 explicitly deferred: process isolation's *safe spawning*, memory caps, FD caps, and network restriction.

**Decision, mechanism by mechanism:**

- **Spawning the child (Days 2 + this note's own fork-safety case study).** An agent server is realistically async and/or thread-pool-backed (Day 3's C10K motivation is *why* it's built that way). That makes it multi-threaded at the moment a tool call fires — exactly the condition this note's `os.fork()` case study warns about. The decision is therefore `subprocess.Popen`, not bare `os.fork()` and not `multiprocessing.Process` with the default fork context: `Popen` performs `fork`+immediate `execve` (or `posix_spawn`) with no Python bytecode running in the gap, so there is no Python-level lock for a stalled sibling thread to be holding.
- **Memory cap (Day 4, reused, not re-derived).** Day 4 gave two mechanisms with opposite failure characters: `resource.setrlimit(RLIMIT_AS, ...)` yields a *catchable* `MemoryError` inside the child, cheap to set, but only bounds virtual address space of that one process and — Day 4's own explicit warning — is not the same guarantee as a container's memory limit. The cgroup-backed limit (`docker run -m`, or a raw `memory.max` write) yields an *uncatchable* `SIGKILL`, which is what actually happens in production containers. The decision: treat the cgroup/container limit as the **enforced boundary** (it's real, it doesn't depend on the sandboxed code cooperating), and additionally set a tighter `RLIMIT_AS` inside the child as a cheap early warning so well-behaved-but-buggy code gets a clean `MemoryError` traceback back to the model instead of a bare `SIGKILL` with no diagnostic. **Verified honestly on this note's own machine:** the `resource` module does not exist on native Windows Python at all (`ModuleNotFoundError: No module named 'resource'`) — this mechanism, and the two below, require the sandbox host to be Linux, whether that's WSL2, a Linux container, or a Linux VM. On Windows-native hosts, the nearest OS-level equivalent is a [Job Object](https://learn.microsoft.com/en-us/windows/win32/procthread/job-objects) (`CreateJobObject` + `JOBOBJECT_EXTENDED_LIMIT_INFORMATION` for a memory cap, `TerminateJobObject` to kill the whole tree) — named here as [AWARE]: treat it as a black box unless a Windows-only deployment target forces you to open it.
- **FD cap (Day 5, reused).** Day 5's own `fds.py` demonstrated the exact API: `resource.getrlimit`/`setrlimit(RLIMIT_NOFILE, (soft, hard))`, observed tripping `EMFILE` after the soft limit was exhausted. Applied here unchanged, on the child, before it can open a single file of its own.
- **Network restriction (genuinely new — none of Days 1–7 covered this).** The strongest, cheapest real enforcement at this tier is a network namespace containing no interface but loopback — the one-line version most people actually reach for is Docker's `--network none`, which does precisely this. Where you can't create a namespace (no root, no container runtime), the fallback is a seccomp filter that denies the `socket()` syscall for `AF_INET`/`AF_INET6` — weaker in that it blocks *new* connections rather than removing network reachability at the kernel-routing level, but still a real, syscall-level enforcement rather than an environment-variable convention (which a sandboxed process could simply ignore).
- **Timeout and cleanup (Day 2 §1.11's shape, inverted).** Day 2's graceful-shutdown pattern is `SIGTERM → drain → deadline → SIGKILL`, built for *cooperative* workers you trust to clean up. A sandboxed child is, by definition, not trusted to cooperate — the correct inversion is a very short or zero grace period, and killing the **process group**, not just the one PID (`os.killpg(pgid, signal.SIGKILL)`, after starting the child with `start_new_session=True` so it gets its own process group) — otherwise a child that itself forked a grandchild before the timeout fired leaves that grandchild running, orphaned, exactly the failure Day 2's own zombie/orphan material describes.

**A safer variant of the naive rlimit-setting approach, worth naming explicitly.** The obvious way to set the child's rlimits is `subprocess.Popen(..., preexec_fn=lambda: resource.setrlimit(...))` — but Python's own `subprocess` documentation states plainly that `preexec_fn` "is not safe to use in the presence of threads in your application," for the identical root cause as this section's fork-safety case study: it runs Python code in the forked-but-not-yet-`exec`'d child of a possibly multi-threaded parent. The safer alternative on Linux is to let the **parent** set the limits on the child *after* `Popen` returns a real PID, using `resource.prlimit(child.pid, resource.RLIMIT_AS, (limit, limit))` — no code runs inside the fork/exec gap at all. *Verify `resource.prlimit`'s exact signature against `docs.python.org/3/library/resource.html` for your Python version before shipping this* — it wraps the Linux-specific `prlimit(2)` syscall and this note could not execute-verify it live (no Linux environment was available on the machine this note was written on).

```python
# sandbox_run.py — Linux/WSL2 only (uses resource, subprocess process groups).
# Not executed in this session (no WSL2/Docker available on the authoring
# machine) — verify against your own Linux/WSL2 environment before trusting it.
# pip install: nothing extra — stdlib only.
import os
import resource
import signal
import subprocess

MEM_LIMIT_BYTES = 256 * 1024 * 1024
FD_LIMIT = 64
TIMEOUT_S = 10


def run_sandboxed(code: str) -> tuple[str, str, int]:
    proc = subprocess.Popen(
        ["python3", "-I", "-c", code],   # -I: isolated mode, ignores env/site tweaks
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE,
        start_new_session=True,          # own process group -> killpg reaches every child
        env={},                          # no inherited secrets in the environment
    )
    try:
        # Parent sets limits on the already-running child: no code executes
        # in the fork/exec gap, sidestepping the preexec_fn hazard above.
        resource.prlimit(proc.pid, resource.RLIMIT_AS, (MEM_LIMIT_BYTES, MEM_LIMIT_BYTES))
        resource.prlimit(proc.pid, resource.RLIMIT_NOFILE, (FD_LIMIT, FD_LIMIT))
        stdout, stderr = proc.communicate(timeout=TIMEOUT_S)
        return stdout.decode(), stderr.decode(), proc.returncode
    except subprocess.TimeoutExpired:
        os.killpg(os.getpgid(proc.pid), signal.SIGKILL)   # kill the whole group, not just proc.pid
        proc.communicate()  # reap it — avoid a zombie, per Day 2
        return "", "sandboxed code timed out", -9
```

**Failure modes, one per mechanism above:**

| Attack / bug | What stops it |
|---|---|
| Infinite loop | `TIMEOUT_S` + `os.killpg(..., SIGKILL)` on the whole process group |
| Memory bomb (`x = [0] * 10**12`) | `RLIMIT_AS` (catchable `MemoryError`) or the outer cgroup limit (uncatchable `SIGKILL`) |
| Fork bomb inside the sandbox | Not covered by the mechanisms above — needs a cgroup `pids.max` controller or a non-root uid's process quota; named here as a genuine gap this design leaves open, not silently assumed away |
| FD exhaustion (`while True: open(...)`) | `RLIMIT_NOFILE` |
| Network exfiltration | `--network none` / netns, or seccomp-denied `socket()` |
| Huge stdout floods the pipe buffer | `Popen.communicate()` reads both streams concurrently precisely to avoid the classic hand-rolled deadlock (write to a full pipe blocks the child while you're blocked reading a different, empty one) — a naive `proc.stdout.read()` followed by `proc.stdin.write()` can deadlock exactly this way; this is Day 5's pipe-buffering material, one layer up |

**Trade-off and flip condition.** Tier 2 (this design) costs on the order of tens of milliseconds to spin up and shares the host kernel with the sandboxed code — a kernel exploit escapes everything. Tier 3 (Firecracker/gVisor) pays real boot latency (Firecracker's own published figures are in the low hundreds of milliseconds; *verify current numbers*) and real operational complexity (a VMM, image management) in exchange for kernel-level blast-radius containment. Day 2's own flip condition still governs the top-level choice — "your own model, your own prompt, blast radius is one tenant's scratch directory" stays at Tier 2; arbitrary user-submitted code as a multi-tenant product flips you to Tier 3. This design adds one more, concrete flip condition specific to this reader's own verified environment: **if your worker fleet runs on Windows-native hosts without WSL2 or a container runtime, this entire design is unbuildable as specified** — `resource`, network namespaces, and seccomp simply don't exist there. The fix isn't a workaround; it's putting the sandbox host on Linux (a WSL2 environment, or a Linux container even if the rest of the stack is Windows), which is exactly why this plan's own Day-0 environment setup asked you to install WSL2 in the first place.

---

## System design ② — a single-machine job runner: queue on disk, worker processes, crash recovery

**The problem.** Build a local background-job runner — the kind that might run behind an agent's "process these 500 documents" background task — on one machine: a fixed pool of worker **processes** pulls jobs from a queue; a submitted job must never be silently lost, even across a worker crash (OOM-killed, `kill -9`, a bug) or a full machine power loss and reboot; and two workers must never both believe they own the same job. Concrete scale: 4 workers on one host, jobs arriving as small JSON payloads, a modest ~50 jobs/sec target — deliberately small enough that "add a real broker" would be over-engineering for the exercise, and deliberately durable enough that "just use an in-memory queue" would fail the actual requirement.

**Alternative already covered, and why it's rejected here.** Day 3's §1.8 already designed an in-process background job scheduler with priority and starvation prevention (`queue.PriorityQueue` + aging), and it's a genuinely good design — for its stated scope. That scope assumes the process itself never dies: the queue lives in one process's heap (Day 4), so the instant that process is `kill -9`'d, every job in it — queued or mid-execution — vanishes with it, no different in kind from Day 4's own "a Python memory leak is a lingering reference" logic applied to jobs instead of bytes. That's an acceptable trade for a scheduler embedded in a server you don't expect to crash-and-lose-work-silently; it fails this problem's explicit "never lose a submitted job" requirement, so it isn't reused here, and the two designs solve genuinely different problems (fairness among *unequal* jobs in one long-lived process vs. surviving the *process itself* dying).

**The alternative a competent engineer would actually reach for.** A real broker — Redis Streams' consumer groups, RabbitMQ, or Postgres-as-a-queue — solves this correctly and is what you'd actually ship. It's deliberately not the answer built here, because the plan scopes today as single-machine specifically so you build the primitive by hand once, with nothing but a filesystem, and *feel* which guarantees a broker is quietly giving you for free — you'll meet the distributed, multi-machine version of this exact problem on Day 49.

**Decision: a durable log for intake, a directory-based lease scheme for ownership.**

Reuse Day 5's own §1.7 system design almost unchanged for the *intake* half: an append-only file, one JSON line per submitted job, `fsync`'d before the submitter is told "accepted" — structurally identical to "append + fsync before ack is the durability primitive," just with a job record instead of a database row. Day 5 already measured the cost of this pattern (300 records: 0.9 ms with no `fsync` at all vs. 251.7 ms `fsync`-per-record vs. 7.4 ms group-committing every 50 — reused here directly, not re-benchmarked) and already worked out the failure table for a torn write (CRC + length check, truncate at the first bad record) — none of that changes for a job payload instead of an order record.

The new half is *ownership*: which of 4 worker processes gets to run a given job, and what happens if that worker dies mid-job. The mechanism is a directory layout plus one POSIX guarantee:

```
queue/pending/<job_id>.json     # submitted, unclaimed
queue/claimed/<job_id>.json     # {"worker_pid": ..., "lease_expires": ...}
queue/done/<job_id>.json        # completed
```

A worker claims a job with a single `os.rename("pending/<id>.json", "claimed/<id>.json")`. `rename(2)` on a POSIX filesystem is atomic with respect to every other process looking at that path — there is no instant where the name exists twice or doesn't exist at all — so if two workers race to claim the same job, **exactly one `rename` succeeds; every other caller gets `FileNotFoundError` because the source path is already gone.** This is not a novel trick: it's the exact mechanism behind the [Maildir format](http://qmail.org/man/man5/maildir.html), invented by Daniel J. Bernstein for qmail in 1995 specifically to guarantee a mail message is delivered exactly once even with multiple concurrent readers — write to `tmp/`, then `rename()` into `new/`, and the atomicity of that one syscall is the entire correctness argument.

**Verified, real, on this note's own machine** (this code needs no `resource`/`unshare`/cgroups, so unlike System design ①'s code it runs identically on Windows, WSL2, or Linux — it was executed for this note): eight worker processes, spawned with `multiprocessing`'s **`spawn`** start method — deliberately, applying this note's own fork-safety case study rather than defaulting to `fork` — all raced to `os.rename()` the same pending job file:

```python
import multiprocessing as mp
import os


def try_claim(worker_id, result_queue):
    src = os.path.join("queue", "pending", "job1.json")
    dst = os.path.join("queue", "claimed", f"job1.worker{worker_id}.json")
    try:
        os.rename(src, dst)
        result_queue.put((worker_id, "WON", None))
    except FileNotFoundError as e:
        result_queue.put((worker_id, "LOST", str(e)))


if __name__ == "__main__":
    mp.set_start_method("spawn")
    q = mp.Queue()
    procs = [mp.Process(target=try_claim, args=(i, q)) for i in range(8)]
    for p in procs:
        p.start()
    for p in procs:
        p.join()
    results = [q.get() for _ in procs]
    print(f"winners: {[r for r in results if r[1] == 'WON']}")
    print(f"losers: {len([r for r in results if r[1] == 'LOST'])}")
```
```
$ python claim_race.py
winners: [(1, 'WON', None)]
losers: 7
```
Eight real, separate OS processes; exactly one winner, every time this was run. No lock, no database, no coordination beyond a single filesystem syscall — because the coordination is happening one layer down, in the kernel's filesystem-namespace code, which is precisely why Day 3's `threading.Lock` machinery is the wrong tool to even reach for here: a `Lock` protects one process's shared memory, and once "shared state" means eight separate address spaces (Day 2) with no shared memory at all, an in-process lock literally cannot reach across them — the filesystem is the only thing all eight workers actually share.

**Crash recovery.** Every claim carries a `lease_expires` timestamp (e.g., now + 60 s). A worker's job loop periodically `os.utime()`s its own claim file to push the lease forward while it's genuinely still working — cheap, and crash-safe by construction: if the worker is `kill -9`'d (Day 2), OOM-killed (Day 4), or the whole machine loses power mid-job, it simply stops renewing, and the lease silently expires. Before pulling a new job, any idle worker first scans `claimed/` for expired leases and `rename()`s them back to `pending/` — the exact same atomic primitive, run in reverse. No heartbeat channel, no separate monitor process, no additional failure mode to reason about beyond "does the lease file's timestamp keep moving."

**At-least-once, not exactly-once — and why that's an explicit, not accidental, guarantee.** A job re-claimed after a crash may run twice: the previous worker could have died *after* producing a real-world side effect (charged a card, sent an email) but *before* writing to `done/`. This is Day 5's own §2.3 lesson, reused rather than re-derived: "the intent survives; the result never happened… reconcile by idempotency key… do not blindly retry." The job runner's contract with every job handler is therefore that handlers must be idempotent (keyed by `job_id`) or must check `done/<job_id>.json` for a prior completion marker before producing an external effect — the same reconciliation Day 5 already worked out for a single durable log, now the runner's standing requirement for every job type.

**Worker lifecycle.** Workers are spawned once at startup via `subprocess.Popen` (Day 2's fork+exec, this note's Case study ② applied directly: never bare `multiprocessing.Process` with the default fork context from a supervisor that might itself be threaded). Sizing the pool at 4 for a mixed CPU/I/O workload is a judgment call Day 3's Little's Law (§1.7) doesn't directly answer — that scenario sized threads for pure I/O wait, not worker processes doing real CPU work, so the honest answer here is "start at core count, measure, adjust," not a formula reused wholesale. Shutdown reuses Day 2's §1.11 pattern unchanged: `SIGTERM` to each worker, a grace period to finish or voluntarily release its current lease, `SIGKILL` at the deadline.

**Trade-off.** This design costs zero new infrastructure and forces genuine understanding of every guarantee it provides — but you now personally own tuning the lease timeout (too short duplicates work on slow jobs; too long delays recovery after a real crash) and the reaper's cost, which is an `O(n)` directory scan over `claimed/` — the same shape of problem `select`/`poll` had before `epoll` (Day 5 §1.6) — and stops scaling comfortably somewhere around the low thousands of pending files on one directory/filesystem. It also gives you none of a real broker's built-in tooling: no dead-letter queue, no cross-machine fanout, no transactional exactly-once outbox.

**Flip condition.** The moment you need more than one machine, or job volume that makes an `O(n)` directory scan a real bottleneck, or producers/consumers in more than one language, move to a real broker — and when you do, you'll recognize every mechanism in this design under a different name: SQS's *visibility timeout* is this design's lease expiry; a Redis Streams *consumer group*'s pending-entries list is the `claimed/` directory; Kafka's *consumer offset commit* is the `done/` marker. That recognition — not the code itself — is the actual point of building this by hand once.

---

## Failure modes & common misconceptions

| Misconception | Reality |
|---|---|
| "Python caches compiled bytecode for every file automatically." | Only *imported* modules get `__pycache__/*.pyc` caching. The `__main__` script named on the command line is recompiled from source on every run, always (PEP 3147). |
| "`preexec_fn` is the normal way to set a sandboxed child's rlimits." | Python's own docs call it unsafe under threads, for the same reason `os.fork()` itself is now deprecated in a threaded process — code running in the fork/exec gap can deadlock on a lock a sibling thread was holding. Use `resource.prlimit()` from the parent after `Popen` returns instead. |
| "A `Lock` will keep two workers from claiming the same job." | A `threading.Lock` only coordinates threads sharing one process's memory. Worker *processes* share no memory (Day 2) — coordination has to go through something all of them can see, like the filesystem's atomic `rename()`. |
| "If the tool timed out and I killed the PID, the sandbox is clean." | Killing only the top-level PID leaves any grandchild the sandboxed code forked before the timeout still running, orphaned. Kill the whole process group (`os.killpg`), not one PID. |
| "Interpreter startup is a fixed, small constant I don't need to think about." | It's dominated by whatever happens to be sitting in `site-packages` on that specific machine — this note measured a ~140 ms `site`-import cost from packages the script never even imports. Measure with `-X importtime`; don't assume. |
| "An in-memory priority queue with aging (Day 3) is durable enough for background jobs." | It's durable against *starvation*, not against the process holding it being killed. Durability against a crash needs something that survives the process — a disk-backed log, at minimum. |

---

## Topic-wide wrap-up

### Glossary

- **Atomic rename** — a `rename(2)` call on a POSIX filesystem, guaranteed to never leave the destination path in a partially-renamed or duplicated state, and to remove the source path the instant it succeeds — the property this note's job-runner design uses as its only concurrency primitive.
- **Cold start** — the latency added when a program (or serverless function) must run its full interpreter-startup trace (Stages 0–3 of this note) because no already-initialized process was available to reuse it.
- **Core initialization** — the second phase of CPython startup (PEP 432): the interpreter state, GIL, and built-in singletons are created and `encodings` is imported, but `sys.path` and `site` are not yet finalized.
- **Fork-safety hazard** — the risk that `fork()`ing a multi-threaded process copies only the calling thread into the child, potentially copying a lock in its held state with no thread left alive to release it; the root cause behind Python 3.12's `os.fork()` deprecation warning and `subprocess`'s `preexec_fn` warning.
- **Frozen module** — a module whose compiled code object is statically embedded in the CPython interpreter binary itself (as of 3.11, for core startup modules), skipping the `__pycache__` read/unmarshal step entirely.
- **Hash-based `.pyc` invalidation** — an opt-in cache-validity check (PEP 552) that compares a stored content hash of the source file, instead of the default timestamp+size check, to decide whether a cached bytecode file is still valid.
- **Lease** — a time-bounded claim on a resource (here, a job file) that a holder must periodically renew to keep, and that automatically becomes available to others if the holder stops renewing (e.g., because it crashed) — the mechanism this note's job runner uses for crash recovery, analogous to a DHCP lease or a distributed lock's TTL.
- **Main interpreter initialization** — the third phase of CPython startup (PEP 432): `sys.path` is computed and the `site` module runs (processing `.pth` files and `site-packages`), concluding with the interpreter fully "Initialized."
- **Process group** — a set of processes (typically a parent and any children it spawned into the same group) that can be signaled together with one call (`killpg`), used here so a timed-out sandboxed process's own children are killed along with it.

### Cheat Sheet

| Stage / Design | Day(s) it draws on | The one new thing Day 8 adds |
|---|---|---|
| Stage 0 — fork+exec | Day 2 | Nothing new — reused verbatim as the entry point |
| Stage 1 — interpreter startup | (none — genuinely new) | PEP 432 phases, `site`/`encodings` ordering, measured cold-start cost |
| Stage 2 — locate/read script | Day 5 | The `__main__`-is-never-cached honesty caveat |
| Stage 3 — compile | Day 6 | Applied to real source + the `__pycache__` short-circuit for imports |
| Stage 4 — execute | Day 2, Day 6 | Showing `print()` and Day 2's traced `write()` are the same event |
| Stage 5 — shutdown | Day 2, Day 4, Day 5 | Showing all three converge at one `exit_group()` |
| Sandbox design | Days 2, 4, 5 | Safe spawning (fork-safety), network restriction (new), `prlimit` over `preexec_fn` |
| Job runner design | Days 2, 3, 5 | Atomic-rename ownership + lease-based crash recovery (new) |

### Build This

A single, cumulative definition of done:

1. Run `python -X importtime yourapp.py 2> importtime.log` on a real script of yours; find the single most expensive import that isn't obviously load-bearing for the code path you just ran, and defer it (move the `import` inside the function that needs it). Re-measure and record the delta.
2. Reproduce this note's `dis` output on your own small function with an `import` inside a module — confirm for yourself that module-scope uses `LOAD_NAME`/`STORE_NAME` and function-scope uses `LOAD_FAST`.
3. In a WSL2/Linux environment, implement `sandbox_run.py` from System design ① against a real snippet of untrusted-looking code (e.g. `while True: pass`, then `[0] * 10**12`, then `open("/etc/passwd")` with `--network none`-style isolation faked via `preexec_fn`-free `RLIMIT` settings) and confirm each attack is actually stopped, not just designed on paper.
4. Reproduce this note's `claim_race.py` exactly (it needs no special OS features — it will run on Windows, WSL2, or Linux); confirm exactly one winner across at least 3 separate runs; then extend it into the full job-runner: an incoming log with `fsync`-before-ack (reuse your Day 5 build), a `claimed/`-directory lease with expiry, and a deliberate `kill -9` of a worker mid-job that you then observe get requeued and re-run by a survivor.
5. Do the teach-backs and cross-day synthesis questions from the consolidation table above, aloud, before checking this note again.

### Active Recall & Self-Test (answer from memory)

1. Name, in order, the four PEP 432 interpreter states between "nothing exists yet" and "your script starts running." What can and can't you import in each?
2. Why is the `__main__` script never cached to `__pycache__`, while a module it imports is?
3. In Day 2's traced `python -c "print('hi')"`, roughly how many syscalls ran before the one `write()` that corresponds to your own code? What were most of the rest doing?
4. Why does `subprocess.Popen` sidestep the fork-safety hazard that plain `os.fork()` (and `preexec_fn`) don't?
5. Which memory-cap mechanism from Day 4 is catchable inside the sandboxed process, and which is not — and which one is this note's sandbox design treating as the *enforced* boundary?
6. What single POSIX guarantee does the job-runner design rely on to prevent two workers from claiming the same job, and why doesn't a `threading.Lock` provide the same guarantee here?
7. What does a job's "lease" expiring actually detect, mechanically — and why does it require no separate heartbeat channel?
8. Why is the job runner described as "at-least-once," not "exactly-once" — and what must a job handler do about that?
9. Name one thing this note measured that will differ meaningfully on your own machine, and explain why it would differ.
10. **60-second teach-back:** without looking at this note, narrate the six stages of `python app.py`'s life out loud, naming which of Days 1–7 each stage belongs to.

### Spaced-Repetition Flashcards

- Q: What are CPython's four PEP 432 startup phases? → A: Uninitialized → Runtime Pre-Initialization → Main Interpreter Initialization → Initialized.
- Q: Which module must import successfully during core initialization, before almost anything else can happen? → A: `encodings`.
- Q: Does the script named on the command line ever get a `__pycache__/*.pyc` file? → A: No — only imported modules do; the main script is always recompiled from source.
- Q: What CPython 3.11 change reduced interpreter startup by 10–15%? → A: Freezing core startup modules' code objects directly into the interpreter binary.
- Q: What warning did Python 3.12 add, and why? → A: A `DeprecationWarning` on `os.fork()` in a multi-threaded process, because only the forking thread is copied, risking a deadlock on a lock a dead sibling thread was holding.
- Q: What POSIX property makes `os.rename()` usable as a distributed job-claiming lock? → A: It's atomic — the source path is removed the instant the rename succeeds, so only one of several racing callers can win.
- Q: What decades-old real system uses exactly the same atomic-rename trick? → A: Maildir (qmail, 1995) — deliver to `tmp/`, then `rename()` into `new/`.
- Q: Why is `preexec_fn` unsafe for setting a sandboxed child's rlimits? → A: It runs Python code in the fork/exec gap of a possibly multi-threaded parent — the same hazard as bare `os.fork()`.
- Q: What's the safer alternative to `preexec_fn` for setting a child's rlimits? → A: `resource.prlimit(child.pid, ...)`, called from the parent after `Popen` returns — no code runs in the fork/exec gap.
- Q: In the job runner, what makes crash recovery work without a separate heartbeat process? → A: A lease's expiry timestamp — a dead worker simply stops renewing it, and it silently expires.

### Primary Sources

- [PEP 432 — Restructuring the CPython Startup Sequence](https://peps.python.org/pep-0432/)
- [PEP 587 — Python Initialization Configuration](https://peps.python.org/pep-0587/)
- [PEP 3147 — PYC Repository Directories](https://peps.python.org/pep-3147/)
- [PEP 552 — Deterministic pycs](https://peps.python.org/pep-0552/)
- [PEP 690 — Lazy Imports](https://peps.python.org/pep-0690/) and [PEP 810 — Explicit Lazy Imports](https://peps.python.org/pep-0810/) (Mercurial's `demandimport` cited as prior art)
- [What's New In Python 3.11 — frozen modules](https://docs.python.org/3/whatsnew/3.11.html)
- `execve(2)`, `rename(2)`, `prlimit(2)` — Linux man pages (`man7.org` or your distribution's `man` pages)
- [Python `subprocess` module docs — `preexec_fn` warning](https://docs.python.org/3/library/subprocess.html)
- [Python `resource` module docs](https://docs.python.org/3/library/resource.html)
- [`maildir(5)` — qmail.org](http://qmail.org/man/man5/maildir.html)
- [AWS Compute Blog — under the hood: how AWS Lambda SnapStart optimizes function startup latency](https://aws.amazon.com/blogs/compute/under-the-hood-how-aws-lambda-snapstart-optimizes-function-startup-latency/) (*verify current cold-start figures and pricing — this drifts*)
- [Microsoft Learn — Job Objects](https://learn.microsoft.com/en-us/windows/win32/procthread/job-objects)
- Real GitHub issues cited for the fork-safety case study: [gunicorn#3289](https://github.com/benoitc/gunicorn/issues/3289), [gevent#2052](https://github.com/gevent/gevent/issues/2052), [buildbot#7276](https://github.com/buildbot/buildbot/issues/7276), [fail2ban#3766](https://github.com/fail2ban/fail2ban/issues/3766)

### Layered explanations

- **10 seconds:** Running a Python program isn't one event — it's a fixed sequence (spawn a process, start the interpreter, read and compile your source, execute it, tear it down) that every one of Days 2–6 taught you one piece of; today's job is seeing the whole sequence at once and using it to design a code sandbox and a crash-safe job queue.
- **1 minute:** `python app.py` triggers, in order: the shell forking and `exec`ing the interpreter (Day 2); the interpreter's own multi-phase startup, importing `encodings` then `site` before it can even see your script (new today, and measurably ~140 ms on a cluttered machine); opening and compiling your script fresh every time, while any modules it imports get compiled once and cached to `__pycache__` (Days 5/6); executing bytecode top to bottom until your `print()` becomes the exact same `write()` syscall Day 2 traced; and a shutdown that unwinds references, closes descriptors, and exits with a status your shell reaps. That same sequence, run as a subprocess instead of by a human, is what a sandboxed agent tool call and a background job worker both are — which is why this day's two system designs (a code sandbox combining Days 2–5's isolation mechanisms, and a disk-backed job runner combining Day 5's durability primitive with Day 2's process model) are really just this trace pointed at two real problems.
- **5 minutes:** The six-stage trace matters because each stage silently assumes the previous one succeeded, and a slow or buggy stage is invisible from every stage after it — a bloated `site-packages` install adds no exception and no visible bytecode, only wall-clock time, which is why `-X importtime` (not `cProfile`, which only sees post-import execution) is the right tool for diagnosing it. The two system designs are worked examples of the same discipline: the sandbox takes Day 2's already-chosen isolation tier and fills in the specific mechanisms (safe process spawning that avoids the fork-in-threads deadlock hazard now formalized in Python 3.12's own `os.fork()` deprecation warning, `RLIMIT_AS`/cgroups for memory, `RLIMIT_NOFILE` for descriptors, network namespaces or seccomp for egress) that Days 2–5 individually named but didn't combine. The job runner takes Day 5's write-ahead-log durability primitive and Day 2's process model and adds exactly one new idea — an atomic-rename-based lease, the same 1995 mechanism Maildir uses — to get crash-safe job ownership across multiple worker processes without any shared memory or locks, explicitly trading exactly-once for at-least-once-plus-idempotency, and explicitly deferring to a real broker (Day 49) the moment the problem outgrows one machine.
- **Expert summary:** CPython's startup sequence formalized by PEP 432/587 (Uninitialized → Runtime Pre-Init → Main Interpreter Init, concluding with `site`, → Initialized) sits upstream of, and gates, the compile pipeline and `ceval` execution model covered independently on Day 6; the `__main__` module is exempted from PEP 3147/552's bytecode-caching scheme entirely, making it the one code path recompiled unconditionally on every invocation. Fork-safety (the root cause behind `os.fork()`'s 3.12 deprecation warning and `subprocess`'s `preexec_fn` warning alike — a single non-forking thread's held lock surviving, unreleased, into a child with no thread left to release it) is the single mechanism connecting Day 2's process model, Day 3's lock semantics, and both of this day's system designs' choice to spawn via `Popen`/`spawn` rather than bare `fork`. The job-runner design's ownership primitive — atomic `rename(2)` as a lock substitute across processes with no shared memory — is the same 1995 Maildir mechanism underlying modern message-queue delivery semantics, making the exercise's real payoff the recognition, on Day 49, of SQS visibility timeouts and consumer-group PELs as the same primitive under a different name.

---

*End of Day 8. Next: Day 9 — What a network is: packets, frames, and switches.*
