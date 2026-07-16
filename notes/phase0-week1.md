# Phase 0 — Week 1: Environment & the Two Foundational Mental Models

> Graduate-level study notes. Self-contained. Every term is explained before it is used.
> Two parallel tracks this week: **(A) the modern Python backend toolchain** and **(B) the two mental models of agentic systems**. They meet at one idea: *the LLM is stateless, so all state, memory, and durability are backend responsibilities.*

---

# PART A — The Modern Python Toolchain

---

## Overview

A "toolchain" is the set of programs that sit between *you writing code* and *code running correctly in production*. For Python in 2026 the modern, fast, opinionated stack is:

| Job | Tool | One-line role |
|---|---|---|
| Install packages, manage virtual environments, pin the Python version | **uv** | The "everything" package/environment manager (replaces pip, venv, pyenv, pip-tools, poetry) |
| Find bugs & enforce style, then auto-format | **Ruff** | Linter + formatter, written in Rust, ~10–100× faster than the tools it replaces |
| Catch type errors *before* running code | **mypy** (in `--strict` mode) | Static type checker |
| Prove the code works | **pytest** + **pytest-asyncio** | Test runner (and its async plugin) |
| Describe the project in one file | **pyproject.toml** (PEP 621) | Standardized project metadata & config |
| Guarantee everyone gets identical dependencies | **committed lockfile** | Exact, reproducible dependency graph |

The deliverable is a **project skeleton**: a reusable, correctly-wired repository you copy for every later week.

---

## Why was it created? (First Principles)

**What problem existed?** Historically, Python development was a pile of loosely-coordinated tools:

- `pip` installed packages but didn't record *exact* versions or resolve conflicts well.
- `venv`/`virtualenv` created isolated environments but were separate from installation.
- `pyenv` managed Python *versions* but was yet another tool.
- `poetry`/`pip-tools` added locking but were slow and sometimes fought each other.
- `flake8`, `black`, `isort`, `pylint` — four separate tools for linting/formatting/import-sorting, each with its own config and its own pass over your code.

Result: slow feedback, "works on my machine" bugs, and a fragmented config spread across `setup.py`, `setup.cfg`, `requirements.txt`, `.flake8`, `tox.ini`, etc.

**Why the modern stack exists:** to *collapse* that fragmentation. One fast tool for environments (uv), one fast tool for lint+format (Ruff), one config file (`pyproject.toml`), one guaranteed-reproducible artifact (lockfile). Speed matters because **fast feedback loops change behavior** — if linting takes 50ms instead of 10s, you actually run it on every save.

---

## Problem it solves

1. **Reproducibility** — "It works on my machine" is caused by different dependency versions. A committed lockfile makes every machine identical.
2. **Speed** — Rust-based tooling (uv, Ruff) removes the "I'll skip the check, it's too slow" temptation.
3. **Correctness before runtime** — a type checker catches a whole class of bugs (passing `None` where a string is expected) *without executing code*.
4. **Single source of truth** — `pyproject.toml` replaces a scatter of config files.

---

## Core Concepts (each explained from scratch)

### 1. Virtual environment
A **virtual environment** is a private folder holding one project's Python interpreter and its packages, isolated from the system Python and from other projects. Without it, installing `requests==2.0` for project X would break project Y that needs `requests==3.0`. Think of it as a *sandbox per project*.

### 2. Dependency *resolution* vs. sequential installation
A **dependency** is a package your code needs. Your dependencies have *their own* dependencies (transitive dependencies). **Resolution** is the act of finding one set of versions that satisfies *every* constraint simultaneously.

- **Old way (`pip install` one by one):** pip installed packages sequentially; installing A then B could leave you with a B that's incompatible with the A already installed. This is the *diamond dependency problem*.
- **Modern way (a resolver):** uv builds a **graph** of all constraints and solves it as a whole — like solving a system of equations — producing a single consistent set of versions, or a clear error if none exists.

```
Sequential (fragile):        Resolver (correct):
 install A  ─┐                 read ALL constraints
 install B  ─┤ hope they         │
 install C  ─┘  agree            ▼
                              solve the whole graph at once
                                 │
                                 ▼
                              one consistent version set
```

### 3. Lockfile
A **lockfile** records the *exact* resolved versions (and cryptographic hashes) of every direct and transitive dependency. You commit it to git. Anyone who checks out the repo gets *byte-identical* dependencies. `pyproject.toml` says "I want FastAPI"; the lockfile says "you will get FastAPI 0.115.4 and starlette 0.41.2 and … with these exact hashes."

### 4. Pinned Python version
"Pinned" = fixed to an exact version (e.g. Python 3.13.1), recorded in the repo, so the interpreter itself is reproducible — not just the packages.

### 5. Linting vs. Formatting
- **Linter** = a program that reads your code and flags likely *bugs* and *bad patterns* (unused variable, undefined name, mutable default argument). It's about *correctness/quality*.
- **Formatter** = a program that rewrites your code into a canonical *style* (spacing, line length, quote style). It's about *appearance*, and it removes all style debates.
- Ruff does **both**.

### 6. Static type checking
"Static" = analyzed *without running the program*. A **type checker** reads your **type hints** (annotations like `def f(x: int) -> str:`) and proves that values flow through the program consistently. `--strict` mode means *every* function must be annotated and no untyped "escape hatches" are silently allowed.

### 7. PEP 621 / `pyproject.toml`
A **PEP** is a "Python Enhancement Proposal" — a formal spec. **PEP 621** standardized how project metadata (name, version, dependencies, Python requirement) is written in `pyproject.toml`. Before it, every tool had its own format; now there's one agreed schema.

### 8. `src/` layout
A project structure where your importable code lives in a `src/` subfolder rather than the repo root:

```
myproject/
├── pyproject.toml
├── uv.lock
├── src/
│   └── myproject/
│       ├── __init__.py
│       └── app.py
└── tests/
    └── test_app.py
```

**Why `src/`?** It forces tests to import your package *the same way a user would after installation* (from the installed location), not accidentally from the working directory. This catches "it worked in dev but the packaging was broken" bugs.

---

## Internal Working (Under the Hood)

### How uv resolves and locks
1. **Read constraints** from `pyproject.toml` (`dependencies = ["fastapi>=0.115"]`).
2. **Fetch metadata** for candidate versions (uv aggressively caches this on disk, so repeat runs are near-instant).
3. **Solve** using a resolution algorithm (PubGrub-style backtracking): pick versions, detect conflicts, backtrack, and either converge on a consistent graph or produce a *human-readable* explanation of why it's impossible.
4. **Write the lockfile** (`uv.lock`) with exact versions + hashes.
5. **Materialize the environment** — uv uses a global cache of package files and hard-links/clones them into the venv instead of re-downloading and re-copying. That's a big part of why it's fast.

uv is written in **Rust**, so the resolver and the file operations run as compiled native code, not interpreted Python.

### How Ruff runs 800+ rules in milliseconds
1. **Parse once**: Ruff parses your file into an **AST** (Abstract Syntax Tree — a tree representation of code structure) a *single* time.
2. **Single traversal**: it walks that one tree and evaluates all enabled rules during the same pass, instead of each rule re-parsing the file (which is what a pile of separate Python tools effectively did).
3. **Rust**: compiled, no Python interpreter startup per file, easy parallelism across files.

```
Old: file → parse(flake8) → parse(isort) → parse(pydocstyle) ...  (N parses)
Ruff: file → parse ONCE → walk tree → [rule1, rule2, ... rule800] → fixes
```

### Why type checking catches real bugs
Type hints + a strict checker turn "boundaries" into *verified contracts*. When you add **Pydantic** (a library that validates data against a typed model at runtime) on top, you get a two-layer defense: mypy proves the *code* is internally consistent, and Pydantic proves the *data crossing into your program* (an HTTP request body, a JSON file) actually matches the declared shape. Bugs like "the API sent a string where we expected a number" are caught at the boundary instead of exploding three functions deep.

---

## Step-by-Step Execution (building the skeleton)

```bash
# 1. Install uv (once, system-wide) — see astral.sh/uv for the current installer

# 2. Create the project
uv init --package myproject      # creates pyproject.toml + src/ layout
cd myproject

# 3. Pin the Python version (writes .python-version, downloads if needed)
uv python pin 3.13

# 4. Add runtime dependencies (updates pyproject.toml AND uv.lock)
uv add fastapi uvicorn pydantic

# 5. Add dev-only tools (isolated to a dev dependency group)
uv add --dev ruff mypy pytest pytest-asyncio

# 6. Lint + format
uv run ruff check .      # lint
uv run ruff format .     # format

# 7. Type-check in strict mode
uv run mypy --strict src

# 8. Run tests
uv run pytest
```

`uv run <cmd>` guarantees the command runs *inside the locked environment* — you never manually "activate" a venv.

### Minimal file contents

`pyproject.toml` (PEP 621 core):
```toml
[project]
name = "myproject"
version = "0.1.0"
requires-python = ">=3.13"
dependencies = ["fastapi", "uvicorn", "pydantic"]

[dependency-groups]
dev = ["ruff", "mypy", "pytest", "pytest-asyncio"]

[tool.ruff]
line-length = 100

[tool.mypy]
strict = true

[tool.pytest.ini_options]
asyncio_mode = "auto"
```

`src/myproject/app.py` — a trivial FastAPI "hello" endpoint:
```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/hello")
async def hello() -> dict[str, str]:
    return {"message": "hello"}
```

`tests/test_app.py` — one passing async test:
```python
import pytest
from httpx import ASGITransport, AsyncClient
from myproject.app import app

@pytest.mark.asyncio
async def test_hello() -> None:
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        resp = await client.get("/hello")
    assert resp.status_code == 200
    assert resp.json() == {"message": "hello"}
```

### Pre-commit hook + CI stub
A **pre-commit hook** is a script git runs automatically *before* a commit is recorded; if it fails, the commit is blocked. This stops bad code from ever entering history.

`.pre-commit-config.yaml`:
```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.8.0
    hooks:
      - id: ruff        # lint
      - id: ruff-format # format
```

CI ("Continuous Integration" — a service that runs your checks on every push) stub, `.github/workflows/ci.yml`:
```yaml
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

---

## Data Flow

```
 you edit code
      │
      ▼
 [pre-commit] ── ruff check + ruff format ──► block if dirty
      │ pass
      ▼
   git commit
      │
      ▼
   git push ──► [CI] uv sync → ruff → mypy --strict → pytest ──► ✅ / ❌
```

---

## Examples

- **Beginner:** `uv add requests` — adds the dep, updates the lockfile, installs it, all in one command.
- **Real-world:** onboarding a new engineer = `git clone && uv sync`. Two commands and they have the *exact* environment CI uses.
- **Industry:** teams standardize on `pyproject.toml` + lockfile so that a security scan of the lockfile can flag a vulnerable transitive dependency precisely.
- **Edge case:** two dependencies require incompatible versions of a third → uv's resolver *fails loudly* with an explanation instead of silently installing something broken.
- **Counter-example (what NOT to do):** `pip install -r requirements.txt` with unpinned versions → next month a transitive dep releases a breaking change and your build breaks with zero code changes.

---

## Analogies

- **Lockfile = a recipe with exact grams**, not "some flour." Anyone reproduces the same cake.
- **Resolver = a seating planner** who places every wedding guest so no two enemies share a table — solved globally, not guest-by-guest.
- **Type checker = spell-check for logic** — it underlines the mistake before you "publish" (run).
- **`src/` layout = testing the packaged product**, not the parts still on your workbench.

---

## Comparisons

| Concern | Old tool(s) | Modern | Why switch |
|---|---|---|---|
| Env + install | pip + venv + pyenv + pip-tools | **uv** | one tool, 10–100× faster, real resolver, built-in Python management |
| Lint | flake8 + pylint | **Ruff** | one pass, Rust speed, 800+ rules |
| Format | black + isort | **Ruff format** | same tool as lint, one config |
| Types | (none / optional) | **mypy --strict** | catches bugs before runtime |
| Config | setup.py, .flake8, tox.ini … | **pyproject.toml** | single file (PEP 621) |

**Common misconceptions:**
- *"A lockfile and `pyproject.toml` are redundant."* No — `pyproject.toml` states *intent* (ranges); the lockfile records the *resolved reality* (exact pins).
- *"Type hints slow down my program."* No — they're erased at runtime; they only affect the static checker.
- *"Ruff is fast because it does less."* No — it does *more* rules; it's fast because of single-parse + Rust.

---

## Advantages
Reproducible builds, fast feedback, fewer runtime bugs, one config file, trivial onboarding, CI parity with local dev.

## Limitations
- The toolchain is new-ish; some legacy corporate setups still assume `requirements.txt`.
- `mypy --strict` on a large untyped legacy codebase is painful to adopt incrementally.
- Ruff doesn't (yet) cover *every* niche plugin from the old ecosystem.

## Trade-offs
- **Strictness vs. speed of writing:** `--strict` slows initial coding but pays back in caught bugs.
- **Locking vs. freshness:** pinned deps are stable but require deliberate updates to get security patches.
- **One-tool consolidation vs. flexibility:** fewer knobs, but you inherit the tool's opinions.

## Industry Usage
Reproducible environments are mandatory for regulated/enterprise software (audit, security scanning of the exact dependency graph). Fast CI reduces cost. `pyproject.toml` + lockfile is the emerging default across new Python projects and internal platform standards.

## Performance Considerations
- uv: near-instant installs via a global content-addressed cache + hard-links; resolution cached on disk.
- Ruff: milliseconds per file; parallel across files.
- mypy: slower (it's a whole type-inference engine); use its incremental cache (`.mypy_cache`).
- Time complexity intuition: resolution is a backtracking search — usually fast, but pathological constraint sets can be expensive; that's why caching and a good resolver matter.

## Common Mistakes
- **Beginner:** forgetting to commit the lockfile → reproducibility lost.
- **Beginner:** editing packages with bare `pip install` inside a uv project → the lockfile drifts from reality.
- **Intermediate:** using the flat (root) layout and getting tests that import the wrong copy of the package.
- **Intermediate:** disabling mypy rules wholesale instead of fixing the underlying type.
- **Expert pitfall:** letting CI use different commands than the pre-commit hook, so "green locally, red in CI." Keep them identical.
- **How to avoid:** one command surface (`uv run …`) used identically in hook and CI.

## Interview Questions
1. What is the diamond dependency problem and how does a resolver solve it?
2. Difference between `pyproject.toml` dependencies and a lockfile?
3. Why is a `src/` layout preferred over a flat layout?
4. What does `mypy --strict` add over plain mypy?
5. Why is Ruff so fast compared to flake8 + isort + black?
6. Static vs. runtime validation — where do mypy and Pydantic each fit?

## Practice Questions
- *Easy:* Which single file (PEP 621) holds project metadata and dependencies?
- *Easy:* What command adds a dev-only dependency with uv?
- *Medium:* Explain why committing `uv.lock` gives "works on every machine."
- *Medium:* Write a pre-commit config that runs Ruff lint + format.
- *Hard:* Two of your dependencies pin incompatible versions of `pydantic`. Walk through exactly what uv does and how you'd resolve it.
- *Hard:* Design a CI pipeline that guarantees parity with local pre-commit checks and explain each stage.

---
---

# PART B — The Two Foundational Mental Models

> Get these *on paper* before touching any agent framework. Everything later in the curriculum is a variation on these two ideas.

---

## Mental Model 1 — An LLM call is a stateless input → output function

### First Principles
- **What is it?** An **LLM** (Large Language Model) call takes text in and produces text out. That's the *entire* interface: `output = LLM(input)`.
- **Stateless** means: the function keeps **no memory** between calls. Call it twice with the same input and it has no idea the first call ever happened. Each call starts from a blank slate.
- **Why it exists this way:** the model is a fixed mathematical function (fixed learned weights) that maps a sequence of tokens to a probability distribution over the next token. There is no place inside it to "remember" your last question — its only input is the text you send *this* call.

### Intuition
Think of the LLM as a **brilliant amnesiac contractor**. Every time you talk to them, they have total expertise but *zero recollection* of your previous conversation. To continue a discussion, you must hand them a written transcript of everything said so far, every single time.

```
Turn 1:  you send:  "My name is Vinay."
         model sees: ["My name is Vinay."]           → "Nice to meet you, Vinay."

Turn 2:  you send:  "What's my name?"
         model sees: ["What's my name?"]  ← NOTHING about Vinay!
         → "I don't know your name."
```

To make turn 2 work, *your backend* must resend the history:
```
Turn 2:  model sees: ["My name is Vinay.",
                      "Nice to meet you, Vinay.",
                      "What's my name?"]
         → "Your name is Vinay."
```

### Under the Hood — how the "illusion of memory" is built
1. Each turn, your backend takes the stored conversation history.
2. It concatenates it into one big prompt (system message + prior user/assistant turns + new message).
3. It sends that whole thing to the model.
4. The model processes the *entire* sequence fresh (there's no saved state to resume), predicts the next tokens, returns them.
5. Your backend **appends** the new user message *and* the model's reply to its stored history — so next turn can resend it.

```
 ┌─────────── YOUR BACKEND (stateful) ───────────┐
 │  history = [sys, u1, a1, u2, a2, ...]         │
 │        │  append new user msg                 │
 │        ▼                                       │
 │  build prompt = join(history + new)            │
 └────────┬───────────────────────────────────────┘
          │  send FULL prompt every turn
          ▼
   ╔══════════════════════╗
   ║  LLM (stateless fn)  ║   output = f(prompt)
   ╚══════════╤═══════════╝
          │ reply
          ▼
  backend appends reply to history
```

### Why designed this way
- **Scalability:** a stateless function can be load-balanced across thousands of servers — any server can handle any request because none holds session state. If the model held memory, requests would be pinned to specific machines.
- **Determinism/debuggability:** output depends only on the input you can see, making behavior reproducible and inspectable.
- **Separation of concerns:** the model does *reasoning*; the backend owns *state*. Clean boundary.

### Consequences (this is the whole point of the week)
- **Memory is a backend feature**, not a model feature. Chat history, user profiles, long-term memory — all stored and re-injected by *you*.
- **Context window is finite.** The model can only "see" a bounded number of tokens. As history grows you must *summarize, truncate, or retrieve* — which is the seed of the entire "memory/RAG" topic later.
- **Cost & latency scale with resent history.** Every turn re-pays for the whole transcript.

---

## Mental Model 2 — An agent is a `while` loop around that function

### First Principles
- **What is it?** An **agent** is a program that repeatedly calls the LLM to *decide and take actions* until a goal is met. Formally, a loop:

```
perceive → reason → act → (repeat) → stop
```

- **Why it exists:** a single LLM call can only *say* things. To *do* things (search the web, read a file, call an API) and to handle multi-step tasks, you wrap the call in a loop that (a) lets the model choose an action, (b) executes it, (c) feeds the result back, and (d) repeats.

### The loop, defined term by term
- **Perceive** — read the current state: the user request, prior messages, and any *tool results* from earlier iterations.
- **Reason** — the LLM decides the next action: either "call tool X with these arguments" or "I'm done, here's the answer."
- **Act** — your backend *executes* the chosen tool (the model itself can't run code; it only emits a request), captures the result.
- **Repeat** — the tool result is appended to state and fed back in (Model 1 in action).
- **Stop condition** — the model answers instead of calling a tool, or a max-iteration/budget limit is hit.

### Intuition
An agent is a **person solving a task with a to-do loop**: look at what you know → decide the next step → do it → look again. A thermostat is the tiniest agent (perceive temperature → decide → act → repeat). An LLM agent is that loop with a reasoning brain in the "decide" box and *tools* in the "act" box.

### Under the Hood — one full iteration
```
LOOP:
  1. state = system + history + tool_results_so_far
  2. reply = LLM(state)                 # Model 1: stateless call, full state resent
  3. if reply is a final answer:
        return reply                    # STOP
     else reply requests tool T(args):
        result = run_tool(T, args)      # backend executes — NOT the model
        history.append(reply)           # the tool *request*
        history.append(result)          # the tool *result*
        goto LOOP
```

### Visual Diagram
```
        ┌──────────────────────────────┐
        │            STATE             │
        │ system + msgs + tool results │
        └───────────────┬──────────────┘
                        │ perceive
                        ▼
                 ┌─────────────┐
                 │  LLM: reason│  "call tool" or "final answer"
                 └──────┬──────┘
              final?    │    tool?
          ┌─────────────┴──────────────┐
          ▼                            ▼
      ┌────────┐                 ┌──────────────┐
      │ RETURN │                 │  ACT: backend│
      └────────┘                 │  runs tool   │
                                 └──────┬───────┘
                                        │ append result
                                        └────► back to STATE (repeat)
```

### Why designed this way
- **Tools extend a text model into the real world** — the model reasons; the backend acts. Same clean split as Model 1.
- **The loop handles the unknown number of steps** a task needs — you don't know in advance whether "book me a flight" takes 1 or 12 tool calls.
- **State persistence between iterations is a backend job** — exactly Model 1's lesson, now inside a loop.

### The Bridge (why Part A is a prerequisite, not a nicety)
Because the LLM is stateless (Model 1) and the agent loops (Model 2), **every durable thing — conversation memory, tool-result history, retries, budgets, persistence between iterations — lives in your backend.** If your backend is sloppy (unreproducible env, untyped boundaries, no tests), your agent is unreliable *for reasons that have nothing to do with the model.* That is why Week 1 forces backend rigor first: **the agent is only as trustworthy as the backend that runs its loop and holds its state.**

---

## Examples (agentic)
- **Beginner:** a loop where the only tool is `calculator`; ask "what's 17% of 2,340?" → model calls calculator → result fed back → model answers.
- **Real-world:** a coding agent: perceive repo → decide to read a file (tool) → read result → decide to edit (tool) → run tests (tool) → repeat until tests pass.
- **Industry:** customer-support agent that reads the ticket, calls an order-lookup API, then a refund API, then replies.
- **Edge case:** the model loops forever calling the same failing tool → a **max-iteration guard** (backend) must stop it.
- **Counter-example:** a single `LLM("summarize this")` call is *not* an agent — no loop, no tools, no state between steps.

## Comparisons
| | Plain LLM call | Agent (loop) |
|---|---|---|
| Steps | one | many, until stop condition |
| Can take actions? | no (text only) | yes, via tools executed by backend |
| State | none | accumulated in backend across iterations |
| Use when | single transform (summarize, classify) | multi-step tasks needing tools/decisions |

**Misconceptions:**
- *"The model runs the tools."* No — the model *requests*; the backend *executes*.
- *"The model remembers the conversation."* No — the backend resends it.
- *"An agent is a special kind of model."* No — it's ordinary code (a loop) around ordinary stateless calls.

## Common Mistakes (agentic)
- Assuming built-in memory → forgetting to resend history → the model "forgets."
- No max-iteration / budget guard → runaway loops and cost blowups.
- Letting history grow unbounded → context-window overflow (later: summarize/retrieve).
- Trusting tool inputs/outputs without validation → this is *exactly* why typed boundaries (Pydantic) from Part A matter inside the loop.

## Interview Questions (agentic)
1. Explain why an LLM is stateless and what that implies for building chat.
2. How is "conversation memory" actually implemented?
3. Draw and explain the agent loop. Where does state live? Who executes tools?
4. What stops an agent loop? Name two stop conditions.
5. Why does the statelessness of the LLM make backend rigor a prerequisite?

## Practice Questions (agentic)
- *Easy:* True/False: the LLM stores your previous messages between calls. (False)
- *Easy:* Name the four phases of the agent loop.
- *Medium:* Given a two-turn chat, show exactly what bytes get sent to the model on turn 2 and why.
- *Medium:* Add a stop condition and a max-iteration guard to a described agent loop.
- *Hard:* Your agent "forgets" the user's name after 20 turns even though you resend history. Give three distinct root causes (context-window limit, truncation strategy, summarization loss) and how you'd diagnose each.
- *Hard:* Explain, end to end, how a single "book me a flight" request becomes 6 stateless LLM calls, naming what state the backend holds between each.

---

## Week 1 Summary / Cheat Sheet

**Toolchain (say it in one breath):**
> uv (env+deps+python+lock) · Ruff (lint+format, Rust, single-parse) · mypy --strict (types before runtime) · pytest+asyncio (proof) · pyproject.toml (PEP 621, one config) · committed lockfile (reproducibility) · src/ layout (test the packaged product) · pre-commit + CI (same commands both places).

**Two mental models (memorize verbatim):**
1. **`output = LLM(input)` — stateless.** Memory/conversation is an illusion the backend builds by resending prior messages every turn.
2. **Agent = `while` loop:** perceive → reason → act → repeat until stop. The model *reasons*; the backend *acts, stores state, and enforces limits*.

**The bridge:** LLM is stateless ⇒ all state/memory/durability is the backend's job ⇒ backend rigor is the prerequisite for reliable agents.

**Memory tricks:**
- "**S**tateless" → the backend **S**ends history every turn.
- "**PRAR**" → **P**erceive, **R**eason, **A**ct, **R**epeat.
- "**Intent vs. Reality**" → `pyproject.toml` = intent (ranges); lockfile = reality (exact pins).

**Build deliverable this week:** a reusable skeleton repo — uv-managed, Ruff+mypy+pytest wired into a pre-commit hook and CI stub, plus a trivial FastAPI `/hello` endpoint and one passing async test.
