# Phase 0 — Week 1: Environment & the Two Foundational Mental Models

> **What this covers:** the modern Python backend toolchain (Part A) and the two mental models every agentic system is built on (Part B).
> **Who it's for:** a reader with zero prior knowledge — every term is defined before it's used.
> **The one idea that unites both Parts:** *an LLM has no memory, so all state, memory, and durability are your backend's job. That is why backend rigor is a prerequisite for building agents — not a nice-to-have.*
>
> **Depth tiers used below:** **[CORE]** = open every black box; **[WORKING]** = use it and know its tradeoffs, but treat its internals as a named black box; **[AWARE]** = know it exists and when to reach for it.

---

# PART A — The Modern Python Toolchain

## Overview & motivation

A **toolchain** is the set of programs that sit between *you writing code* and *code running correctly on someone else's machine*. Two everyday failures motivate the whole of Part A:

1. **"It works on my machine."** Your code runs perfectly, then breaks for a teammate or a server because they have a slightly different package version or Python. This is a *reproducibility* failure — and it is entirely preventable.
2. **"I'll skip the check, it's too slow."** If linting or type-checking takes 10 seconds, you stop running it, and it catches nothing. Fast tools change behavior: a 50-millisecond check gets run on every save.

The modern stack fixes both. Here is the whole thing at a glance (each row is explained below):

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

*(Defined before use — nothing here assumes you've programmed before.)*

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
5. **Materialize the environment** — instead of re-downloading/re-copying, uv keeps a **global content-addressed cache** (files stored under the hash of their contents, so identical files are stored once) and **hard-links** them into the venv. This, plus being compiled **Rust**, is why it's fast.

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

## Performance, trade-offs & comparisons

- **Performance:** uv installs are near-instant on a warm cache (hard-links, not copies); Ruff is milliseconds/file; mypy is the slowest (a full inference engine) but caches incrementally in `.mypy_cache/`. First cold resolve is a backtracking search — usually fast, occasionally expensive on pathological constraints, which is *why* caching matters. Type hints are **erased at runtime** — they cost nothing while the program runs.
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

## Interview & practice questions (Part A)

1. What is the diamond dependency problem, and how does a resolver solve it?
2. Difference between `pyproject.toml` dependencies and a lockfile?
3. Why is `src/` preferred over a flat layout?
4. What does `mypy --strict` add over plain mypy? Where does Pydantic fit vs. mypy?
5. Why is Ruff faster than flake8 + isort + black?

*Practice — Easy:* which PEP 621 file holds metadata + config? • *Medium:* explain why committing `uv.lock` gives "works on every machine." • *Hard:* two deps pin incompatible `pydantic` versions — walk through exactly what uv does.

---

# PART B — The Two Foundational Mental Models

> Get these *on paper* before touching any agent framework. Everything later in the curriculum is a variation on these two ideas.

## Mental Model 1 — An LLM call is a stateless input → output function **[CORE]**

**First principles.** An **LLM** (Large Language Model) call takes text in and produces text out: `output = LLM(input)`. **Stateless** means it keeps *no memory* between calls — call it twice identically and the second call has no idea the first happened. Why: the model is a fixed mathematical function (frozen learned weights) mapping a sequence of **tokens** (chunks of text) to a probability distribution over the next token. Its *only* input is the text you send *this* call. There is no memory slot. (This falls out of the **Transformer** architecture — see Primary Sources.)

**Intuition.** A **brilliant amnesiac contractor**: total expertise, *zero* recollection of previous conversation. To continue, you must hand them the whole transcript, every time.

```
Turn 1: you send "My name is Vinay."   model sees ["My name is Vinay."]     → "Nice to meet you."
Turn 2: you send "What's my name?"      model sees ["What's my name?"]        → "I don't know."  ← no memory!
        To fix, the BACKEND resends:    ["My name is Vinay.", "...", "What's my name?"] → "Vinay."
```

**Under the hood — how the "illusion of memory" is built:**
```
 ┌─────────── YOUR BACKEND (stateful) ───────────┐
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

**Consequences (the point of the week).** Memory is a **backend feature**, not a model feature. The **context window** (max tokens the model can see in one call) is finite, so growing history must eventually be summarized, truncated, or retrieved — the seed of the later "memory/RAG" topic. And cost/latency scale with resent history: every turn re-pays for the whole transcript.

## Mental Model 2 — An agent is a `while` loop around that function **[CORE]**

**First principles.** An **agent** is ordinary code that repeatedly calls the LLM to decide and take actions until a goal is met: **perceive → reason → act → repeat → stop.** A single LLM call can only *say* things; to *do* things (search, read a file, call an API) and handle multi-step tasks, you wrap the call in a loop.

**The loop, term by term:**
- **Perceive** — read current state: the request, prior messages, and any *tool results* so far.
- **Reason** — the LLM chooses the next action: "call tool X with these args" or "final answer."
- **Act** — *your backend* executes the chosen **tool** (the model only *emits a request*; it can't run code) and captures the result.
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

**Failure modes → recovery.** Assuming built-in memory (forgetting to resend history) → the model "forgets" • no max-iteration/budget guard → runaway loop + cost blowup • unbounded history → context-window overflow (summarize/truncate/retrieve) • trusting tool I/O without validation → crash deep in the loop (validate with Pydantic — Part A rigor *inside* the loop).

## Comparison & confusions (Part B)

| | Plain LLM call | Agent (loop) |
|---|---|---|
| Steps | one | many, until a stop condition |
| Takes real-world actions? | no (text only) | yes, via tools the *backend* executes |
| State | none | accumulated in the backend across iterations |
| Use when | single transform (summarize, classify) | multi-step tasks needing tools/decisions |

**Confusions settled:** the model *requests* tools, the backend *runs* them • the model doesn't remember — the backend resends • an agent isn't a special model, it's ordinary loop code around ordinary stateless calls.

## Interview & practice questions (Part B)

1. Why is an LLM stateless, and what does that imply for building chat?
2. How is "conversation memory" actually implemented?
3. Draw the agent loop. Where does state live? Who executes tools? What are two stop conditions?

*Practice — Easy:* T/F: the LLM stores your previous messages between calls. *(False.)* • *Medium:* show exactly what bytes get sent on turn 2 of a chat, and why. • *Hard:* your agent "forgets" the user's name after 20 turns *despite resending history* — give three root causes (context-window limit, truncation strategy, summarization loss) and how you'd diagnose each.

---

# THE BRIDGE — why Part A is a prerequisite, not a nicety

Because the LLM is stateless (Model 1) and the agent loops (Model 2), **every durable thing — conversation memory, tool-result history, retries, budgets, persistence between iterations — lives in your backend.** Notice the architecture is the *same split* in both Parts: a **stateful backend wrapped around a stateless computational core** (uv/typing/tests make that backend reproducible and validated; the mental models locate all state inside it). If your backend is sloppy — unreproducible env, untyped boundaries, no tests — your agent is unreliable *for reasons that have nothing to do with the model's intelligence.* That is why Week 1 forces backend rigor first: **an agent is only as trustworthy as the backend that runs its loop and holds its state.**

---

# Cheat Sheet

**Toolchain in one breath:** uv (Python + venv + deps + resolution + lockfile + cache) · Ruff (lint+format, Rust, single-parse) · mypy --strict (types before runtime) · pytest+asyncio (proof) · pyproject.toml (PEP 621, one config) · committed lockfile (reproducibility) · `src/` layout (test the packaged product) · pre-commit + CI (identical commands both places).

**Two models (verbatim):** (1) `output = LLM(input)` — **stateless**; memory is the backend resending history every turn. (2) **Agent = `while` loop:** perceive → reason → act → repeat until stop; the model *reasons*, the backend *acts, stores state, enforces limits*.

**Bridge:** LLM stateless ⇒ all state/memory/durability is the backend's job ⇒ backend rigor is the prerequisite.

**Mnemonics:** "**S**tateless" → the backend **S**ends history • "**PRAR**" → Perceive, Reason, Act, Repeat • "Intent vs. Reality" → `pyproject.toml` (ranges) vs. lockfile (exact pins) • "At rest vs. at runtime" → mypy vs. Pydantic.

---

# Build This

**Definition of done — a skeleton repo that goes fully green:**
1. `uv init --package`, `uv python pin 3.13`, add the deps and dev tools from Part A. Commit `uv.lock`.
2. Add the FastAPI `/hello` endpoint and the async test above.
3. Wire the pre-commit hook and CI stub. Confirm **all four pass**: `uv run ruff check .`, `uv run ruff format --check .`, `uv run mypy --strict src`, `uv run pytest`.
4. **Prove reproducibility:** delete `.venv/`, run `uv sync`, and confirm the tests still pass.
5. **Prove statelessness (Part B):** write a ~30-line script that keeps a `messages` list, calls any LLM API twice, and shows that *omitting* the history makes it "forget" your name while *including* it makes it remember. You have now implemented "memory" by hand.

*Stretch:* turn the script into a minimal ReAct loop with one tool (a calculator), a `MAX_ITERS` guard, and history appended each turn.

---

# Active Recall & Self-Test

**Answer from memory (no looking):**
1. What exactly does a lockfile record that `pyproject.toml` doesn't, and why commit it?
2. Why is Ruff fast — name the two independent reasons.
3. Where does mypy stop and Pydantic start?
4. Write `output = LLM(input)` and explain why "stateless" makes conversation memory the backend's job.
5. List the four phases of the agent loop and two stop conditions. Who executes the tool?

**Teach-back (60 seconds, aloud):** explain the Bridge — *why does an LLM being stateless make backend rigor a prerequisite for reliable agents?* If you stumble, re-read THE BRIDGE and try again.

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

- Reproducibility is *engineered*, not hoped for: committed lockfile + pinned Python + `uv sync`.
- Fast tools get used; slow tools get skipped — Rust tooling makes rigor cheap.
- Correctness has two layers: mypy (code, at rest) + Pydantic (data, at runtime).
- The LLM is a pure, stateless function; memory is something *you* construct.
- An agent is just a loop around it; the backend acts, stores state, and enforces limits.
- The bridge is the whole point: a rigorous backend *is* the foundation of a reliable agent.

**10-second:** Set up a fast, reproducible Python project, and understand that an LLM has no memory while an agent is a loop that adds memory and tools around it.

**1-minute:** Part A builds a reusable skeleton — uv manages Python/packages/resolution and a committed lockfile for identical environments; Ruff lints+formats in one Rust pass; mypy --strict catches type bugs before runtime; pytest proves behavior; `pyproject.toml` is the one config; `src/` layout tests the packaged product; pre-commit + CI run the same checks everywhere. Part B fixes the model: `output = LLM(input)` is stateless, so "memory" is the backend resending history; an agent is a while loop — perceive, reason, act, repeat — where the model reasons and the backend executes tools and enforces limits.

**5-minute:** read Part A "Under the hood" + Part B both models + The Bridge, in order.

**Expert summary:** both Parts share one architecture — a stateful backend around a stateless core. The toolchain makes that backend reproducible, fast to check, and validated at its boundaries; the mental models locate all durability inside it. Therefore backend rigor isn't adjacent to agent reliability — it *is* the substrate of it.

---

# Suggested Next Topics

- **Async Python** — the event loop, `async`/`await`, why I/O-bound LLM/web work is async-first.
- **The raw ReAct loop** — hand-roll perceive→reason→act with real tools, guards, and budgets, *before* a framework.
- **FastAPI + Pydantic v2 internals** — request/response models, dependency injection, validated boundaries.
- **The LLM API contract** — messages, roles, tokens, temperature, tool-calling schemas.
- **Context & memory strategies** — truncation, summarization, and the entry point to RAG.
