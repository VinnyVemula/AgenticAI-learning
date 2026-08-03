# Day 1: What a Computer Is - CPU, Memory, and the Latency Pyramid
> Framing: This note is for a beginner who wants to build fast, reliable backends and agents. The unifying idea is simple: computation happens where the data is. A CPU can do arithmetic incredibly quickly, but waiting for data from farther away dominates time. Caches, batching, databases, CDNs, and LLM prompt caches are all ways to avoid an expensive trip down the same hierarchy.

# PART 1 - BACKEND

## 1. Start with the only performance question that matters

**Depth: [CORE] - latency and the memory hierarchy.**

### Why this exists

"Fast" is not a property of a programming language or a server. It is the elapsed time between asking for a result and receiving it. A backend spends that time either **computing** (the CPU is actively executing instructions) or **waiting** (usually for data from memory, storage, or another machine).

Without this model, "add a cache" is cargo cult. With it, you can estimate that replacing a network round trip with an in-process value can remove milliseconds, while replacing an arithmetic instruction rarely matters.

### Definitions before mechanism

- **Latency**: time for one operation to finish. It answers, "How long does this request wait?"
- **Throughput**: completed work per unit of time. It answers, "How many requests per second can we finish?" A service may have good throughput but bad latency if requests queue.
- **Memory hierarchy**: several places to hold data, ordered from tiny and fast near the CPU to huge and slow farther away.
- **Cache**: a smaller, faster store holding a copy of data expected to be used again.

### The pyramid

Typical numbers are order-of-magnitude estimates, not promises. CPU model, load, access pattern, storage type, and geography all change them. Memorize the *gaps*, then measure the system you operate.

| Where the needed data is | Rough latency | Why it differs |
|---|---:|---|
| CPU register | less than 1 ns | physically inside the CPU core |
| L1 cache | about 1 ns | tiny memory beside the core |
| L2/L3 cache | a few to a few dozen ns | larger, farther, and often shared |
| RAM | about 100 ns | off-core electrical trip to memory chips |
| SSD random read | about 100 microseconds | storage-controller and flash work |
| nearby network service | about 1-10 ms | protocol, queues, switches, and distance |
| cross-ocean network service | tens to hundreds of ms | speed-of-light distance plus everything above |

`1 ms = 1,000 microseconds = 1,000,000 nanoseconds.` So a 100 ms network wait is roughly one million times a 100 ns RAM access. This is why a single unnecessary remote call can outweigh a huge amount of local arithmetic.

### Analogy: a workbench and a warehouse

Your register is the item already in your hand. L1 cache is the workbench. RAM is shelving across the room. SSD is a warehouse in the same building. A network service is a warehouse in another city. The worker may be fast, but the job is slow when they spend it walking.

**Where the analogy breaks:** real hardware moves data automatically and in blocks, not by a person carrying one item. Several requests can also overlap; an application can be waiting on the network while the CPU serves another request.

### Worked example - choose the bottleneck

An endpoint does all of the following in sequence:

1. Adds and compares local values: 0.001 ms.
2. Looks up a value already in RAM: 0.0001 ms.
3. Reads a cache-miss value from SSD: 0.1 ms.
4. Calls a payment provider: 80 ms.

Total is approximately `80.1011 ms`. The payment call contributes more than 99.8% of the time. Making local arithmetic twice as fast changes the total by about 0.001 ms; avoiding, parallelizing, or moving that remote call changes the user experience. This is **Amdahl's law** in practice: speed up the part that occupies meaningful time.

### Visual - the hierarchy is a cost/capacity trade-off

```text
                         fastest, smallest, most expensive per byte
    CPU core:  registers
                    |
                  L1 cache
                    |
                  L2 cache
                    |
                  L3 cache
                    |
                  RAM
                    |
                  SSD
                    |
                network / remote storage
                         slowest, largest, cheapest per byte
```

You cannot put every byte in a register: registers are extremely limited, costly silicon and lose their useful contents when execution moves on. Each lower tier buys capacity and lower cost per byte by accepting more distance and delay. That three-way pressure - **capacity, cost, and volatility** - creates the pyramid.

### Under the hood

When the CPU requests data, it first checks the nearest cache. A **cache hit** means the copy is there; a **cache miss** means hardware fetches a larger unit called a **cache line** from a lower tier. Modern machines commonly use 64-byte cache lines, but treat that as architecture-specific, not a universal rule.

The CPU benefits when code reads adjacent data: after fetching bytes 0-63 for `items[0]`, the next item may already be present or be predicted by a **hardware prefetcher**. Jumping randomly through a large data structure instead creates more cache misses. The Intel Optimization Reference Manual documents these cache and prefetch effects for Intel processors; other CPU families have different details but the same locality principle.

**Deliberate stopping point:** cache replacement policies, cache coherence between CPU cores, virtual memory, and page tables are real mechanisms, but belong to Days 3-4. Today, retain the interface: nearby/reused/sequential data is usually cheaper than distant/random data.

## 2. Bits, bytes, and why computers use binary

**Depth: [WORKING] - binary representation.**

### Intuition

A physical computer needs a dependable way to distinguish symbols. Two voltage ranges are cheap to build and tolerate electrical noise better than trying to distinguish ten precise levels. We label the two states `0` and `1`; they do not mean false and true unless a program gives them that meaning.

- A **bit** is one binary digit: `0` or `1`.
- A **byte** is eight bits. It can represent `2^8 = 256` distinct bit patterns, from `00000000` to `11111111`.
- A **binary number** uses powers of two: `10110110` means `128 + 32 + 16 + 4 + 2 = 182` in decimal.

### Analogy: light switches

Eight switches make one byte. Each switch is either off or on, and the switch positions together encode one of 256 patterns.

**Where the analogy breaks:** bits are not independent household switches. A CPU, memory controller, and storage device agree on encodings, timing, error detection, and voltage thresholds; physical electronics can also be in transitional or invalid states.

### Worked example - a file size calculation

An API receives a 5 MiB upload. In binary storage units, `1 MiB = 1,024 * 1,024` bytes, so the payload is `5,242,880` bytes. If the network sent it one byte at a time it would need that many transfers; real systems group bytes into larger packets and buffers because per-operation overhead matters. Networking will make that precise on Days 9-11.

**Deliberate stopping point:** text encodings such as UTF-8 and signed-integer representation matter a great deal, but are separate concepts. Here, know that all program data is ultimately bit patterns interpreted by rules.

## 3. The CPU: a machine that executes instructions

**Depth: [CORE] - CPU execution.**

### Why a CPU exists

Stored programs replaced custom wiring. Instead of building a new physical circuit for every task, we store a list of instructions and data in memory; the CPU repeatedly reads and carries out the next instruction. A **program** is that passive instruction-and-data representation. A **process** is the running instance of a program, with resources assigned by the operating system; Day 2 opens that OS boundary.

### Core vocabulary

- **Instruction**: a tiny encoded command such as load, add, compare, jump, or store.
- **Register**: a tiny, extremely fast CPU storage location.
- **ALU (arithmetic logic unit)**: circuitry that performs arithmetic and logical operations.
- **Program counter**: a register holding the address of the next instruction.
- **Clock**: electrical timing signal coordinating processor steps. A GHz is billions of cycles per second, but one instruction does not always take one cycle and modern CPUs can overlap work.

### Analogy: a literal recipe reader

The program counter is the recipe bookmark. Registers are bowls beside the cook. The ALU is the cook's arithmetic skill. The CPU reads one recipe step, interprets it, uses nearby bowls, records the result, and moves the bookmark.

**Where the analogy breaks:** modern CPUs do not execute like a single patient cook. They pipeline, speculate, execute some independent operations out of order, and may retire several instructions per clock cycle. The simple cycle below is still the correct first model for *meaning*.

### Worked example - add two numbers

Imagine instructions equivalent to this pseudocode:

```text
load R1, [address_of_a]  # R1 <- 7
load R2, [address_of_b]  # R2 <- 35
add  R3, R1, R2          # R3 <- 42
store [address_of_sum], R3
```

The **fetch-decode-execute cycle** is:

```text
1. Fetch: read the instruction at the program-counter address.
2. Decode: determine that it is, for example, an ADD and which registers it names.
3. Execute: the ALU adds the values.
4. Write back: place the result in R3.
5. Advance or change the program counter; repeat.
```

The `load` operations are where the latency pyramid enters the story. If `a` and `b` are in cache, the add arrives quickly. If they cause a RAM miss, the ALU may be idle waiting for them.

### Under the hood

The instruction encoding is architecture-specific: x86-64 and ARM64 have different instruction sets. A compiler or interpreter turns source-level operations into many lower-level operations; Python source is not directly one CPU instruction. CPython first compiles Python to bytecode and its interpreter executes that bytecode, which eventually invokes machine code in the interpreter and libraries. Do not infer Python speed from a single `ADD` instruction.

**Deliberate stopping point:** compiler optimization, instruction pipelines, branch prediction, and CPU vector units are valuable advanced material. The load-bearing model for now is: instructions consume data; the location of that data determines whether execution computes or waits.

## 4. CPU-bound versus I/O-bound work

**Depth: [WORKING] - bottleneck classification.**

### Intuition and worked comparison

A thumbnail-resize job spends most of its time decoding and transforming pixels. More CPU cores or a faster algorithm can help: it is **CPU-bound**. A request that waits 200 ms for an HTTP response spends little CPU time and mostly waits for **I/O (input/output)**: it is **I/O-bound**.

| Workload | Dominant wait | First useful response |
|---|---|---|
| Hash 1 GB locally | CPU + memory bandwidth | profile algorithm; use appropriate parallelism |
| Read a cold 1 GB file | SSD + OS cache | stream; avoid needless rereads |
| Call three independent services | network latency | issue bounded concurrent requests; set timeouts |
| Parse a cached 1 KB JSON object | neither much | leave it alone until measurement says otherwise |

### Analogy: a chef and deliveries

A chef chopping vegetables is CPU-bound: a better knife or another chef may help. A chef waiting for an ingredient delivery is I/O-bound: another chef cannot make the truck arrive sooner, but can work on another order while waiting.

**Where the analogy breaks:** software can combine both kinds of work, and adding workers has costs: contention, memory use, context switches, and upstream overload. Day 3 and Day 21 cover those trade-offs.

## 5. Runnable example - measure a local lookup, file read, and HTTP call

Save as `measure_latency.py`. The absolute values are deliberately not asserted: your CPU, filesystem cache, connection, and target server differ. The classification and gap are the lesson.

```python
# Python standard library only
from __future__ import annotations

import statistics
import tempfile
import time
import urllib.request
from pathlib import Path
from typing import Callable


def measure_ms(operation: Callable[[], None], repeats: int = 10) -> float:
    samples: list[float] = []
    for _ in range(repeats):
        started = time.perf_counter_ns()
        operation()
        samples.append((time.perf_counter_ns() - started) / 1_000_000)
    return statistics.median(samples)


def main() -> None:
    values = {"answer": 42}
    dict_ms = measure_ms(lambda: values["answer"], repeats=100_000)

    with tempfile.TemporaryDirectory() as directory:
        file_path = Path(directory) / "one-megabyte.bin"
        file_path.write_bytes(b"x" * (1024 * 1024))
        file_ms = measure_ms(lambda: file_path.read_bytes())

    # A public documentation domain; network timing and availability vary.
    def fetch() -> None:
        with urllib.request.urlopen("https://example.com/", timeout=10) as response:
            response.read()

    http_ms = measure_ms(fetch, repeats=3)

    print(f"dict lookup median: {dict_ms:.6f} ms")
    print(f"1 MiB file read median: {file_ms:.3f} ms")
    print(f"HTTPS request median: {http_ms:.3f} ms")
    print("Interpret the ratios, not one machine's absolute values.")


if __name__ == "__main__":
    main()
```

```bash
python measure_latency.py
# Verified output in this workspace; your numbers will differ:
# -> dict lookup median: 0.000300 ms
# -> 1 MiB file read median: 1.218 ms
# -> HTTPS request median: 161.983 ms
# -> Interpret the ratios, not one machine's absolute values.
```

`perf_counter_ns()` uses a high-resolution monotonic timer suitable for measuring short durations. The first measurement makes the dictionary lookup; repeated measurements make timer overhead and ordinary scheduling noise less misleading, and the median rejects an occasional outlier. `write_bytes` creates a 1 MiB file, then `read_bytes` measures a likely warm filesystem read; it may be RAM-backed after the first read, which is an honest demonstration that operating systems cache file data. `urlopen` performs DNS, TCP/TLS setup as applicable, and an HTTP request, so its timing includes far more than remote server work. Benchmarking itself is a measurement discipline; use a profiler before changing production code.

## 6. System-design reps

### Scenario 1 - rank these operations

**Problem:** rank a register read, sequential 1 MiB RAM read, random 1 MiB RAM read, 1 MiB SSD read, and 1 KiB cross-continent fetch.

**Decision:** the usual order is register, sequential RAM, random RAM, SSD, remote fetch. Sequential RAM wins because cache lines and prefetching make one memory trip serve nearby future bytes; random access wastes much of each fetched line. The remote fetch dominates despite being only 1 KiB because latency is dominated by communication and queues, not payload size.

**Failure mode:** confusing bandwidth with latency. A large sequential SSD read can have high throughput after it starts; a tiny remote request can still have dreadful time-to-first-byte.

### Scenario 2 - why no universal fastest store exists

**Problem:** design storage for 10 TB of user data that must survive a power loss and also serve 1 ns reads.

**Decision:** impossible in one tier. Registers and cache are too small and volatile; RAM is also volatile and expensive at 10 TB; SSD offers durable capacity but cannot meet cache latency. Use layers: durable source of truth below, RAM and cache copies above, and explicit invalidation/consistency rules later in the plan.

**Trade-off:** every promoted copy improves speed but increases cost and creates a risk that a copy is stale. A cache is therefore not free performance; it is a correctness responsibility.

## 7. Case studies

### Jeff Dean's latency-number heuristic

Google engineer Jeff Dean's widely circulated latency estimates trained engineers to make rough, early calculations before architecture hardens. Treat the table as a reasoning tool rather than a hardware specification: the relative gulf between cache, RAM, storage, and network calls tells you where an optimization can plausibly matter. Re-measure for a real deployment. The Intel architecture and optimization manuals are the primary source for processor-specific behavior; Dean-style tables are derived, intentionally approximate teaching aids.

### LMAX Disruptor and mechanical sympathy

LMAX Exchange published the Disruptor as an event-processing design for high-throughput trading workloads. Its design deliberately considers cache lines, contiguous memory access, and inter-core communication rather than treating the CPU as an infinitely fast black box. The lesson is not "copy the Disruptor"; it is to know that at very high rates, memory layout and coordination can become the architecture. That is awareness-level context for now; revisit after concurrency and queues.

## 8. In production

- Measure before optimizing. Record p50, p95, and p99 latency, because an average can hide painful tail waits.
- Put timeouts around network and storage dependencies. A request that waits forever consumes a worker and eventually creates a queue.
- Cache only after deciding what is allowed to be stale and how the copy is invalidated. "Cache it" without an invalidation policy is a data-corruption plan wearing a speed hat.
- Avoid N+1 remote calls: one request per list item turns a 30 ms page into a serial chain. Batch or fetch in bounded parallelism when semantics allow it.
- Identify CPU-bound and I/O-bound work separately. More async helps I/O waits; it does not accelerate an expensive calculation on one CPU core.

# PART 2 - AGENTIC AI

## 9. An LLM request also obeys the pyramid

**Depth: [AWARE] - physical latency behind an agent call.**

An LLM call may look like a single function call, but your program sends bytes to a remote service, waits for queued and accelerated computation, then receives bytes back. It is neither magic nor local Python execution. This is why an agent needs timeouts, budgets, streaming, and later caching.

### Worked example

Suppose a model starts returning its first token 900 ms after an agent calls it, while your local code takes 0.2 ms to assemble the request. Rewriting that 0.2 ms of Python cannot make the answer feel quick. Reducing prompt size, choosing a nearer/faster route or model, reusing a computed prefix, or streaming the first token can matter. The details of tokens, attention, and KV caches begin on Days 60-64; do not pretend Day 1 has opened those boxes.

### Analogy: a very capable remote specialist

The agent's backend is a coordinator asking a remote specialist for work. The coordinator is fast at preparing the folder, but it still pays delivery time and the specialist's queue.

**Where the analogy breaks:** an LLM service is distributed computer infrastructure, not one human. It can batch work, use accelerator hardware, and stream partial output; its behavior is probabilistic rather than a fixed service function.

**Deliberate stopping point:** do not study model internals today. Keep only this constraint: a model call is a remote dependency whose latency and cost must be measured and controlled.

# PART 3 - THE BRIDGE

## 10. One request, two layers, one latency budget

The backend constructs and sends an agent request; the agentic layer depends on that backend for every byte, retry, timeout, state load, and response stream. The agent cannot make the network faster, and the backend cannot make a model answer deterministic, but both share the same end-to-end latency budget.

```text
user request
    -> backend validates and loads state (CPU/cache/RAM/SSD)
    -> backend calls model or tool (network + remote queue/compute)
    -> backend records result and responds (RAM/SSD/network)

Slow state load  -> model sees stale/missing context or user waits
Slow model/tool  -> backend workers queue unless bounded
Bad cache policy -> fast answer using incorrect state
```

The dependency map is deliberately short: the backend controls the data paths and resource limits; the agentic code is one consumer of those paths. Day 1's discipline is to ask, at every arrow, "where is the data, how long does it take to get here, and can I safely avoid the trip?"

# Topic-wide wrap-up

## Cheat Sheet

| Question | Answer |
|---|---|
| What is a program? | Instructions and data stored for execution. |
| What is a process? | A running program instance with OS-managed resources. |
| Why binary? | Two physical states are cheap and noise-tolerant. |
| Why is sequential data often faster? | Cache lines and prefetching turn adjacent accesses into fewer lower-tier trips. |
| CPU-bound? | Computation is the limiting resource. |
| I/O-bound? | Waiting for storage, network, or another external system dominates. |
| Core rule | Optimize the slowest meaningful tier, after measuring. |

## Build This

Run `measure_latency.py`, capture your output in a short `README` or commit message, and draw your own pyramid with the three median measurements. Then answer: which operation dominates, what did the benchmark actually measure, and which proposed optimization would be a waste? **Definition of done:** the script runs, you can explain why its file timing may be served from RAM, and you do not compare your absolute milliseconds to someone else's as a universal constant.

## Active Recall and Self-Test

1. Why can a 1 KiB network request be slower than a 1 MiB RAM read?
2. Walk the fetch-decode-execute cycle for an `add` instruction.
3. What is the difference between a cache hit and a cache miss?
4. Why does sequential access usually beat random access?
5. Name the capacity, cost, and volatility trade-off behind the hierarchy.
6. Classify image resizing and waiting for an HTTP API as CPU-bound or I/O-bound.
7. Why is a cached answer also a correctness decision?
8. Teach back in 60 seconds: explain why a fast CPU cannot make a slow network request fast.

## Spaced-Repetition Flashcards

| Q | A |
|---|---|
| One byte contains how many bits? | Eight. |
| What is latency? | Time for one operation to complete. |
| What is throughput? | Amount of completed work per time unit. |
| What does a cache store? | A fast copy of data likely to be reused. |
| Why do cache lines matter? | One fetch brings adjacent bytes, rewarding locality. |
| CPU-bound means? | Active computation limits progress. |
| I/O-bound means? | Waiting for external data or devices limits progress. |
| First question before optimizing? | What does measurement say dominates time? |

## Primary Sources

- [Intel 64 and IA-32 Architectures Software Developer's Manuals](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html) - processor architecture, instruction execution, and memory-system reference; verify behavior for a specific CPU generation.
- [Intel 64 and IA-32 Architectures Optimization Reference Manual](https://www.intel.com/content/www/us/en/developer/articles/technical/intel64-and-ia32-architectures-optimization.html) - locality, caches, prefetching, and performance guidance.
- [Python `time` documentation](https://docs.python.org/3/library/time.html#time.perf_counter_ns) - `perf_counter_ns` timing semantics.
- [Python `urllib.request` documentation](https://docs.python.org/3/library/urllib.request.html) - the standard-library HTTP example used above.
- [LMAX Disruptor technical paper](https://lmax-exchange.github.io/disruptor/files/Disruptor-1.0.pdf) - the named mechanical-sympathy case study.
- Jeff Dean's latency-number teaching material is useful for estimates but not a hardware contract; check current hardware documentation and benchmark your deployment before using a number in a design.

## Layered explanations

**10 seconds:** A computer is a CPU executing stored instructions. It is fast only when the needed data is nearby; the farther the data, the more waiting dominates.

**1 minute:** Bits encode data; programs are instructions plus data. The CPU fetches, decodes, and executes those instructions using tiny registers and an ALU. Data lives in a hierarchy from cache to RAM to SSD to remote machines. Each lower tier is larger and cheaper but slower. Sequential access works well because cache lines and prefetching reuse a trip. Identify whether work is CPU-bound or I/O-bound before choosing an optimization.

**5 minutes:** Add that latency is per-operation time while throughput is completed work per second. Remote latency can exceed RAM access by orders of magnitude, so a dependency call often dominates endpoint time even when the payload is tiny. The hierarchy exists because no technology gives unlimited, durable, cheap, nanosecond storage. Caches trade correctness complexity for speed: copies need invalidation and must tolerate staleness. That same physical reality later explains backend caches, CDNs, database buffer pools, LLM model calls, and prompt/KV caching.

**Expert:** Performance is a data-movement problem constrained by hierarchy, locality, bandwidth, queues, and coordination. The first model is serialized latency; production systems add concurrency, tail distributions, cache coherence, page faults, protocol handshakes, and work-conserving scheduling. Use the model to form a falsifiable bottleneck hypothesis, then profile and trace the specific workload. Never promote a coarse latency table into an SLO or capacity model without measurement.
