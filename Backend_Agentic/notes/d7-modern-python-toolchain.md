# Day 7 — The Modern Python Toolchain + Your Permanent Project Skeleton

> **Framing.** Today you stop hand-assembling projects and build the *one* repository skeleton you will copy, commit into, and extend for the next 93 days. It is a small pile of files — `pyproject.toml`, a `src/` tree, a FastAPI endpoint, a test — but each file encodes a hard-won lesson about how software rots and how teams stop it from rotting. You will learn four tools that have quietly replaced a decade of Python tooling: **`uv`** (environments, dependencies, and Python versions, lockfile-first), **`ruff`** (lint + format in one Rust pass), **`mypy --strict`** (static types as *checked contracts*, not decoration), and **`pytest` + `pytest-asyncio`** (tests, including async ones). Then you wire them together with **pre-commit hooks** and a **GitHub Actions CI stub** so that no un-formatted, un-typed, or un-tested line can reach `main`.
>
> **Who it's for.** Someone who has installed a package with `pip install`, been burned by "works on my machine," and never understood what a lockfile actually locks. We assume you can write basic Python (Days P1–P3) and use git and a shell (Day P4). Nothing else.
>
> **The ONE idea that unites the backend and agentic layers:** *a dependency graph you did not pin is a distributed system you do not control, and a boundary you did not type is a runtime crash you chose to defer.* The exact same `pyproject.toml` + lockfile + type gate that keeps a FastAPI service reproducible is what keeps an **agent's tool-execution environment** from silently pulling a malicious transitive dependency and what stops a mistyped `tool_result` from blowing up three iterations deep in the agent loop. The toolchain is not ceremony — it is the *validated boundary* around code you will later hand a non-deterministic model the keys to. Per the plan's thesis, an agent is a distributed backend system with one non-deterministic component; today you build the substrate that makes the *deterministic* 95% trustworthy, so that on Day 24 the only thing you have to worry about is the model.

**A note on tooling versions.** Tool versions drift fast, and this is a fast-moving corner of the ecosystem (Astral shipped `uv` and `ruff` and is now building a type checker, `ty`). Every version number, flag, and transcript below is written against **verify-current** releases — roughly `uv` 0.9.x, `ruff` 0.14.x, `mypy` 1.18.x, `pytest` 8.x, `pytest-asyncio` 1.x, FastAPI 0.121.x, Pydantic 2.12.x as of mid-2026. The *commands and concepts* are stable; the exact output strings and default rule sets are the thing most likely to have shifted by the time you read this. Where a fact is version-sensitive I say "verify current" and cite the doc.

**Platform.** Unlike Day 5, none of this is Linux-only — `uv`, `ruff`, `mypy`, and `pytest` are cross-platform. But the plan runs builds in **WSL2/Ubuntu**, so the transcripts use a Unix shell. On native Windows PowerShell the same commands work; only the prompt (`$` vs `PS>`) and a couple of path separators differ.

**Reading order.** Part 1 builds the toolchain and the skeleton as a *backend* project. Part 2 puts an agent behind the same skeleton and shows why every one of these gates matters *more*, not less, when a model is in the loop. Part 3 is the dependency map where the two share one CI pipeline and one lockfile — and fail together when a transitive dep breaks.

---

# PART 1 — BACKEND

## 1.1 First, the problem: why does any of this exist?

**Depth: [CORE]**

### Intuition

You write `import requests` and it works on your laptop. Your colleague clones the repo and it explodes: they have a different `requests`, which pulls a different `urllib3`, which behaves differently on their Python 3.11 than your 3.12. Nobody changed any code. This is **dependency hell**, and it has three separate faces that beginners conflate:

1. **Which Python?** 3.11 vs 3.12 vs 3.13 — different syntax, different stdlib, different bugs.
2. **Which packages, and which versions of *their* packages?** You asked for `fastapi`; it dragged in `starlette`, `pydantic`, `anyio`, `typing-extensions`… a *transitive* graph of dozens you never named.
3. **Reproducibly?** Can a machine you've never touched — a colleague's, a CI runner's, a production container's — install the *exact* same bytes you tested against, today and in six months?

Before the modern toolchain, the answers were a scattered pile of tools that each solved one face and fought the others: `python` (version — managed by `pyenv`), `venv` (isolation), `pip` (install), `pip freeze > requirements.txt` (a *flat, un-hashed, platform-specific* snapshot masquerading as a lockfile), `setup.py`/`setup.cfg` (metadata), plus `flake8`, `black`, `isort`, `pylint`, `mypy`, `pytest` each configured in its *own* file with its own syntax. A new project meant ten decisions and ten config files before you wrote a line of logic.

The modern toolchain's thesis: **one metadata file (`pyproject.toml`), one lockfile (`uv.lock`), one fast linter/formatter (`ruff`), one type gate (`mypy`), one test runner (`pytest`), wired once and enforced automatically.** Configuration becomes a decision you make *once per skeleton* and copy forever. That's why today's build is "permanent."

### Analogy — the shipping container

Before 1956, cargo was loaded piece by piece — barrels, sacks, crates — each handled differently, each a chance to drop, mislabel, or lose. The **shipping container** standardized the *interface*: whatever's inside, the box is the same size, stacks the same way, and every crane, truck, and ship in the world handles it identically. A project's `pyproject.toml` + lockfile is that container: whatever your dependencies are, the *interface* to build and run your project is identical on every machine, and the lockfile is the manifest sealed to the box saying *exactly* what's inside, down to the cryptographic hash.

**Where the analogy breaks (non-negotiable to state):** two ways.
1. **A container's contents are inert; your dependencies execute code, on install and at import.** A shipping container can't reach out and grab a different barrel while at sea. A Python dependency's *transitive* graph can change under you the moment a version is unpinned — a sub-dependency ships a new release and your "sealed box" now contains something you never audited. The lockfile exists precisely to freeze the transitive graph, which no physical container needs to do. (This is the whole xz-utils story in §1.11.)
2. **The container standard is one global authority (ISO); Python's is an evolving stack of PEPs** implemented at different speeds by different tools. `pyproject.toml` metadata is PEP 621; dependency groups are PEP 735; the *standard* lockfile is PEP 751 (`pylock.toml`), which arrived *after* tools like `uv` shipped their own lockfile formats. So "the standard" is real but younger and more fragmented than a shipping container — you're standardizing on a moving target, which is why "verify current" is a recurring note.

### Worked example — the same project, two machines, without and with a lockfile

Concretely, here is the failure this whole day prevents. `requirements.txt` produced by `pip freeze` on machine A in January:

```
fastapi==0.109.0
starlette==0.35.1
```

You did **not** pin `anyio`. In March, machine B runs `pip install -r requirements.txt`. Between January and March, `anyio` released `4.3.0` with a behavior change, and `starlette==0.35.1` allows `anyio>=3.4.0`, so pip resolves `anyio==4.3.0` on machine B and `anyio==4.2.0` on machine A. Same `requirements.txt`, different bytes running, a bug that reproduces on B and not A. Multiply by ~40 transitive packages and you have the median Python onboarding experience circa 2020.

A lockfile records the *entire resolved graph* with hashes:

```
# excerpt of the idea (real uv.lock is larger and TOML)
anyio == 4.2.0   --hash=sha256:e1875bb4b4e2de1669f4bc7869b6d3f54231cda71b65f...
starlette == 0.35.1  --hash=sha256:3e2139...
fastapi == 0.109.0   --hash=sha256:8c7bab...
```

Now machine B installs `anyio==4.2.0`, verifies the hash, and runs the identical bytes. That is the entire value proposition, and it is a **security control** as much as a reproducibility one (§1.11).

**Deliberate stop.** I am not going to teach `pip`, `venv`, `pyenv`, `poetry`, or `pip-tools` as tools you'll use — `uv` subsumes all of them for this course. I name them so you recognize them in older repos and blog posts; treat them as the barrels-and-sacks era. The one exception worth knowing: `uv pip …` is a `pip`-compatible interface inside `uv` for when you must interoperate with pip-shaped workflows.

---

## 1.2 `uv` — one tool for Python versions, environments, and a locked dependency graph

**Depth: [CORE]**

### Intuition

`uv` (from Astral, written in Rust) collapses `pyenv` + `venv` + `pip` + `pip-tools` + `pipx` + `poetry` into a single binary that does three jobs, all lockfile-first:

- **Python versions:** `uv python install 3.12` downloads a standalone Python build; `requires-python` in `pyproject.toml` says which versions your project supports, and `uv` fetches one that satisfies it — you never install Python from python.org again.
- **Environments:** `uv` creates and manages a `.venv` *for you*, transparently. You almost never activate it by hand; `uv run <cmd>` runs `<cmd>` inside it, syncing first if anything drifted.
- **Dependencies as a locked graph:** `uv add fastapi` records the *constraint* (`fastapi>=…`) in `pyproject.toml` and writes the *exact resolved graph of everything* to `uv.lock`. That lockfile is **universal** — it captures the resolution across all platforms and Python versions your project supports, not just the one you happen to be on (this is the key advance over `pip freeze`, which snapshots only your current machine).

The mental model: **`pyproject.toml` is your intent ("I want FastAPI, roughly this new"); `uv.lock` is the reality ("here is the one exact set of 41 packages that satisfies that intent, hashed"); `.venv` is a disposable materialization of the lock you can delete and rebuild in seconds.** You edit intent, `uv` derives reality, you commit both, and every machine reconstructs the same `.venv`.

### Analogy — the recipe, the shopping list, and the stocked kitchen

`pyproject.toml` is a **recipe** written loosely: "flour, some good butter, a couple of eggs." `uv.lock` is the **itemized shopping receipt** from one specific shop on one specific day: "King Arthur bread flour, 500g, lot #A22; Kerrygold butter, 227g; …" — precise enough that anyone can reproduce the exact dish. The `.venv` is the **kitchen counter** stocked with those exact ingredients, ready to cook; you can wipe it down and re-stock from the receipt any time.

**Where the analogy breaks:** a shopping receipt is a *record of what you already bought*; `uv.lock` is computed *ahead of buying* by a **resolver** that must solve a constraint-satisfaction problem across the *entire transitive graph* simultaneously — "find one version of each of 41 packages such that every package's requirements about every other package are all satisfied at once." A receipt never had to prove that the flour and the butter are *compatible*; the resolver does exactly that, and it can fail ("no set of versions satisfies all constraints") in ways a shopping trip never can. That solving step is the "under the hood" below.

### Runnable example — initialize the skeleton, add deps, and watch the lockfile appear

```bash
# uv installs as a single binary; see docs.astral.sh/uv for your platform.
# (macOS/Linux) curl -LsSf https://astral.sh/uv/install.sh | sh
# Verify:
uv --version
# -> uv 0.9.2 (verify current)

# 1. Create the project skeleton with a src/ layout and a pinned Python.
uv init --package --python 3.12 skeleton
cd skeleton
# `--package` gives us a src/ layout + build metadata (see §1.3); plain `uv init`
# gives a flat script-style layout, which we do NOT want for the permanent skeleton.

# 2. Add runtime dependencies. Each `uv add` re-resolves and updates the lock.
uv add "fastapi>=0.121" "uvicorn[standard]>=0.38"
# -> Resolved 41 packages in 210ms
# -> Prepared 39 packages in 1.2s
# -> Installed 39 packages in 84ms
# ->  + fastapi==0.121.1
# ->  + pydantic==2.12.3
# ->  + starlette==0.49.1
# ->  + uvicorn==0.38.0
# ->  ... (35 more)

# 3. Add DEV-only tools to a dependency group (PEP 735) — not shipped to prod.
uv add --dev "ruff>=0.14" "mypy>=1.18" "pytest>=8.4" "pytest-asyncio>=1.2" "httpx>=0.28" "pre-commit>=4.3"
# -> Resolved 63 packages in 180ms
# ->  + ruff==0.14.0
# ->  + mypy==1.18.2
# ->  + pytest==8.4.1
# ->  ...

# 4. Look at what got created.
ls -a
# -> .  ..  .git  .gitignore  .python-version  .venv  README.md  pyproject.toml  src  uv.lock

# 5. Run something IN the managed env without activating anything.
uv run python -c "import fastapi, sys; print(fastapi.__version__, sys.version.split()[0])"
# -> 0.121.1 3.12.7
```

Now the two files that matter, side by side. `pyproject.toml` (intent):

```toml
[project]
dependencies = [
    "fastapi>=0.121",
    "uvicorn[standard]>=0.38",
]
[dependency-groups]
dev = ["ruff>=0.14", "mypy>=1.18", "pytest>=8.4", "pytest-asyncio>=1.2", "httpx>=0.28", "pre-commit>=4.3"]
```

`uv.lock` (reality — one excerpted entry of 63):

```toml
[[package]]
name = "anyio"
version = "4.11.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "idna" },
    { name = "sniffio" },
]
sdist = { url = "...anyio-4.11.0.tar.gz", hash = "sha256:82a8d0b81e3...", size = 219038 }
wheels = [
    { url = "...anyio-4.11.0-py3-none-any.whl", hash = "sha256:7c1a1...", size = 90420 },
]
```

**Why this works, line by line.**

- `uv init --package --python 3.12 skeleton` does four things at once: creates the directory, writes a `pyproject.toml` with `requires-python = ">=3.12"`, drops a `.python-version` file pinning the interpreter, and lays out `src/skeleton/`. The `--package` flag is the load-bearing one — it selects the *packaged* (`src/`) layout with a build backend instead of a loose script. Verify the exact scaffolding against `docs.astral.sh/uv` — the defaults have changed across 0.x releases.
- `uv add "fastapi>=0.121"` writes the **constraint** to `[project.dependencies]`, then runs the **resolver** over the whole graph, writes `uv.lock`, and syncs `.venv`. "Resolved 41 packages" means the resolver found a consistent assignment for FastAPI plus its 40 transitive deps. The `>=0.121` is *your floor*; the lock pins the *exact* `0.121.1` it chose.
- `uv add --dev …` writes to `[dependency-groups] dev` (PEP 735), a group that is installed for development and CI but **excluded from production installs** (`uv sync --no-dev`). This separation is why your deployed image doesn't ship `pytest` and `ruff` — a smaller image and a smaller attack surface.
- `.venv` is created and kept in sync automatically; it is `.gitignore`d (uv writes that ignore for you). **You never commit `.venv`; you always commit `uv.lock`.** That single rule is the difference between reproducible and not.
- `uv run python -c …` runs inside `.venv` without `source .venv/bin/activate`. Before running, `uv` checks the lock is current and syncs if not — so `uv run` can never execute against a stale environment. This is why every command in this note is `uv run <tool>`, not bare `<tool>`.
- Each `uv.lock` package entry carries `dependencies` (its edges in the graph) and per-artifact `hash`es. On install, `uv` verifies the downloaded bytes against the hash — a supply-chain check that a bare `requirements.txt` cannot do. The `source = { registry = ... }` records *where* it came from, so a swapped index is detectable.

### Under the hood — the resolver produces a *graph*, not a sequence

This is the [CORE] mechanism, and the plan calls it out explicitly. **`pip`'s classic install is sequential and greedy:** it processes requirements roughly in order, installing the first version of each package that looks satisfiable, and only *later* discovering that an earlier choice conflicts with a later requirement — at which point older pip would happily install an inconsistent set (the "pip doesn't have a real resolver" era; modern pip added backtracking, but it's still slower and per-platform).

`uv`'s resolver instead treats the whole thing as a **constraint-satisfaction / SAT-like problem over a dependency graph**:

```
        your intent (pyproject.toml)
        fastapi>=0.121 ── uvicorn[standard]>=0.38
              │                    │
        ┌─────┴─────┐        ┌─────┴─────┐
   starlette     pydantic   httptools  websockets ...
        │            │
      anyio     pydantic-core
        │
   ┌────┴────┐
  idna    sniffio        ← the resolver must pick ONE version of each node
                            such that EVERY edge's version constraint holds,
                            simultaneously, across ALL target platforms.
```

It builds the graph, gathers every version constraint on every node from every parent, and searches (with backtracking, guided by a **PubGrub**-style algorithm — the same family `poetry` uses, but implemented in Rust and heavily optimized) for an assignment that satisfies all edges at once. When it can't, it reports a *human-readable conflict* ("because A requires B<2 and C requires B>=2, no version works") instead of installing something broken.

Two properties make `uv.lock` special (primary source: `docs.astral.sh/uv/concepts/projects/layout` and `.../resolution`):

1. **It's universal.** The resolver doesn't just solve for *your* OS/arch/Python; it solves across the full matrix your `requires-python` and platform markers allow, and records conditional entries (e.g. "`colorama` only on Windows"). One `uv.lock`, committed once, is correct on Linux CI, a Mac laptop, and a Windows box — unlike `pip freeze`, which is a snapshot of one machine and silently wrong on the others.
2. **It's hash-verified and human-readable TOML** — auditable in code review, but **managed by uv, never hand-edited**. Note the honesty caveat: `uv.lock` is uv-specific and *not* consumable by other tools. The tool-agnostic standard is **PEP 751 `pylock.toml`**, which `uv` can *export* to (`uv export -o pylock.toml`) but does not use as its native project lockfile, because some of uv's information (e.g. full universal resolution) can't yet be expressed in `pylock.toml`. So "the standard lockfile" exists but is younger than `uv.lock`; verify the state of PEP 751 support before relying on cross-tool lock interchange.

**Why it's also fast:** beyond the algorithm, `uv` caches aggressively (a global cache of downloaded wheels and metadata, hard-linked into each `.venv` so N projects sharing a dep store it once), parallelizes network fetches, and is compiled Rust rather than interpreted Python. The result is resolutions and installs that are routinely 10–100× faster than `pip` — fast enough that `uv run` re-checking the lock on *every* command is imperceptible, which is what makes the "never run against a stale env" guarantee practical. **Deliberate stop:** I'm not opening PubGrub's clause-learning internals or uv's cache-layout on disk; the graph-plus-backtracking model above is enough to reason about every resolution success and failure you'll hit. Primary source for the algorithm family: Natalie Weizenbaum's PubGrub write-up; for uv specifically, `docs.astral.sh/uv/reference/internals/resolver`.

---

## 1.3 `pyproject.toml` (PEP 621) and the `src/` layout — one config file, one import boundary

**Depth: [CORE]**

### Intuition

**`pyproject.toml` is the single file that every modern Python tool reads.** Before it, a project had `setup.py` (executable metadata — a *security* problem, since installing ran arbitrary code), `setup.cfg`, `requirements.txt`, `MANIFEST.in`, plus `.flake8`, `.isort.cfg`, `mypy.ini`, `pytest.ini`, `.coveragerc` — up to a dozen dotfiles. **PEP 518** introduced `pyproject.toml`; **PEP 621** standardized *project metadata* (name, version, dependencies, Python requirement) inside it under `[project]`; and every tool worth using (`uv`, `ruff`, `mypy`, `pytest`, `coverage`) now reads its own config from a `[tool.<name>]` table in that same file. One file, declarative (TOML, not executable), version-controlled.

The **`src/` layout** is the second half of the boundary. Instead of putting your package directory at the repo root next to `tests/`, you nest it one level down in `src/`:

```
skeleton/                 ← repo root
├── pyproject.toml
├── uv.lock
├── src/
│   └── skeleton/         ← THE package (what gets imported and shipped)
│       ├── __init__.py
│       └── app.py
└── tests/
    └── test_app.py
```

Why the extra `src/` directory? Because of a subtle, real bug: with a **flat** layout (package at root), `import skeleton` finds the *source directory* even when the package was never installed — Python adds the current directory to `sys.path`. So your tests pass locally against un-built source, then fail in production because you forgot to include a file in the actual package, or a relative import only worked by accident. The `src/` layout makes the source directory **not importable by accident** — the only way to `import skeleton` is to have *installed* it (even in editable mode, `uv` does this for you). Your tests therefore run against the package *as users will get it*, catching packaging bugs before release. This is a documented, recommended practice (Python Packaging User Guide, "src layout vs flat layout").

### Analogy — the shop floor vs the stockroom

A flat layout is a shop where the stockroom and the showroom are the same space: customers (your tests, your imports) can grab things straight off the delivery pallet before they've been unpacked, priced, and shelved. Sometimes the pallet has an item the shelf doesn't — so it "works" in the shop and fails when shipped to a customer's home. The `src/` layout separates stockroom (`src/skeleton/`, raw source) from showroom (the *installed* package): customers can only buy what's actually been put on the shelf through the proper process (installation), so what you test is exactly what ships.

**Where the analogy breaks:** the shop's separation is physical and permanent, but Python's is *only* a `sys.path` convention. Nothing stops someone from `sys.path.insert(0, "src")` or setting `PYTHONPATH=src` and re-importing the raw source, re-creating the flat-layout bug on top of a `src/` tree. The protection is real but conventional, not enforced by the language — which is why the tooling (`uv`, the build backend) has to cooperate by installing the package for you rather than you relying on the directory structure alone.

### Runnable example — the complete permanent `pyproject.toml`

This is the file you copy for 93 more days. Every block is load-bearing and annotated.

```toml
# pyproject.toml — the permanent skeleton. Verify tool versions against current docs.

[project]                                    # PEP 621 metadata (standard, tool-agnostic)
name = "skeleton"
version = "0.1.0"
description = "Permanent project skeleton for the 100-day backend+agentic plan"
readme = "README.md"
requires-python = ">=3.12"                   # the resolver honors this; CI matches it
dependencies = [                             # RUNTIME deps only (shipped to prod)
    "fastapi>=0.121",
    "uvicorn[standard]>=0.38",
    "pydantic>=2.12",
]

[dependency-groups]                          # PEP 735 — dev tools, NOT shipped to prod
dev = [
    "ruff>=0.14",
    "mypy>=1.18",
    "pytest>=8.4",
    "pytest-asyncio>=1.2",
    "httpx>=0.28",
    "pre-commit>=4.3",
]

[build-system]                               # how the package is built (PEP 517/518)
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]             # tell the backend where the package lives
packages = ["src/skeleton"]

# ---- one config file, one table per tool ---------------------------------

[tool.ruff]
line-length = 100
src = ["src", "tests"]                        # so import-sorting knows first-party code

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP", "SIM", "ASYNC"]  # rule families — see §1.4

[tool.mypy]
strict = true                                # the whole point — see §1.5
files = ["src", "tests"]

[tool.pytest.ini_options]
asyncio_mode = "auto"                        # pytest-asyncio: await tests with no decorator
testpaths = ["tests"]
```

And the minimal package + init so the tree is importable:

```bash
# src/skeleton/__init__.py may be empty; its presence makes `skeleton` a package.
uv run python -c "import skeleton; print(skeleton.__file__)"
# -> /home/you/skeleton/.venv/lib/python3.12/site-packages/skeleton/__init__.py   (installed!)
#    Note the path is in site-packages, NOT src/ — proof the src/ layout works:
#    we imported the INSTALLED package, exactly what a user gets.
```

**Why this works, line by line.**

- `[project]` is the *standard* table — `uv`, `pip`, `build`, and PyPI all read it. `requires-python = ">=3.12"` is consumed by the resolver (§1.2) *and* is the source of truth CI should mirror. Keep it in one place; a mismatch between this and your CI matrix is a classic "green in CI, broken for a user on 3.11" bug.
- `dependencies` vs `[dependency-groups] dev` is the **prod/dev boundary**. `uv sync` installs both; `uv sync --no-dev` (what your Dockerfile runs) installs only runtime deps. Shipping `pytest` to production is both bloat and needless attack surface.
- `[build-system]` with `hatchling` tells any PEP 517 frontend how to build a wheel. `[tool.hatch.build.targets.wheel] packages = ["src/skeleton"]` is the line that makes the `src/` layout buildable — it points the backend at the real package dir. (Different backends discover `src/` differently; `hatchling` needs this hint. Verify against the Hatch docs.)
- The four `[tool.*]` tables replace four separate dotfiles. `ruff`'s `src = [...]` teaches its import-sorter which modules are first-party (so `import skeleton` sorts after third-party imports); `mypy`'s `strict = true` is §1.5; `pytest`'s `asyncio_mode = "auto"` is §1.6. **This is the payoff of PEP 518/621:** all configuration for the whole toolchain in one reviewable file.
- The final `import skeleton` printing a path inside `.venv/.../site-packages/` — *not* `src/` — is the `src/`-layout proof. If it had printed a `src/skeleton/...` path, the flat-layout footgun would still be armed.

**Deliberate stop.** I'm not covering `[project.scripts]` entry points, dynamic versioning (`hatch-vcs`), or building/publishing wheels to PyPI — you won't publish this skeleton, you'll *deploy* it (Day 8+ and Phase 8). The metadata above is everything the toolchain needs. Primary sources: PEP 621 (project metadata), PEP 518 (build-system table), PEP 735 (dependency groups), Python Packaging User Guide "src layout vs flat layout."

---

## 1.4 `ruff` — lint and format in one Rust pass

**Depth: [CORE]**

### Intuition

Two separate jobs, historically two-to-five separate tools, now one:

- **Formatting** = mechanically rewriting code to a canonical style (line length, quotes, spacing, trailing commas) so *nobody argues about it in review*. This was `black`'s job, plus `isort` for import ordering.
- **Linting** = detecting *likely bugs and bad patterns* without running the code — unused imports, undefined names, mutable default arguments, `== None` instead of `is None`, un-awaited coroutines. This was `flake8` (a wrapper over `pycodestyle` + `pyflakes` + `mccabe`) plus a zoo of `flake8-*` plugins, plus `pylint`, plus `pyupgrade`, plus `autoflake`.

**`ruff` replaces that entire stack** — `flake8` and its plugins, `black`, `isort`, `pydocstyle`, `pyupgrade`, `autoflake` — with one binary exposing >900 rules, and it does it 10–100× faster (Astral's own headline; independent testimonials in the Ruff docs report 150–1000× on large codebases). `ruff format` is the formatter (near-drop-in for `black`); `ruff check` is the linter (with `--fix` to auto-repair). One tool, one config table, one pass.

Why speed matters *pedagogically*, not just ergonomically: a linter that runs in 200ms can live in your editor-on-save and your pre-commit hook and your CI, giving instant feedback at every layer. A linter that takes 20 seconds (the flake8+pylint stack on a large repo) gets run "later," which means "never," which means style debates leak into code review. FastAPI's creator literally said Ruff is *so* fast he adds bugs on purpose to check it's running (Ruff docs, Testimonials). Speed changes behavior.

### Analogy — the print shop's automated proofreader

Imagine a newspaper where, before the old days, one person checked spelling (`pyflakes`), another checked column width and font (`black`), another alphabetized the classifieds (`isort`), and another flagged outdated phrasings (`pyupgrade`) — four passes, four people, four desks, and the paper waited on all of them. `ruff` is one machine that does all four checks in a single pass of the page through the scanner, in the time it took the old team to pick up their coffee.

**Where the analogy breaks:** a proofreader reads the *text as strings* — it can spell-check without understanding grammar. `ruff` does **not** operate on your code as text; it parses it into an **Abstract Syntax Tree** (a structured tree of "this is a function definition containing an if-statement containing a comparison") and reasons about *structure*. That's why it can catch "you compared to `None` with `==`" or "this default argument is a mutable list" — semantic facts invisible to a string scanner. The single-pass AST is the whole "under the hood," below.

### Runnable example — catch and auto-fix real issues, then format

Drop this deliberately-messy file into the skeleton:

```python
# src/skeleton/messy.py — intentionally full of problems, to see ruff work.
import os
import sys
from typing import List


def greet(names: List[str], prefix = "Hello"):
    result = []
    for n in names:
        result.append(prefix + ", " + n)
    return result
```

```bash
# LINT: find problems (no changes made).
uv run ruff check src/skeleton/messy.py
# -> messy.py:2:8: F401 [*] `os` imported but unused
# -> messy.py:3:8: F401 [*] `sys` imported but unused
# -> messy.py:4:1: UP035 [*] `typing.List` is deprecated, use `list` instead
# -> messy.py:7:23: UP006 [*] Use `list` instead of `List` for type annotation
# -> Found 4 errors.
# -> [*] 4 fixable with the `--fix` option.

# LINT + AUTO-FIX: apply the safe fixes.
uv run ruff check --fix src/skeleton/messy.py
# -> Found 4 errors (4 fixed, 0 remaining).

# FORMAT: canonicalize whitespace, the `prefix =` default, etc.
uv run ruff format src/skeleton/messy.py
# -> 1 file reformatted
```

The file is now:

```python
# src/skeleton/messy.py — after ruff --fix and ruff format
def greet(names: list[str], prefix: str = "Hello") -> list[str]:
    result = []
    for n in names:
        result.append(f"{prefix}, {n}")   # (SIM/UP would suggest this; shown for effect)
    return result
```

And the CI-shaped invocation — check without fixing, and check formatting without rewriting:

```bash
uv run ruff check .              # lint the whole tree; nonzero exit if anything found
uv run ruff format --check .     # FAIL if any file isn't already formatted (no writes)
# -> 3 files already formatted
```

**Why this works, line by line.**

- `ruff check` reports `F401` (unused import), `UP035`/`UP006` (use built-in `list` instead of `typing.List` — the modern PEP 585 generics). The rule *families* come from your `select = ["E","F","I","B","UP","SIM","ASYNC"]` in `pyproject.toml`: `E`=pycodestyle style, `F`=pyflakes bugs, `I`=isort imports, `B`=flake8-bugbear likely-bugs, `UP`=pyupgrade modernization, `SIM`=flake8-simplify, `ASYNC`=async-specific footguns (directly relevant to Part 2's agent loop). Each family is a former standalone tool, now a prefix.
- `[*]` marks a rule as **auto-fixable**. `--fix` applies exactly those, and *only* the ones Ruff deems safe (it distinguishes safe from unsafe fixes; unsafe ones need `--unsafe-fixes`). Removing unused imports and rewriting `List`→`list` are safe; deleting a variable that *might* have a side effect is not.
- `ruff format` handles the parts a linter shouldn't auto-rewrite as "errors" — it added the space normalization and would normalize quotes/trailing commas. Formatter and linter are separate subcommands because they have different contracts: format *always rewrites to canonical*, lint *reports and optionally fixes*.
- `ruff format --check .` is the **CI form**: it makes no changes and exits nonzero if any file *would* change. This is how you enforce "all code is formatted" without CI editing your code. `ruff check .` similarly exits nonzero on any lint finding. Both are what §1.7's pre-commit and §1.8's CI run.

### Under the hood — one AST, one pass, in Rust

The plan's specific claim: *Ruff's single-pass Rust AST is why it's ~100× faster than the flake8/black/isort stack it replaces.* Here's the mechanism.

The old stack was slow for **structural** reasons, not just because Python is slower than Rust:

```
OLD STACK (flake8 + black + isort + pyupgrade), each a separate process:
  flake8:    read file → tokenize → parse to AST → run pyflakes → run pycodestyle → ...
  isort:     read file → tokenize → parse → reorder imports → write
  black:     read file → tokenize → parse to ITS OWN AST → reformat → write
  pyupgrade: read file → tokenize → parse → rewrite → write
  → the SAME file is read and parsed 4+ times, in 4+ processes, in Python.

RUFF, one process:
  read file ONCE → parse to ONE AST (Rust) → run ALL 900+ rule checks over
  that one tree in a single traversal → collect diagnostics → (optionally) fix → format
```

Three compounding wins:

1. **Parse once, not N times.** Parsing (tokenize → build AST) is the expensive part, and the old stack paid it separately in every tool. Ruff builds the AST **once** and every rule inspects the same in-memory tree. This is the "single-pass AST" the plan names.
2. **Rust, not Python.** The parser and the tree-walk are compiled native code with no interpreter overhead and no per-node `PyObject` allocation (recall Day 6: a Python object carries a type pointer + refcount and is ~28 bytes for an int). Walking a million AST nodes in Rust is cache-friendly and allocation-light; in Python it's a million object dereferences.
3. **Parallelism + caching.** Ruff processes files across all cores and caches results per-file (keyed on content), so re-running on an unchanged tree is nearly free. The old tools were mostly single-threaded per invocation.

The result: linting the entire CPython codebase "from scratch" in a fraction of a second (Ruff's front-page benchmark), where the flake8/pylint stack takes minutes. **Deliberate stop:** I'm not opening Ruff's rule-implementation API or its red-knot type-aware analysis; you use Ruff as a configured black box (that's the [WORKING] way to treat *its internals*), while understanding *why* the architecture is fast (the [CORE] concept the plan asked for). Primary source: `docs.astral.sh/ruff` (overview + FAQ "how does Ruff compare to flake8/black/isort"), and Charlie Marsh's "Python tooling could be much, much faster."

**Honesty caveat.** Ruff's formatter is "near-drop-in" for `black`, not byte-identical in every edge case, and its default rule set is **not** all 900 rules — you opt in via `select`. Enabling everything (`select = ["ALL"]`) will flood you with conflicting and pedantic rules; the curated `select` in the skeleton is a sane starting set. Verify the current default rules at `docs.astral.sh/ruff/default-rules` — they evolve.

---

## 1.5 `mypy --strict` — static typing as validated boundaries, not ceremony

**Depth: [CORE]**

### Intuition

Python is dynamically typed: a variable can hold anything, and a wrong type is only discovered when the code *runs* into it — often in production, three call-frames deep, at 2 a.m. A **type hint** (`def f(x: int) -> str:`) is a *claim* about what flows through a boundary. But a claim is worthless until something **checks** it. `mypy` is that checker: it reads your type hints and, *without running the code*, proves that the claims are consistent — that you never pass a `str` where an `int` is required, never return `None` from a function annotated `-> str`, never call `.upper()` on something that might be `None`.

The framing that matters — the plan's phrase, "**typed code as validated boundaries, not ceremony**": don't think of types as decoration you add to please a linter. Think of every function signature as a **contract at a boundary**, and `mypy` as the compiler-like tool that *validates the contract holds across the whole program before you ship*. A boundary you typed and checked is a class of bug that *cannot* reach runtime. A boundary you left `Any` is a runtime crash you chose to defer. This is the exact same idea as a lockfile — pin the thing now, in a checkable form, or debug it later in production.

`--strict` turns on the full set of checks (no implicit `Any`, no untyped function bodies silently skipped, no untyped decorators, warn on unreachable code, etc.). Without `--strict`, mypy is lenient enough to let large holes through; **for the permanent skeleton, strict is the point** — you want the boundary fully sealed from day one, because retrofitting strictness onto an untyped codebase later is agony (§1.11's monorepo case study).

### Analogy — the customs checkpoint

A function boundary is a border crossing. The type signature is the declared manifest: "this truck carries `list[str]`, returns `int`." Dynamic typing with no checker is a border with a sign but *no guards* — anyone can drive through with anything, and you only find the contraband when it explodes downstream. `mypy --strict` is the checkpoint with guards who read every manifest and inspect every truck *at the border*, refusing anything whose cargo doesn't match its declaration — **before** it gets into the country (production).

**Where the analogy breaks (two ways):**
1. **The guards never see the actual trucks — only the paperwork.** `mypy` is *static*: it checks that the *annotations* are mutually consistent, not that runtime values obey them. If you *lie* in an annotation (`x: int` but you actually pass a `str` via an untyped external call, or you use `cast()` or `# type: ignore`), mypy believes the paperwork. Types are checked *assuming the annotations are honest*; they don't validate data crossing an *untyped* boundary at runtime. That runtime validation is a *different* tool — **Pydantic** — which is exactly why Part 2 leans on Pydantic for tool I/O: mypy guards code-to-code boundaries, Pydantic guards data-from-the-outside-world (JSON, an LLM's tool call) boundaries. They're complementary halves of "validated boundaries."
2. **Passing customs isn't proof of safety, only of *declared* consistency.** A green mypy run means "no type contradiction I can prove," not "no bugs." Logic errors, off-by-ones, and wrong-but-well-typed values sail straight through. mypy shrinks one *class* of bug (type mismatches) to zero; it says nothing about the others. Claiming "it type-checks, so it's correct" is the beginner error.

### Runnable example — mypy catching a real bug before runtime

A plausible bug: a function that can return `None`, used as if it always returns a value.

```python
# src/skeleton/lookup.py
def find_user(user_id: int, users: dict[int, str]) -> str | None:
    return users.get(user_id)          # dict.get returns None if the key is absent


def greeting(user_id: int, users: dict[int, str]) -> str:
    name = find_user(user_id, users)
    return "Hello, " + name.upper()    # BUG: name may be None -> AttributeError at runtime
```

```bash
uv run mypy src/skeleton/lookup.py
# -> src/skeleton/lookup.py:8: error: Item "None" of "str | None" has no
# ->     attribute "upper"  [union-attr]
# -> Found 1 error in 1 file (checked 1 source file)
```

Fix it by *handling* the boundary the type made explicit:

```python
# src/skeleton/lookup.py — fixed
def greeting(user_id: int, users: dict[int, str]) -> str:
    name = find_user(user_id, users)
    if name is None:                   # narrow str | None  ->  str
        return "Hello, stranger"
    return f"Hello, {name.upper()}"    # mypy now KNOWS name is str here
```

```bash
uv run mypy src/skeleton/lookup.py
# -> Success: no issues found in 1 source file
```

And what `--strict` adds that lenient mypy would miss — an *untyped* function silently accepting anything:

```python
# src/skeleton/loose.py
def add(a, b):            # no annotations
    return a + b
```

```bash
uv run mypy --strict src/skeleton/loose.py
# -> src/skeleton/loose.py:2: error: Function is missing a type annotation  [no-untyped-def]
# -> Found 1 error in 1 file (checked 1 source file)
# (Without --strict, mypy would say "Success" and this hole would ship.)
```

**Why this works, line by line.**

- `find_user` is annotated `-> str | None` because `dict.get` returns `None` on a miss — the type *tells the truth* about a real runtime possibility. This is the boundary being declared honestly.
- In the buggy `greeting`, `name` has type `str | None`, and `name.upper()` is only valid on `str`. mypy proves — statically, without running — that on the `None` branch there is no `.upper()`, and emits `union-attr`. **This is a `NoneType has no attribute 'upper'` production crash caught at author-time.** The single most common Python runtime error class (`AttributeError` / `TypeError` on `None`) is exactly what optional-type checking eliminates.
- The fix's `if name is None:` is **type narrowing**: after that guard, mypy *knows* `name` is `str` on the following line, so `.upper()` type-checks. You didn't just silence the checker — you handled the boundary it forced you to acknowledge. That's the difference between "types as validated boundaries" (fix the logic) and "types as ceremony" (`# type: ignore`).
- The `loose.py` example shows why `--strict` matters: `no-untyped-def` refuses functions with no annotations. Un-annotated functions are invisible holes where `Any` leaks in and infects everything it touches, silently disabling checking downstream. Strict mode seals them. Primary source: the mypy docs, "Strict mode and configuration" and the flag list under `mypy --help`.

**Honesty caveat.** `--strict` on a *fresh* skeleton is painless because there's nothing to retrofit. On an existing untyped codebase it can emit thousands of errors; the real-world path is incremental (per-module `[[tool.mypy.overrides]]`, `disallow_untyped_defs` module-by-module). Also note Astral is building **`ty`**, a Rust type checker meant to be to mypy what Ruff is to flake8 (much faster) — it is in preview as of this writing. Treat `ty` as **[AWARE]**: know it exists and may become the default checker; use `mypy` today because it's mature and stable. Verify current status at the ty repo.

---

## 1.6 `pytest` + `pytest-asyncio` — tests, including the async ones

**Depth: [CORE]**

### Intuition

A test is a small program that runs your real code with known inputs and *asserts* the output is what you expect — automated, repeatable proof that a behavior works, and (more importantly) a tripwire that fires the day someone breaks it. `pytest` is the de-facto Python test runner: you write plain functions named `test_*` containing plain `assert` statements, and `pytest` discovers, runs, and reports them. No boilerplate `class TestCase(unittest.TestCase)`, no `self.assertEqual` — just `assert x == y`, with pytest rewriting the assert so a failure shows you *both* values.

`pytest-asyncio` is the plug-in that lets you write `async def test_*` and `await` inside them — necessary because your FastAPI handlers are `async` (Part 1 of the plan; Day 21 opens the event loop), and to test them realistically you make **async HTTP calls into the app**. Without the plugin, pytest doesn't know how to run a coroutine test and silently skips or errors on it.

The skeleton's one passing async test does something specific and valuable: it spins the **real FastAPI app** in-process and makes a real HTTP request to `/health` through an async client, asserting the status and body. That single test exercises the whole request path (routing → handler → Pydantic response model → serialization) without a network socket or a running server — fast enough to run on every save.

### Analogy — the smoke detector

A test suite is a house full of smoke detectors. You don't install them because you expect a fire today; you install them so that *if* a fire ever starts — someone refactors, upgrades a dependency (§1.10), or the LLM in your agent loop starts returning a new shape — an alarm goes off *immediately and loudly*, at the exact spot, instead of you discovering the damage after the house has burned (a customer files a bug). The async test specifically is the detector in the room where your `async` handlers live.

**Where the analogy breaks:** a smoke detector only ever *reports*; it can't tell you *why* there's smoke, and a passing detector doesn't mean the house is well-built. A test's green checkmark proves the *asserted* behavior holds for the *inputs you chose* — nothing about inputs you didn't test, and nothing about design quality. Worse, a badly written test can pass while the code is broken (it asserted the wrong thing, or mocked away the very code under test). "All tests pass" means "no tripwire I set has been tripped," which is exactly as strong as the tripwires you bothered to set — a subtler guarantee than a smoke detector's binary "smoke / no smoke."

### Runnable example — the FastAPI hello-world endpoint and its passing async test

The application (part of the permanent skeleton):

```python
# src/skeleton/app.py
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI(title="skeleton", version="0.1.0")


class Health(BaseModel):        # a typed, validated response boundary (preview of Part 2)
    status: str
    service: str


@app.get("/health")
async def health() -> Health:   # async handler; return type is a Pydantic model
    return Health(status="ok", service="skeleton")
```

The async test — real HTTP into the real app, no live server:

```python
# tests/test_app.py
import httpx
from skeleton.app import app          # importable because the package is INSTALLED (§1.3)


async def test_health_returns_ok() -> None:
    transport = httpx.ASGITransport(app=app)          # drive the ASGI app in-process
    async with httpx.AsyncClient(transport=transport, base_url="http://test") as client:
        response = await client.get("/health")        # a real awaited HTTP request
    assert response.status_code == 200
    assert response.json() == {"status": "ok", "service": "skeleton"}
```

Run it:

```bash
uv run pytest
# -> ======================= test session starts =======================
# -> platform linux -- Python 3.12.7, pytest-8.4.1, pluggy-1.6.0
# -> rootdir: /home/you/skeleton
# -> configfile: pyproject.toml
# -> plugins: asyncio-1.2.0, anyio-4.11.0
# -> asyncio: mode=auto
# -> collected 1 item
# ->
# -> tests/test_app.py .                                        [100%]
# ->
# -> ======================== 1 passed in 0.14s ========================
```

**Why this works, line by line.**

- `async def test_health_returns_ok()` is picked up as an async test because `asyncio_mode = "auto"` is set in `pyproject.toml` (§1.3). In `auto` mode, `pytest-asyncio` treats every `async def test_*` as a coroutine to run on an event loop — **no `@pytest.mark.asyncio` decorator needed**. (In `strict` mode you'd decorate each one; `auto` is less boilerplate for an all-async codebase. Verify the mode names against the pytest-asyncio docs — they've changed across 0.x→1.x.)
- `from skeleton.app import app` works *only because the package is installed* into `.venv` (the `src/` layout, §1.3). If this import fails with `ModuleNotFoundError`, you skipped `uv sync`/`uv run`, or your `[tool.hatch...] packages` is wrong — a good early test that the skeleton's boundary is correct.
- `httpx.ASGITransport(app=app)` is the key trick: it lets `httpx` speak to the FastAPI app **directly through the ASGI interface**, in the same process, with no TCP socket, no `uvicorn`, no port. The request goes through FastAPI's real routing and serialization — a genuine integration test of the handler — but runs in 140ms. (FastAPI's own docs recommend this async-client pattern for testing async endpoints; the older `TestClient` is sync and wraps this.)
- `await client.get("/health")` is a real awaited request; `pytest-asyncio` provides the event loop it runs on. `response.json()` deserializes the body FastAPI produced by serializing the `Health` Pydantic model — so the assert checks the *whole* path including the response-model boundary.
- The report line `asyncio: mode=auto` and `plugins: asyncio-1.2.0` confirm the plugin loaded and the mode is active — if you ever see an async test reported as *passed instantly with a warning about coroutine never awaited*, the plugin isn't engaged and your test asserted nothing. Always confirm the plugin line.

**Deliberate stop.** I'm not covering fixtures (`@pytest.fixture`), parametrization (`@pytest.mark.parametrize`), coverage (`pytest-cov`), mocking, or property-based testing (`hypothesis`) — those are their own days and you'll meet them when the code needs them (Phase 3+ and Day 87). The skeleton needs exactly *one* honest async integration test to prove the toolchain and the request path work end to end. Primary sources: `docs.pytest.org`, `pytest-asyncio` docs, FastAPI "Testing" and "Async Tests" pages, `python-httpx.org` `ASGITransport`.

---

## 1.7 Pre-commit hooks — the gate that runs *before* the commit exists

**Depth: [WORKING]** — you must configure and reason about it, not reimplement git hooks.

### Intuition

You've got four checks: `ruff format`, `ruff check`, `mypy`, `pytest`. Running them by hand before every commit is a discipline that *will* lapse. A **git hook** is a script git runs automatically at a lifecycle moment; the `pre-commit` hook runs *before a commit is finalized* and can **abort it** if a check fails. The `pre-commit` framework (a Python tool, `pre-commit`) manages these hooks declaratively from a `.pre-commit-config.yaml`: it installs each tool in an isolated environment, runs them only on changed files, and blocks the commit on failure.

The value: **a broken-formatting or type-error commit never enters history in the first place.** It shifts every check from "hopefully someone runs it" to "impossible to skip by accident." It's the same checks as CI (§1.8), but at the earliest, cheapest moment — on your machine, on the files you touched, in the seconds before the commit. CI is the backstop; pre-commit is the front line.

### Analogy — spellcheck that won't let you hit "send"

Pre-commit is an email client that refuses to send until spellcheck passes: you *can't* fire off the typo-ridden draft, because the "send" button is gated on the check. CI is the recipient's assistant who also reads it and bounces it back if something slipped through — a second line of defense, slower and more embarrassing (public failure) than the client catching it locally.

**Where the analogy breaks:** you can *bypass* pre-commit trivially with `git commit --no-verify`, and hooks only run on machines where someone ran `pre-commit install`. So pre-commit is a **convenience and a fast local signal, not an enforcement boundary** — it can't be trusted to *guarantee* anything, because it lives on developers' machines and is skippable. The actual enforcement (the thing that *cannot* be bypassed) is CI running on the server against every PR (§1.8). Treating pre-commit as the guarantee is a classic mistake; it's the helpful nag, CI is the wall.

### Runnable example — the skeleton's `.pre-commit-config.yaml` and a blocked commit

```yaml
# .pre-commit-config.yaml — verify hook rev tags against each repo's releases.
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.14.0
    hooks:
      - id: ruff-check          # lint, with autofix
        args: [--fix]
      - id: ruff-format         # format

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.18.2
    hooks:
      - id: mypy
        additional_dependencies: [pydantic, fastapi]   # so mypy sees these types

  - repo: local
    hooks:
      - id: pytest
        name: pytest
        entry: uv run pytest
        language: system
        pass_filenames: false
        always_run: true
```

```bash
# One-time: install the git hook into .git/hooks (per clone).
uv run pre-commit install
# -> pre-commit installed at .git/hooks/pre-commit

# Now try to commit a file with an unused import and a type error:
git add src/skeleton/messy.py
git commit -m "add messy module"
# -> ruff-check..............................................Failed
# -> - hook id: ruff-check
# -> - files were modified by this hook
# ->   Fixed src/skeleton/messy.py (removed unused import `os`)
# -> ruff-format.............................................Passed
# -> mypy....................................................Failed
# -> - hook id: mypy
# ->   src/skeleton/messy.py:8: error: ... [union-attr]
# -> pytest..................................................Passed
# ->
# -> (commit ABORTED — fix mypy, `git add` the ruff autofix, commit again)
```

**Why this works, line by line.**

- Each `repo:` block pins a **`rev`** — the exact tagged version of the hook tool. This is a lockfile-by-another-name: an un-pinned hook rev means your checks silently change when the upstream releases, re-introducing the "works on my machine" problem *for the gate itself*. Pin it; bump it deliberately (Renovate can, §1.10).
- `ruff-check` with `args: [--fix]` auto-repairs and, if it *modified* files, **fails the commit** so you notice and re-`git add`. `ruff-format` reformats. `mypy` runs with `additional_dependencies` because the hook runs in its *own* isolated env that doesn't have your project's deps — you must tell it which type stubs/packages to install or it can't resolve `fastapi`/`pydantic` types.
- The `local` + `language: system` pytest hook runs `uv run pytest` in *your* environment (`pass_filenames: false, always_run: true` because tests aren't a per-file operation). Running the full suite on every commit is fine for a small skeleton; on a large repo you'd move pytest to CI-only and keep pre-commit to the fast checks. That trade-off is the [WORKING] judgment call.
- The commit is **aborted** because mypy failed. Nothing entered history. You fix, re-add, re-commit — and the bad state never existed in a commit anyone else could pull.

**In production (condensed, [WORKING] tier).** Best practice: keep pre-commit hooks *fast* (sub-second where possible) so they don't tempt people into `--no-verify`; put slow/expensive checks (full test suite, integration tests) in CI only. Top failure mode: hooks that pass locally but fail in CI because the environments differ — mitigate by having CI *also* run `pre-commit run --all-files` (§1.8) so the same hook definitions execute on the server, making pre-commit and CI agree by construction. Common beginner mistake: not pinning `rev`, then blaming "flaky CI" when an upstream hook changed. Primary source: `pre-commit.com`.

---

## 1.8 CI stub — GitHub Actions, the enforcement wall

**Depth: [WORKING]**

### Intuition

**Continuous Integration (CI)** is automation that, on every push and pull request, checks out your code on a clean server and runs your gates — lint, format, types, tests — reporting pass/fail on the PR. Its defining property, the one pre-commit lacks: **it runs on infrastructure the developer doesn't control and cannot skip.** A PR that fails CI cannot be merged (with branch protection on). This is the actual enforcement boundary — the wall that makes "all code on `main` is formatted, typed, and tested" a *fact* rather than a hope.

The clean-server part matters: CI installs *from the lockfile* on a fresh runner, so it catches exactly the "works on my machine, my machine has a stray global package" bugs that §1.1 is about. If it passes in CI, it passes on any lockfile-faithful machine — including production.

### Analogy — the airport security checkpoint (vs the pat-down at home)

Pre-commit is you checking your own bag before leaving the house — helpful, skippable, self-administered. CI is the **airport checkpoint**: staffed by someone else, on their equipment, and *you do not board without clearing it*. It doesn't matter that you checked at home; everyone goes through the same gate, the same way, every time. Branch protection is the rule that there's no door to the plane except through the checkpoint.

**Where the analogy breaks:** an airport checkpoint is a one-time pass/fail at a single moment; CI re-runs the *entire* battery on *every* push, and its result can go stale — a PR that passed last week may fail today because `main` moved under it or an unpinned dependency shifted (§1.10, §3). CI is a *continuously re-evaluated* gate, not a single stamp; "it passed once" is not "it passes now," which is why merge queues re-run CI against the latest `main` before merging.

### Runnable example — the skeleton's `.github/workflows/ci.yml`

```yaml
# .github/workflows/ci.yml — runs on every push and PR. Verify action versions.
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v5
        with:
          enable-cache: true              # cache the uv download cache across runs

      - name: Set up Python and sync from the lockfile
        run: uv sync --frozen --all-groups
        # --frozen: FAIL if uv.lock is out of date with pyproject.toml
        #           (i.e. someone changed deps but didn't commit the lock)

      - name: Ruff format (check only)
        run: uv run ruff format --check .

      - name: Ruff lint
        run: uv run ruff check .

      - name: Mypy (strict)
        run: uv run mypy

      - name: Pytest
        run: uv run pytest
```

A passing run's summary, as GitHub shows it:

```
CI / quality (pull_request)                                     ✓ Success in 38s
  ✓ Install uv
  ✓ Set up Python and sync from the lockfile
  ✓ Ruff format (check only)          3 files already formatted
  ✓ Ruff lint                         All checks passed!
  ✓ Mypy (strict)                     Success: no issues found in 3 source files
  ✓ Pytest                            1 passed in 0.14s
```

**Why this works, line by line.**

- `on: [push to main, pull_request]` fires the workflow on every PR and every push to `main` — so no change reaches `main` un-checked, and (with branch protection requiring this job) no PR merges red.
- `astral-sh/setup-uv@v5` installs `uv` on the runner and caches its download cache, so dependency installs are fast on repeat runs. (Verify the action's major version — Astral bumps it. `enable-cache` keys on the lockfile.)
- **`uv sync --frozen --all-groups`** is the load-bearing line. `--frozen` makes CI *refuse to proceed if `uv.lock` doesn't match `pyproject.toml`* — this catches the "I added a dep but forgot to commit the updated lock" mistake, which would otherwise let CI silently resolve something *different* from what teammates have. It installs the **exact** locked graph (reproducibility, §1.1). `--all-groups` includes the `dev` tools CI needs.
- The four `uv run …` steps are the *same commands* you run locally and in pre-commit — `ruff format --check` (no rewriting, fail if unformatted), `ruff check`, `mypy` (strict, config from `pyproject.toml`), `pytest`. Each nonzero exit fails the job. **Sameness is the design goal:** identical commands locally, in pre-commit, and in CI means "green locally" reliably predicts "green in CI."
- The whole thing runs in ~38s on a fresh Ubuntu runner from the lockfile — proof the project is reproducible on a machine that has never seen your laptop.

**In production (condensed, [WORKING] tier).** Real CI adds: a **matrix** across Python versions (`3.12`, `3.13`) that must match `requires-python`; caching keyed on `uv.lock`; a separate job for slow integration tests; required-status-check **branch protection** so red PRs can't merge; and often `pre-commit run --all-files` as a CI step so the hooks and CI can't drift. Top failure mode: CI that installs *without* `--frozen` (or with `pip install .` and no lock), which passes while resolving different bytes than developers have — defeating the entire point. Cost note: CI minutes are metered; keep the fast path fast (the whole reason Ruff and uv's speed matters — a 38s pipeline invites frequent PRs, a 15-minute one invites giant risky ones). Primary sources: GitHub Actions docs; `docs.astral.sh/uv/guides/integration/github`.

---

## 1.9 System design ① — Monorepo vs polyrepo for a 5-service platform

**The problem.** You're building a platform with five services: `api-gateway`, `auth`, `billing`, `agent-runtime`, and a shared `common` library (types, clients, utils). Decide whether to put them in **one repository (monorepo)** or **five (polyrepo)**, and design the CI implications either way. Requirements: (a) a change to `common` must not silently break a consumer; (b) CI must stay fast as the platform grows; (c) a new engineer should be productive quickly; (d) you can release services independently.

**Requirements → the key decision.** The pivotal tension is requirement (a) — cross-service consistency of the shared `common` library — versus (d) — independent release. Everything else follows from how you resolve it.

**Option A — Polyrepo (five repos).**

```
auth-repo/       api-gateway-repo/   billing-repo/   agent-runtime-repo/   common-repo/
  pyproject.toml   pyproject.toml      ...             ...                   (published to
  depends on         depends on                                              an index)
  common==2.1.0      common==1.9.0   ← DIFFERENT versions of common in flight
```

- `common` is *published* (to a private package index) and each service depends on a *pinned version* in its own lockfile. Independent release is trivial: each repo deploys on its own schedule.
- **The cost:** version skew. `auth` is on `common==2.1.0`, `api-gateway` still on `1.9.0`. A breaking change to `common` is discovered *service by service*, weeks apart, whenever each team gets around to upgrading — exactly the §1.10 upgrade-lag problem, multiplied by five. Requirement (a) is *hard*: you cannot atomically change `common` and all its consumers in one reviewable change.
- CI is simple per-repo (each repo runs the §1.8 pipeline on itself) but there's **no CI that tests `common` against all its consumers at once**. You'd bolt that on with cross-repo integration tests, which are awkward.

**Option B — Monorepo (one repo, five packages).**

```
platform/                        one uv workspace, one uv.lock for everything
├── pyproject.toml               (workspace root)
├── uv.lock                      ← ONE locked graph for all services
├── packages/common/
├── services/auth/               depends on common via workspace (path), not a version
├── services/api-gateway/
├── services/billing/
└── services/agent-runtime/
```

- `uv` **workspaces** (verify current: `docs.astral.sh/uv/concepts/projects/workspaces`) let multiple packages share one lockfile and depend on each other by path. A change to `common` and all five consumers is **one atomic PR**, tested together in **one CI run**. Requirement (a) becomes trivial: you literally cannot merge a `common` change that breaks a consumer, because their tests run in the same pipeline.
- **The cost:** independent release needs discipline. Everything shares one lockfile, so a dependency bump affects all services at once (a feature for consistency, a constraint for autonomy). And naive CI re-runs *everything* on every change — which kills requirement (b) as the repo grows.
- **CI implication (the real design work):** you need **change detection / affected-targets** — only test the packages affected by a diff. GitHub Actions `paths:` filters handle the simple case ("only run `billing` CI if `services/billing/**` or `packages/common/**` changed"); at scale you reach for a build system that models the dependency graph (Bazel, Pants, Nx) so a `common` change tests exactly its transitive consumers and nothing else. This is precisely what Google/Meta do (§1.11).

**The trade-off made, and why.** For a *5-service platform with a shared library and one team*, choose the **monorepo**. The dominant risk is `common` skew (a) and onboarding friction (c) — a monorepo makes cross-cutting changes atomic and gives a new engineer *one* clone, *one* `uv sync`, *one* toolchain config to learn. You pay for it with CI change-detection complexity and release coordination, both of which are *solvable engineering* rather than *recurring human coordination cost*. Polyrepo wins when teams are large and independent enough that release autonomy dominates and the shared surface is small — i.e. when the skew cost is low and the coordination cost of a monorepo is high. **Failure mode of getting it wrong:** a monorepo without change-detection CI becomes a 40-minute pipeline everyone hates and routes around; a polyrepo without a disciplined upgrade policy (§1.10) becomes five services on five incompatible versions of `common`, and an incident where a "fixed" bug is still live in three of them.

**Cross-reference.** The upgrade-lag failure this section keeps pointing at is exactly §1.10's subject. The "test `common` against all consumers atomically" property is what makes the §1.11 monorepo CI-gate case study possible.

---

## 1.10 System design ② — A dependency-upgrade policy: lockfiles, automated PRs, security-patch SLAs

**The problem.** Your platform depends, transitively, on ~300 packages. Some will ship security fixes; some will ship breaking changes; all will drift. Design a **policy** (not a one-off action) that keeps dependencies current and *auditable* without either (a) breaking prod on every bump or (b) sitting on a known-vulnerable version for months. Requirements: patch critical CVEs within a bounded time; never merge an upgrade that fails the gates; keep the lockfile the single source of truth; make upgrades *small and frequent* rather than *huge and rare*.

**Requirements → key decisions.**

**Decision 1 — the lockfile is the control point.** Because `uv.lock` pins the *entire* transitive graph with hashes (§1.2), an upgrade is *exactly* "a diff to `uv.lock`" (and sometimes to a floor in `pyproject.toml`). That makes upgrades **reviewable**: a PR that changes `uv.lock` shows precisely which package moved from which version to which, and CI (§1.8) runs the full gate against the new graph *before* it can merge. The lockfile turns "dependencies drifted mysteriously" into "here is a diff someone approved."

**Decision 2 — automate the PRs, don't chase them manually.** Configure **Dependabot** or **Renovate** to open PRs automatically when a dependency has a newer version. Each bot PR: bumps the version, updates the lock, and triggers CI. Humans review a small, green, single-purpose PR instead of periodically doing a terrifying bulk upgrade.

```yaml
# .github/dependabot.yml  (Dependabot; Renovate is the more configurable alternative)
version: 2
updates:
  - package-ecosystem: "uv"          # verify: uv support/ecosystem name is evolving
    directory: "/"
    schedule:
      interval: "weekly"
    groups:
      dev-dependencies:              # batch low-risk dev bumps into one PR
        dependency-type: "development"
    open-pull-requests-limit: 10
```

- **Grouping** matters: batch low-risk dev-tool bumps (ruff, mypy) into one PR; keep runtime and major-version bumps as *separate* PRs so a risky change is isolated and revertable. Renovate's grouping/scheduling is richer (verify against its docs; `uv` documents both — `docs.astral.sh/uv/guides/integration/{dependabot,renovate}`).
- **Auto-merge policy:** patch-level bumps of *dev* dependencies that pass CI can auto-merge; anything touching runtime deps or a major version requires human review. The gate (§1.8) is what makes auto-merge safe at all — you're trusting the tests, not the bot.

**Decision 3 — security-patch SLAs, tied to severity.** Wire a vulnerability scanner (GitHub's **Dependabot security alerts** / `pip-audit` / `osv-scanner`) that watches your lockfile against advisory databases (OSV, GitHub Advisory) and *opens a PR the moment a dep in your graph has a known CVE*. Attach a policy with teeth:

| Severity (CVSS) | Patch SLA | Mechanism |
|---|---|---|
| Critical (9.0–10) | **24–48h** | security PR auto-prioritized; page on-call if no fixed version exists |
| High (7.0–8.9) | **1 week** | security PR, normal review, expedited |
| Medium (4.0–6.9) | next scheduled upgrade cycle | batched |
| Low | best-effort | batched |

**The trade-off made.** Frequent small automated upgrades trade a steady trickle of tiny review PRs (a real, recurring cost) against the alternative: rare, giant, high-risk upgrades where 40 packages jump versions at once and *something* breaks with no way to bisect which. Small-and-frequent is almost always right — it keeps each change bisectable and keeps you close to current, so a security patch is a one-version hop, not a six-month migration. **The one exception:** pin and *hold* during a release freeze; the policy should have a documented "freeze window" switch. **Failure mode of no policy:** you discover you're 14 months and a major version behind on a package with a critical CVE, the fixed version has breaking changes, and now the "security patch" is a multi-week migration under incident pressure — the xz/left-pad lesson (§1.11) is that the *time to patch* is itself a security property, and it's bounded by how current you keep your graph.

**Honesty note.** The exact CVSS→SLA numbers above are a *reasonable industry-shaped* policy, not a standard you can cite — real orgs set their own thresholds (verify against your org's security policy). What *is* firm: (1) the lockfile is the reviewable control point, (2) automate the PRs, (3) bound patch time by severity. Cross-reference: this policy is what prevents the Part 3 shared failure mode — an unpinned transitive dep breaking a deployed agent.

---

## 1.11 Case studies

### ① The left-pad (2016) and xz-utils (CVE-2024-3094) supply-chain incidents

**left-pad (March 2016) — how 11 lines broke the internet.** A developer, in a naming dispute with npm, *unpublished* his packages — including `left-pad`, an eleven-line function that pads a string. Thousands of packages (including Babel, React tooling) depended on it *transitively*; overnight their installs failed with `npm ERR! 404 'left-pad' not found`, and CI and deploys broke across the industry until npm took the unprecedented step of *un-unpublishing* it. **What it proved:** your build depends on code you never chose, from people you've never heard of, and its *availability* is part of your build's reliability. A lockfile with a local/cached copy or a vendored mirror means a disappeared upstream can't break your existing builds; an un-pinned, un-cached `install-latest` build is hostage to every transitive author's whims. Primary source: contemporaneous npm blog post and Chris Williams' *The Register* coverage, March 2016.

**xz-utils (CVE-2024-3094, March 2024) — the postmortem, and the scarier one.** Over ~2–3 years, an attacker using the persona "Jia Tan" gained maintainer trust on `xz`/`liblzma` (a compression library in essentially every Linux distro) through patient, legitimate-looking contributions, then planted an **obfuscated backdoor** in the *release tarballs* (not the git source) of versions 5.6.0/5.6.1. The backdoor hooked into `sshd` (via a systemd→liblzma linkage) to allow remote code execution against a huge fraction of internet-facing Linux servers. It was caught almost by accident: Andres Freund, a Postgres developer, noticed `sshd` was ~500ms slower and used ~0.5s more CPU, investigated, and unraveled a nation-state-grade supply-chain attack. **The engineering lessons, all directly on today's topic:**

1. **Pinned, hash-verified dependencies are a security control, not pedantry.** A lockfile that pins *and hashes* exact versions (§1.2) means you install the *audited* bytes, and a swapped or newly-poisoned release is a *detectable diff* requiring human approval — not a silent auto-upgrade. The attack shipped in a *new release*; environments that auto-pulled "latest" were exposed the instant it landed, while pinned environments had a change to review.
2. **The build/release artifact can differ from the source you read.** xz's backdoor was in the tarball, not the git tree — so "I read the source on GitHub" wasn't enough. This is why hash-verified artifacts and reproducible builds matter, and why `uv.lock`'s recording of artifact hashes and *source index* is load-bearing.
3. **Transitive trust is the attack surface.** You didn't choose `liblzma`; `sshd` did, transitively. Your *entire* dependency graph is your trust boundary, which is exactly why the lockfile pins the *whole graph* and why the §1.10 upgrade policy must be able to *audit* what's in it (`uv tree`, SBOM export).
4. **Speed of response depends on being current and auditable.** Distros could respond fast *because* they knew exactly which versions were affected and could pin back. An org that couldn't enumerate its dependency versions couldn't even answer "are we exposed?"

Primary sources: the CISA/NVD entry for CVE-2024-3094; Andres Freund's original `oss-security` mailing-list disclosure (2024-03-29); the extensive `research!rsc` (Russ Cox) timeline write-up. **Verify current** — the forensic details were still being reconstructed after disclosure.

*Engineering lesson tying both together:* left-pad shows dependencies are a *reliability* liability (they can vanish); xz shows they're a *security* liability (they can be poisoned). The lockfile + pinned, hashed, auditable graph + a bounded upgrade/audit policy (§1.10) is the single control that addresses both. That's why the plan says lockfiles are "a security control, not pedantry."

### ② How large Python monorepos enforce format/type gates in CI so review is about design, not style

**What it is (real, documented practice).** Google and Meta run enormous monorepos (Google's is one of the largest codebases on earth) with a defining property: **automated gates enforce style and types so human code review can be spent entirely on design and correctness.** Google's public engineering writing (the "Software Engineering at Google" book; the Google Python Style Guide; their internal `Critique`/`Tricorder` analysis tooling described in published papers) documents the principle: a *presubmit* (their pre-merge CI) runs formatters and static analysis, and a change **cannot merge** until it's clean. Formatting is not a review comment because a *machine* already guaranteed it; type/static-analysis findings surface as automated review comments before a human looks. Meta's equivalent stack (the Pyre type checker, the `arc`/Phabricator/Sapling presubmit flow) enforces types across their Python at scale for the same reason.

**The mechanism, mapped to today's skeleton.** What Google/Meta do at planetary scale is *exactly* your §1.7 + §1.8 gates, scaled up: format + lint + type-check + test run automatically pre-merge; nothing un-clean reaches the trunk; humans review *design*. The monorepo (§1.9) is what makes trunk-wide enforcement possible — one config, one gate, applied uniformly, with change-detection CI (Bazel-style affected-targets) so a 100-million-line repo doesn't re-test itself on every diff.

**Engineering lessons.**
1. **Style should never be a human review comment.** If a person is typing "please run the formatter" on a PR, you've mis-invested a human on a machine's job. `ruff format --check` in CI (§1.8) makes that comment impossible and structurally reserves review for what humans are good at.
2. **Enforce at the trunk gate, because local hooks are skippable.** Google's presubmit, like your CI, is the un-bypassable wall (§1.8) — pre-commit is the courtesy, CI is the guarantee.
3. **Types-at-scale require the gate from day one.** Meta/Google enforce types because retrofitting them onto millions of untyped lines is prohibitive — which is exactly why the skeleton turns on `mypy --strict` on line one (§1.5), so the boundary is never allowed to rot in the first place.

Primary sources: *Software Engineering at Google* (Winters, Manshreck, Wright — the "Static Analysis" and "Code Review" chapters); the Google Python Style Guide; Meta Engineering's posts on Pyre and Sapling. **Verify current** — internal tooling names and specifics drift, but the *presubmit-enforces-style-and-types* principle is stable and public.

---

## 1.12 In production (Part 1)

**Best practices, beginner → senior.**

| Level | Habit |
|---|---|
| Beginner | Always commit `uv.lock`, never `.venv`. Run everything via `uv run <tool>`. Use the `src/` layout. Turn on `mypy --strict` from the first commit. |
| Intermediate | One `pyproject.toml` for all tool config. Separate runtime deps from the `dev` group. Wire pre-commit *and* the same checks in CI. Pin pre-commit hook `rev`s. `uv sync --frozen` in CI. |
| Senior | A written dependency-upgrade policy with CVE SLAs (§1.10). Automated Dependabot/Renovate PRs with grouping + selective auto-merge. Monorepo change-detection CI. SBOM export + vulnerability scanning of the lockfile. Treat "time-to-patch a CVE" as a monitored metric. |

**Monitoring / observability — for a toolchain, this is *supply-chain* observability.**
1. **Lockfile drift:** CI with `uv sync --frozen` fails if `uv.lock` and `pyproject.toml` disagree — alert on it, it means someone bypassed the flow.
2. **Vulnerability count in the graph:** Dependabot/`pip-audit`/`osv-scanner` against `uv.lock`; track open advisories by severity and *age* (age is the SLA-breach signal).
3. **Dependency freshness:** how many versions / how many days behind latest, per dep — the leading indicator of a painful future upgrade.
4. **CI pipeline time:** if it creeps past a few minutes, people batch changes and route around the gate — pipeline latency is a cultural risk, not just a cost.

**Failure modes and recovery.**

| Symptom | Likely cause | First move |
|---|---|---|
| "Works on my machine," fails in CI | env not built from lock; stray global package | `uv sync --frozen` locally; reproduce on a clean checkout |
| CI resolves different bytes than dev | CI installs without `--frozen`, or uses `pip install .` | switch CI to `uv sync --frozen`; commit the lock |
| `ModuleNotFoundError: skeleton` in tests | package not installed (flat-layout habit) / wrong `packages` | `uv sync`; verify `[tool.hatch...] packages`; confirm import resolves to `.venv` |
| pre-commit passes, CI fails | hook rev drift; different envs | run `pre-commit run --all-files` in CI too; pin `rev`s |
| Sudden mass mypy/ruff errors after a bump | tool released new default rules | pin tool versions; bump deliberately via a reviewed PR |
| A dep vanished / install 404s | unpinned/uncached transitive dep removed upstream (left-pad) | rely on lock + cache/mirror; restore from a pinned artifact |

**Scaling behavior.** Toolchain cost scales with *repo size and CI frequency*, not request rate. Ruff/uv's speed is what keeps the gate sub-minute as the repo grows; the moment checks are slow, they get skipped, and the guarantee evaporates. On a monorepo, change-detection CI (§1.9) is what keeps pipeline time flat as package count grows.

**Cost.** CI minutes are metered and dependency-review is human time — both are minimized by *fast tools* (frequent cheap runs) and *small automated upgrade PRs* (cheap reviews) rather than rare giant ones (expensive, risky reviews). The largest hidden cost is a *skipped* upgrade that becomes an emergency CVE migration (§1.10).

## 1.13 Failure modes and common misconceptions (Part 1)

| Misconception | Reality |
|---|---|
| "`requirements.txt` from `pip freeze` is a lockfile." | It's a flat, un-hashed, single-platform snapshot. A real lockfile pins the *whole graph*, with hashes, universally (`uv.lock`). |
| "Types are ceremony to please the linter." | Types are *validated boundaries*; `mypy --strict` turns a class of runtime crash into a compile-time error. Untyped = deferred crash. |
| "It type-checks, so it's correct." | mypy proves *no type contradiction*, not *no bug*. Logic errors sail through. |
| "mypy validates my data at runtime." | mypy is static; it trusts your annotations. Runtime data validation is Pydantic's job (Part 2). |
| "pre-commit guarantees clean code on `main`." | pre-commit is skippable (`--no-verify`) and machine-local. CI with branch protection is the guarantee. |
| "Ruff is just black + flake8 bundled." | It's a single-pass Rust AST reimplementation — same jobs, 10–100× faster, one config, and it *replaces* black/isort/flake8/pyupgrade/etc. |
| "Pinning versions is pedantic." | Pinning + hashing is a *security and reliability control* (left-pad, xz). Unpinned = hostage to every transitive author. |
| "I'll add types/tests later." | Retrofitting strict types and gates onto a grown untyped codebase is prohibitively expensive (Google/Meta lesson). Start clean. |
| "Commit `.venv` so it's reproducible." | Never. Commit `uv.lock`; rebuild `.venv` from it in seconds. `.venv` is platform-specific and huge. |
| "Faster tools are a nice-to-have." | Speed is *behavioral*: sub-second checks run everywhere; 20s checks get skipped, and a skipped gate is no gate. |

## 1.14 Interview & practice questions (Part 1)

1. What does a lockfile lock that `requirements.txt` from `pip freeze` doesn't? *(The full transitive graph, with hashes, universally across platforms — vs a flat, un-hashed, single-machine snapshot.)*
2. Why is `uv`'s resolution a *graph* problem and pip's classic install a *sequential* one? *(uv solves all version constraints simultaneously with backtracking (PubGrub-style); old pip installed greedily and discovered conflicts too late.)*
3. Why does the `src/` layout catch packaging bugs a flat layout hides? *(Source isn't importable by accident; tests run against the *installed* package, so missing-file/relative-import bugs surface pre-release.)*
4. Give the mechanism for Ruff being ~100× faster than the flake8/black/isort stack. *(Parse to one AST once, run all rules in a single Rust pass, parallel + cached — vs each old tool re-parsing in its own Python process.)*
5. `--strict` mypy flags a function with no annotations. Why is that hole dangerous? *(Untyped bodies leak `Any`, which silently disables checking downstream.)*
6. What's the difference between what mypy validates and what Pydantic validates? *(mypy: static code-to-code type consistency, trusting annotations. Pydantic: runtime validation of external data crossing a boundary.)*
7. pre-commit passed but CI failed on the same check. Two causes and the fix. *(Un-pinned hook rev drift; different environments. Pin revs; also run `pre-commit run --all-files` in CI.)*
8. Which single CI line prevents "CI resolved different bytes than the developer"? *(`uv sync --frozen` — fails if the lock is stale, installs the exact locked graph.)*
9. For a 5-service platform with a shared lib and one team, monorepo or polyrepo — and the main CI cost of your choice? *(Monorepo, for atomic cross-service changes; cost is change-detection CI so you don't re-test everything.)*
10. In one sentence each, the engineering lesson of left-pad and of xz. *(left-pad: deps are a reliability liability — they can vanish; xz: deps are a security liability — they can be poisoned; a pinned, hashed, auditable lockfile addresses both.)*

---

# PART 2 — AGENTIC AI

> **Reading note.** Part 1 built the toolchain and skeleton as a *backend* project. Here we put an **agent** behind the same skeleton and show that every gate you just wired matters *more* when a non-deterministic model is in the loop. We treat the backend (FastAPI, the endpoint, the request path) as a black box from Part 1 and cross-reference it rather than re-explaining. The agent loop itself is Day 24's subject; today we care only about how *the toolchain* meets *the agent*.

## 2.1 The same skeleton, now with an agent behind it

**Depth: [WORKING]**

### Intuition

An agent, stripped of mystique, is: **a `while` loop that sends messages to an LLM, the model *describes* a tool call, *your code* runs the tool, you append the result, and you loop** (the plan's ReAct shape, Day 24). Structurally it is a backend service — it has an HTTP surface (a FastAPI endpoint that accepts a user message), it calls out to dependencies (the model provider's SDK, and whatever the tools touch), and it must be tested, typed, and deployed. So the *same skeleton* from Part 1 hosts it: the FastAPI app gains an `/agent` endpoint instead of just `/health`, the dependencies gain the provider SDK (`anthropic`, `openai`, or `google-genai`), and the tests gain agent-loop tests.

The plan's thesis restated for today: an agent is a distributed backend system with **one** non-deterministic component (the model). Part 1's toolchain exists to make the *deterministic* parts — the loop, the tool implementations, the request handling, the dependency graph — *trustworthy and reproducible*, so that when something misbehaves you can be certain the culprit is the model's output, not a version skew or an untyped boundary or an un-run test. **The toolchain shrinks the search space of "what went wrong" down to the one component that's genuinely uncertain.** Without it, every agent bug is a suspect list of "model? or my code? or a dependency? or my environment?"; with it, three of those four are ruled out by construction.

### Analogy — the lab around the reactor

The model is a reactor: powerful, useful, and *inherently unpredictable* at the core. Everything around it — the containment vessel, the sensors, the interlocks, the tested control rods — is boring, deterministic, and rigorously validated *precisely so that the one unpredictable thing is safely contained*. Part 1's toolchain is that lab: the pinned dependency graph, the typed boundaries, the tests, the CI gate. You don't make the reactor predictable; you make *everything else* so trustworthy that the reactor's uncertainty is the only variable you have to manage.

**Where the analogy breaks:** a physical reactor's uncertainty is bounded by physics and stays roughly constant. A model's behavior can shift *discontinuously* when you change the model version, the prompt, or even a dependency that alters how tools serialize — and it can be *adversarially* driven (prompt injection). So the "containment" isn't just about a steady random core; it must also defend a boundary that an *attacker* may try to push malicious data through (the tool inputs), which is why §2.2's typed/validated boundary and §2.3's locked execution environment are load-bearing in a way a reactor's passive shielding isn't.

---

## 2.2 Typed Pydantic models as the validated boundary for tool I/O

**Depth: [CORE]**

### Intuition

A **tool** is a function the model can ask your code to run — `get_weather(city)`, `search_orders(customer_id, status)`, `transfer_funds(from, to, amount)`. The model doesn't run it; it *emits a JSON object* claiming to be the arguments, and *your code* runs the real function. That JSON comes from a **non-deterministic, sometimes-adversarially-steered source** — the model might hallucinate a field, omit a required one, pass a string where you need an int, or (under prompt injection) supply a hostile value. This is the *sharpest* untrusted boundary in the whole system.

§1.5 established the split: **mypy validates code-to-code boundaries statically; it cannot validate data arriving at runtime from the outside world.** An LLM's tool-call JSON *is* outside-world data. So the tool boundary needs the *other* half of "validated boundaries": **Pydantic**, which validates and coerces *runtime data* against a typed schema and *rejects* what doesn't conform. You define the tool's arguments as a Pydantic model; the model's JSON is parsed *through* it; malformed calls fail *at the boundary* with a structured error instead of crashing three lines into your tool or, worse, executing with garbage.

And there's a second gift: a Pydantic model can **emit its own JSON Schema**, which is exactly the format every provider's tool/function-calling API wants for the tool definition you send *to* the model. So one typed Python class is simultaneously (a) the schema you advertise to the model, (b) the runtime validator for what comes back, and (c) a mypy-checked type for your own code. One definition, three boundaries sealed. This is the preview of tool schemas the plan promises for Day 24+.

### Analogy — the API's typed order form

A tool schema is a strict order form at a warehouse counter. The customer (the model) fills it out; the form has typed fields ("quantity: integer 1–100", "sku: string matching this pattern"). A clerk (Pydantic) checks the *filled form* against the spec *before* anyone walks into the warehouse: wrong type, missing field, out-of-range — rejected at the counter with a specific complaint. Only a valid form reaches the floor, so the warehouse code never has to defend against a garbage order.

**Where the analogy breaks:** a human clerk applies judgment and can phone the customer to clarify. Pydantic applies *only* the schema you wrote — it validates *shape and constraints*, not *intent* or *safety semantics*. A perfectly-valid `transfer_funds(from="victim", to="attacker", amount=1000000)` passes validation cleanly; the *values* are well-typed and in-range, but the *action* is catastrophic. Schema validation stops malformed calls, **not** malicious-but-well-formed ones — those need authorization checks, allow-lists, and human-in-the-loop confirmation *on top of* validation (Days 79+ security). Confusing "it validated" with "it's safe" is the same beginner error as "it type-checks so it's correct" (§1.5), and it's more dangerous here because the input is adversarial.

### Runnable example — a typed tool boundary that rejects a hallucinated call

```python
# src/skeleton/tools.py — a validated tool boundary. Provider-agnostic.
# (deps already in the skeleton: pydantic. The LLM SDK is added in Day 24.)
from pydantic import BaseModel, Field, ValidationError


class WeatherArgs(BaseModel):
    """Arguments for the get_weather tool. This IS the tool's schema."""
    city: str = Field(min_length=1, description="City name, e.g. 'Hyderabad'")
    units: str = Field(default="celsius", pattern="^(celsius|fahrenheit)$")


def get_weather(args: WeatherArgs) -> dict[str, str | float]:
    # By the time we're here, args is VALIDATED: city non-empty, units in the enum.
    return {"city": args.city, "units": args.units, "temp": 34.0}


# 1. The schema you send TO the model (every provider wants JSON Schema):
print(WeatherArgs.model_json_schema())
# -> {'type': 'object',
# ->  'properties': {'city': {'type': 'string', 'minLength': 1, ...},
# ->                 'units': {'type': 'string', 'pattern': '^(celsius|fahrenheit)$',
# ->                           'default': 'celsius'}},
# ->  'required': ['city'], ...}

# 2. A GOOD tool call from the model — validates and runs:
good = '{"city": "Hyderabad", "units": "celsius"}'
print(get_weather(WeatherArgs.model_validate_json(good)))
# -> {'city': 'Hyderabad', 'units': 'celsius', 'temp': 34.0}

# 3. A HALLUCINATED / malformed call — the model invented a bad `units` and
#    dropped `city`. Validation REJECTS it at the boundary:
bad = '{"units": "kelvin"}'
try:
    get_weather(WeatherArgs.model_validate_json(bad))
except ValidationError as e:
    print(e.error_count(), "validation errors")
    for err in e.errors():
        print(" ", err["loc"], err["msg"])
# -> 2 validation errors
# ->   ('city',) Field required
# ->   ('units',) String should match pattern '^(celsius|fahrenheit)$'
```

**Why this works, line by line.**

- `WeatherArgs(BaseModel)` with `Field(...)` constraints *is* the tool schema — `min_length=1`, the `units` regex, the default. This single class is the source of truth for the boundary.
- `WeatherArgs.model_json_schema()` emits standards JSON Schema — the exact shape you put in `tools=[...]` for Anthropic/OpenAI/Gemini function calling (field names differ per provider; the *schema* is shared). You never hand-write and hand-maintain a separate JSON schema that can drift from your Python; they're the same object. (Verify per-provider wrapping against current SDK docs — Day 24 does this concretely.)
- `model_validate_json(good)` parses *and validates* the model's JSON in one step, returning a typed `WeatherArgs`. Inside `get_weather`, `args.city` is *known* to be a non-empty str and `args.units` *known* to be one of two values — no defensive `if not city:` clutter, because the boundary already guaranteed it. That guarantee is also what `mypy --strict` type-checks against (`get_weather(args: WeatherArgs)`), so §1.5 and Pydantic reinforce each other.
- The malformed `bad` call — invented `units="kelvin"`, missing `city` — raises `ValidationError` **at the boundary**, with a structured, per-field error you can feed *back to the model* ("your call was invalid: city is required") to let it self-correct in the next loop iteration. The alternative — no validation — is the tool crashing with a `KeyError` deep inside, or worse, silently proceeding with a bad `units` and returning wrong data the agent then reasons on. **The validation error is a feature of the agent loop, not just a guard.**

**Cross-reference.** This is the runtime half of "validated boundaries"; §1.5's mypy is the static half. Together they seal the tool boundary from both sides — mypy proves *your code* uses `WeatherArgs` consistently; Pydantic proves *the model's data* conforms to it. Day 24 wires this into the actual ReAct loop.

---

## 2.3 Lockfiles + pinned deps as an even sharper security control for agent tool-execution

**Depth: [CORE]**

### Intuition

§1.11 established that a pinned, hashed lockfile is a supply-chain security control for *any* project. For an **agent that executes tools**, the stakes ratchet up for two compounding reasons:

1. **The agent's environment is a higher-value target.** A tool-executing agent often has credentials, network access, filesystem access, or a code-execution sandbox — precisely the capabilities an attacker wants. Compromising a dependency in the *agent's* environment (via an xz-style poisoned transitive dep, §1.11) hands the attacker those capabilities directly. The lockfile that pins and hashes the agent's entire graph is the control that turns "auto-pulled a poisoned latest" into "a reviewable, hash-verified diff."
2. **The catastrophic anti-pattern: an agent that `pip install`s at runtime.** A tempting agent capability is "install whatever package you need to solve the task" — the model decides it wants `numpy`, so your code runs `pip install numpy` mid-loop. **This is a gaping supply-chain hole.** You've handed a non-deterministic, injectable component the ability to pull *arbitrary code from the internet and execute it* inside your trusted, credentialed environment. A prompt injection ("install this helpful package") becomes remote code execution. The model might typo a package name into a **typosquat** (a malicious package named `reqeusts`), or be steered to a hostile one. There is no lockfile discipline possible when the dependency set is chosen *at runtime by the model*.

The principle: **an agent's dependency set must be fixed, pinned, hashed, and audited *ahead of time* (in `uv.lock`, §1.2), exactly like any other production service — and the agent must never be able to mutate it at runtime.** If a tool genuinely needs a library, it goes through the *human* upgrade policy (§1.10) and the CI gate (§1.8), not through the model's whim. The agent runs against a *frozen, known* graph.

### Analogy — the surgeon's pre-counted instrument tray

Before surgery, every instrument is counted onto the tray and verified; nothing enters the sterile field that wasn't checked in advance, and instruments are counted again at the end so nothing is left behind. A pinned, hashed lockfile is that pre-counted, verified tray for the agent's environment: the exact, audited set of code, sealed before the operation. An agent that `pip install`s at runtime is a surgeon reaching out of the sterile field mid-operation to grab an unverified, unsterilized instrument a stranger handed through the door — and using it inside the patient.

**Where the analogy breaks:** a surgical instrument is inert and its risk is contamination; a runtime-installed *package executes code on install and import* — it's not a dirty scalpel, it's a scalpel that can *act on its own* inside the patient. The failure isn't just "unsterile"; it's active hostile capability running with the agent's privileges. And unlike a countable tray, a transitive dependency graph is dozens deep, so "counting the instruments" means the lockfile enumerating the *entire* transitive set with hashes — a job no human could do by eye, which is exactly why the tooling does it.

### Worked example — the two agent deployment postures, contrasted

```
POSTURE A (unsafe): agent can install at runtime
────────────────────────────────────────────────
  user/injected prompt ──► model: "I'll pip install `helpfullib`"
     ──► your loop runs: subprocess(["pip", "install", "helpfullib"])
     ──► arbitrary code from PyPI executes with the AGENT's credentials
     ──► typosquat / poisoned dep = RCE inside your trusted environment
  Dependency set: UNKNOWN until runtime. Un-auditable. Un-lockable.

POSTURE B (safe): frozen, pinned environment
────────────────────────────────────────────────
  all deps resolved AHEAD of time  ──► uv.lock (hashed, universal)
     ──► CI installs `uv sync --frozen`  ──► image built once, immutable
     ──► agent runs against a KNOWN graph; cannot install anything new
  A new library requires: a human PR → §1.10 policy → §1.8 CI gate → redeploy
  Dependency set: fixed, hashed, auditable, reviewed. The model cannot change it.
```

The contrast is the whole lesson: Posture A makes the model's non-determinism reach *all the way into your supply chain*; Posture B confines the model's non-determinism to *choosing among tools you pre-approved*, running in an environment whose code was pinned and audited before the model ever ran. Cross-reference Day 2's process-isolation sandbox and Day 8's agent-sandbox design — the frozen dependency graph is the *supply-chain* layer of that same defense-in-depth; process/network isolation is the *runtime* layer.

**Honesty note.** "Never install at runtime" is the strong default. There are legitimate sandboxed-code-execution agents (data-analysis agents that run model-written code in a *disposable, network-isolated, unprivileged* container). The reconciliation: such execution happens in a **throwaway sandbox with no credentials and no network egress** (so a poisoned install can't exfiltrate or persist), *not* in the agent's own trusted, credentialed process. The lockfile pins *your* environment; the sandbox contains *the model's* code. Those are two different environments with two different threat models — conflating them is the mistake.

---

## 2.4 `mypy --strict` catching bugs in the agent loop before runtime

**Depth: [WORKING]**

### Intuition

The agent loop is fiddly, stateful code: you accumulate a `messages` list of dicts/objects with specific shapes, you branch on a `stop_reason`, you must append **one** `tool_result` per `tool_use_id` with the right keying, and you must guard the loop with `MAX_ITERS`. The provider SDKs have precise types for all of this. A wrong-shaped message, a mis-keyed tool result, a `None` where a block was expected — these are the bugs that surface *three iterations deep*, mid-conversation, after you've already spent tokens (and money) getting there. They're expensive to reproduce because the model's non-determinism means the failing path isn't the same twice.

`mypy --strict` (§1.5) catches this entire class *before you run the loop*. If the SDK types a message as a `list[MessageParam]` and you append a bare `str`, mypy fails at author-time. If your tool-dispatch function is annotated to return `ToolResultBlockParam` and you return a `dict` missing a field, mypy fails. You pay for the model's non-determinism *once* (at runtime, unavoidably); you should not *also* be paying for deterministic type bugs at runtime when a static check would have caught them for free. In an agent loop, "shift the bug left" is worth more than in ordinary code because runtime reproduction is so much dearer.

### Runnable example — a mis-keyed tool result, caught statically

```python
# src/skeleton/agent_shapes.py — provider-agnostic illustration of the shape bug.
from typing import Literal, TypedDict


class ToolResult(TypedDict):
    type: Literal["tool_result"]
    tool_use_id: str
    content: str


def make_result(tool_use_id: str, output: str) -> ToolResult:
    # BUG: typo'd the key as `tool_id` instead of `tool_use_id`, and dropped `content`.
    return {"type": "tool_result", "tool_id": tool_use_id}   # type: ignore-free on purpose


def loop_step(results: list[ToolResult]) -> None:
    for r in results:
        print(r["tool_use_id"])
```

```bash
uv run mypy --strict src/skeleton/agent_shapes.py
# -> src/skeleton/agent_shapes.py:13: error: Missing key "content" for TypedDict "ToolResult"  [typeddict-item]
# -> src/skeleton/agent_shapes.py:13: error: Extra key "tool_id" for TypedDict "ToolResult"  [typeddict-unknown-key]
# -> Found 2 errors in 1 file (checked 1 source file)
```

**Why this works.**

- Modeling the tool-result shape as a `TypedDict` (the SDKs use real typed params; `TypedDict` illustrates the mechanism provider-agnostically) means mypy *knows the required keys*. The typo `tool_id` and the missing `content` are flagged as `typeddict-unknown-key` and `typeddict-item` — **before the loop ever runs**, before a single token is spent. Had this shipped, it would surface only when the model *actually* requested a tool, N iterations in, as a provider API 400 or a `KeyError`.
- This is §1.5's "validated boundaries" applied to the *internal* boundary of the loop: the contract between "code that builds messages" and "code that consumes them." mypy proves the two halves agree. Combined with §2.2's Pydantic (the *external* model-data boundary) and §2.3's frozen deps (the *supply-chain* boundary), all three of the agent's deterministic boundaries are sealed, leaving exactly one uncertain thing — the model's actual output — which is where your runtime attention belongs.

**Cross-reference.** Real SDK types (Anthropic's `MessageParam`/`ToolUseBlock`, OpenAI's typed params) make this sharper than the `TypedDict` sketch; Day 24 uses them directly. The point today is that turning on `--strict` in the skeleton (§1.5) means the agent loop you build later is type-checked from its first line.

---

## 2.5 Case study & In production (Part 2)

### Case study — the AutoGPT-era "runtime-install / unbounded autonomy" failures (real, 2023)

**What happened.** The 2023 wave of autonomous-agent projects (AutoGPT and kin) demonstrated the failure modes this Part warns about in the wild. Two are directly on-topic: (1) agents that could **install packages and execute arbitrary code at runtime** as a "capability," turning the agent into an unsandboxed code-execution surface driven by model output (and by injected content it read from the web) — the Posture A hole of §2.3; and (2) agents with **no stop conditions** looping until they exhausted budgets. The community response — sandboxing code execution in disposable containers, pinning environments, adding iteration/cost ceilings — is precisely the discipline of Part 1's frozen lockfile plus Day 24's loop guards. **Engineering lesson:** the moment you give a non-deterministic component the ability to mutate its own execution environment (install code) or run unbounded, you've built an un-auditable, un-securable system. The lockfile-frozen, pre-audited environment (§2.3) and the typed/validated boundaries (§2.2, §2.4) are what make an agent a *system you can reason about* rather than a liability. Primary source: Anthropic's "Building Effective Agents" (the case for simple, bounded, well-scoped workflows over open-ended autonomy — the Day 24 taxonomy); contemporaneous AutoGPT issue trackers documenting runaway loops and code-exec risks. **Verify current** — the specific projects evolved rapidly.

*(Per Principle 7: I'm not aware of a single canonical, named *company* postmortem specifically about an agent's dependency supply-chain compromise that I can cite accurately, so I don't invent one. The real, on-topic material is the AutoGPT-era pattern above plus the general supply-chain postmortem (xz, §1.11), which applies with *more* force to a credentialed agent environment.)*

### In production (condensed, [WORKING] tier)

- **Freeze the agent's environment like any prod service** — `uv.lock`, `uv sync --frozen`, immutable image, `--no-dev` for the runtime. The agent never `pip install`s (§2.3).
- **Validate every tool input through Pydantic** (§2.2); feed validation errors back to the model for self-correction rather than crashing. Validate *outputs* too if a tool returns data the loop reasons on.
- **`mypy --strict` the loop and tool dispatch** (§2.4) so shape bugs die at author-time, not token-time.
- **Top failure mode:** the same one from §1.7 of the Day 5 lesson-family — a new SDK/HTTP client created *per loop iteration* instead of a shared pooled client, leaking sockets and re-doing TLS. In an agent this compounds because a single user turn can trigger many iterations. Reuse one client.
- **Cost note:** every runtime bug the toolchain catches statically is a bug you *don't* pay model tokens to reproduce. In agentic systems the toolchain's ROI is partly measured in saved API spend.

---

# PART 3 — THE BRIDGE

> Where the two layers become one system. No new concepts here — only how Part 1's toolchain and Part 2's agent depend on each other, share one pipeline, and fail together. The skeleton is not "a backend skeleton" and "an agent skeleton"; it is *one* substrate that both code paths commit into.

## 3.1 The dependency map — one gate protects both code paths

The FastAPI API and the agent loop are not two projects. They are two code paths in **one repository**, sharing **one `pyproject.toml`**, **one `uv.lock`**, **one `mypy --strict` config**, **one pre-commit gate**, and **one CI pipeline**. Draw the dependencies:

```
                          ONE  pyproject.toml  +  ONE  uv.lock
                          (the pinned graph both paths import from)
                                      │
              ┌───────────────────────┴────────────────────────┐
              ▼                                                 ▼
     PART 1 code path                                  PART 2 code path
     FastAPI /health, /agent endpoint                 agent loop + tools
     Pydantic request/response models  ◄── shared ──► Pydantic tool-arg models (§2.2)
     async handlers (tested §1.6)                     ReAct loop, tool dispatch (§2.4)
              │                                                 │
              └───────────────────────┬────────────────────────┘
                                      ▼
                    ONE gate:  ruff → mypy --strict → pytest
                    (pre-commit §1.7  +  CI §1.8, uv sync --frozen)
                                      │
                                      ▼
                    merges to main only if BOTH paths are
                    formatted, typed, tested, and lock-faithful
```

What the agent *calls into* on the backend side: the FastAPI request path (routing, the async event loop — Day 21, serialization), the Pydantic models, the pooled clients. What the backend *serves back*: a validated request in, a validated response out, an HTTP surface for the agent. The **toolchain sits underneath both** and gates them together — a single `mypy --strict` run type-checks the endpoint handler *and* the tool-dispatch function; a single `uv.lock` pins FastAPI *and* the model SDK; a single `pytest` run exercises the `/health` endpoint *and* the agent loop's shape contracts.

The crucial consequence: **you cannot merge a change that breaks either path.** A refactor of the shared Pydantic models that satisfies the API but breaks a tool schema fails the *one* mypy gate. A dependency bump that upgrades a transitive dep used by both fails the *one* CI run if either path's tests break. There is no "the backend is fine, the agent team will notice later" — the monorepo/one-gate design (§1.9) makes the two paths' correctness *atomic*, exactly the property that made the monorepo the right call.

## 3.2 The shared failure mode — an unpinned transitive dep breaks a deployed agent

This is the through-line of the whole day, made concrete as *one* failure that hits *both* layers at once.

**The scenario.** Suppose the skeleton did *not* commit `uv.lock` (or CI installed without `--frozen`). Both the FastAPI app and the agent loop depend, transitively, on `pydantic` → `pydantic-core` → some low-level package. A transitive dependency ships a new minor version with a subtle serialization change. On the next deploy, the fresh install resolves the *new* transitive version:

```
un-pinned transitive dep bumps  ──►  fresh prod install pulls the new version
       │
       ├──► PART 1: FastAPI response serialization changes shape slightly →
       │            a client contract breaks, 200s become malformed bodies
       │
       └──► PART 2: the SAME serialization path is how tool-arg JSON is parsed →
                    the agent's Pydantic tool validation (§2.2) now accepts/rejects
                    differently → the loop mis-parses a tool call three iterations in
```

**One un-pinned dependency, two simultaneous production failures**, in code nobody touched, discovered at 2 a.m., non-reproducible on the developer's laptop (which still has the *old* transitive version). This is §1.1's dependency-hell nightmare, now with an agent's non-determinism layered on top to make reproduction even harder — you can't even reliably re-trigger the failing tool-call path.

**How the skeleton prevents it, at every layer we built:**
- `uv.lock` pins the transitive version, so prod installs the *same* bytes as the laptop (§1.2).
- `uv sync --frozen` in CI (§1.8) fails if the lock is stale, so the bump can't sneak in un-reviewed.
- The dependency-upgrade policy (§1.10) surfaces the bump as an *automated PR* that runs the *full gate* — so if the serialization change breaks either path, **the PR goes red and never merges**, and a human sees exactly which transitive version moved.
- `mypy --strict` and `pytest` (running both the API test and the agent-shape tests) are what turn "red" into a *specific, localized* failure instead of a mystery.

The lockfile is where the two layers' fates are joined: it is *literally one file* that determines the exact code both the API and the agent run. Get it right and both are reproducible; get it wrong and both break together.

## 3.3 The skeleton as the permanent substrate

Everything the plan builds for the next 93 days **commits into this skeleton**. Day 8 adds the "life of `python app.py`" trace; Phase 1 puts networking tools beside the FastAPI app; Phase 3 grows the API properly; Day 24 drops the hand-rolled agent loop into `src/skeleton/agent.py` behind the `/agent` endpoint; Phase 4 adds a database and vector store to the *same* `pyproject.toml`; Phase 8 hardens the *same* CI. Each day is a commit that passes the *same* gate.

That is why today's build is "permanent" and foundational: it is not one day's exercise but the **container** (§1.1's analogy) every later day's cargo ships in. The toolchain you wired — pinned graph, typed boundaries, one gate — is the substrate that lets you add a non-deterministic model on Day 24 and *still* reason about the system, because everything around the model is deterministic, reproducible, and tested. The plan's thesis — an agent is a distributed backend system with one non-deterministic component — is *operationalized* by this skeleton: it makes the deterministic 95% trustworthy so the 5% that's genuinely uncertain is the only thing you have to watch.

---

# Topic-wide wrap-up

## Cheat Sheet — the whole toolchain at a glance

| Tool | Job | Key command | Config (in `pyproject.toml`) | The one thing to remember |
|---|---|---|---|---|
| **uv** | Python + envs + locked deps | `uv add`, `uv sync --frozen`, `uv run` | `[project]`, `[dependency-groups]` | Commit `uv.lock`, never `.venv`. Universal, hashed graph. |
| **ruff** | lint + format | `ruff check --fix`, `ruff format` | `[tool.ruff]`, `[tool.ruff.lint]` | One Rust single-pass AST → 10–100× faster; replaces black/isort/flake8/… |
| **mypy** | static type check | `mypy` (strict) | `[tool.mypy] strict = true` | Validated *code-to-code* boundary. Not runtime; not correctness. |
| **pytest (+asyncio)** | tests, incl. async | `pytest` | `[tool.pytest.ini_options] asyncio_mode="auto"` | Async test via `httpx.ASGITransport`, no live server. |
| **pre-commit** | local gate | `pre-commit install` | `.pre-commit-config.yaml` | Fast, skippable convenience. Pin hook `rev`s. |
| **GitHub Actions** | enforcement wall | `.github/workflows/ci.yml` | `uv sync --frozen` + same 4 checks | The un-bypassable gate. Branch protection makes it real. |
| **Pydantic** | runtime data validation | `model_validate_json` | (models in code) | Validated *external-data* boundary; tool-arg schema (§2.2). |

**The mental model in one line:** `pyproject.toml` = intent, `uv.lock` = reality (pinned + hashed), `.venv` = disposable materialization; mypy seals code boundaries, Pydantic seals data boundaries, the CI gate seals the merge.

## Build This — runnable definition of done

Build the permanent skeleton from scratch. **Definition of done** (every command exits 0):

```bash
uv init --package --python 3.12 skeleton && cd skeleton
uv add "fastapi>=0.121" "uvicorn[standard]>=0.38" "pydantic>=2.12"
uv add --dev "ruff>=0.14" "mypy>=1.18" "pytest>=8.4" "pytest-asyncio>=1.2" "httpx>=0.28" "pre-commit>=4.3"
# ... write pyproject.toml tool tables (§1.3), src/skeleton/app.py (§1.6),
#     tests/test_app.py (§1.6), .pre-commit-config.yaml (§1.7), .github/workflows/ci.yml (§1.8)
uv run ruff format --check .    # -> N files already formatted
uv run ruff check .             # -> All checks passed!
uv run mypy                     # -> Success: no issues found in N source files
uv run pytest                   # -> 1 passed
uv run pre-commit install && git add -A && git commit -m "skeleton"   # gate runs, commit succeeds
```

You're done when: (1) all four checks pass locally; (2) a deliberately-broken commit (unused import + a `None`-attribute bug) is *blocked* by pre-commit; (3) the repo pushes and CI goes green from the lockfile on a clean runner; (4) `uv.lock` is committed and `.venv` is git-ignored. **Stretch:** add a `.github/dependabot.yml` (§1.10) and confirm it opens a grouped dev-dependency PR.

## Active Recall & Self-Test (answer from memory)

1. What three things does `uv` unify, and what does its lockfile capture that `pip freeze` doesn't?
2. Give the mechanism (not just "it's Rust") for Ruff being ~100× faster than the flake8/black/isort stack.
3. Why does the `src/` layout catch packaging bugs a flat layout hides?
4. mypy vs Pydantic: what boundary does each validate, and why does an agent need *both*?
5. Which single CI line guarantees prod installs the same bytes the developer tested, and what does it do on a stale lock?
6. Why is "an agent that `pip install`s at runtime" a supply-chain hole, and what's the safe posture?
7. Explain, in the shared-failure-mode terms of Part 3, how *one* unpinned transitive dep can break both the API and the agent at once.
8. Monorepo vs polyrepo for 5 services + a shared lib: which, and what's the main CI cost you take on?

**60-second teach-back:** Explain to a beginner why "the toolchain isn't ceremony." Hit: lockfile = reproducibility + security control (left-pad/xz); types = validated boundaries that shift a bug class from runtime to author-time; one CI gate = the un-bypassable wall so review is about design; and why all of this matters *more* once a non-deterministic model is in the loop (it shrinks "what went wrong" to the one uncertain component).

## Spaced-Repetition Flashcards

- **Q:** Commit `uv.lock` or `.venv`? → **A:** `uv.lock` always; `.venv` never (rebuild it from the lock).
- **Q:** What makes `uv.lock` "universal"? → **A:** It captures resolution across *all* target platforms/Python versions, not just the current machine.
- **Q:** Ruff's speed mechanism? → **A:** Parse to one AST once, run all 900+ rules in a single Rust pass, parallel + cached.
- **Q:** `mypy --strict` catches what class of bug at author-time? → **A:** Type mismatches, incl. `None`-attribute access, missing/extra TypedDict keys, untyped functions leaking `Any`.
- **Q:** mypy validates ___; Pydantic validates ___. → **A:** static code-to-code type consistency; runtime external-data conformance.
- **Q:** The CI line that prevents "different bytes in prod"? → **A:** `uv sync --frozen`.
- **Q:** Lockfile as security control — name the two incidents. → **A:** left-pad (availability), xz-utils CVE-2024-3094 (poisoned release).
- **Q:** One Pydantic model gives you which three things at the tool boundary? → **A:** JSON Schema to send the model, runtime validation of its call, a mypy-checked type for your code.
- **Q:** Safe vs unsafe agent dependency posture? → **A:** Frozen pre-audited `uv.lock` (safe) vs runtime `pip install` chosen by the model (RCE hole).
- **Q:** pre-commit vs CI — which is the guarantee? → **A:** CI (un-bypassable, server-side, branch-protected); pre-commit is the skippable local convenience.

## Primary Sources (verify against these; versions drift)

- **uv:** `docs.astral.sh/uv` — Projects/Structure, Resolution, Locking & Syncing, GitHub Actions/Dependabot/Renovate integration guides.
- **ruff:** `docs.astral.sh/ruff` — overview, linter, formatter, FAQ (vs flake8/black/isort), default rules; Charlie Marsh, "Python tooling could be much, much faster."
- **mypy:** mypy docs — strict mode & config; `ty` repo for the [AWARE] preview checker.
- **pytest:** `docs.pytest.org`; `pytest-asyncio` docs; FastAPI "Testing"/"Async Tests"; `python-httpx.org` `ASGITransport`.
- **PEPs:** 518 (build-system), 621 (project metadata), 735 (dependency groups), 751 (`pylock.toml` standard lockfile), 585 (built-in generic types).
- **Packaging:** Python Packaging User Guide — "src layout vs flat layout."
- **Supply chain:** NVD/CISA CVE-2024-3094 (xz); Andres Freund's `oss-security` disclosure (2024-03-29); Russ Cox's xz timeline; npm/`The Register` left-pad coverage (2016).
- **Monorepo gates:** *Software Engineering at Google* (Static Analysis, Code Review chapters); Google Python Style Guide; Meta Engineering on Pyre/Sapling.
- **Agents:** Anthropic, "Building Effective Agents."
- **pre-commit / CI:** `pre-commit.com`; GitHub Actions docs.

## Layered explanations

- **10 seconds:** One repo skeleton — `uv` for pinned deps, `ruff` for lint/format, `mypy --strict` for types, `pytest` for tests, wired into pre-commit + CI — so nothing un-formatted, un-typed, or un-tested reaches `main`.
- **1 minute:** `pyproject.toml` is your intent; `uv.lock` is the exact, hashed, universal reality; `.venv` is disposable. Ruff is one Rust single-pass AST replacing a whole tool stack at 10–100× speed. mypy seals code-to-code boundaries statically; Pydantic seals data boundaries at runtime; together that's "validated boundaries, not ceremony." pre-commit is the fast local nag, CI (`uv sync --frozen` + the four checks, branch-protected) is the un-bypassable wall. Lockfiles are a *security* control (left-pad, xz), not pedantry.
- **5 minutes:** The whole of §1.1–1.8: dependency hell has three faces (which Python, which transitive versions, reproducibly) that the modern toolchain solves with one metadata file + one lockfile + fast gates. `uv`'s resolver solves the *whole graph* at once (PubGrub-style backtracking) and writes a universal, hashed lock — the reproducibility and supply-chain control. `src/` layout forces tests to run against the *installed* package. Ruff's single-AST single-pass Rust architecture is the speed. `mypy --strict` turns a class of runtime crash into author-time errors. pytest+asyncio tests the real async request path in-process. pre-commit + CI enforce it, with `--frozen` guaranteeing prod parity. Then §1.9–1.11: monorepo (atomic cross-service changes) vs polyrepo (release autonomy); an upgrade policy with CVE SLAs and automated PRs; and why left-pad and xz make pinned/hashed/auditable deps a security control.
- **Expert:** The toolchain is the *deterministic containment* around a system that will, on Day 24, gain one non-deterministic component. Every gate — pinned hashed universal lockfile, static type boundaries, runtime Pydantic validation, one enforcement wall — exists to reduce the failure-attribution search space to that single component. For an agent this is sharper: the lockfile is supply-chain defense for a *credentialed, injectable* execution environment (never `pip install` at runtime); Pydantic is the adversarial-input boundary for tool calls (one model → JSON Schema out, validation in, mypy type in code); mypy shifts loop-shape bugs left of the expensive token-spending runtime. Part 3's join is the lockfile: one file fixes the exact bytes both the API and the agent run, so they are reproducible together and, when it's wrong, fail together. The skeleton is the permanent substrate all 93 remaining days commit into — operationalizing the plan's thesis that an agent is a distributed backend system with one non-deterministic component.
