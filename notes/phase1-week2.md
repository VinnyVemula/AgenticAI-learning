# Phase 1 — Week 2: The Concurrency Model & the Raw ReAct Loop

> **What this covers:** how one computer does many things at "once" — processes, threads, coroutines, the GIL, and the asyncio event loop (backend) — and how to hand-build an agent from nothing: the ReAct loop with real function calling (agentic). Then: why they are the *same* workload.
> **Prerequisite:** [Phase 0](phase0-week1.md) — the toolchain, and the two mental models (LLM = stateless function; agent = `while` loop).
> **The one idea that unites this week:** *the agent loop is an async, I/O-bound workload — every step is a network wait — so the event loop is not a backend detail an agent-builder may skip; it is the engine the loop runs on.*
>
> **How to read this:** every concept moves through the same ladder — **intuition → analogy → a concrete worked example → a diagram → the formal/under-the-hood truth**. If a concept ever feels abstract, jump to its "**Example**" block; that is where it becomes real.
>
> **Depth tiers:** **[CORE]** = open every black box · **[WORKING]** = use it, know its tradeoffs, internals stay a named black box · **[AWARE]** = know it exists and when to reach for it.

---

# PART 1 — BACKEND: The Concurrency Model

## Overview & motivation — the problem before concurrency

**The intuition.** Picture one cashier at a supermarket. A customer's card machine is slow — it takes 30 seconds to authorize. During those 30 seconds the cashier just… stands there. Behind them, a queue of ten people, each holding a single apple, waits. The cashier is not *busy* — they are *blocked*, waiting on a machine. The store's throughput has collapsed not because the work is hard, but because one person is idle while others could be served.

This is *exactly* the situation of a web server or an AI agent. The overwhelming majority of their life is spent **waiting on the network** — waiting for a database, an external API, or a large language model to respond. A naive server, like the naive cashier, freezes during every wait. **Concurrency** is the discipline of *not standing idle*: serving customer B while customer A's card is authorizing.

**Why there are three different tools (this is the crux).** "Doing more at once" is not one problem — it is two, and they need opposite solutions:

- **I/O-bound work** — the bottleneck is *time spent waiting* for something external (network, disk). You do **not** need more cashiers; you need the one cashier to *stop standing idle during the wait*. → the fix is **coroutines / async**.
- **CPU-bound work** — the bottleneck is *raw calculation* (resizing 10,000 images, hashing a huge file). Here the cashier is genuinely working the whole time; the only way to go faster is **more cashiers working in parallel**. → the fix is **processes**.

> **The single most important sentence in Part 1:** *the right concurrency tool depends entirely on whether your bottleneck is waiting (I/O) or computing (CPU).* Confusing the two is the root of most Python performance disasters.

Everything below is an unpacking of that sentence.

---

## 1. Processes, threads, coroutines — the three units of execution **[CORE]**

### Intuition & first principles

- A **process** is a running program with its **own private memory** the operating system fences off from every other process. Think of it as a *separate house* — its own rooms, its own furniture, no shared walls.
- A **thread** is an independent stream of execution *inside* one process. Threads of the same process **share that process's memory** — like *people living in the same house*, sharing the same fridge and rooms. Cheap to coordinate, but if two people write on the same whiteboard at once, you get garbage (a **race condition**).
- A **coroutine** is a single function that can **pause itself** at marked points, hand control back to a scheduler, and later **resume exactly where it paused** — all inside *one thread*. Think of *one very organized person* who, whenever they hit a wait, sets that task aside with a bookmark and picks up another, never needing a second person.

### Analogy (and where it breaks)

A restaurant kitchen:
- **Processes** = separate kitchens in separate buildings. True parallelism, but sharing an ingredient means physically driving it across town (expensive).
- **Threads** = several cooks in *one* kitchen sharing counters and the fridge. Fast to share, but two cooks grabbing the same knife collide (race conditions; you need locks).
- **Coroutines** = *one* cook who, while the pasta boils, chops vegetables, then checks the oven — never idle, but only ever doing one thing at any instant.

*Where it breaks:* real cooks have two hands (a little true parallelism); a single coroutine thread has exactly one "hand" — it interleaves, it does not truly parallelize.

### The distinctions that matter, side by side

| | Process | Thread | Coroutine |
|---|---|---|---|
| Memory | private (isolated) | shared with siblings | shared (same thread) |
| Created/switched by | the OS (expensive) | the OS (medium) | *your program* (cheapest) |
| True parallelism? | **yes** (separate cores) | **no** in Python (the GIL — §2) | no |
| Switch trigger | OS preempts anytime | OS preempts anytime | **only at `await`** (cooperative) |
| Cost to hold 10,000 of them | impossible (GBs) | hard (~10 GB of stacks) | easy (~tens of MB) |
| Best for | CPU-bound | I/O-bound w/ blocking libs | I/O-bound (the default for agents/web) |

### Concrete example — the same task, three ways

**Task:** download 3 web pages, each taking 1 second (all *waiting*, no CPU).

- **Serial (no concurrency):** download 1, then 2, then 3 → **3 seconds**. The cashier stands idle through each wait.
- **Threads:** start 3 threads; while each waits on its socket, the GIL is released so the others proceed → all three waits overlap → **~1 second**. Three cooks, but they spend the whole time *waiting*, so sharing one kitchen is fine.
- **Coroutines (async):** one thread launches all 3 downloads, and at each `await` hands control to the next → all three waits overlap → **~1 second**, using a *fraction* of the memory of threads.

The lesson: for pure waiting, threads and coroutines both win, but coroutines win *cheaper*. For pure computing, neither threads nor coroutines help in Python — only processes do. §2 explains *why*.

---

## 2. The GIL — the one fact that shapes all Python concurrency **[CORE]**

### What it is (first principles)

CPython (the standard Python interpreter from python.org) has a single lock called the **GIL — Global Interpreter Lock**. Its rule: **only one thread may execute Python bytecode at any instant.** *Bytecode* is the low-level instruction stream the interpreter runs after compiling your `.py` file.

### Why it exists (motivation — never skip the why)

CPython tracks memory with **reference counting**: every object stores a count of how many things point to it, and is freed the moment that count hits zero. Now imagine two threads both doing `x = shared_object` and later discarding it. Each must increment then decrement that counter. If two threads touch the counter *simultaneously* without coordination, the increments/decrements interleave and corrupt it — leading to objects freed while still in use (a crash) or never freed (a leak). The GIL was the simple, fast fix: make the interpreter itself single-threaded for bytecode, so the counter is never touched by two threads at once. It made single-threaded Python fast and C-extensions easy — a deliberate 1990s tradeoff we still live with.

### The two consequences you must burn into memory

**(1) Threads give NO speedup for CPU-bound pure-Python work.** They take turns holding the GIL; you get *interleaving*, not *parallelism*.

**(2) The GIL is RELEASED during I/O.** When a thread makes a blocking call that waits on the OS (read a socket, read a file), CPython hands the GIL to another thread while the first waits. **This is why threads DO help I/O-bound work.**

### Concrete example — measure it yourself (worked)

**CPU-bound test:** sum the numbers 1..50,000,000 twice.

```python
# Serial: do it twice, one after the other
def count():
    total = 0
    for i in range(50_000_000): total += i

# ~2.0s serial (0.5s? no — say each takes 1.0s → 2.0s total)

# With 2 THREADS (expecting 2× speedup):
t1 = Thread(target=count); t2 = Thread(target=count)
t1.start(); t2.start(); t1.join(); t2.join()
# RESULT: ~2.0s — NO speedup! (often slightly SLOWER from switching overhead)
#   Why: both threads want the GIL; only one runs bytecode at a time.

# With 2 PROCESSES:
p1 = Process(target=count); p2 = Process(target=count)
p1.start(); p2.start(); p1.join(); p2.join()
# RESULT: ~1.0s — real 2× speedup! Each process has its OWN GIL on its OWN core.
```

**I/O-bound test:** two functions that each `time.sleep`-equivalent wait 1s on a network call.

```python
# 2 THREADS on I/O-bound work:
# RESULT: ~1.0s — real speedup! The GIL is released during the wait,
#   so thread B runs while thread A waits on its socket.
```

**The takeaway in one line:** threads are useless for CPU, useful for I/O; processes are the only route to CPU parallelism in Python.

### Runnable example — measure the GIL yourself

The snippets above used illustrative timings. Here is the complete, runnable version — run it and read the real numbers off your own machine.

```python
# gil_cpu.py — run with:  python gil_cpu.py
import time
from threading import Thread
from multiprocessing import Process


def count():                       # pure CPU work — holds the GIL the whole time
    total = 0
    for _ in range(50_000_000):
        total += 1


def run_two(worker_cls) -> float:
    a, b = worker_cls(target=count), worker_cls(target=count)
    t0 = time.perf_counter()
    a.start(); b.start(); a.join(); b.join()
    return time.perf_counter() - t0


if __name__ == "__main__":                    # guard REQUIRED for multiprocessing
    t0 = time.perf_counter(); count(); count()
    print(f"serial (2x):  {time.perf_counter() - t0:.2f}s")
    print(f"2 threads:    {run_two(Thread):.2f}s")     # ≈ serial — the GIL serializes bytecode
    print(f"2 processes:  {run_two(Process):.2f}s")    # ≈ half — each process has its OWN GIL/core
```

```python
# gil_io.py — the SAME two-worker shape, but the work is WAITING, not computing
import time, threading


def io_task():
    time.sleep(1)                  # a stand-in for a network wait — sleep RELEASES the GIL


t0 = time.perf_counter()
threads = [threading.Thread(target=io_task) for _ in range(3)]
for t in threads: t.start()
for t in threads: t.join()
print(f"3 threads on I/O: {time.perf_counter() - t0:.2f}s")   # ≈ 1s, not 3s
```

**What the numbers prove.** In `gil_cpu.py`, "2 threads" comes out roughly equal to serial because both threads want the GIL and only one runs bytecode at a time — you get interleaving, not parallelism. "2 processes" comes out ≈ half because each process is a separate interpreter with its *own* GIL on its own core: genuine parallelism. Flip to `gil_io.py` and threads suddenly win — three 1-second waits finish in ≈1 second total — because `time.sleep` (like any blocking I/O call) *releases* the GIL, so the other threads run during each wait. Same threads, opposite result: the deciding factor is whether the bottleneck is computing (GIL blocks you) or waiting (GIL steps aside). The `if __name__ == "__main__"` guard isn't optional — `multiprocessing` re-imports the module in each child, and without the guard that re-import would spawn processes recursively.

### Visual

```
CPU-bound, 2 threads:        I/O-bound, 2 threads:
 A ██░░██░░██  (holds GIL)     A ██····wait(GIL freed)····██
 B ░░██░░██░░  (waits GIL)     B ░░██··(runs while A waits)··██
   total time UNCHANGED          the WAITS overlap → faster
```

### Decision table **[CORE]**

| Your workload | Use | Why |
|---|---|---|
| I/O-bound, many waits (web, **agents**, DB, API/LLM calls) | **coroutines / asyncio** | cheapest concurrency; one thread overlaps thousands of waits |
| I/O-bound but stuck with blocking libraries | **threads** | GIL releases on I/O, so waits still overlap |
| CPU-bound (image/number crunching) | **processes** | separate GILs → true multi-core parallelism |
| Mixed | async core + offload CPU/blocking to a pool | keep the event loop free (§4) |

*Verify-before-relying: CPython 3.13+ ships an experimental **free-threaded (no-GIL)** build (PEP 703). It may eventually change the CPU-threading story, but it is opt-in and maturing — treat as **[AWARE]**; design against the GIL model above.*

---

## 3. The asyncio event loop — under the hood **[CORE]**

### Intuition

A coroutine is a worker who never stands idle. But *someone* has to decide which coroutine runs next and wake the right one when its wait finishes. That someone is the **event loop**: a single-threaded scheduler that runs coroutines, parks each one when it hits a wait, and resumes it the instant its wait completes.

### Analogy — the short-order cook (developed fully)

One cook (the single thread) runs the whole line by following one rule: *never wait idly.*

1. Take order A (a steak — needs 8 min on the grill). Put it on the grill. **This is `await grill(steak)`** — the cook doesn't stand watching it.
2. Immediately take order B (a salad). Start chopping.
3. A timer dings — steak A is ready. The cook plates it (**resumes coroutine A exactly where it left off**), then goes back to the salad.

One cook, zero idle time, many meals "in progress at once." That is the event loop. *Where it breaks:* a real cook can chop *while* the steak grills (a bit of true parallelism); the event loop's single thread does exactly one thing per instant — it just never wastes an instant waiting.

### First principles & terminology

- `async def` defines a **coroutine function**; calling it returns a coroutine object that does nothing until scheduled on the loop.
- `await x` means: **"pause me here; the loop may run others; wake me when `x` is done, and resume me on this exact line."** Control can switch **only** at an `await` — nowhere else. (This is *cooperative* scheduling, and it's the source of the trap in §4.)
- The loop knows *which* wait finished via an **OS readiness API** — `epoll` (Linux), `kqueue` (macOS), `IOCP` (Windows) — a kernel facility that watches thousands of sockets and reports exactly which became ready. **[WORKING]** — know it exists; you rarely touch it.

### Concrete example — two coroutines, traced step by step (worked)

```python
async def fetch(name, seconds):
    print(f"{name}: start")
    await asyncio.sleep(seconds)     # a stand-in for a network wait
    print(f"{name}: done")
    return name

async def main():
    # gather runs BOTH concurrently on the ONE loop
    await asyncio.gather(fetch("A", 2), fetch("B", 1))

asyncio.run(main())
```

**Exact timeline of what the single thread does:**
```
t=0.00  A: start          → A hits `await sleep(2)` → loop parks A ("wake at t=2")
t=0.00  B: start          → B hits `await sleep(1)` → loop parks B ("wake at t=1")
t=0.00  loop has nothing ready → it SLEEPS in epoll(), using ~0% CPU
t=1.00  B's timer fires   → loop resumes B → prints "B: done"
t=2.00  A's timer fires   → loop resumes A → prints "A: done"
total wall-clock ≈ 2.0s   (NOT 3.0s — the two waits overlapped)
```

Serial would have been 2 + 1 = 3s. The loop got it in 2s (the *longest* wait), on **one thread**, at **~0% CPU** during the waits. That is the entire value proposition.

### `asyncio.gather` **[WORKING]**

`results = await asyncio.gather(coro1, coro2, coro3)` schedules all coroutines concurrently and returns when all finish. **Wall-clock ≈ the slowest one, not the sum.** It is *the* primitive for firing independent network calls at once (and, in Part 3, independent tool calls).

### Runnable example — serial vs. `gather`, timed

```python
# gather.py — run with:  python gather.py
import asyncio, time


async def fetch(name: str, seconds: float) -> str:
    print(f"{name}: start")
    await asyncio.sleep(seconds)          # stand-in for a network wait
    print(f"{name}: done")
    return name


async def serial():
    t0 = time.perf_counter()
    await fetch("A", 1); await fetch("B", 1); await fetch("C", 1)   # one after another
    print(f"serial: {time.perf_counter() - t0:.2f}s")               # ≈ 3s


async def concurrent():
    t0 = time.perf_counter()
    await asyncio.gather(fetch("A", 1), fetch("B", 1), fetch("C", 1))  # all at once
    print(f"gather: {time.perf_counter() - t0:.2f}s")                  # ≈ 1s


asyncio.run(serial())
asyncio.run(concurrent())
```

**Why `gather` is ≈1s, not ≈3s.** In `serial()`, each `await fetch(...)` fully completes before the next one starts — the loop parks on A, wakes it, *then* starts B. In `concurrent()`, `gather` schedules all three coroutines before awaiting any of them, so all three hit their `await asyncio.sleep(1)` at t=0 and park together; the three waits overlap and the whole thing finishes at ≈1s (the *longest* wait), all on one thread at ~0% CPU. This is the exact primitive Part 3 uses to fire a model's independent tool calls at once.

### Memory perspective — why coroutines scale where threads can't

A parked coroutine stores only its tiny stack frame (its local variables + the resume point) — a few kilobytes. A thread reserves a full OS stack, often ~1 MB. So:
- **10,000 threads** ≈ ~10 GB of stack memory → impossible.
- **10,000 coroutines** ≈ tens of MB → trivial.

This is *why* one async process can hold hundreds of thousands of concurrent connections (or agent sessions) while a threaded one caps out in the thousands.

---

## 4. The blocking-call trap — the single most important async bug **[CORE]**

### The intuition (why it happens)

The loop is **cooperative**: a coroutine keeps the one thread until it *voluntarily* yields with `await`. So if a coroutine runs a **synchronous, blocking** call — one that never `await`s — it **never gives the thread back.** The loop is frozen. And because it's *one* thread serving *everyone*, freezing it freezes **every** concurrent request on that worker, not just the one that misbehaved.

### Concrete example — with the actual latency math (worked)

An `async def` handler accidentally calls the **blocking** `time.sleep(5)` (or the blocking `requests.get(...)`, or a heavy CPU loop). 100 requests arrive at the same instant.

```
CORRECT (await asyncio.sleep(5)):        BROKEN (time.sleep(5)):
 all 100 park at the await               request 1 runs sleep(5) — loop FROZEN 5s
 all 100 waits overlap                   request 2 can't even start until 1 returns
 total ≈ 5 seconds                       requests serialize: 100 × 5s = 500 seconds
 throughput: 100 req / 5s = 20 req/s      throughput: 100 req / 500s = 0.2 req/s
```

One wrong function call turned a 20 req/s service into a 0.2 req/s one — a **100× collapse** — and *every* endpoint on the worker times out, not just the slow one.

### Diagnosis & fix

- **Symptoms:** throughput falls off a cliff under load; p99 latency explodes; endpoints *unrelated* to the slow code also time out; CPU is often near 0% (everyone's just *stuck*, not working).
- **Fix / prevention:**
  - Use **async I/O libraries**: `httpx`/`aiohttp` not `requests`; `asyncpg` not a blocking DB driver.
  - Never `time.sleep()` in async code → use `await asyncio.sleep()`.
  - Push unavoidable blocking or CPU work off the loop: `await asyncio.to_thread(blocking_fn, ...)` (threadpool) or a process pool for CPU work.

```python
# BROKEN: freezes the whole loop
@app.get("/report")
async def report():
    data = requests.get("https://slow-api")   # blocking! never awaits
    return heavy_cpu_crunch(data)              # blocking CPU too

# FIXED: async I/O + offload CPU
@app.get("/report")
async def report():
    async with httpx.AsyncClient() as c:
        data = await c.get("https://slow-api")             # yields to the loop
    return await asyncio.to_thread(heavy_cpu_crunch, data) # CPU off the loop
```

### Runnable example — watch one blocking call serialize everything

```python
# blocking_trap.py — run with:  python blocking_trap.py
import asyncio, time


async def good():
    await asyncio.sleep(1)          # yields the thread back to the loop


async def bad():
    time.sleep(1)                   # BLOCKS the one thread — never yields


async def run(task, label: str):
    t0 = time.perf_counter()
    await asyncio.gather(*(task() for _ in range(5)))   # 5 concurrent tasks
    print(f"{label}: {time.perf_counter() - t0:.2f}s")


asyncio.run(run(good, "await asyncio.sleep"))   # ≈ 1s — the 5 waits overlap
asyncio.run(run(bad,  "blocking time.sleep  ")) # ≈ 5s — they serialize; loop frozen each second
```

**The whole trap in five lines of difference.** Both versions `gather` five tasks. `good()` uses `await asyncio.sleep(1)`, which parks the coroutine and returns the single thread to the loop — so all five park at t=0 and finish together at ≈1s. `bad()` uses the *synchronous* `time.sleep(1)`, which holds the one thread and never yields — so task 1 must fully finish before task 2 can even start, and five tasks serialize to ≈5s. In a web server or agent this is catastrophic: the frozen thread is shared by *every* concurrent request, so one blocking call stalls all of them (the 100× collapse above). The fix is always the same — use an async library, or push the unavoidable blocking/CPU work off the loop with `await asyncio.to_thread(...)`.

---

## Performance, trade-offs, industry usage, common mistakes (Part 1)

- **Performance:** async shines when work is I/O-dominated (agents, web APIs) — one core serves thousands of connections. It gives **nothing** for CPU-bound work (still one thread). Switch costs: coroutine ≈ nanoseconds, thread ≈ microseconds, process ≈ more.
- **Industry usage:** every high-concurrency Python service (FastAPI, aiohttp) is event-loop-based; CPU-heavy pipelines use `multiprocessing` or task queues (Celery) instead. "Async for I/O, processes for CPU" is universal.
- **Common mistakes:** *beginner* — believing `async` makes code "faster" (it only overlaps *waiting*); *intermediate* — one blocking call freezing the loop; *senior* — using threads for CPU-bound work and being baffled by zero speedup (the GIL).

## Interview & practice (Part 1)

1. Why do threads not speed up CPU-bound Python but *do* help I/O-bound work? Name the GIL and its I/O release.
2. Give the exact latency math for 100 concurrent requests when one `async def` handler calls `time.sleep(10)`.
3. Why can one process hold 100k coroutines but only a few thousand threads? (Answer in memory terms.)
4. When would you reach for `multiprocessing` over `asyncio`? Give a concrete workload.

*Practice — Easy:* define concurrency vs parallelism in one sentence each. • *Medium:* rewrite three serial `await api.get()` calls to run concurrently and state the new wall-clock. • *Hard:* a service is fast in isolation but collapses under load with CPU near 0% — give the top-two root causes and how you'd confirm each.

---

# PART 2 — AGENTIC AI: The Raw ReAct Loop

> Backend is a black box here: whenever "your code runs the tool" appears, that's the async machinery of Part 1. We focus on the model-facing side.

## Overview & motivation — why build it by hand, from scratch

[Phase 0](phase0-week1.md) told you *what* an agent is (a loop around a stateless model). This week you *build* one with **no framework** — deliberately. Frameworks (Phase 4) hide the three things you must be able to debug: how the model is made to *request* a tool, how you *parse and execute* that request, and how *state accumulates*. You cannot trust or fix a black box you have never seen opened. By the end you will have written the ~30 lines that every agent framework is, underneath, a wrapper around.

## 1. First principles & terminology **[CORE]**

- **Message list** — the conversation as an ordered list of role-tagged messages: `system` (standing instructions), `user`, `assistant` (the model's turns), and `tool` (a tool's result). **This list *is* the agent's entire state** — because the model is stateless ([Phase 0](phase0-week1.md), Model 1), you resend the whole list every call.
- **Tool / function** — a real function your code exposes to the model (a calculator, a web search, a DB query).
- **Tool definition** — a machine-readable description sent with the prompt: `name`, `description`, and a **JSON Schema** of the parameters (JSON Schema = a standard vocabulary for describing the shape of JSON: types, required fields, allowed values).
- **Tool call** — the structured object the model emits *instead of* prose when it wants an action: `{name, arguments}`.
- **ReAct** — **Rea**son + **Act**: interleave the model's reasoning with tool actions in a loop until it produces a final answer.

## 2. How function calling actually works — step by step **[CORE]**

The model **cannot execute anything.** Function calling is the protocol that turns the model's *intent* into your code's *action*:

```
1. YOU → model:  the prompt + a list of tool definitions (name, description, JSON-Schema)
2. model → YOU:  EITHER final prose, OR a structured tool-call: {name, arguments}
3. YOU:          parse it, run the REAL function (Part 1: an async I/O call)
4. YOU → model:  append the result as a `tool`-role message, then call the model again
5. model → YOU:  reads the result, then either requests another tool or gives the final answer
```

> **The most important sentence in Part 2:** *the model only emits **text describing an intended tool call**; your code does the actual execution.* Everything an agent "does" in the world is your code acting on the model's request. Forgetting this is the #1 beginner confusion.

### Concrete example — one function call, showing the real payloads (worked)

User asks: *"What's the weather in Austin?"*

**Step 1 — you send** the prompt plus a tool definition:
```json
{ "name": "get_weather",
  "description": "Get current weather for a city. Use when the user asks about weather.",
  "parameters": {"type":"object",
    "properties":{"city":{"type":"string","description":"City name"}},
    "required":["city"]} }
```
**Step 2 — the model replies, NOT with prose, but with a tool-call:**
```json
{"tool_call": {"name": "get_weather", "arguments": {"city": "Austin"}}}
```
**Step 3 — YOUR code executes** `get_weather("Austin")` (a real HTTP call, Part 1) → returns `{"temp_c": 34, "sky": "clear"}`.

**Step 4 — you append that result** as a `tool` message and call the model again. The message list is now:
```
[system, user("weather in Austin?"), assistant(tool_call get_weather),
 tool(get_weather → {"temp_c":34,"sky":"clear"})]
```
**Step 5 — the model reads the result and answers in prose:** *"It's 34°C and clear in Austin right now."*

Notice: the model never touched the network. It said "please call get_weather with city=Austin," *you* called it, and *you* fed the answer back.

## 3. The ReAct loop, in real code **[CORE]**

```python
messages = [system_prompt, user_message]          # the ENTIRE state (resent every turn)

for step in range(MAX_ITERS):                      # STOP #2: the loop guard (mandatory)
    reply = llm(messages, tools=TOOL_DEFS)         # Part 1 black box: the network call
    messages.append(reply)                         # record the assistant's turn

    if reply.tool_calls:                            # the model chose to ACT
        for call in reply.tool_calls:              # it may request several at once
            result = run_tool(call.name, call.arguments)   # Part 1: YOUR code executes
            messages.append(tool_message(call.id, result)) # feed the result back
        continue                                    # loop → model REASONS again with new info
    return reply.content                            # STOP #1: the model gave a final answer

raise LoopGuardTripped("hit MAX_ITERS")             # ran out of steps → hard stop
```

Mapping to [Phase 0](phase0-week1.md)'s loop: **Perceive** = the growing `messages`; **Reason** = the model's choice; **Act** = `run_tool`; **Repeat** = append result + `continue`; **Stop** = a final answer *or* `MAX_ITERS`.

### Concrete example — a full multi-step run, traced (worked)

User: *"What is 17% of 2,340, and is that more than 400?"* Tools: `calculator`.

```
step 0: messages = [system, user(question)]
  → llm() replies: tool_call calculator("2340 * 0.17")
  → run_tool → 397.8
  → messages += [assistant(call), tool(397.8)]        (continue)

step 1: llm() sees 397.8, replies: tool_call calculator("397.8 > 400")
  → run_tool → False
  → messages += [assistant(call), tool(False)]        (continue)

step 2: llm() now has both facts, replies with PROSE (no tool_call):
  "17% of 2,340 is 397.8, which is NOT more than 400."
  → return (STOP #1)
```

Three passes through the loop, two tool calls, one final answer. Each pass re-sent the *entire* growing `messages` list (Part 1's I/O cost — see Part 3).

### Runnable example — the hand-rolled ReAct loop, end to end (Anthropic SDK)

The loop above is pseudocode (`llm`, `run_tool`, `tool_message` are stand-ins). Here is the *complete, runnable* ~30 lines every framework wraps — the same "17% of 2,340" run against Claude.

```python
# react.py — pip install anthropic ; export ANTHROPIC_API_KEY=...
import anthropic

client = anthropic.Anthropic()


def calculator(expression: str) -> str:
    # Demo only — NEVER eval untrusted input in production.
    return str(eval(expression, {"__builtins__": {}}, {}))


TOOL_DEFS = [{
    "name": "calculator",
    "description": "Evaluate an arithmetic expression, e.g. '2340 * 0.17' or '397.8 > 400'.",
    "input_schema": {
        "type": "object",
        "properties": {"expression": {"type": "string"}},
        "required": ["expression"],
    },
}]


def run_tool(name: str, args: dict) -> str:      # <-- YOUR code executes, not the model
    if name == "calculator":
        return calculator(args["expression"])
    return f"unknown tool: {name}"


messages = [{"role": "user",                      # the ENTIRE state — resent every turn
             "content": "What is 17% of 2,340, and is that more than 400?"}]
MAX_ITERS = 6

for step in range(MAX_ITERS):                     # STOP #2: mandatory loop guard
    reply = client.messages.create(model="claude-opus-4-8", max_tokens=1024,
                                   tools=TOOL_DEFS, messages=messages)   # the network call
    messages.append({"role": "assistant", "content": reply.content})     # record the turn

    tool_calls = [b for b in reply.content if b.type == "tool_use"]
    if not tool_calls:                            # STOP #1: model produced a final answer
        print(next(b.text for b in reply.content if b.type == "text"))
        break

    results = []
    for call in tool_calls:                       # the model may request several at once
        out = run_tool(call.name, call.input)
        print(f"[step {step}] {call.name}({call.input}) -> {out}")
        results.append({"type": "tool_result", "tool_use_id": call.id, "content": out})
    messages.append({"role": "user", "content": results})   # feed results back -> loop again
else:
    raise RuntimeError("hit MAX_ITERS")           # guard tripped -> hard stop
```

Typical output:

```
[step 0] calculator({'expression': '2340 * 0.17'}) -> 397.8
[step 1] calculator({'expression': '397.8 > 400'}) -> False
17% of 2,340 is 397.8, which is not more than 400.
```

**Mapping code to the mental model.** `messages` is the whole state (Perceive), re-sent on every `create` call because the model is stateless. `client.messages.create` is the Reason step — it returns either a `tool_use` block (the model chose to act) or plain `text` (a final answer). `run_tool` is the Act step: **your** code runs the calculator; the model only *asked* for it. The two stops are visible: `if not tool_calls` is STOP #1 (the natural finish), and `for step in range(MAX_ITERS)` with the `else` clause is STOP #2 (the guard that fires if the model gets stuck looping — remove it and a flaky tool could burn tokens forever). Two mechanical rules the API enforces: append the assistant's `content` (with its `tool_use` blocks) **before** the results, and return exactly one `tool_result` per `tool_use`, keyed by `tool_use_id` — get the order wrong and the next `create` call errors.

### Why two stop conditions? (the "why", not just the "what")

The natural stop is the model answering (STOP #1). The **guard** (STOP #2) exists because a model can get stuck: it repeatedly calls a tool that keeps returning an error, or ping-pongs between two tools, never converging. Without a hard cap, that is an **infinite loop burning tokens and real money** every iteration. Example: a `search` tool is down and returns an error; a guardless agent may call it 10,000 times. **The guard is mandatory, not optional.** Ideally you also add a *budget* guard (max tokens / max dollars).

## Under the hood — what one iteration actually costs

- The `llm()` call re-processes the **entire** `messages` list every time ([Phase 0](phase0-week1.md), Model 1: statelessness ⇒ resend everything). So iteration *N* pays for all tokens accumulated in iterations 0..N-1. **Cost and latency grow with both the number of steps and the length of history.** In the worked example above, step 2 re-sent the question, both tool calls, and both results.
- Every tool result you append becomes **permanent context** re-read on every later turn — which is why bloated tool outputs quietly wreck later reasoning and cost (made precise in [Week 3](phase1-week3.md)).

## Failure modes & common mistakes (Part 2)

- **No guard → runaway loop** (the cardinal sin). *Symptom:* the same tool call repeats forever. *Fix:* `MAX_ITERS` + a token/cost budget.
- **Assuming the model executed the tool.** It never does. *Symptom:* "the agent said it searched but nothing happened" — your `run_tool` wasn't wired to the tool-call. 
- **Swallowing tool errors.** Feeding the model nothing (or a raw stack trace) leaves it unable to recover. *Fix:* return a short, structured, model-readable message ([Week 3](phase1-week3.md) makes this precise).
- **Wrong message order.** Most APIs require the assistant's tool-*call* message *and then* the tool-*result* message, in order. Appending the result without the call errors on the next request.

## Interview & practice (Part 2)

1. Walk the five steps of a function call, showing the payload at each step. Who executes the tool?
2. What is the agent's "state," and why must you resend it every call? What does that cost?
3. Name the two stop conditions and explain, with an example, why the guard is non-negotiable.

*Practice — Easy:* T/F — the model runs the tool. *(False — it emits intent; your code runs it.)* • *Medium:* hand-write the loop and annotate perceive/reason/act/stop. • *Hard:* your agent returns prose immediately and never calls a defined tool — give three causes (tool not passed to `llm`, weak `description`, model judged it unnecessary) and how to test each.

---

# PART 3 — THE BRIDGE: The Agent Loop *Is* an Async I/O-Bound Workload

*Every claim here references Part 1 or Part 2 — no new concepts.*

## The core realization

Look again at the loop in Part 2 and ask: what does each iteration *actually do* with its time? It **waits on the network** — first for the model (`llm(...)`), then for each tool (`run_tool` is an API/DB call). Waiting is the whole job. That is the textbook definition of an **I/O-bound workload** from Part 1. Therefore:

> **An agent is not a special kind of program. It is an async, I/O-bound client — and the event loop (Part 1) is the engine it should run on.**

This is *why* Part 1 came first: you cannot build a concurrent, production agent without the concurrency model underneath it.

## The dependency map

```
     AGENTIC LOOP (Part 2)                    BACKEND ENGINE (Part 1)
 ┌──────────────────────────────┐     ┌──────────────────────────────────┐
 │ each iteration = network wait ┼────►│ asyncio event loop (overlap waits)│
 │ run_tool() = an I/O call      ┼────►│ async libs; NEVER block the loop  │
 │ N independent tool calls      ┼────►│ asyncio.gather (fire concurrently)│
 │ model "does" nothing itself   ┼────►│ your code performs every action   │
 │ a CPU-heavy tool (parsing)    ┼────►│ offload with to_thread / process  │
 └──────────────────────────────┘     └──────────────────────────────────┘
```

## Interlink 1 — the event loop is what makes agents *concurrent* **[CORE]**

Because each loop step `await`s the model and tools (Part 2), **one server can run many users' agents at once on a single core** (Part 1 event loop). While user A's agent is parked at `await llm(...)`, the loop advances user B's agent.

**Concrete example (worked):** 50 users each start an agent whose loop spends ~10s waiting on model+tool calls.
- *Blocking/serial design:* 50 × 10s = **500s** for the last user — unusable.
- *Async design:* all 50 agents' waits overlap on one loop → the last user finishes in **~10s**. Same one core.

Skip the event loop and you serialize every user; understand it and one machine serves the whole fleet.

## Interlink 2 — run independent tool calls concurrently **[CORE]**

When the model requests several tools in one turn (Part 2's `for call in reply.tool_calls`) and they don't depend on each other, run them with `asyncio.gather` (Part 1).

**Concrete example (worked):** the model asks, in one turn, for weather in 3 cities (3 independent API calls, ~1s each).
```python
# Serial:   await w("Austin"); await w("Dallas"); await w("Houston")   → ~3s
# gather:   await asyncio.gather(w("Austin"), w("Dallas"), w("Houston")) → ~1s
```
Same result, one-third the wall-clock — a direct application of Part 1's `gather`.

### Runnable example — fan out a turn's tool calls concurrently

```python
# concurrent_tools.py — run with:  python concurrent_tools.py
import asyncio, time


async def get_weather(city: str) -> dict:
    await asyncio.sleep(1)                 # stand-in for a ~1s weather API call
    return {"city": city, "temp_c": 34}


async def main():
    cities = ["Austin", "Dallas", "Houston"]   # what the model asked for in ONE turn

    t0 = time.perf_counter()                    # serial: one after another
    serial = [await get_weather(c) for c in cities]
    print(f"serial: {time.perf_counter() - t0:.2f}s")          # ≈ 3s

    t0 = time.perf_counter()                    # concurrent: all three at once
    together = await asyncio.gather(*(get_weather(c) for c in cities))
    print(together, f"gather: {time.perf_counter() - t0:.2f}s")  # ≈ 1s


asyncio.run(main())
```

**Wiring this into the ReAct loop.** When a turn returns several independent `tool_use` blocks (Part 2's `for call in tool_calls`), don't `await` them one at a time — build a coroutine per call and `await asyncio.gather(*coros)`. The three 1-second waits overlap into ≈1 second on the one thread, exactly as in Part 1. The only precondition is *independence*: if tool B needs tool A's result, they must stay sequential. This is the concrete payoff of recognizing the agent loop as an async I/O-bound workload — the same `gather` that speeds up any batch of network calls speeds up a model's parallel tool requests.

## Interlink 3 — the blocking trap becomes a *fleet-wide outage* **[CORE]**

If **one** tool in `run_tool` makes a **blocking** call (Part 1 trap) — a synchronous `requests.get`, a heavy CPU parse — it freezes the whole event loop, so **every concurrent user's agent stalls**, not just the one that triggered the slow tool.

**Concrete example (worked):** 30 users' agents run fine. One user's agent calls a tool that does a blocking 8s `requests.get`. During those 8 seconds:
- The one loop is frozen → *all 30* agents stop advancing (even those doing unrelated work).
- Symptom: adding users makes *every* agent slow, yet CPU is ~0% (everyone's just stuck).
- Fix (Part 1): async HTTP (`httpx`) or `await asyncio.to_thread(blocking_tool, ...)`.

This is the single most important operational lesson of the week: **a blocking tool is not a local slowdown; it is a shared-resource outage.**

## Interlink 4 — "the model never executes" is why reliability is backend work

Because the model only *emits intent* (Part 2) and your code (Part 1) performs every I/O, the agent's **speed, concurrency, and safety are set by the backend, not the model.** The model supplies decisions; the event loop supplies throughput. A brilliant model on a blocking loop is a slow, fragile agent.

## Under the hood — one agent turn on the event loop **[CORE]**

```
1. [LOOP]  await llm(messages)          → this coroutine parks; loop serves OTHER users  (Part 1)
2. [MODEL] returns a tool-call JSON                                                       (Part 2)
3. [LOOP]  await gather(run_tool(a), run_tool(b))  → both I/O waits overlap               (Part 1)
4. [LOOP]  append results; check MAX_ITERS                                                (Part 2)
5. goto 1, or return the final answer
```
While *this* user's agent is parked at step 1 or 3, the loop is busy advancing *other* users' agents. One thread, many concurrent agents — the payoff of putting Part 2 on top of Part 1.

## Interview & practice (Part 3)

1. Why is "the agent loop is an async I/O-bound workload" the key insight — and what three things follow from it?
2. The model requests three independent tools in one turn. How do you run them, and what's the exact wall-clock difference vs. serial?
3. Explain, with numbers, why one blocking tool degrades *all* concurrent agents, not just its own.

*Practice — Medium:* map the four dependency arrows to their Part 1 + Part 2 concepts. • *Hard:* an agent service is snappy for one user, unusable at 30 concurrent users, CPU near 0% — locate the bug across both Parts and give the fix.

---

# Cheat Sheet

**Part 1:** CPU-bound → **processes** (own GIL, true parallelism) · I/O-bound → **coroutines/event loop** (cheapest) or **threads** (GIL frees on I/O) · GIL = one thread runs bytecode at a time, released during I/O · event loop = single-thread scheduler, switches only at `await` · **never block the loop** (async libs or `to_thread`) · `gather` = concurrent awaits (wall-clock ≈ slowest).

**Part 2:** function calling = send JSON-Schema tool defs → model emits a tool-call → **your code** executes → append result → loop · ReAct = perceive/reason/act/repeat with a **mandatory** `MAX_ITERS` guard · the model never runs anything; state = the message list, resent every call (so cost grows each step).

**Part 3:** the agent loop is an async I/O-bound client → the event loop lets one core run the whole fleet; run independent tools with `gather`; **one blocking tool freezes every concurrent agent**.

**Mnemonics:** "CPU→process, I/O→async" · "block the loop, drop them all" · "the model emits intent; your code acts."

---

# Build This

**Definition of done:**
1. **(Part 1, feel the GIL)** Write the CPU-bound count from §2 and time it: serial, then 2 threads (expect *no* speedup), then 2 processes (expect ~2×). Then do an I/O-bound version with 2 threads (expect a real speedup). Write one sentence explaining each result.
2. **(Part 1, feel the loop)** Fire 5 slow HTTP GETs (a public delay endpoint) serially with `httpx`, then concurrently with `asyncio.gather`; confirm wall-clock drops from ~5×delay to ~1×delay.
3. **(Part 1, prove the trap)** In an `async` function, run blocking `time.sleep(3)` in 5 concurrent tasks (confirm they serialize to ~15s), then swap for `await asyncio.sleep(3)` (confirm ~3s).
4. **(Part 2)** Hand-roll a ReAct loop (no framework) with a `calculator` tool and a `MAX_ITERS` guard. Run the worked example ("17% of 2,340, more than 400?") and print the message list after each step so you *see* it grow.
5. **(Bridge)** Add a real **async** HTTP tool and `await` both the model and tools. Then make that tool blocking, run several agents concurrently, watch the freeze, and fix it with `asyncio.to_thread`. You have now *seen* the event loop stall and recover under an agent workload.

---

# Active Recall & Self-Test

**From memory:** (1) Why do threads help I/O but not CPU in Python? (2) Give the latency math for 100 concurrent requests when an `async` handler blocks for 5s. (3) Draw the event-loop cycle and trace two coroutines. (4) Walk the five steps of function calling with payloads — who executes the tool? (5) Why are two stop conditions needed, and what breaks without the guard? (6) Why is an agent an I/O-bound workload, and what does `gather` buy you (with numbers)?

**Teach-back (60s, aloud):** explain why understanding the event loop is a prerequisite for building concurrent agents, and why one blocking tool can take down every user's agent. Stumble → re-read Part 3.

---

# Primary Sources

- **asyncio & the event loop** — `docs.python.org/3/library/asyncio.html` (esp. "Developing with asyncio" and the event-loop reference).
- **The GIL** — `docs.python.org/3/glossary.html#term-global-interpreter-lock`; PEP 703 (free-threaded CPython) — **verify status before relying on it.**
- **OS readiness APIs** — `man epoll` (Linux), kqueue (BSD/macOS), IOCP (Windows) — background, [AWARE].
- **ReAct** — Yao et al., "ReAct: Synergizing Reasoning and Acting in Language Models" (2022).
- **Function calling / tool use** — your model provider's tool-use API docs (Anthropic/OpenAI); **verify the exact schema and message-ordering rules against current docs — they drift.**

---

# Key Takeaways & Summary

- **Part 1:** Two bottlenecks, two tools — CPU→processes, I/O→coroutines; the GIL is *why* (only one thread runs bytecode, released on I/O). The event loop overlaps waits on one thread and switches only at `await`; **never block it**, or you serialize everyone.
- **Part 2:** Function calling = schema-in, tool-call-out, *your code* executes; the ReAct loop needs a mandatory guard; the model never acts — it emits intent, and the message list (resent each turn) is the whole state.
- **Part 3:** The agent loop is an async I/O-bound client, so the event loop is its engine — one core runs the whole fleet, independent tools run with `gather`, and one blocking tool freezes every concurrent agent.

**10-second:** Learn how Python does concurrency (CPU→process, I/O→async, never block the loop), hand-build a ReAct agent, and see that the agent loop is just an async I/O client.

**1-minute:** Concurrency splits into CPU-bound (needs *processes* for real parallelism, because the GIL lets only one thread run bytecode) and I/O-bound (needs *coroutines* on the single-threaded event loop, which overlaps waits and switches only at `await` — so one blocking call freezes everything and serializes all requests). An agent is a hand-rolled ReAct loop: you send JSON-Schema tool definitions, the model emits a tool-call which *your* code executes and feeds back, looping with a mandatory `MAX_ITERS` guard; the model never executes anything, and the message list it re-sends each turn is the whole state. Because each iteration is a network wait, the agent is an async I/O-bound client — run it on the event loop so one core serves the fleet, fire independent tools with `gather`, and never let a blocking tool freeze everyone.

**5-minute:** read Part 1 §2 (GIL, worked timing), §3 (event-loop trace), §4 (blocking-trap math), Part 2 §2–3 (function-calling payloads + the traced multi-step run), then Part 3's four interlinks with their worked numbers.

**Expert summary:** the agent loop and the async web server are the same object — a single-threaded scheduler overlapping I/O waits — with a stateless model call in place of a database read. Internalize "CPU→process, I/O→async, never block the loop," and building concurrent, reliable agents becomes an application of it, not a new skill.

---

# Next: [Week 3 — HTTP semantics & tool-call internals](phase1-week3.md)
