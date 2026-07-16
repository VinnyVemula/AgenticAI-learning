# Phase 0 — Week 1: Environment & the Two Foundational Mental Models

> **What this covers:** Part 1 is the modern Python **backend toolchain** (uv, Ruff, mypy, pytest, `pyproject.toml`, `src/` layout, lockfile, pre-commit/CI). Part 2 is the two **agentic-AI** mental models every agent is built on (a stateless LLM call; an agent as a `while` loop). Part 3 is the **Bridge**: how these two layers depend on and interlink into one system.
> **Who it's for:** a reader with zero prior knowledge — every term is defined before it is used.
> **The one idea that unites both layers:** *an LLM has no memory, so all state, memory, and durability are the backend's job. Backend rigor is therefore a prerequisite for building agents, not a nice-to-have.*
>
> **Depth tiers used below:** **[CORE]** = open every black box; **[WORKING]** = use it and know its tradeoffs, but treat its internals as a named black box; **[AWARE]** = know it exists and when to reach for it.

---

# PART 1 — BACKEND

*The modern Python toolchain: the machinery that makes your code reproducible, checked, and trustworthy on machines that are not yours.*

## Overview & motivation

A **toolchain** is the set of programs that sit between *you writing code* and *that code running correctly on someone else's machine*. Two everyday failures motivate this entire Part:

1. **"It works on my machine."** Your code runs perfectly, then breaks for a teammate or a server because they have a slightly different package version or Python interpreter. This is a *reproducibility* failure — and it is entirely preventable.
2. **"I'll skip the check, it's too slow."** If linting or type-checking takes 10 seconds, you stop running it, and it catches nothing. Fast tools change behavior: a 50-millisecond check gets run on every save.

The modern stack fixes both. Here is the whole thing at a glance — each row is explained below:

| Job | Tool | Depth | One-line role |
|---|---|---|---|
| Packages, environments, Python version, locking | **uv** | [CORE] | one manager replacing pip + venv + pyenv + pip-tools/Poetry |
| Find likely bugs + enforce style + auto-format | **Ruff** | [WORKING] | linter **and** formatter, in Rust, ~10–100× faster |
| Catch type errors before running | **mypy** `--strict` | [CORE] | static type checker |
| Prove the code works | **pytest** (+ **pytest-asyncio**) | [WORKING] | test runner (and its async plugin) |
| Describe the project in one file | **pyproject.toml** (PEP 621) | [CORE] | standardized metadata + tool config |
| Guarantee identical dependencies everywhere | **committed lockfile** (`uv.lock`) | [CORE] | the exact, reproducible dependency graph |
| Isolate importable code | **`src/` layout** | [WORKING] | forces tests to import the package as a user would |

**Deliverable:** a reusable *project skeleton* you copy at the start of every future week.

## First principles & terminology

*(Defined before use — nothing here assumes you have programmed before.)*

- **Package** (a.k.a. *library*) — reusable code someone else wrote that your program uses (e.g. `fastapi`). A **dependency** is any package your code needs. Packages have their own dependencies — **transitive dependencies**.
- **Environment** — "where installed packages live + which Python runs them." If every project shares one global environment, installing `requests` v2 for project X breaks project Y that needs v3.
- **Virtual environment** — a private per-project folder holding that project's interpreter and packages, isolated from everything else. A sandbox per project.
- **Dependency resolution** — finding *one* set of versions that satisfies *every* constraint (yours + all transitive) *simultaneously*, versus installing one-by-one and hoping they agree. The **diamond dependency problem**: A needs `C>=2`, B needs `C<2`, and you need both — a resolver must find a compromise or fail *loudly*.
- **Lockfile** — a file (`uv.lock`) recording the *exact* version **and cryptographic hash** of every dependency, direct and transitive. Committed to Git, it makes checkouts byte-identical. (A **hash** is a fingerprint of a file's bytes; matching hashes prove the bytes are unchanged.)
- **Linter** vs **formatter** — a *linter* flags likely bugs/bad patterns (correctness); a *formatter* rewrites code into one canonical style (appearance, ending style debates). Ruff does both.
- **Static type checking** — analysis done *without running the program*. A **type checker** reads **type hints** (`def f(x: int) -> str:`) and proves values flow consistently. `--strict` = every function annotated, no silent untyped escape hatches.
- **PEP** — a *Python Enhancement Proposal*, a formal spec. **PEP 621** standardized project metadata in `pyproject.toml`.
- **`src/` layout** — importable code lives under `src/` (not the repo root), so tests import the *installed* package, catching packaging bugs (diagram below).

## Intuition & analogies

Think of a professional kitchen. The toolchain is the set of standardized stations so *any* cook produces the *same* dish. `uv` stocks the pantry with exact ingredients (the lockfile is the recipe *in grams*, not "some flour"). Ruff is the line cook who plates identically and spots a spoiled ingredient. mypy is the food-safety inspector who checks the *plan* before a pan is lit. pytest is the taste test before the plate leaves.

- **Lockfile = a recipe in exact grams.** *Works:* anyone reproduces the same cake. *Breaks:* a recipe can't detect swapped flour — the lockfile's hashes can (better than the analogy).
- **Resolver = a wedding seating planner** placing every guest so no two feuding relatives share a table — solved *globally*. *Breaks:* a planner bends rules; a resolver either finds a valid arrangement or declares it impossible.
- **Type checker = spell-check for logic** — underlines the mistake before you "publish" (run). *Breaks:* it reasons about how *values flow*, which is deeper than word-matching.

## Visual diagrams

**The `src/` skeleton:**
```
myproject/
├── pyproject.toml          # PEP 621 metadata + tool config (one file)
├── uv.lock                 # exact, hashed dependency graph (committed)
├── .python-version         # pinned interpreter (e.g. 3.13)
├── .pre-commit-config.yaml # checks that block a bad commit
├── .github/workflows/ci.yml# the same checks, on every push
├── src/myproject/
│   ├── __init__.py         # marks the folder as an importable package
│   └── app.py              # the FastAPI "hello" endpoint
└── tests/test_app.py       # one passing async test
```

**Sequential install (fragile) vs. resolver (correct):**
```
Sequential (hope):        Resolver (proof):
 install A ─┐               read ALL constraints
 install B ─┤ hope they        │
 install C ─┘  agree           ▼
                            solve the whole graph at once
                            → one consistent set, or a clear failure
```

## Under the hood **[CORE]**

**How uv resolves and locks (not sequential pip installs):**
1. **Read constraints** from `pyproject.toml` (e.g. `fastapi>=0.115`).
2. **Fetch metadata** for candidate versions; cached on disk, so repeat runs are near-instant.
3. **Solve** with a **PubGrub-style backtracking resolver**: tentatively pick versions, detect a conflict, backtrack, and either converge on one consistent graph *or* emit a human-readable explanation of why none exists.
4. **Write `uv.lock`** with exact versions + hashes.
5. **Materialize the environment** — instead of re-downloading/re-copying, uv keeps a **global content-addressed cache** (files stored under the hash of their contents, so identical files are stored once) and **hard-links** them into the venv. This, plus being compiled **Rust**, is why it is fast.

**Why Ruff is fast [WORKING]:** it parses each file **once** into an **AST** (Abstract Syntax Tree — a tree of the code's structure) and evaluates *all* rules in that single traversal, in parallel across files, as compiled Rust — versus a pile of separate Python tools each re-parsing the file.

**Why static typing catches real bugs [CORE]:** mypy does **type inference** (deducing types you didn't annotate) and **flow analysis** (narrowing a value's type through `if` branches) to prove — without running anything — that you never pass a `None` where a `str` is required. Add **Pydantic** (validates data against a typed model *at runtime*) and you get two layers: *mypy proves the code is consistent (at rest); Pydantic proves the data entering the program (at runtime) matches its declared shape.* A bug like "the API sent a string where a number was expected" is caught **at the boundary**, not three functions deep.

## Step-by-step execution — building the skeleton

`uv run <cmd>` always runs `<cmd>` *inside the locked environment*, so you never manually "activate" anything.

```bash
# 1. Install uv once, system-wide (installer at astral.sh/uv).

uv init --package myproject      # 2. create pyproject.toml + src/ layout
cd myproject
uv python pin 3.13               # 3. pin the interpreter (writes .python-version)
uv add fastapi uvicorn pydantic  # 4. runtime deps — updates pyproject.toml AND uv.lock
uv add --dev ruff mypy pytest pytest-asyncio httpx  # 5. dev-only tools

uv run ruff check .              # 6. lint
uv run ruff format .             #    format
uv run mypy --strict src         # 7. type-check (without running)
uv run pytest                    # 8. tests
```

**`pyproject.toml` (PEP 621 core + tool config):**
```toml
[project]
name = "myproject"
version = "0.1.0"
requires-python = ">=3.13"
dependencies = ["fastapi", "uvicorn", "pydantic"]

[dependency-groups]
dev = ["ruff", "mypy", "pytest", "pytest-asyncio", "httpx"]

[tool.ruff]
line-length = 100

[tool.mypy]
strict = true

[tool.pytest.ini_options]
asyncio_mode = "auto"
```

**`src/myproject/app.py`:**
```python
from fastapi import FastAPI

app = FastAPI()


@app.get("/hello")
async def hello() -> dict[str, str]:
    return {"message": "hello"}  # fully annotated → mypy --strict is satisfied
```

**`tests/test_app.py`:**
```python
import pytest
from httpx import ASGITransport, AsyncClient

from myproject.app import app  # imported the way an installed user would


@pytest.mark.asyncio
async def test_hello() -> None:
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        resp = await client.get("/hello")
    assert resp.status_code == 200
    assert resp.json() == {"message": "hello"}
```

**Pre-commit + CI [WORKING].** A *pre-commit hook* is a script Git runs before recording a commit; if it fails, the commit is blocked, so broken code never enters history. *CI* (Continuous Integration) runs the same checks on a clean machine on every push. Use the **same commands in both** so "green locally, red in CI" can't happen.

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.8.0
    hooks:
      - id: ruff         # lint
      - id: ruff-format  # format
```
```yaml
# .github/workflows/ci.yml
name: ci
on: [push, pull_request]
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
      - run: uv sync --dev
      - run: uv run ruff check .
      - run: uv run mypy --strict src
      - run: uv run pytest
```
From here, onboarding is two commands: `git clone …` then `uv sync`.

## Data flow, memory & runtime behavior

- **At rest (checked, not running):** `pyproject.toml` → resolver → `uv.lock` → materialized `.venv/`. Ruff and mypy read your source and produce diagnostics; nothing of yours executes.
- **At runtime:** type hints are **erased** — they cost nothing while the program runs. The only runtime validation is Pydantic, which builds validator objects once and checks incoming data at the boundary.
- **Caches that make it fast:** uv's content-addressed cache (on disk, shared across projects), mypy's incremental `.mypy_cache/`, Ruff's per-run in-memory AST. These are why the second run of everything is near-instant.

## Performance, trade-offs & comparisons

- **Performance:** uv installs are near-instant on a warm cache (hard-links, not copies); Ruff is milliseconds/file; mypy is the slowest (a full inference engine) but caches incrementally. First cold resolve is a backtracking search — usually fast, occasionally expensive on pathological constraints, which is *why* caching matters.
- **Trade-offs:** `--strict` slows initial coding but repays it in bugs caught early • pinned deps are stable but need deliberate updates for security patches • one opinionated tool means fewer knobs but you inherit its opinions.

| Concern | Old tool(s) | Modern | Why switch |
|---|---|---|---|
| Env + install + versions | pip + venv + pyenv + pip-tools/Poetry | **uv** | one tool, 10–100× faster, real resolver |
| Lint / format | flake8 + pylint / black + isort | **Ruff** | single parse, Rust, 800+ rules, one config |
| Types | (none) | **mypy --strict** | catches a class of bugs before runtime |
| Config | setup.py, .flake8, tox.ini … | **pyproject.toml** | one file (PEP 621) |

**Confusions settled:** `pyproject.toml` states *intent* (version ranges); the lockfile records *resolved reality* (exact pins + hashes) — not redundant. Ruff is fast because it does *more in one pass*, not less.

## Failure modes, industry usage, best practices, common mistakes

- **Failure modes → recovery:** uncommitted lockfile → "works on my machine" (commit `uv.lock`) • bare `pip install` inside a uv project → lockfile drifts (use `uv add`) • flat layout → tests import the wrong copy (adopt `src/`) • different commands in hook vs. CI → "green locally, red in CI" (one `uv run` surface).
- **Industry usage:** reproducible environments are *mandatory* in regulated/enterprise software — auditors and security scanners need the exact dependency graph (the lockfile) to flag a vulnerable transitive package precisely. `pyproject.toml` + committed lockfile is the emerging default.
- **Common mistakes:** *beginner* — forgetting to commit the lockfile; *intermediate* — disabling mypy rules wholesale instead of fixing the type; *senior* — hook and CI running different commands.

## Interview & practice questions (Part 1)

1. What is the diamond dependency problem, and how does a resolver solve it?
2. Difference between `pyproject.toml` dependencies and a lockfile?
3. Why is `src/` preferred over a flat layout?
4. What does `mypy --strict` add over plain mypy? Where does Pydantic fit vs. mypy?
5. Why is Ruff faster than flake8 + isort + black?

*Practice — Easy:* which PEP 621 file holds metadata + config? • *Medium:* explain why committing `uv.lock` gives "works on every machine." • *Hard:* two deps pin incompatible `pydantic` versions — walk through exactly what uv does.

---

# PART 2 — AGENTIC AI

*The two mental models every agent is built on. Get them on paper before touching any framework — everything later in the curriculum is a variation on these two ideas.*

> Throughout this Part, the **backend is a black box**: whenever "the backend stores / resends / executes" appears, that is the machinery from Part 1 (a reproducible, typed, tested service). Part 3 opens that box and wires the two together; here we only need to know it exists and is trustworthy.

## Overview & motivation

An LLM by itself can only produce text. It cannot remember a conversation, look anything up, or take an action in the world. Yet products built on LLMs *do* remember, *do* search, *do* act. Where does that capability come from? Not from the model — from the code around it. These two mental models name that code precisely, so you always know which capability is the model's and which is yours to build.

## First principles & core terminology

- **LLM** (Large Language Model) — a fixed mathematical function that maps a sequence of **tokens** (chunks of text) to a probability distribution over the next token.
- **Stateless** — keeps *no memory* between calls; call it twice identically and the second call has no idea the first happened.
- **Context window** — the maximum number of tokens the model can see in a single call. Finite.
- **Agent** — ordinary code that repeatedly calls the LLM to decide and take actions until a goal is met.
- **Tool** — a function the agent can invoke (search, read a file, call an API). The model only *emits a request* to use a tool; it never runs code itself.

## Mental Model 1 — An LLM call is a stateless input → output function **[CORE]**

**First principles.** An LLM call takes text in and produces text out: `output = LLM(input)`. It keeps no memory between calls. Why: the model is a fixed function (frozen learned weights) whose *only* input is the text you send *this* call. There is no memory slot. (This falls out of the **Transformer** architecture — see Primary Sources.)

**Intuition.** A **brilliant amnesiac contractor**: total expertise, *zero* recollection of previous conversation. To continue, you must hand them the whole transcript, every time.

```
Turn 1: you send "My name is Vinay."   model sees ["My name is Vinay."]     → "Nice to meet you."
Turn 2: you send "What's my name?"      model sees ["What's my name?"]        → "I don't know."  ← no memory!
        To fix, the BACKEND resends:    ["My name is Vinay.", "...", "What's my name?"] → "Vinay."
```

**Under the hood — how the "illusion of memory" is built (backend = black box from Part 1):**
```
 ┌─────────── BACKEND (stateful — see Part 1) ────┐
 │  history = [system, u1, a1, u2, a2, ...]       │
 │        │  append new user message              │
 │        ▼                                        │
 │  prompt = join(history + new message)           │
 └────────┬────────────────────────────────────────┘
          │  send the FULL prompt every turn
          ▼
   ╔══════════════════════╗
   ║  LLM (stateless fn)  ║   output = f(prompt)   ← processes the whole thing FRESH
   ╚══════════╤═══════════╝
          │ reply
          ▼
   backend APPENDS reply to history (so next turn can resend it)
```

**Why designed this way.** A stateless function can be load-balanced across thousands of servers because no request is pinned to a machine holding its session; behavior depends only on visible input (reproducible, debuggable); and the model *reasons* while the backend owns *state* — a clean split.

**Consequences (the point of the week).** Memory is a **backend feature**, not a model feature. The context window is finite, so growing history must eventually be summarized, truncated, or retrieved — the seed of the later "memory/RAG" topic. And cost/latency scale with resent history: every turn re-pays for the whole transcript.

## Mental Model 2 — An agent is a `while` loop around that function **[CORE]**

**First principles.** An agent is ordinary code that repeatedly calls the LLM to decide and take actions until a goal is met: **perceive → reason → act → repeat → stop.** A single LLM call can only *say* things; to *do* things (search, read a file, call an API) and handle multi-step tasks, you wrap the call in a loop.

**The loop, term by term:**
- **Perceive** — read current state: the request, prior messages, and any *tool results* so far.
- **Reason** — the LLM chooses the next action: "call tool X with these args" or "final answer."
- **Act** — *the backend* executes the chosen tool (the model only *emits a request*; it can't run code) and captures the result.
- **Repeat** — the result is appended to state and fed back (Model 1 in action).
- **Stop** — the model answers instead of calling a tool, *or* a guard trips (max iterations / budget).

**Intuition.** A person with a to-do checklist: look at what you know → pick the next step → do it → look again. A thermostat is the tiniest agent; an LLM agent puts a reasoning brain in the "decide" box and *tools* in the "do" box.

**Under the hood — one iteration, and the loop diagram:**
```
LOOP:
  1. state = system + history + tool_results_so_far
  2. reply = LLM(state)                 # Model 1: stateless, full state resent
  3. if reply is a final answer: return reply          # STOP #1
     elif reply requests tool T(args):
        result = run_tool(T, args)       # backend executes — NOT the model
        history.append(reply); history.append(result)  # request + result
        if iterations > MAX: raise Stop  # STOP #2 (guard)
        goto LOOP
```
```
   ┌───── STATE: system + msgs + tool results ─────┐
   └───────────────────┬───────────────────────────┘
                       │ perceive
                       ▼
                ┌─────────────┐  "call tool"  ┌──────────────┐
                │ LLM: reason │──────────────►│ ACT: backend │
                └──────┬──────┘               │  runs tool   │
              "final"  │                       └──────┬───────┘
                       ▼                              │ append result
                   ┌────────┐                         └───► back to STATE
                   │ RETURN │
                   └────────┘
```

**Examples.** *Beginner:* one `calculator` tool — "what's 17% of 2,340?" → model calls it → result fed back → model answers. *Real-world:* a coding agent — read file (tool) → edit (tool) → run tests (tool) → repeat until green. *Counter-example:* a single `LLM("summarize this")` call is **not** an agent — no loop, no tools, no state between steps.

## Performance, trade-offs & comparison

- **Performance:** every loop iteration is a full model call over the *entire* accumulated state, so cost and latency grow with the number of steps *and* history length. Fewer, well-chosen tool calls beat many redundant ones.
- **Trade-offs:** more autonomy (longer loops, more tools) means more capability but more ways to spiral, overspend, or take a wrong action — which is exactly why guards and (later) approval gates exist.

| | Plain LLM call | Agent (loop) |
|---|---|---|
| Steps | one | many, until a stop condition |
| Takes real-world actions? | no (text only) | yes, via tools the *backend* executes |
| State | none | accumulated in the backend across iterations |
| Use when | single transform (summarize, classify) | multi-step tasks needing tools/decisions |

## Failure modes & common confusions

**Failure modes → recovery.** Assuming built-in memory (forgetting to resend history) → the model "forgets" • no max-iteration/budget guard → runaway loop + cost blowup • unbounded history → context-window overflow (summarize/truncate/retrieve) • trusting tool I/O without validation → crash deep in the loop (validate at the boundary — Part 1 rigor *inside* the loop).

**Confusions settled:** the model *requests* tools, the backend *runs* them • the model doesn't remember — the backend resends • an agent isn't a special model, it's ordinary loop code around ordinary stateless calls.

## Interview & practice questions (Part 2)

1. Why is an LLM stateless, and what does that imply for building chat?
2. How is "conversation memory" actually implemented?
3. Draw the agent loop. Where does state live? Who executes tools? What are two stop conditions?

*Practice — Easy:* T/F: the LLM stores your previous messages between calls. *(False.)* • *Medium:* show exactly what bytes get sent on turn 2 of a chat, and why. • *Hard:* your agent "forgets" the user's name after 20 turns *despite resending history* — give three root causes (context-window limit, truncation strategy, summarization loss) and how you'd diagnose each.

---

# PART 3 — THE BRIDGE (how backend and agentic layers depend on and interlink)

*Where Part 1 and Part 2 stop being two separate skill sets and become one system. Every concept here references something already taught above — nothing new is introduced.*

## Overview & motivation — why this Part exists

Parts 1 and 2 can each feel self-contained: one is "Python tooling," the other is "how LLMs work." Learned in isolation, they produce a common failure — an engineer who understands agents conceptually but ships one that is flaky, unreproducible, and impossible to debug, *for reasons that have nothing to do with the model's intelligence.* This Part exists to prevent that: it shows that the agentic layer (Part 2) **runs on top of** and **depends on** the backend layer (Part 1), and that the quality of the second is capped by the quality of the first.

## First principles — the single shared shape

Reduce both Parts to their skeleton and they are the *same* architecture:

> **A stateful backend wrapped around a stateless computational core.**

- In **Part 1**, the stateless core is the *code you run* and the backend machinery (uv/typing/tests) makes that surrounding system reproducible and validated.
- In **Part 2**, the stateless core is the *LLM call* (Model 1) and the backend is what holds all state across the loop (Model 2).

Recognizing this one shape is the payoff of Week 1: agent frameworks you meet later are all *variations on where this stateful/stateless boundary is drawn and who runs it.*

## Intuition & analogy

The LLM is a **brilliant amnesiac contractor** (Part 2). The backend (Part 1) is the **office around them**: the filing cabinet that remembers every past conversation, the assistant who re-briefs them each morning, the phone lines they use to actually *do* anything, and the manager who stops them working past a budget. The contractor supplies intelligence; the office supplies *memory, action, and control.* An agent is that whole office — not just the contractor. *Where the analogy breaks:* the office (backend) is deterministic and testable; the contractor (LLM) is the single non-deterministic part — which is exactly why you make everything *around* it as rigorous as possible.

## The dependency map (the core diagram of the week)

Every arrow is a place the agentic layer *reaches into* the backend layer:

```
        AGENTIC LAYER (Part 2)                 BACKEND LAYER (Part 1)
   ┌──────────────────────────────┐      ┌────────────────────────────────┐
   │  agent loop (Model 2)         │      │  reproducible env  (uv, lock)  │
   │    │ needs state persisted ───┼─────►│  stores conversation history   │
   │    │ needs a tool executed ───┼─────►│  runs the tool = an API call   │
   │    │ needs validated I/O   ───┼─────►│  Pydantic validates boundaries │
   │    │ needs a stop guard    ───┼─────►│  budget / max-iter enforcement │
   │  LLM call (Model 1) ──────────┼──────┤  (the one stateless core)      │
   └──────────────────────────────┘      └────────────────────────────────┘
     supplies: reasoning, decisions          supplies: memory, action, control
```

Read it as four dependencies, each traceable to a concept you already have:

1. **Memory depends on persistence.** Because the LLM is stateless (Model 1), conversation "memory" is just history the backend re-sends each turn — data the backend must store and reload reliably.
2. **Action depends on the backend executing tools.** The model only *requests* a tool (Model 2); the backend runs it. A tool *is an API call*, so every backend virtue (typed inputs, correct semantics, structured errors) becomes an agent virtue.
3. **Correctness depends on validated boundaries.** Tool inputs/outputs flow into the loop's state; if they are malformed, the next LLM call reasons over garbage. Pydantic validation from Part 1 belongs *inside* the loop.
4. **Safety/cost depends on backend control.** Stop conditions, budgets, and iteration caps are ordinary backend logic — the model cannot be trusted to stop itself.

## Under the hood — how a single turn crosses the boundary **[CORE]**

Trace one iteration and watch control pass back and forth between the two Parts:

```
1. [BACKEND]  load history from store         ← Part 1 (persistence + reproducible env)
2. [BACKEND]  assemble full prompt             ← Part 1 (Model 1 says: resend everything)
3. [LLM]      reason → "call tool T(args)"      ← Part 2 (stateless core)
4. [BACKEND]  validate args (Pydantic)          ← Part 1 (typed boundary)
5. [BACKEND]  execute T = an API call           ← Part 1 (the tool IS backend code)
6. [BACKEND]  validate result, append to state  ← Part 1
7. [BACKEND]  check budget / max-iters          ← Part 1 (control)
8. loop back to step 2, or return               ← Part 2 (loop shape)
```

Only **one** of eight steps (step 3) is the model. The other seven are Part 1. That ratio *is* the thesis of the week: an agent is mostly backend engineering wrapped around a small non-deterministic core.

## Data flow, runtime & failure modes across the boundary

- **Data flow:** user input → backend state → LLM → tool request → backend executes → result → backend state → LLM … The LLM never touches storage or the network directly; the backend mediates every crossing.
- **Coupled failure modes:** an unreproducible environment (Part 1 failure) makes an agent bug non-reproducible even though the *model* is deterministic given identical input • an untyped tool boundary (Part 1 failure) surfaces as the model "reasoning badly" when really it was fed malformed data • a missing stop guard (Part 1 failure) shows up as runaway cost (a Part 2 symptom). **The lesson:** many "the agent is dumb" bugs are actually backend bugs in disguise.

## Performance & trade-offs of the coupling

- Because every loop step re-pays for the whole resent history (Model 1) *and* runs through backend validation/persistence (Part 1), the two layers' costs compound. Optimizations therefore also compound: caching a stable prompt prefix (a Part 1 caching idea, later weeks) directly cuts Part 2 token cost.
- **Trade-off:** pushing more responsibility into the backend (validation, guards, persistence) adds code but is the *only* place reliability can come from — you cannot make the stateless, non-deterministic core more reliable, so you invest in everything around it.

## Interview & practice questions (Part 3)

1. In one sentence, why is backend rigor a prerequisite for reliable agents rather than a nice-to-have?
2. Given "the agent forgot the user's name," list two *backend* root causes before blaming the model.
3. Of the eight steps in one agent turn, how many are the model? Why does that ratio matter?
4. Name the single architectural shape shared by Part 1 and Part 2.

*Practice — Medium:* map each of the four dependency arrows to the specific Part 1 concept it relies on. • *Hard:* an agent double-charges a customer on a retried tool call — locate the failure on the dependency map and name which layer must fix it.

---

# Cheat Sheet

**Part 1 — toolchain in one breath:** uv (Python + venv + deps + resolution + lockfile + cache) · Ruff (lint+format, Rust, single-parse) · mypy --strict (types before runtime) · pytest+asyncio (proof) · pyproject.toml (PEP 621, one config) · committed lockfile (reproducibility) · `src/` layout (test the packaged product) · pre-commit + CI (identical commands both places).

**Part 2 — two models (verbatim):** (1) `output = LLM(input)` — **stateless**; memory is the backend resending history every turn. (2) **Agent = `while` loop:** perceive → reason → act → repeat until stop; the model *reasons*, the backend *acts, stores state, enforces limits*.

**Part 3 — the bridge:** LLM stateless ⇒ all state/memory/durability/control is the backend's job ⇒ an agent is mostly backend engineering around a small non-deterministic core ⇒ backend rigor is the prerequisite. Shared shape: *a stateful backend around a stateless core.*

**Mnemonics:** "**S**tateless" → the backend **S**ends history • "**PRAR**" → Perceive, Reason, Act, Repeat • "Intent vs. Reality" → `pyproject.toml` (ranges) vs. lockfile (exact pins) • "At rest vs. at runtime" → mypy vs. Pydantic • "1 of 8" → only one step of an agent turn is the model.

---

# Build This

**Definition of done — a skeleton repo that goes fully green *and* demonstrates the bridge:**
1. `uv init --package`, `uv python pin 3.13`, add the deps and dev tools from Part 1. Commit `uv.lock`.
2. Add the FastAPI `/hello` endpoint and the async test above.
3. Wire the pre-commit hook and CI stub. Confirm **all four pass**: `uv run ruff check .`, `uv run ruff format --check .`, `uv run mypy --strict src`, `uv run pytest`.
4. **Prove reproducibility (Part 1):** delete `.venv/`, run `uv sync`, confirm the tests still pass.
5. **Prove statelessness (Part 2):** write a ~30-line script that keeps a `messages` list, calls any LLM API twice, and shows that *omitting* the history makes it "forget" your name while *including* it makes it remember. You have now implemented "memory" by hand.
6. **Prove the bridge (Part 3):** turn the script into a minimal ReAct loop with one tool (a calculator), a `MAX_ITERS` guard, and history appended each turn — with the tool's input validated by Pydantic. Deliberately feed the tool a malformed arg and watch the *backend* (not the model) catch it. You have now seen all four dependency arrows fire.

---

# Active Recall & Self-Test

**Answer from memory (no looking):**
1. What exactly does a lockfile record that `pyproject.toml` doesn't, and why commit it? *(Part 1)*
2. Why is Ruff fast — name the two independent reasons. *(Part 1)*
3. Where does mypy stop and Pydantic start? *(Part 1)*
4. Write `output = LLM(input)` and explain why "stateless" makes conversation memory the backend's job. *(Part 2)*
5. List the four phases of the agent loop and two stop conditions. Who executes the tool? *(Part 2)*
6. Draw the dependency map: name the four things the agentic layer needs from the backend layer. *(Part 3)*

**Teach-back (60 seconds, aloud):** explain the Bridge — *why does an LLM being stateless make backend rigor a prerequisite for reliable agents?* If you stumble, re-read Part 3 and try again.

---

# Primary Sources

- **uv** — docs at `docs.astral.sh/uv` (resolution, locking, caching, Python management).
- **Ruff** — docs at `docs.astral.sh/ruff` (rules, formatter, config).
- **mypy** — `mypy.readthedocs.io`; strict mode reference.
- **PEP 621** — "Storing project metadata in pyproject.toml" (peps.python.org/pep-0621).
- **pytest / pytest-asyncio** — `docs.pytest.org`, `pytest-asyncio.readthedocs.io`.
- **FastAPI / Pydantic** — `fastapi.tiangolo.com`, `docs.pydantic.dev`.
- **Transformer / statelessness** — Vaswani et al., "Attention Is All You Need" (2017). Verify token/context-window specifics against your model provider's current API docs (they change across versions).

---

# Key Takeaways & Summary

- **Part 1:** Reproducibility is *engineered*, not hoped for — committed lockfile + pinned Python + `uv sync`; correctness has two layers (mypy at rest, Pydantic at runtime); fast tools get used, slow tools get skipped.
- **Part 2:** The LLM is a pure, stateless function; memory is something *you* construct; an agent is just a loop where the model reasons and the backend acts, stores state, and enforces limits.
- **Part 3:** The two layers are one system — a stateful backend around a stateless core. Only one of eight steps in an agent turn is the model; the rest is backend engineering. Therefore a rigorous backend *is* the substrate of a reliable agent.

**10-second:** Set up a fast, reproducible Python project (Part 1), understand that an LLM has no memory and an agent is a loop that adds memory and tools around it (Part 2), and see that the loop *runs on* the backend (Part 3).

**1-minute:** Part 1 builds a reusable skeleton — uv manages Python/packages/resolution and a committed lockfile for identical environments; Ruff lints+formats in one Rust pass; mypy --strict catches type bugs before runtime; pytest proves behavior; `pyproject.toml` is the one config; `src/` layout tests the packaged product; pre-commit + CI run the same checks everywhere. Part 2 fixes the model: `output = LLM(input)` is stateless, so "memory" is the backend resending history; an agent is a while loop — perceive, reason, act, repeat — where the model reasons and the backend executes tools and enforces limits. Part 3 connects them: memory, action, correctness, and control are all backend jobs the agent loop depends on, so backend rigor caps agent reliability.

**5-minute:** read Part 1 "Under the hood" + Part 2 both models + Part 3 dependency map and the single-turn trace, in order.

**Expert summary:** both layers share one architecture — a stateful backend around a stateless core. The toolchain makes that backend reproducible, fast to check, and validated at its boundaries; the mental models locate all durability inside it; the bridge shows the agent loop is seven parts backend to one part model. Backend rigor isn't adjacent to agent reliability — it *is* the substrate of it.

---

# Suggested Next Topics

- **Async Python** — the event loop, `async`/`await`, why I/O-bound LLM/web work is async-first.
- **The raw ReAct loop** — hand-roll perceive→reason→act with real tools, guards, and budgets, *before* a framework.
- **FastAPI + Pydantic v2 internals** — request/response models, dependency injection, validated boundaries.
- **The LLM API contract** — messages, roles, tokens, temperature, tool-calling schemas.
- **Context & memory strategies** — truncation, summarization, and the entry point to RAG.
