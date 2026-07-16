# Phase 0 — Week 1: Environment & the Two Foundational Mental Models

> **Self-contained graduate-level notes.** Zero prior knowledge assumed. Every technical term is defined the moment it appears. Every "what" is preceded by a "why." No external textbook, course, or tutorial is required.
>
> This week runs **two parallel tracks** that meet at a single idea:
> - **Track A — the modern Python backend toolchain** (`uv`, Ruff, mypy, pytest, `pyproject.toml`, lockfiles, `src/` layout).
> - **Track B — the two mental models of agentic systems** (the stateless LLM; the agent loop).
>
> **The bridge that unites them:** *An LLM has no memory. Therefore every durable thing — state, memory, retries, budgets — is your backend's responsibility. That is why backend rigor is a prerequisite, not a nicety.*

---

## Table of Contents

1. [Overview](#1-overview)
2. [Motivation](#2-motivation)
3. [Historical Background](#3-historical-background)
4. [Problem Statement](#4-problem-statement)
5. [First Principles](#5-first-principles)
6. [Fundamental Building Blocks & Terminology](#6-fundamental-building-blocks--terminology)
7. [Intuition](#7-intuition)
8. [Analogies](#8-analogies)
9. [Visual Diagrams](#9-visual-diagrams)
10. [Internal Architecture](#10-internal-architecture)
11. [Step-by-Step Execution — Building the Skeleton](#11-step-by-step-execution--building-the-skeleton)
12. [Under the Hood](#12-under-the-hood)
13. [Data Flow](#13-data-flow)
14. [Memory & Runtime Behavior](#14-memory--runtime-behavior)
15. [Performance Analysis](#15-performance-analysis)
16. [Trade-offs](#16-trade-offs)
17. [Comparisons](#17-comparisons)
18. [Failure Modes](#18-failure-modes)
19. [Industry Usage](#19-industry-usage)
20. [Best Practices](#20-best-practices)
21. [Common Mistakes](#21-common-mistakes)
22. [Interview Questions](#22-interview-questions)
23. [Practice Questions](#23-practice-questions)
24. [Cheat Sheet](#24-cheat-sheet)
25. [Build This — Definition of Done](#25-build-this--definition-of-done)
26. [Active Recall & Self-Test](#26-active-recall--self-test)
27. [Primary Sources](#27-primary-sources)
28. [Key Takeaways](#28-key-takeaways)
29. [Summary](#29-summary)
30. [Suggested Next Topics](#30-suggested-next-topics)

---

## 1. Overview

By the end of this week you will have built **one reusable thing** and understood **two unshakeable ideas**.

**The thing:** a *project skeleton* — a small, correctly-wired Git repository you will copy at the start of every future week. It manages its own Python version, its own packages, formats and lints itself, checks its own types, runs its own tests, and refuses to let broken code be committed.

**The two ideas:**
1. A call to a Large Language Model (**LLM** — a program that takes text in and produces text out) is a *stateless function*: it remembers nothing between calls.
2. An *agent* is a `while` loop wrapped around that function: **perceive → reason → act → repeat**, until a stop condition.

Everything in the rest of the curriculum is a variation on those two ideas, and everything you *build* will sit inside the skeleton. So Week 1 is deliberately foundational: get the tools and the mental models right, and the following months are "just" elaboration.

Here is the whole toolchain at a glance (each term is fully defined later — this table is only a map):

| Job to be done | Tool | One-line role |
|---|---|---|
| Install packages, manage isolated environments, pin the Python version, lock dependencies | **uv** | The "everything" package/environment manager (replaces pip, venv, pyenv, pip-tools, poetry) |
| Find likely bugs & enforce style, then auto-format | **Ruff** | Linter **+** formatter, written in Rust, ~10–100× faster than the tools it replaces |
| Catch type errors *before* the program runs | **mypy** (`--strict`) | Static type checker |
| Prove the code actually works | **pytest** + **pytest-asyncio** | Test runner (and its plugin for `async` code) |
| Describe the whole project in one file | **pyproject.toml** (PEP 621) | Standardized project metadata + tool config |
| Guarantee every machine gets identical dependencies | **committed lockfile** (`uv.lock`) | The exact, reproducible dependency graph |
| Keep importable code isolated from the repo root | **`src/` layout** | Forces tests to import the package the way a user would |

---

## 2. Motivation

**Why does any of this exist? What problem came first?**

Building software that uses AI is *not primarily an AI problem — it is a systems problem.* The model is a component you rent; the reliability, cost, correctness, and memory of the overall system are yours to engineer. Two failure patterns motivate the whole week:

1. **"It works on my machine."** You write code, it runs perfectly, you hand it to a teammate or a server, and it breaks — because they have a slightly different package version, a different Python, or a different operating system. Hours vanish chasing ghosts. This is a *reproducibility* failure, and it is entirely preventable.

2. **"The AI forgot what I just said."** You build a chatbot, it greets you by name, and two questions later it has no idea who you are. Beginners conclude the model is broken. It is not — the model is *stateless by design*, and "memory" was never the model's job. It was always yours. Not understanding this produces subtly broken agents forever.

The week fixes both at their root: **make the environment reproducible and rigorous (Track A)**, and **understand exactly where state lives (Track B)**. The motivation for doing them *together, first,* is the bridge: because the LLM holds no state, an unreliable backend makes an unreliable agent — for reasons that have nothing to do with the model's intelligence.

---

## 3. Historical Background

**Track A — how Python tooling fragmented, then re-consolidated.**

Python (created by Guido van Rossum, first released 1991) had no built-in notion of project isolation for years. Tools accreted one at a time to patch each gap:

- **`pip`** (2008) installed packages — but recorded no exact versions and resolved conflicts poorly.
- **`virtualenv`** (2007) and later the built-in **`venv`** (2012, PEP 405) created isolated environments — but as a *separate* step from installing.
- **`pyenv`** managed multiple *Python versions* — yet another tool.
- **`pip-tools`** and **Poetry** (2018) added *locking* (recording exact versions) — but were slow and sometimes fought each other and pip.
- **`flake8`**, **`pylint`** (linting), **`black`** (formatting, 2018), **`isort`** (import sorting) — four more tools, each with its own config file and its own separate pass over your source.

The result by ~2020: to start one project you juggled `pip` + `venv` + `pyenv` + `pip-tools` + `flake8` + `black` + `isort`, configured across `setup.py`, `setup.cfg`, `requirements.txt`, `.flake8`, `tox.ini`, and more.

The consolidation came from a company called **Astral**, which asked "what if these were rewritten in **Rust** (a compiled, fast, memory-safe systems language) as *one or two* fast tools?" They shipped **Ruff** (2022) — one tool replacing flake8 + black + isort + more — and **uv** (2024) — one tool replacing pip + venv + pyenv + pip-tools + Poetry. Both are 10–100× faster than what they replaced, which changes behavior: *a check that takes 50 milliseconds gets run on every save; a check that takes 10 seconds gets skipped.*

**Track B — where the "stateless model" came from.**

Modern LLMs are built on the **Transformer** architecture (introduced in the 2017 paper *"Attention Is All You Need"* by Vaswani et al. at Google). A Transformer is a fixed mathematical function: given a sequence of tokens (chunks of text), it outputs a probability distribution over the next token. Crucially, its only input is the sequence you hand it *right now*. There is no hidden slot inside the trained weights that accumulates your conversation. Statelessness is not a bug or an oversight — it falls directly out of how the architecture works, and (as we'll see) it is what makes serving these models at planetary scale possible.

---

## 4. Problem Statement

We must solve, precisely:

1. **Reproducibility.** Given a repository, *any* person or server must be able to obtain a *byte-for-byte identical* set of dependencies and the *same* Python interpreter, with two commands.
2. **Fast correctness feedback.** Style, likely bugs, and type errors must be caught in *milliseconds to seconds*, locally *and* in automated checks, using *identical* commands in both places (so "green locally, red in CI" cannot happen).
3. **Validated boundaries.** Data entering the program (an HTTP request body, a JSON file, a tool's output) must be checked against a declared shape before it flows deeper.
4. **A correct mental model of state.** We must know, unambiguously, *who* holds conversation memory, *who* executes tools, and *what* stops an agent loop — because getting this wrong produces agents that are unreliable in ways no amount of prompt-tweaking fixes.

---

## 5. First Principles

We build every idea from absolute zero. Nothing below assumes you have programmed before.

**What is a program's "dependency"?** A **package** (also called a *library*) is reusable code someone else wrote that your program uses — for example, `fastapi` for building web servers. A **dependency** is any package your code needs to run. Packages have *their own* dependencies, called **transitive dependencies** (dependencies of dependencies).

**What is an "environment"?** When you install a package, it has to go *somewhere* on disk, associated with *some* Python interpreter. An **environment** is that "somewhere + which Python." If all projects share one global environment, installing `requests` version 2 for project X can break project Y that needs version 3.

**What is a Transformer / LLM at the level we need?** A function. Text sequence in → next-token probabilities out. Run repeatedly to generate a full response. The function's behavior is *fully determined by its input sequence plus its frozen internal numbers (weights)*. Same input ⇒ same behavior (modulo deliberate randomness we control). **There is no memory slot.**

**What is a "loop"?** A loop is code that repeats a block of steps until a condition tells it to stop. That single primitive — *repeat until done* — is the entire mechanism that turns a one-shot text generator into an agent that can take many steps toward a goal.

From these four primitives — *package*, *environment*, *stateless function*, *loop* — everything this week is assembled.

---

## 6. Fundamental Building Blocks & Terminology

Each term is defined before it is ever used elsewhere. Per the depth-calibration rule (context.txt Rule 6), each concept is tagged **[CORE]** (open every black box), **[WORKING]** (use it correctly + know its failure modes, but don't reimplement it), or **[AWARE]** (know it exists and when to reach for it).

**Depth map for this week:**

| Concept | Tier | Why this tier |
|---|---|---|
| The stateless LLM (`output = f(input)`) | **[CORE]** | The single idea the whole curriculum rests on; must be understood to the metal. |
| The agent loop (perceive → reason → act → repeat) | **[CORE]** | Same — every later pattern is a variation on this. |
| Dependency resolution / lockfile | **[CORE]** | Reproducibility is the backend prerequisite; understand *why* a resolver ≠ sequential installs. |
| Static type checking + runtime validation (mypy + Pydantic) | **[CORE]** | The "validated boundaries" that make an agent's tool layer safe. |
| `uv` internals (PubGrub resolver, content-addressed cache) | **[WORKING]** | Use it daily and reason about its behavior; don't reimplement a SAT solver. |
| Ruff internals (single-parse AST, Rust) | **[WORKING]** | Know why it's fast and what it checks; the rule engine itself is a black box. |
| `src/` layout, `pyproject.toml`, PEP 621 | **[WORKING]** | Standard project shape; adopt correctly, no need to study the packaging spec line-by-line. |
| pre-commit / CI wiring | **[WORKING]** | Wire it once, understand parity; the runner internals stay closed. |
| Transformer / token / context window | **[AWARE]** this week | You only need "text in → next-token out, no memory slot, finite window" now. Deep internals are an optional later module (see §27, Suggested Next Topics). |

### Track A terms

- **Virtual environment** — a private folder holding *one project's* Python interpreter and packages, isolated from the system Python and from every other project. A sandbox per project. Without it, projects poison each other's package versions.
- **Dependency resolution** — the act of finding *one* set of package versions that satisfies *every* constraint (yours + all transitive ones) *simultaneously*. Contrast with installing packages one-by-one and *hoping* they agree.
- **Diamond dependency problem** — package A needs `C>=2`, package B needs `C<2`; both A and B are needed. A resolver must detect this and either find a compromise or fail *loudly* with an explanation. Installing sequentially silently leaves you broken.
- **Lockfile** — a file (`uv.lock`) recording the *exact* resolved version **and cryptographic hash** of every direct and transitive dependency. You commit it to Git. Anyone who checks out the repo reproduces *identical* packages. (A **hash** is a short fingerprint of a file's bytes; if the fingerprint matches, the bytes are guaranteed unchanged — this detects tampering or corruption.)
- **Pinned Python version** — the interpreter itself fixed to an exact version (e.g. `3.13`), recorded in the repo (`.python-version`), so the *language runtime* is reproducible, not just the packages.
- **Linter** — a program that reads your code *without running it* and flags likely bugs and bad patterns (unused variable, undefined name, a dangerous mutable default argument). Concerned with *correctness & quality*.
- **Formatter** — a program that rewrites your code into one canonical *style* (spacing, line length, quote style). Concerned with *appearance*; it ends all style arguments because there is only one output.
- **Static type checking** — analysis performed *without running the program* ("static" = at rest). A **type checker** reads your **type hints** (annotations like `def f(x: int) -> str:` meaning "`x` is an integer, this returns a string") and proves values flow consistently. **`--strict`** means *every* function must be annotated and no untyped escape hatches are silently tolerated.
- **PEP** — a **Python Enhancement Proposal**, a formal specification document. **PEP 621** standardized how project metadata lives in `pyproject.toml`, replacing a scatter of per-tool formats.
- **`pyproject.toml`** — one standardized file describing the project (name, version, dependencies, required Python) *and* configuring the tools (Ruff, mypy, pytest). TOML is a simple, human-readable config format.
- **`src/` layout** — a structure where importable code lives under a `src/` subfolder instead of the repo root (diagram in §9). Forces tests to import the *installed* package, catching packaging bugs.
- **Pre-commit hook** — a script Git runs automatically *before* a commit is recorded; if it fails, the commit is blocked, so broken code never enters history.
- **CI (Continuous Integration)** — a service that automatically runs your checks (lint, types, tests) on every push to the shared repository, on a clean machine.
- **`async` / asynchronous code** — code that can *pause* while waiting (e.g. for a network reply) and let other work proceed, instead of blocking. Web servers and LLM calls are I/O-heavy, so this is the norm. `pytest-asyncio` is the plugin that lets pytest run `async` tests.
- **Pydantic** — a library that validates data against a *typed model* at **runtime** (while the program runs). Complements mypy, which validates at rest.

### Track B terms

- **Token** — a chunk of text (roughly a word-piece) that the model reads and emits. Models see text as a *sequence of tokens*.
- **Stateless** — keeps *no memory* between calls. Call it twice identically and the second call has no idea the first happened.
- **Context window** — the maximum number of tokens the model can "see" in one call. Finite. History that exceeds it must be summarized, truncated, or retrieved.
- **Tool** — a function your backend exposes to the model (search the web, read a file, look up an order). The model *requests* a tool with arguments; **your backend executes it** — the model cannot run code itself.
- **Agent** — ordinary code (a loop) that repeatedly calls the LLM to decide and take actions until a goal is met.
- **Stop condition** — what ends the loop: the model gives a final answer, or a guard (max iterations / budget) trips.

---

## 7. Intuition

**Track A intuition.** Think of a professional kitchen. The *toolchain* is the set of standardized stations — the same knives, the same measured recipes, the same cleaning checklist — so that *any* cook, on *any* night, produces the *same* dish. `uv` stocks the pantry with exact ingredients (the lockfile is the recipe in grams, not "some flour"). Ruff is the line cook who plates every dish identically and spots a spoiled ingredient instantly. mypy is the food-safety inspector who checks the *plan* before a single pan is lit. pytest is the taste test before the plate leaves the kitchen. Without standardized stations, every cook improvises and the restaurant is unpredictable.

**Track B intuition.** Picture a **brilliant amnesiac contractor**. Every time you speak to them they have world-class expertise but *zero* recollection of anything you said before. To continue a discussion you must hand them a written transcript of the whole conversation, *every single time*. That is the stateless LLM. Now imagine you want them to actually *do* a multi-step job: you give them a notepad and a rule — "look at everything known so far, decide the single next action, do it, write the result down, then look again." That repeating rule is the agent loop.

The reader should now *feel* both ideas before any formalism: tooling exists to make outcomes *repeatable*; the LLM exists as a *pure function*, so memory is something *you* construct.

---

## 8. Analogies

Multiple analogies, each with where it works and where it breaks.

- **Lockfile = a recipe measured in exact grams.** *Works:* anyone reproduces the identical cake. *Breaks:* a recipe doesn't verify the flour wasn't swapped — the lockfile's hashes *do* (better than the analogy).
- **Resolver = a wedding seating planner** who seats every guest so no two feuding relatives share a table — solved *globally*, not one guest at a time. *Works:* captures "satisfy all constraints at once." *Breaks:* a planner can bend rules; a resolver cannot — it either finds a valid seating or declares it impossible.
- **Type checker = spell-check for logic.** *Works:* underlines the mistake before you "publish" (run). *Breaks:* spell-check only knows words; mypy reasons about how values *flow*, which is deeper.
- **`src/` layout = testing the finished, boxed product** rather than the parts still on the workbench. *Works:* you test what the user actually gets. *Breaks:* the physical metaphor understates *why* — it's about Python's import path, not shipping cartons.
- **Stateless LLM = an amnesiac contractor.** *Works:* nails "no memory; resend the transcript." *Breaks:* a human tires as the transcript grows; the model instead hits a hard *context-window* wall and pays linearly in cost.
- **Agent loop = a person with a to-do checklist** (look → pick next step → do it → look again). *Works:* captures perceive/reason/act/repeat. *Breaks:* a person self-limits; an agent will loop forever unless *you* add a stop guard.

---

## 9. Visual Diagrams

**`src/` layout of the skeleton repo:**
```
myproject/
├── pyproject.toml          # PEP 621 metadata + tool config (one file)
├── uv.lock                 # exact, hashed dependency graph (committed)
├── .python-version         # pinned interpreter (e.g. 3.13)
├── .pre-commit-config.yaml # checks that block a bad commit
├── .github/workflows/ci.yml# same checks, on every push
├── src/
│   └── myproject/
│       ├── __init__.py     # marks the folder as an importable package
│       └── app.py          # the FastAPI "hello" endpoint
└── tests/
    └── test_app.py         # one passing async test
```

**Sequential install (fragile) vs. resolver (correct):**
```
Sequential (hope):            Resolver (proof):
 install A ─┐                   read ALL constraints
 install B ─┤ hope they           │
 install C ─┘  agree              ▼
                               solve the whole graph at once
                                  │
                                  ▼
                               one consistent version set
                               (or a clear, explained failure)
```

**Ruff's single-parse advantage:**
```
Old:  file → parse(flake8) → parse(isort) → parse(black) → ...   (N parses)
Ruff: file → parse ONCE → walk the tree → [rule1 … rule800] → fixes   (1 parse)
```

**The stateless LLM and the backend's illusion of memory:**
```
 ┌─────────── YOUR BACKEND (stateful) ────────────┐
 │  history = [system, u1, a1, u2, a2, ...]        │
 │        │  append new user message               │
 │        ▼                                         │
 │  prompt = join(history + new message)            │
 └────────┬─────────────────────────────────────────┘
          │  send the FULL prompt every turn
          ▼
   ╔══════════════════════╗
   ║  LLM  (stateless fn) ║   output = f(prompt)
   ╚══════════╤═══════════╝
              │ reply
              ▼
   backend APPENDS reply to history (so next turn can resend it)
```

**The agent loop:**
```
        ┌──────────────────────────────┐
        │            STATE             │
        │ system + msgs + tool results │
        └───────────────┬──────────────┘
                        │ perceive
                        ▼
                 ┌─────────────┐
                 │ LLM: reason │  → "call tool" OR "final answer"
                 └──────┬──────┘
              final?    │    tool?
          ┌─────────────┴──────────────┐
          ▼                            ▼
      ┌────────┐                 ┌───────────────┐
      │ RETURN │                 │ ACT: backend  │
      └────────┘                 │ runs the tool │
                                 └──────┬────────┘
                                        │ append result
                                        └────► back to STATE (repeat)
```

---

## 10. Internal Architecture

**Track A — the toolchain as a layered system:**
```
        You edit source in src/
                 │
   ┌─────────────┴─────────────────────────────┐
   │  uv  — owns: Python version, venv, deps,   │
   │        resolution, lockfile, caching       │
   └─────────────┬─────────────────────────────┘
                 │ provides a locked, isolated environment to:
   ┌─────────────┼───────────────┬──────────────┐
   ▼             ▼               ▼              ▼
 Ruff          mypy           pytest        FastAPI app
 (lint+fmt)   (types)       (+asyncio)     (runs in that env)
   │             │               │
   └──── all invoked as `uv run <tool>` (same env every time) ──┘
                 │
        pre-commit (local)  ═══ identical commands ═══  CI (remote)
```

**Track B — the agent as a system with a clear split of duties:**
```
  ┌────────────────── BACKEND (you own) ──────────────────┐
  │  • holds conversation + tool-result history            │
  │  • builds the prompt each turn                         │
  │  • executes tools                                      │
  │  • enforces stop conditions / budgets / retries        │
  │  • validates tool inputs & outputs (Pydantic)          │
  └───────────────┬────────────────────────────▲──────────┘
                  │ prompt (full state)         │ reply / tool request
                  ▼                             │
          ┌───────────────── LLM (rented) ──────┴──┐
          │  • reasons over the prompt              │
          │  • emits text OR a tool request         │
          │  • holds NO state between calls         │
          └─────────────────────────────────────────┘
```
Notice the architecture is *the same split* in both tracks: a stateful backend around a stateless computational core. That is not a coincidence — it is the bridge.

---

## 11. Step-by-Step Execution — Building the Skeleton

Every command is explained. `uv run <cmd>` always runs `<cmd>` *inside the locked environment*, so you never manually "activate" anything.

```bash
# 1. Install uv once, system-wide (see astral.sh/uv for the current installer).

# 2. Create the project with a src/ layout and a pyproject.toml
uv init --package myproject
cd myproject

# 3. Pin the Python version (writes .python-version; downloads that Python if missing)
uv python pin 3.13

# 4. Add runtime dependencies (updates BOTH pyproject.toml AND uv.lock, then installs)
uv add fastapi uvicorn pydantic

# 5. Add dev-only tools into an isolated dependency group
uv add --dev ruff mypy pytest pytest-asyncio httpx

# 6. Lint, then format
uv run ruff check .      # find likely bugs / style issues
uv run ruff format .     # rewrite into canonical style

# 7. Type-check in strict mode (analyzes without running)
uv run mypy --strict src

# 8. Run the tests (proof the code works)
uv run pytest
```

**Minimal `pyproject.toml` (PEP 621 core + tool config):**
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

**`src/myproject/app.py` — the trivial FastAPI "hello" endpoint:**
```python
from fastapi import FastAPI

app = FastAPI()


@app.get("/hello")
async def hello() -> dict[str, str]:
    # Fully type-annotated so `mypy --strict` is satisfied.
    return {"message": "hello"}
```

**`tests/test_app.py` — one passing async test:**
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

**`.pre-commit-config.yaml` — block a bad commit locally:**
```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.8.0
    hooks:
      - id: ruff         # lint
      - id: ruff-format  # format
```

**`.github/workflows/ci.yml` — the *same* checks on every push:**
```yaml
name: ci
on: [push, pull_request]
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
      - run: uv sync --dev           # reproduce the exact locked env
      - run: uv run ruff check .
      - run: uv run mypy --strict src
      - run: uv run pytest
```

**Onboarding, from here on, is two commands:** `git clone …` then `uv sync` — and the new machine has the *exact* environment CI uses.

---

## 12. Under the Hood

No black boxes. Here is what actually happens inside each piece.

### How uv resolves and locks (not sequential pip installs)
1. **Read constraints** from `pyproject.toml` (e.g. `fastapi>=0.115`).
2. **Fetch metadata** for candidate versions of each package and its transitive deps. uv aggressively caches this on disk, so repeat runs are near-instant.
3. **Solve** with a **PubGrub-style backtracking resolver**: it tentatively picks versions, detects a conflict, backtracks, and either converges on one consistent graph *or* produces a *human-readable* explanation of why no solution exists (this is the diamond problem, resolved).
4. **Write `uv.lock`** with exact versions + hashes.
5. **Materialize the environment** — instead of re-downloading and re-copying files, uv keeps a **global content-addressed cache** and **hard-links** (or reflink-clones) package files into the project's venv. "Content-addressed" means files are stored under the hash of their contents, so identical files are stored once and shared. This is a large part of why installs are near-instant. All of this is compiled **Rust**, not interpreted Python.

### How Ruff runs 800+ rules in milliseconds
1. **Parse once** into an **AST** (Abstract Syntax Tree — a tree that represents code structure: functions, arguments, expressions as nodes).
2. **Single traversal** — Ruff walks that one tree and evaluates *all* enabled rules during the same pass, rather than each rule re-parsing the file (which is what a pile of separate Python tools effectively did).
3. **Rust + parallelism** — compiled native code, no per-file Python interpreter startup, and files are checked in parallel across CPU cores.

### Why static type checking catches real bugs
Type hints + a strict checker turn every function boundary into a *verified contract*. mypy performs **type inference** (deducing types even where you didn't annotate) and **flow analysis** (tracking how a value's type narrows through `if` branches) to prove, without running anything, that you never pass a `None` where a `str` is required. Add **Pydantic** and you get a two-layer defense: *mypy proves the code is internally consistent (at rest); Pydantic proves the data entering the program at runtime actually matches the declared shape.* A bug like "the API sent a string where we expected a number" is caught **at the boundary** instead of exploding three functions deep.

### How the LLM's "illusion of memory" is built (Track B under the hood)
1. Each turn, the backend takes the stored conversation history.
2. It concatenates it into one prompt: `system message + prior user/assistant turns + new message`.
3. It sends that *whole* thing to the model.
4. The model processes the *entire* sequence *fresh* — there is no saved state to resume — predicts the next tokens, and returns them.
5. The backend **appends** both the new user message *and* the model's reply to its stored history, so the next turn can resend it.

There is no step where the model "remembers." Every turn is a cold start over the full transcript.

### One full iteration of the agent loop (under the hood)
```
LOOP:
  1. state = system + history + tool_results_so_far
  2. reply = LLM(state)                 # Model 1: stateless, full state resent
  3. if reply is a final answer:
        return reply                    # STOP condition #1
     elif reply requests tool T(args):
        result = run_tool(T, args)      # backend executes — NOT the model
        history.append(reply)           # record the tool *request*
        history.append(result)          # record the tool *result*
        if iterations > MAX: raise Stop  # STOP condition #2 (guard)
        goto LOOP
```

---

## 13. Data Flow

**Track A — from keystroke to green build:**
```
 you edit code
      │
      ▼
 [pre-commit] ── ruff check + ruff format ──► BLOCK commit if issues
      │ pass
      ▼
   git commit  ──►  git push
                      │
                      ▼
                   [CI] uv sync → ruff check → mypy --strict → pytest
                      │
                      ▼
                   ✅ merge   /   ❌ fix and repeat
```

**Track B — data moving through one agent turn:**
```
user request
      │
      ▼
 backend builds prompt  (system + history + tool results)
      │
      ▼
   LLM reasons ──► emits: tool request  OR  final answer
      │                        │
      │ (tool)                 │ (final)
      ▼                        ▼
 backend validates args   return answer to user
 (Pydantic) → runs tool
      │
      ▼
 validate result (Pydantic) → append to history → loop back
```
Notice Pydantic (Track A) sits *inside* the agent's data flow (Track B). That is the bridge made concrete.

---

## 14. Memory & Runtime Behavior

**Track A — what lives where, when:**
- The **global uv cache** lives once per machine, on disk, persisting across projects (content-addressed package files).
- Each project's **venv** holds hard-links into that cache — small on disk, created quickly, destroyed when you delete the folder.
- **mypy's incremental cache** (`.mypy_cache/`) persists between runs so re-checks only re-analyze changed files.
- At runtime the FastAPI app is an ordinary Python process; type hints are **erased at runtime** (they cost nothing while the program runs — they exist only for the static checker and for tools like Pydantic that *choose* to read them).

**Track B — what state exists, who owns it, how long:**
- The **conversation history** lives in *your backend* (memory, a database, wherever you put it), for as long as *you* decide. The model owns none of it.
- The model's **context window** is a *per-call* budget of tokens — it exists only for the duration of that single call and is gone the instant the call returns.
- **Cost and latency scale with resent history:** every turn re-pays (in tokens, time, and money) for the *entire* transcript you resend. This is why, later, you will *summarize, truncate, or retrieve* instead of resending everything — the seed of the "memory / RAG" topic.

---

## 15. Performance Analysis

**Track A:**
- **uv install:** near-instant on warm cache (hard-links, not copies); resolution cached on disk. First cold resolve is a backtracking search — usually fast, but pathological constraint sets can be expensive, which is *why* caching and a good resolver matter.
- **Ruff:** milliseconds per file; parallel across files; single parse.
- **mypy:** the slowest of the set — it is a full type-inference engine — but its incremental cache keeps re-runs cheap. Complexity grows with codebase size and typing depth.
- **General principle:** *fast feedback loops change behavior.* Sub-second checks get run constantly; multi-second checks get skipped, and skipped checks catch nothing.

**Track B:**
- **Latency & throughput** are dominated by the LLM call, which grows with the number of tokens in the resent context. An agent that resends a bloated history is slow *and* expensive.
- **Time/space intuition:** each agent iteration is one LLM call over `O(history length)` tokens; a task of `k` steps costs roughly `k` calls, each over a *growing* context — so naive agents are super-linear in cost. Managing context size is the main performance lever.
- **Bottleneck:** almost always the model call (network + generation), not your Python. This is why `async` matters — while one call is in flight, other work proceeds.

---

## 16. Trade-offs

**Track A:**
- **Strictness vs. speed of writing** — `mypy --strict` slows initial coding but repays it in bugs caught before runtime.
- **Locking vs. freshness** — pinned deps are stable and reproducible but require *deliberate* updates to pull in security patches.
- **Consolidation vs. flexibility** — one opinionated tool (uv, Ruff) means fewer knobs; you inherit the tool's opinions, gaining simplicity at the cost of niche customization.

**Track B:**
- **Autonomy vs. control** — more loop iterations and tools make an agent more capable but harder to bound and predict; guards (max iterations, budgets) trade some autonomy for safety.
- **Context completeness vs. cost** — resending the full history maximizes the model's awareness but maximizes tokens/cost/latency; summarizing saves money but risks losing detail.
- **Where to put intelligence** — do more work in deterministic backend code (cheap, testable, reproducible) or defer to the model (flexible, but stateless and costed per token)? A core design tension of the whole curriculum.

---

## 17. Comparisons

**Old Python tooling vs. modern:**

| Concern | Old tool(s) | Modern | Why switch |
|---|---|---|---|
| Env + install + versions | pip + venv + pyenv + pip-tools/Poetry | **uv** | one tool, 10–100× faster, real resolver, built-in Python management |
| Lint | flake8 + pylint | **Ruff** | single parse, Rust speed, 800+ rules |
| Format | black + isort | **Ruff format** | same tool as lint, one config |
| Types | (none / optional) | **mypy --strict** | catches a class of bugs before runtime |
| Config | setup.py, .flake8, tox.ini … | **pyproject.toml** | one file (PEP 621) |

**Plain LLM call vs. agent:**

| | Plain LLM call | Agent (loop) |
|---|---|---|
| Steps | exactly one | many, until a stop condition |
| Can take real-world actions? | no (text only) | yes, via tools the *backend* executes |
| State | none | accumulated in the backend across iterations |
| Use when | a single transform (summarize, classify) | multi-step tasks needing tools/decisions |

**mypy vs. Pydantic** (frequently confused): mypy checks *code* consistency *at rest, before running*; Pydantic validates *data* against a model *at runtime, as it enters*. They are complementary, not alternatives.

---

## 18. Failure Modes

**Track A — what breaks, why, symptom, recovery:**
- **Uncommitted lockfile** → "works on my machine." *Symptom:* teammate/CI gets different versions. *Recovery:* commit `uv.lock`; re-run `uv sync`.
- **Bare `pip install` inside a uv project** → lockfile drifts from reality. *Symptom:* env has packages the lockfile doesn't know. *Recovery:* use `uv add`; regenerate the lock.
- **Flat (root) layout** → tests import the wrong copy of the package. *Symptom:* "passes in dev, broken when installed." *Recovery:* adopt `src/` layout.
- **Different commands in hook vs. CI** → "green locally, red in CI." *Recovery:* one command surface (`uv run …`) used identically in both.

**Track B — what breaks, why, symptom, recovery:**
- **Assuming built-in memory** (forgetting to resend history) → the model "forgets." *Recovery:* the backend must resend the transcript every turn.
- **No max-iteration / budget guard** → runaway loop calling a failing tool forever → cost blowup. *Recovery:* add a hard iteration cap and a token/dollar budget guard.
- **Unbounded history growth** → context-window overflow / rising cost. *Symptom:* errors past N turns, or the model "forgets" the oldest facts. *Recovery:* summarize / truncate / retrieve.
- **Trusting tool inputs/outputs without validation** → malformed data crashes the loop deep inside. *Recovery:* validate every boundary with Pydantic — *exactly* why Track A rigor matters inside the loop.

---

## 19. Industry Usage

**Track A.** Reproducible environments are *mandatory* in regulated and enterprise software: auditors and security scanners need the *exact* dependency graph (the lockfile) to flag a vulnerable transitive package precisely. `pyproject.toml` + a committed lockfile is the emerging default for new Python projects and internal platform standards. Fast CI (Rust tooling) directly cuts cloud compute cost, because checks that used to take minutes now take seconds. New-engineer onboarding collapses to `git clone && uv sync`.

**Track B.** Every production "AI feature" you have used is these two models in disguise:
- A **customer-support agent** reads a ticket, calls an order-lookup API (tool), then a refund API (tool), then replies — an agent loop with validated tool boundaries.
- A **coding agent** perceives a repo, reads a file (tool), edits it (tool), runs tests (tool), and repeats until tests pass.
- A **chatbot with "memory"** is a stateless model plus a backend that stores and re-injects history (and, at scale, summarizes it).

The reason these systems are reliable at scale is *precisely* the stateless design: a stateless function can be load-balanced across thousands of servers because no request is pinned to a machine that holds its session — the backend holds all the state.

---

## 20. Best Practices

- **Commit the lockfile.** Reproducibility is worthless if the lock isn't in Git.
- **One command surface everywhere.** Use `uv run …` in the pre-commit hook *and* in CI so local and remote are identical.
- **Adopt `src/` layout from day one.** Cheap now; painful to retrofit.
- **Turn on `mypy --strict` at the start** of a new project; retrofitting strictness onto a large untyped codebase is the painful path.
- **Validate every boundary with Pydantic** — HTTP bodies, config files, and *tool inputs/outputs inside the agent loop*.
- **Always add two guards to any agent loop:** a max-iteration cap and a budget cap. Never ship a loop that can run forever.
- **Treat history as a managed resource,** not an ever-growing log — plan for summarization/truncation before you need it.
- **Keep the model's job (reason) and the backend's job (state, execute, guard, validate) cleanly separated.**

---

## 21. Common Mistakes

**Beginner:** forgetting to commit the lockfile • bare `pip install` inside a uv project • assuming the model remembers the conversation • thinking "the model runs the tools" (it only *requests*; the backend runs them).

**Intermediate:** flat layout causing wrong-import test bugs • disabling mypy rules wholesale instead of fixing the underlying type • letting agent history grow unbounded • no max-iteration guard.

**Senior / subtle:** hook and CI running *different* commands → "green locally, red in CI" (fix: identical `uv run` surface) • trusting tool outputs without validation and debugging the crash three layers deep • over-summarizing history and silently losing a fact the agent needed.

**Interview traps / misconceptions:** *"Type hints slow the program"* (no — erased at runtime) • *"Ruff is fast because it does less"* (no — it does *more* rules; it's fast via single-parse + Rust) • *"An agent is a special kind of model"* (no — it's ordinary loop code around ordinary stateless calls) • *"A lockfile and `pyproject.toml` are redundant"* (no — `pyproject.toml` states *intent*/ranges; the lockfile records *resolved reality*/exact pins).

---

## 22. Interview Questions

*With the kind of answer an interviewer wants.*

1. **What is the diamond dependency problem and how does a resolver solve it?** — Two deps require incompatible versions of a third; a resolver reads *all* constraints and solves the whole graph at once (backtracking), finding a consistent set or failing with an explanation.
2. **Difference between `pyproject.toml` dependencies and a lockfile?** — Intent (ranges) vs. resolved reality (exact versions + hashes, committed for reproducibility).
3. **Why is a `src/` layout preferred over flat?** — It forces tests to import the *installed* package, catching packaging bugs that a root layout hides.
4. **What does `mypy --strict` add over plain mypy?** — Requires full annotation and forbids silent untyped escape hatches, so more bugs are provable before runtime.
5. **Why is Ruff so fast vs. flake8 + isort + black?** — Single parse into one AST, all rules in one traversal, compiled Rust, parallel across files.
6. **Static vs. runtime validation — where do mypy and Pydantic each fit?** — mypy = code consistency at rest; Pydantic = data shape at runtime as it enters.
7. **Why is an LLM stateless, and what does that imply for chat?** — It's a fixed function over its input sequence with no memory slot; chat "memory" must be re-sent by the backend each turn.
8. **How is conversation memory actually implemented?** — The backend stores history and concatenates it into the prompt every turn; it appends each new message and reply.
9. **Draw and explain the agent loop. Where does state live? Who executes tools?** — perceive → reason → act → repeat; state lives in the backend; the *backend* executes tools (the model only requests them).
10. **What stops an agent loop? Name two conditions.** — The model returns a final answer; or a guard trips (max iterations / budget).
11. **Why does LLM statelessness make backend rigor a prerequisite?** — Because all durability (memory, retries, budgets, persistence) is the backend's job, so a sloppy backend yields an unreliable agent regardless of model quality.

---

## 23. Practice Questions

**Easy**
- Which single file (PEP 621) holds project metadata *and* tool config?
- What `uv` command adds a *dev-only* dependency?
- True/False: the LLM stores your previous messages between calls. *(False.)*
- Name the four phases of the agent loop.

**Medium**
- Explain why committing `uv.lock` gives "works on every machine."
- Write a pre-commit config that runs Ruff lint + format.
- Given a two-turn chat, show exactly what gets sent to the model on turn 2, and why.
- Add a stop condition *and* a max-iteration guard to a described agent loop.

**Hard / scenario**
- Two of your dependencies pin incompatible versions of `pydantic`. Walk through exactly what uv does and how you'd resolve it.
- Design a CI pipeline that guarantees parity with local pre-commit checks; justify each stage.
- Your agent "forgets" the user's name after 20 turns *even though you resend history*. Give three distinct root causes (context-window limit, truncation strategy, summarization loss) and how you'd diagnose each.
- Explain end-to-end how one "book me a flight" request becomes six stateless LLM calls, naming what state the backend holds between each.

**Thought experiments**
- If the LLM *did* hold session memory, what would break about serving it to a million concurrent users? (Hint: load balancing, pinning, failover.)
- Why does making a check 100× faster change *whether people run it at all* — and what does that imply about tool design generally?

---

## 24. Cheat Sheet

**Toolchain in one breath:**
> **uv** (Python + venv + deps + resolution + lockfile + cache) · **Ruff** (lint + format, Rust, single-parse) · **mypy --strict** (types before runtime) · **pytest + asyncio** (proof) · **pyproject.toml** (PEP 621, one config) · **committed lockfile** (reproducibility) · **`src/` layout** (test the packaged product) · **pre-commit + CI** (identical commands both places).

**Two mental models (memorize verbatim):**
1. `output = LLM(input)` — **stateless.** Memory/conversation is an illusion the backend builds by resending prior messages every turn.
2. **Agent = `while` loop:** perceive → reason → act → repeat, until a stop condition. The model *reasons*; the backend *acts, stores state, and enforces limits*.

**The bridge:** LLM is stateless ⇒ all state/memory/durability is the backend's job ⇒ backend rigor is the prerequisite for a reliable agent.

**Mnemonics:**
- "**S**tateless" → the backend **S**ends the history every turn.
- "**PRAR**" → **P**erceive, **R**eason, **A**ct, **R**epeat.
- "**Intent vs. Reality**" → `pyproject.toml` = intent (ranges); lockfile = reality (exact pins).
- "**At rest vs. at runtime**" → mypy (rest) vs. Pydantic (runtime).

**Common confusions, settled:** the model *requests* tools, the backend *runs* them · type hints are *erased at runtime* · Ruff is fast because it does *more in one pass*, not less · an agent is *ordinary loop code*, not a special model.

---

## 25. Build This — Definition of Done

**Task:** the reusable **project skeleton** you will copy at the start of every future week. Every command and file is in §11 — this section is the *acceptance test*, not new material.

**Runnable definition of done (all must pass on a clean machine):**

1. `git clone <repo> && cd <repo> && uv sync --dev` reproduces the environment with **zero** manual steps and **no** global Python required.
2. `uv run ruff check .` → **0 errors**; `uv run ruff format --check .` → **already formatted**.
3. `uv run mypy --strict src` → **Success: no issues found**.
4. `uv run pytest` → the one async test passes (`1 passed`).
5. `uv run uvicorn myproject.app:app` then `curl localhost:8000/hello` → `{"message":"hello"}`.
6. `uv.lock` and `.python-version` are **committed** to Git (`git status` shows them tracked, not ignored).
7. A deliberately broken commit (e.g. an unused import or an untyped function) is **blocked** by the pre-commit hook — verify by trying to commit one, then fix it.
8. Pushing triggers CI that runs the *same four commands* and goes green.

**Stretch (proves the mental models, not just the tooling):** add a `scripts/statelessness_demo.py` that calls any chat API **twice with no history resend** and shows the model "forgetting" the first message — then a second run that resends history and shows it "remembering." This makes Track B's core idea *observable*, not just asserted.

**Why this is the definition of done:** if all eight pass, you have engineered reproducibility (not hoped for it), and onboarding for the next 23 weeks is two commands. If any fail, the week has not landed — fix it before Week 2 (per the plan's rule of thumb: no running artifact ⇒ the week didn't stick).

---

## 26. Active Recall & Self-Test

Retrieval — not re-reading — is what builds durable memory. Answer these **from memory, notes closed**, then check against the section named.

**Active-recall questions (answer aloud or on paper, no peeking):**
1. Write the two mental models as one line each. *(→ §24 Cheat Sheet)*
2. What is the difference between `pyproject.toml` and `uv.lock`? *(→ §17)*
3. Who executes a tool — the model or your backend? *(→ §6, §10)*
4. Name the two things that can stop an agent loop. *(→ §6, §18)*
5. Why is Ruff fast — because it does *less* or *more*? Explain the mechanism. *(→ §12)*
6. Where does conversation "memory" actually live, and what does the backend do every turn? *(→ §12)*
7. mypy vs. Pydantic: what does each check, and *when* (at rest / at runtime)? *(→ §17)*
8. Why does LLM statelessness make it *easy* to serve to millions of users? *(→ §19)*

**60-second Teach-Back prompt (say it out loud, as if to a colleague):**
> "An LLM call is a stateless function — text in, text out, no memory. So every 'conversation' is my backend resending the full transcript each turn, and every durable thing (memory, retries, budgets) is the backend's job, not the model's. An agent is just a `while` loop around that function: perceive the state, let the model reason, my code acts (runs a tool), append the result, repeat until a final answer or a guard trips. That's why I set up a rigorous, reproducible backend *first* — uv for byte-identical environments, Ruff + mypy for fast correctness, pytest for proof — because the backend is the substrate the whole agent depends on."

If you stumble or need to peek, you haven't learned it yet — loop back to §7 (Intuition) and §10 (Internal Architecture).

**Spaced-repetition flashcards** (drop into Anki/any SRS; review across weeks):

| Front (Q) | Back (A) |
|---|---|
| `output = LLM(input)` implies what about chat? | The model has no memory; the backend resends history every turn. |
| Agent loop, four phases? | Perceive → Reason → Act → Repeat (until stop condition). |
| `pyproject.toml` vs `uv.lock`? | Intent/ranges vs. resolved reality/exact pins + hashes (committed). |
| Why commit the lockfile? | Byte-identical dependencies on every machine ("works everywhere"). |
| Why is Ruff ~10–100× faster? | Single parse into one AST + all rules in one pass + compiled Rust + parallel. |
| mypy vs Pydantic? | Code consistency **at rest** vs. data shape **at runtime** as it enters. |
| Two agent stop conditions? | Model emits final answer; or a guard trips (max iterations / budget). |
| Who runs the tool? | The backend. The model only *requests* it with structured args. |
| Why `src/` layout? | Tests import the *installed* package, catching packaging bugs. |
| Are type hints a runtime cost? | No — erased at runtime; read only by static checkers and tools like Pydantic. |

---

## 27. Primary Sources

Verify every non-obvious or version-specific claim here against the primary source (context.txt Rule 7). Tooling names/versions drift — re-check before relying.

**Track A — toolchain:**
- **uv** — official docs & resolver behavior: <https://docs.astral.sh/uv/> · repo: <https://github.com/astral-sh/uv>
- **Ruff** — docs (rules, formatter): <https://docs.astral.sh/ruff/> · repo: <https://github.com/astral-sh/ruff>
- **PubGrub** — the resolution algorithm uv's resolver is based on: <https://nex3.medium.com/pubgrub-2fb6470504f> and the Dart/pub original writeup.
- **mypy** — docs & `--strict`: <https://mypy.readthedocs.io/> · **PEP 484** (type hints): <https://peps.python.org/pep-0484/>
- **Pydantic v2** (Rust core `pydantic-core`): <https://docs.pydantic.dev/latest/>
- **PEP 621** — project metadata in `pyproject.toml`: <https://peps.python.org/pep-0621/>
- **PEP 405** — the built-in `venv`: <https://peps.python.org/pep-0405/>
- **pytest** & **pytest-asyncio**: <https://docs.pytest.org/> · <https://pytest-asyncio.readthedocs.io/>
- **FastAPI** (async, on Starlette) & **`src/` layout** guidance: <https://fastapi.tiangolo.com/> · packaging guide: <https://packaging.python.org/en/latest/discussions/src-layout-vs-flat-layout/>
- **pre-commit**: <https://pre-commit.com/>

**Track B — models & agents:**
- **"Attention Is All You Need"** (Vaswani et al., 2017 — the Transformer): <https://arxiv.org/abs/1706.03762>
- **ReAct** (the reason+act loop, Yao et al., 2022): <https://arxiv.org/abs/2210.03629>
- **Anthropic — "Building effective agents"** (agent = loop; when to use one): <https://www.anthropic.com/engineering/building-effective-agents>
- **Anthropic Messages API** (roles, statelessness, tool use — you resend history each call): <https://docs.claude.com/en/api/messages> · tool use: <https://docs.claude.com/en/docs/build-with-claude/tool-use>
- **OpenAI function calling / tool use** (for the Azure OpenAI SDK path the plan uses): <https://platform.openai.com/docs/guides/function-calling>

> ⚠️ **Verify before relying:** exact `uv`/`Ruff` versions, `.python-version` pins, API model IDs, and provider pricing move monthly (mid-2026). The *concepts* above are stable; the *product names/versions* are not.

---

## 28. Key Takeaways

- **Reproducibility is engineered, not hoped for.** A committed lockfile + pinned Python + `uv sync` make every machine identical.
- **Fast tools get used; slow tools get skipped.** Rust-based uv/Ruff make rigor cheap enough to run on every save.
- **Correctness has two layers:** mypy proves the code (at rest), Pydantic proves the data (at runtime).
- **The LLM is a pure, stateless function.** Memory is something *you* construct by resending history.
- **An agent is just a loop** around that function: perceive → reason → act → repeat, with the *backend* executing tools and enforcing stops.
- **The bridge is the whole point:** because state lives in the backend, a rigorous backend *is* the foundation of a reliable agent.

---

## 29. Summary

- **10-second explanation:** Set up a fast, reproducible Python project (uv, Ruff, mypy, pytest); understand that an LLM has no memory and that an agent is a loop that adds memory and tools around it.
- **1-minute explanation:** Track A builds a reusable skeleton: `uv` manages Python, packages, resolution, and a committed lockfile for byte-identical environments; Ruff lints+formats in one Rust pass; `mypy --strict` catches type bugs before runtime; pytest proves behavior; `pyproject.toml` holds one config; `src/` layout tests the packaged product; pre-commit + CI run the *same* checks locally and remotely. Track B fixes the mental model: `output = LLM(input)` is stateless, so conversation "memory" is the backend resending history each turn; an agent is a `while` loop — perceive, reason, act, repeat — where the model reasons and the backend executes tools and enforces limits.
- **5-minute explanation:** All of §5–§13, read in order.
- **Expert summary:** A stateful backend wrapped around a stateless computational core is the shared architecture of *both* tracks. The toolchain guarantees that backend is reproducible, fast to check, and validated at its boundaries; the mental models locate *all* durability (memory, tool results, retries, budgets, persistence) in that backend. Therefore backend rigor is not adjacent to agent reliability — it *is* the substrate of it. The deliverable — a uv-managed skeleton with Ruff/mypy/pytest wired into pre-commit and CI, plus a FastAPI `/hello` endpoint and a passing async test — is the reusable substrate for everything that follows.

**This week's build deliverable:** a reusable skeleton repo — uv-managed, Ruff + mypy + pytest wired into a pre-commit hook and a CI stub, plus a trivial FastAPI `/hello` endpoint and one passing async test.

---

## 30. Suggested Next Topics

- **Async Python in depth** — the event loop, `async`/`await`, and why I/O-bound LLM/web work is async-first.
- **FastAPI + Pydantic v2** — request/response models, dependency injection, validated boundaries in practice.
- **The LLM API contract** — messages, roles (system/user/assistant), tokens, temperature, and tool-calling schemas.
- **Building the agent loop by hand** — implement perceive→reason→act→repeat with real tools, guards, and budgets, *before* adopting a framework.
- **Context & memory strategies** — truncation, summarization, and the entry point to Retrieval-Augmented Generation (RAG).
- **Observability for agents** — logging every turn, tracing tool calls, and measuring cost/latency per iteration.
