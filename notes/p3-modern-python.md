# Day P3 — Modern Python: Types, Virtual Environments & the Async Syntax You'll Need

> **What this covers:** the four pieces of "modern Python" that every later day of this plan silently assumes — **type hints** (declared contracts that tools can check), **virtual environments & pip** (per-project dependency isolation), the **`async`/`await` syntax** at recognition level, and the **everyday tools** (`f-strings`, `enumerate`, `zip`, `with`). Part 2 then shows each of these four appearing, unchanged, in real agentic/LLM code.
> **Prerequisites:** [Day P1](p1-python-fundamentals-1.md) (values, control flow, functions) and [Day P2](p2-python-fundamentals-2.md) (collections, classes, modules, exceptions).
> **The ONE idea that unites this day:** *reliable software — backend or agentic — is built from two disciplines: **explicit contracts** (a function declares what it accepts and returns; a tool declares what inputs it takes) and **isolated, reproducible environments** (each project carries its own dependencies). Type hints and venvs are Python's native versions of those two disciplines, and the agentic stack (Pydantic, FastAPI, the Anthropic SDK, tool schemas) is built directly on top of them.*
>
> **How to read this:** every concept runs the ladder **intuition → analogy (with where it breaks) → concrete worked example → diagram → under-the-hood**. When something feels abstract, jump to its "**Worked example**" or "**Runnable example**" block and run the code.
>
> **Depth tiers:** **[CORE]** open every box · **[WORKING]** use it correctly, know the tradeoffs · **[AWARE]** know it exists and when to reach for it.
>
> **Calibration note for today:** `async`/`await` is deliberately taught at **syntax-recognition level only**. What it actually *does* — the event loop, cooperative scheduling, why it makes servers fast — is Days 19–21. Today you only need to be able to *read* async code without flinching, because the Anthropic SDK examples you'll meet from here on use it.
>
> *(P-days are pure-fundamentals on-ramp material: per the study plan, the `### System design`, `### Case studies`, and `### In production` blocks begin on Day 1, not here. Everything else — worked examples, runnable code, recall drills — applies in full.)*

---

# PART 1 — BACKEND: The Modern Python Toolkit

## Overview & motivation — why "modern" Python is different from P1/P2 Python

On P1 and P2 you learned Python the language: values, control flow, functions, collections, classes, modules. That Python has existed since the 1990s. But the Python you will actually write for the next 100 days — the Python that FastAPI, Pydantic, and the Anthropic SDK are written in — carries three additions that transformed how professional Python is engineered:

1. **Type hints** (2014, PEP 484) turned Python functions from "call it and hope" into **readable, machine-checkable contracts**. Every framework in this course is built on them.
2. **Virtual environments** solved the "two projects need two different versions of the same library" disaster that plagued a decade of Python development. You will never again install a project's packages globally.
3. **`async`/`await`** (2015, PEP 492) gave Python first-class syntax for "do useful work while waiting on the network" — the reason a single FastAPI process can serve thousands of simultaneous requests, and the reason an agent can fire off five LLM calls at once.

Plus a handful of small, everyday tools (`f-strings`, `enumerate`, `zip`, `with`) that appear in essentially every file you will read or write.

The through-line of all of them: **make intent explicit, and let machinery enforce it.** A type hint makes the function's intent explicit so a checker can enforce it. A venv makes the project's dependencies explicit so the interpreter can isolate them. A `with` block makes a resource's lifetime explicit so Python can guarantee cleanup. Keep that framing — it recurs all day.

---

## 1. Type hints — declared contracts for functions and data **[CORE]**

**Depth: [CORE]** — the entire course rides on these. Pydantic (Day 8) *is* type hints with runtime enforcement; FastAPI generates validation, docs, and serialization *from* type hints; the Anthropic SDK is fully typed so your editor can autocomplete every field of an API response.

### 1.1 The problem before type hints — what pre-PEP-484 Python was like

Python is **dynamically typed**: a variable is just a name bound to an object, and the object's type is checked only at the moment an operation actually runs. That's what made P1 so pleasant — no ceremony, just `x = 5`. But it has a brutal cost at scale.

**Worked example — the 2 a.m. bug.** Here is a function from a pre-2014 codebase:

```python
def apply_discount(price, discount):
    return price - price * discount
```

Questions you cannot answer by reading it:

- Is `price` a `float` in rupees? An `int` in paise? A `str` like `"499.00"` parsed from a form?
- Is `discount` a fraction (`0.1` = 10%) or a percentage (`10` = 10%)?
- What does it return?

Now trace what happens when a teammate calls it wrong, with real values:

```
call:   apply_discount("499.00", 0.1)
step 1: price * discount  →  "499.00" * 0.1  →  TypeError: can't multiply sequence by non-int
```

That `TypeError` fires **at runtime** — maybe on your machine if you're lucky, maybe in production at 2 a.m. if you're not. And this is the *good* case, because it crashes loudly. The evil case:

```
call:   apply_discount(499.00, 10)      # caller thought discount was a percentage
step 1: 499.00 - 499.00 * 10  →  -4491.0
result: the function happily returns -4491.0 — a negative price, no error, silently wrong
```

**What people did before type hints** (the pain that motivated them):

- **Docstring conventions** — `:param str name:` / `:type price: float` in the docstring. Pure prose: nothing checked it, and docstrings drifted out of sync with the code within weeks.
- **Naming conventions** — `str_name`, `price_float` (a Hungarian-notation habit). Ugly, unenforced, and wrong the moment someone refactored.
- **Defensive `isinstance` checks** — `if not isinstance(price, float): raise TypeError(...)` at the top of every function. Boilerplate that ran *at runtime*, on *every call*, and still only caught the bug when the bad call actually executed.

All three share the fatal flaw: **the contract lived outside the language**, so no tool could verify it *before the program ran*.

**PEP 484** (2014, Guido van Rossum, Jukka Lehtosalo, Łukasz Langa — building on the annotation *syntax* from PEP 3107, 2006) fixed this by giving the annotations a standard *meaning*: a machine-readable type vocabulary that external tools — chiefly **mypy** (Lehtosalo's checker) and your editor — can verify without running anything. Crucially, they chose **gradual typing**: hints are optional, file by file, function by function. You can annotate one function in a million-line codebase and get value immediately.

### 1.2 Intuition — a contract stapled to the function

A type hint changes the function from an unlabeled black box into a signed contract:

```python
def apply_discount(price: float, discount: float) -> float:
    """discount is a fraction: 0.1 == 10% off."""
    return price - price * discount
```

Read it aloud: *"apply_discount takes a `float` called price and a `float` called discount, and returns a `float`."* Every question from the worked example above now has an answer **in the signature itself**. Two distinct payoffs:

1. **Humans read contracts, not implementations.** You now know how to call this function without opening its body. Multiply that by the thousands of functions you'll call this year.
2. **Tools catch violations before runtime.** `mypy` reads `apply_discount("499.00", 0.1)` and flags it as an error *while you're still in your editor* — the 2 a.m. crash becomes a red squiggle at 2 p.m.

### 1.3 Analogy — the shipping label (and where it breaks)

A type hint is like the **label on a shipping box**: "FRAGILE — GLASS — 12 kg — THIS SIDE UP." The label tells every handler in the chain what's inside and how to treat it, without them opening the box. A warehouse with labeled boxes runs itself; a warehouse of unlabeled boxes requires opening everything, constantly.

**Where the analogy breaks — and this is the single most important fact about type hints:** a real shipping company *weighs the box* and rejects a mislabeled one. **Python does not.** The label is advisory: at runtime, the interpreter reads the hints, stores them, and then **completely ignores them**. You can put a live cat in a box labeled "GLASS" and Python will ship it without blinking. Enforcement comes only from *external* checkers (mypy, pyright, your editor) — or, later, from libraries like Pydantic that deliberately read the labels at runtime and *do* weigh the box. Never forget this split:

```
   who reads the hint?          when?          does it enforce?
   ─────────────────────        ───────        ────────────────
   you / your teammates         reading        no (but informs)
   your editor (Pylance…)       as you type    warns (squiggles)
   mypy / pyright               pre-run check  yes — fails the check
   Python interpreter           runtime        NO — ignores them
   Pydantic (Day 8 preview)     runtime        YES — validates data
```

### 1.4 The syntax, piece by piece — worked examples with real values

**Parameters and return** — a colon after the name, an arrow before the colon of the body:

```python
def repeat(word: str, times: int) -> str:
    return word * times

repeat("ha", 3)        # -> "hahaha"     (matches the contract)
repeat(3, "ha")        # mypy: error — arguments swapped. Runtime: also "hahaha"! (str * int is
                       # commutative) — proof that "it ran" ≠ "it's called correctly"
```

**Variables** (PEP 526) — same colon syntax, usually only needed when the value doesn't make the type obvious:

```python
retries: int = 3
total: float = 0.0          # without the hint, `total = 0` would be inferred as int
names: list[str] = []       # an empty list is a mystery — the hint says what it will hold
```

**Collections** — the container type, then the element type(s) in square brackets (built-in generics, Python ≥ 3.9):

```python
scores: list[int] = [90, 85, 77]
prices: dict[str, float] = {"idli": 40.0, "dosa": 60.0}   # key type, value type
point: tuple[float, float] = (17.4, 78.5)                 # fixed shape: exactly two floats
tags: set[str] = {"python", "backend"}
```

**"Maybe missing" values** — the single most common real-world pattern. `X | None` (PEP 604, Python ≥ 3.10) means "either an X or `None`":

```python
def find_user(user_id: int) -> dict[str, str] | None:
    """Returns the user record, or None if no such user exists."""
    ...
```

The contract now *forces the caller to handle the None case* — a checker will flag `find_user(7)["name"]` because the result might be `None`. (You'll also see the older spelling `Optional[dict[str, str]]` from `typing` — identical meaning; `Optional[X]` is just `X | None`. "Optional" is a misleading name: it means *maybe-None*, not *optional argument*.)

**Functions that return nothing** — `-> None`, explicitly:

```python
def log_event(message: str) -> None:
    print(f"[event] {message}")
```

**Default values** — hint first, default after the `=`:

```python
def fetch(url: str, timeout_seconds: float = 10.0) -> str: ...
```

**The escape hatch** — `Any` from `typing` means "no contract here; anything goes; checker, look away." It's honest when data genuinely has no fixed shape (e.g. freshly parsed JSON, before validation):

```python
from typing import Any
def parse_json_body(raw: str) -> dict[str, Any]: ...   # keys are strings; values: anything
```

Every `Any` is a hole in your safety net — use it at the boundaries where untyped data enters (network, files), then convert to real types as fast as possible. (Pydantic, Day 8, is precisely the machine that converts `Any`-shaped JSON into typed, validated objects.)

**Worked example — one signature, read as a sentence.** This is the shape you'll write in the Build task today:

```python
def fetch_json(url: str, timeout_seconds: float = 10.0) -> dict[str, Any]:
```

> "`fetch_json` takes a `str` URL and an optional `float` timeout defaulting to 10.0, and returns a dict with string keys and unconstrained values."

One line; the whole calling convention. That's the payoff.

### 1.5 Visual — where the hints sit

```
        parameter hints                       return hint
        ┌─────┴──────────────┐               ┌───┴────┐
def name(param: TYPE, param: TYPE = default) -> TYPE:
    body...

variable hint:      name: TYPE = value
collection hints:   list[ELEM]   dict[KEY, VAL]   tuple[A, B]   set[ELEM]
maybe-missing:      TYPE | None            (older code: Optional[TYPE])
no contract:        Any                    (the deliberate hole)
```

### 1.6 Runnable example — hints are *ignored at runtime* (the surprise you must see once)

The most common beginner misconception is that hints make Python check types when the program runs. Prove to yourself that they don't:

```python
# hints_ignored.py — no install needed
def add(a: int, b: int) -> int:
    return a + b

print(add.__annotations__)   # where Python stores the hints
print(add("ab", "cd"))       # violates the contract on purpose
```

```bash
python hints_ignored.py
# -> {'a': <class 'int'>, 'b': <class 'int'>, 'return': <class 'int'>}
# -> abcd
```

**Why this works, line by line.** `def add(a: int, b: int) -> int` attaches the hints to the function object — Python evaluates them once at definition time and stores them in a plain dict, `add.__annotations__`, which the first `print` shows verbatim: `{'a': int, 'b': int, 'return': int}`. Then `add("ab", "cd")` hands the function two strings — a flat violation of the declared contract — and Python **runs it anyway**: `+` on two strings is concatenation, so you get `"abcd"` and no error of any kind. That's the whole runtime story: hints are *stored metadata*, not *checks*. The interpreter is the courier that never weighs the box. Everything useful about hints comes from tools that read that metadata — your editor and mypy read it *statically* (next example), and Pydantic (Day 8) reads `__annotations__` *at runtime* to actually validate data. Same metadata, three consumers.

### 1.7 Runnable example — mypy catching the bug before you run

`mypy` is the reference type checker: a program that reads your source, checks every call against every signature, and reports violations — without executing anything.

```python
# greet.py
def greet(name: str) -> str:
    return "Hello, " + name

print(greet("Vinay"))
print(greet(42))          # bug: an int where the contract says str
```

Real terminal transcript (inside a venv — see §2):

```bash
pip install mypy
# -> Successfully installed mypy-1.13.0 mypy_extensions-1.0.0 typing_extensions-4.12.2
#    (version numbers drift — yours will be newer)

python greet.py           # first: what the INTERPRETER thinks
# -> Hello, Vinay
# -> Traceback (most recent call last):
# ->   File "greet.py", line 6, in <module>
# ->     print(greet(42))
# ->   File "greet.py", line 3, in greet
# ->     return "Hello, " + name
# -> TypeError: can only concatenate str (not "int") to str

mypy greet.py             # now: what the CHECKER thinks, WITHOUT running
# -> greet.py:6: error: Argument 1 to "greet" has incompatible type "int"; expected "str"  [arg-type]
# -> Found 1 error in 1 file (checked 1 source file)
```

Fix the call (`print(greet(str(42)))` or pass a real name), then:

```bash
mypy greet.py
# -> Success: no issues found in 1 source file
```

**Why this works, line by line.** Running `python greet.py` executes top-to-bottom: the good call prints, then the bad call **crashes mid-program** — and note *where*: the `TypeError` erupts on line 3, *inside* `greet`, at the `+` — not at line 6 where the actual mistake lives (P1's traceback skill applies: read bottom-up, then walk up the call chain to find the bad caller). `mypy greet.py`, by contrast, never executes anything — it parses the file, sees `greet`'s declared contract `(str) -> str`, sees the call `greet(42)`, and reports the mismatch **at the call site, line 6**, with the exact category tag `[arg-type]`. That's the trade in one screen: the interpreter finds the bug *late, at the symptom*; the checker finds it *early, at the cause*. On a 5-line script the difference is cute; on a 40-step agent run that costs real API money per attempt (Part 3), it's the difference between a red squiggle and a burned afternoon.

### 1.8 Under the hood — what a hint actually is, and who consumes it

Open the black box fully ([CORE]):

- **A hint is an expression, evaluated at `def` time, stored in `__annotations__`.** `def f(x: int)` evaluates the expression `int` (yielding the class object `int`) and stores it in `f.__annotations__["x"]`. That's it. No wrapper, no guard, no changed bytecode for the body.
- **Static checkers never run your code.** mypy builds a syntax tree, infers a type for every expression (`"a" + name` where `name: str` → `str`), and checks assignments/calls/returns against declarations. This is why it can check code with side effects, missing credentials, or network calls — nothing executes.
- **Gradual typing is the design philosophy.** Unannotated code defaults to `Any`, which is compatible with everything — so mypy stays silent about it. Value appears function-by-function as you annotate. This is why hints could be retrofitted onto a 25-year-old language.
- **Runtime consumers exist, and they're the course's backbone.** Pydantic reads `__annotations__` on your classes and *generates validators* from them; FastAPI reads the hints on your endpoint functions and *generates request parsing, validation, and API docs* from them. When you write `def create_user(body: UserIn) -> UserOut:` on Day 9, the hints aren't documentation — they're the machine.

### 1.9 Why hints and not something else? — comparisons

| Approach | Checked by a tool? | When errors surface | Runtime cost | Verdict |
|---|---|---|---|---|
| Docstrings (`:type x: int`) | No | Never (prose) | none | drifts out of sync; dead convention |
| `isinstance` checks in every function | Sort of (by you) | runtime, on the bad call | every call | boilerplate; catches late |
| Rewrite in a static language (Go/Java) | Yes | compile time | none | loses Python + its ML ecosystem |
| **Type hints + mypy** | **Yes** | **pre-run, in-editor** | **~none** (metadata only) | contracts without leaving Python |
| Type hints + Pydantic (Day 8) | Yes, at runtime | runtime, at the data boundary | small, on validation | the complement: for *external* data |

The last two rows are a team, not rivals: **mypy checks your code against itself; Pydantic checks the outside world against your code.** mypy cannot know what JSON an API will return tonight; Pydantic can't catch a swapped argument you never run. You need both, and this course uses both.

### 1.10 Trade-offs & common mistakes

- **Cost:** hints add visual noise and a little typing time; complex types (nested dicts, callbacks) can take thought. Verdict of the industry: overwhelmingly worth it — every major Python codebase (and every library in this course) is typed.
- **Mistake — believing hints validate at runtime.** They don't (§1.6). If someone can pass you bad data at runtime, you need Pydantic-style validation, not hints alone.
- **Mistake — `Any` everywhere.** `dict[str, Any]` on every function is typing theater: it silences the checker without creating a contract. Use precise types inside your own code; reserve `Any` for genuine unknowns at boundaries.
- **Mistake — annotating the trivial.** `count: int = 0` is fine; `x: int = 5` two lines before `x` disappears is noise. Annotate function signatures always; variables only where inference fails or the reader needs help.
- **Mistake — old-style spellings in new code.** You'll see `List[int]`, `Dict[str, int]`, `Optional[X]` (capital, from `typing`) in older tutorials. In Python ≥ 3.10 write `list[int]`, `dict[str, int]`, `X | None`. Both work; know both; write the modern one.

---

## 2. Virtual environments & packages — per-project isolation **[CORE]**

**Depth: [CORE]** — every Build in this course starts with "create a venv." Getting this wrong produces the most demoralizing class of beginner failure: code that worked yesterday breaking today for invisible reasons.

### 2.1 First principles — packages, pip, and PyPI (defining terms before the problem)

From P2 you know a **module** is a `.py` file you can `import`, and a **package** is a folder of modules. A **third-party package** is simply a package *someone else wrote* and published so you can install and import it — `requests` (HTTP for humans), `fastapi`, `anthropic`.

- **PyPI** (the Python Package Index, pypi.org) is the public warehouse: ~half a million packages, each with every published version.
- **pip** is the installer that ships with Python: `pip install requests` downloads the package from PyPI (plus everything *it* depends on) and copies it into a folder called **`site-packages`** — the directory the interpreter searches when you write `import requests`.

*(Term necessity: before pip, installing a package meant downloading a tarball and running its `setup.py` by hand, or using the flaky `easy_install`; there was no reliable uninstall and no standard way to record what a project needed. pip + PyPI standardized install/uninstall/versioning — and in doing so made the next problem enormous, because now everyone installed lots of packages.)*

The critical question pip forces: **which `site-packages`?** By default — with no venv — there is exactly *one*, attached to your global Python installation. Every project on your machine shares it. That single shared folder is the villain of this section.

### 2.2 The problem — dependency hell in one shared `site-packages`

**Worked example — the version conflict, traced with real numbers.** You maintain two projects on one laptop:

```
t0  Project A (built in 2019) requires requests 2.19 — it relies on behavior that
    changed in later versions.
t1  You install it globally:      pip install requests==2.19.0
    → global site-packages now contains requests 2.19.0. Project A works.

t2  You start Project B (today). It needs a feature added in requests 2.31:
                                  pip install --upgrade requests
    → pip UPGRADES the one global copy to 2.32.3. Project B works.

t3  A week later you run Project A again:
    → import requests grabs the global copy — now 2.32.3, not 2.19.0
    → Project A crashes (or worse: silently misbehaves).
    Nothing in Project A changed. It broke because Project B exists.

t4  You "fix" it:                 pip install requests==2.19.0
    → downgrade. Project A works. Project B is now broken.
```

There is **no ordering of installs that satisfies both projects**, because one folder can hold only one version of a package. This is *dependency hell*, and pre-venv Python developers lived in it — the era's folklore is full of "never touch the system Python," `sudo pip install` disasters that broke OS tools (Linux distributions themselves use the system Python!), and "works on my machine" bugs that were really "my global site-packages happens to differ from yours."

The fix is conceptually obvious once stated: **stop sharing. Give every project its own private `site-packages`.** That is literally all a virtual environment is.

### 2.3 Intuition & analogy — one toolbox per job site (and where it breaks)

A venv is a **dedicated toolbox for one job site**. The plumbing job gets a toolbox with a 2019-model wrench because those pipes need it; the new construction site gets a toolbox with the 2024 wrench. Nobody upgrades "the one wrench everyone shares" and silently breaks the other site. Buying a second wrench is cheap; debugging a cross-site tool conflict is not.

**Where the analogy breaks:** (1) tools cost money, whereas duplicating packages costs only disk space — megabytes, the cheapest resource you have, which is why "isn't copying wasteful?" is the wrong worry; (2) a toolbox is passive, but a venv also carries its own *interpreter entry point* — activating it changes **which `python` runs**, not just which packages are visible; (3) real toolboxes get shared across sites when someone's lazy — a venv physically can't leak into another project unless you deliberately point at it.

### 2.4 Under the hood — what `python -m venv` actually creates **[CORE]**

Run `python3 -m venv .venv` and Python creates a small directory tree (`.venv` is the conventional name — the dot keeps it visually out of the way):

```
your-project/
└── .venv/
    ├── pyvenv.cfg               ← tiny text file: "home = /usr/bin" — marks this as a venv
    │                              and points back at the base Python it was made from
    ├── bin/                     ← Scripts\ on Windows
    │   ├── python               ← symlink/copy of the real interpreter
    │   ├── pip                  ← a pip wired to THIS venv
    │   └── activate             ← a shell script (see below)
    └── lib/python3.12/
        └── site-packages/       ← THE point: this project's private package folder
                                    (Windows: .venv\Lib\site-packages)
```

Two mechanisms make it work, and both are pleasingly dumb:

1. **The interpreter locates `site-packages` relative to its own executable.** When any `python` starts, it looks at the path it was launched from; if it finds `pyvenv.cfg` next door, it says "I'm in a venv" and sets its package search path to *this venv's* `site-packages` instead of the global one. So running `.venv/bin/python` **is already the whole trick** — activation is optional convenience.
2. **`activate` just edits your shell's `PATH`.** The `source .venv/bin/activate` command prepends `.venv/bin` to `PATH` (so a bare `python` or `pip` now resolves to the venv's copies) and adds the `(.venv)` marker to your prompt. `deactivate` undoes it. No magic, no registry, no config files touched — which is also why **deleting the `.venv` folder is a complete, safe uninstall** of everything in it.

Consequence worth engraving: `pip install` always installs into **whichever Python's `site-packages` the running `pip` belongs to**. Forget to activate, and packages land in the global folder — the #1 cause of "I installed it but Python says `ModuleNotFoundError`."

### 2.5 Runnable example — the full venv lifecycle (real terminal transcript)

This is the exact sequence you'll repeat at the start of every project. (Transcript from WSL2/Ubuntu per the course environment; Windows PowerShell differences noted inline. Version numbers drift — yours will be newer.)

```bash
mkdir p3 && cd p3

# 1. Create the venv (one time per project)
python3 -m venv .venv

# 2. Activate it (every time you open a new terminal for this project)
source .venv/bin/activate                # Windows PowerShell: .venv\Scripts\Activate.ps1
# -> prompt changes:  (.venv) vinay@machine:~/p3$

# 3. Prove the switch happened: which python runs now?
which python                             # Windows: Get-Command python
# -> /home/vinay/p3/.venv/bin/python     ← the venv's interpreter, not the system one

# 4. A fresh venv is EMPTY except pip itself
pip list
# -> Package    Version
# -> ---------- -------
# -> pip        24.2

# 5. Install this project's first dependency
pip install requests
# -> Collecting requests
# ->   Downloading requests-2.32.3-py3-none-any.whl (64 kB)
# -> Collecting charset-normalizer<4,>=2 (from requests)
# -> Collecting idna<4,>=2.5 (from requests)
# -> Collecting urllib3<3,>=1.21.1 (from requests)
# -> Collecting certifi>=2017.4.17 (from requests)
# -> Successfully installed certifi-2024.8.30 charset-normalizer-3.4.0 idna-3.10
#    requests-2.32.3 urllib3-2.2.3

# 6. Verify: importable HERE...
python -c "import requests; print(requests.__version__)"
# -> 2.32.3

# 7. ...and invisible OUTSIDE (this is the isolation, demonstrated)
deactivate
python3 -c "import requests"
# -> ModuleNotFoundError: No module named 'requests'
#    (unless you polluted your global site-packages in a past life)
```

**Why this works, line by line.** Step 1 builds the directory tree from §2.4 — nothing more. Step 2 (`source .venv/bin/activate`) edits `PATH`, which step 3 proves: `which python` now resolves to the venv's binary. Step 4 shows the blank slate — a new venv contains *nothing* but pip, which is the point: your project's dependencies will be exactly and only what you install. Step 5 is pip doing **dependency resolution**: you asked for one package, and the `Collecting ... (from requests)` lines show pip walking `requests`' own declared dependencies (four of them) and installing the whole closure into `.venv/lib/python3.12/site-packages`. Step 6 proves `import` finds it (because this `python` searches the venv's `site-packages`). Step 7 is the money shot: after `deactivate` restores your old `PATH`, the *system* interpreter searches the *global* `site-packages`, finds no `requests`, and fails — proving the install never leaked out. Two projects, two venvs, two `requests` versions, zero conflict: dependency hell dissolved by a folder convention.

### 2.6 Reproducibility — `requirements.txt` **[WORKING]**

Isolation solves conflicts on *your* machine; **`requirements.txt`** solves "how does a teammate (or the you of next month, or a server) rebuild this exact environment?" It's a plain text file listing pinned dependencies:

```bash
pip freeze > requirements.txt      # snapshot the venv's exact contents
cat requirements.txt
# -> certifi==2024.8.30
# -> charset-normalizer==3.4.0
# -> idna==3.10
# -> requests==2.32.3
# -> urllib3==2.2.3
```

Anyone can now recreate the environment from scratch:

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt   # -r: read the list from a file; == pins exact versions
```

**Worked example — why pinning matters.** Your `requirements.txt` says `requests==2.32.3`. Six months from now `pip install -r requirements.txt` still installs **2.32.3** — not whatever 2.35 changed. Without the pin (`requests` bare), the server deploy in March silently gets a different library than the laptop test in January, and you've reinvented dependency hell *across time* instead of across projects. Convention you'll follow: **the venv is disposable and never committed to git** (add `.venv/` to `.gitignore` on Day P4); **`requirements.txt` is the durable record and always committed.**

### 2.7 pip — the six commands you actually use **[WORKING]**

| Command | What it does | Note |
|---|---|---|
| `pip install requests` | install newest version + its dependencies | into the *active* environment |
| `pip install requests==2.32.3` | install an exact version | `==` is the pin |
| `pip install -r requirements.txt` | install everything a file lists | the rebuild command |
| `pip list` | table of installed packages | "what's in this venv?" |
| `pip show requests` | one package's version, deps, location | note the `Location:` line — it names *which* `site-packages`, your forgotten-to-activate detector |
| `pip freeze` | pinned `name==version` lines | pipe to `requirements.txt` |

**[WORKING] boundary, closed deliberately:** how pip resolves conflicting version constraints between packages, builds wheels, or caches downloads is real machinery we won't open — treat pip as a reliable black box that "installs a package and its dependency closure into the active environment's `site-packages`."

### 2.8 `uv` — a preview **[AWARE]**

**Depth: [AWARE].** `uv` is a modern, Rust-built replacement for the venv+pip workflow — same concepts, dramatically faster (10–100×), with project-level lockfiles (`uv init`, `uv add requests`, `uv run app.py` — it creates and manages the venv for you). It exists because pip's resolver is slow on large dependency trees and the venv/pip/freeze dance is three tools where one would do. **You'll adopt it properly on Day 7**; the course's backend code is `uv`-managed from then on. Until then, plain venv+pip — everything you learned today transfers 1:1, because `uv` manages the *same* venvs and the *same* `site-packages`; only the driver changes. Treat as a black box until Day 7.

### 2.9 Common mistakes

- **Installing without activating** → package lands in global `site-packages` → the venv's python can't see it (`ModuleNotFoundError`) *or* the global pollution masks a missing pin. Diagnostic reflex: `which python` / `pip show <pkg>` and read the `Location:`.
- **`sudo pip install` on Linux/WSL** → writes into the *operating system's* Python and can break OS tooling. Never. A venv makes sudo unnecessary by design.
- **Committing `.venv/` to git** → thousands of binary files, platform-specific, useless to teammates. Commit `requirements.txt`; gitignore `.venv/`.
- **One giant venv for all projects** → recreates the global-site-packages problem one level down. One project, one venv. They're free.
- **Recreating vs. repairing** → a venv behaving weirdly is not worth archaeology: `deactivate`, delete `.venv/`, recreate, `pip install -r requirements.txt`. Two minutes, guaranteed-clean.

---

## 3. `async` / `await` — recognize the syntax (mechanics on Days 19–21) **[WORKING]**

**Depth: [WORKING]** for the *syntax*; the machinery underneath (the event loop) is **explicitly deferred to Days 19–21** and treated as a sealed black box today. The goal is narrow: when you see `async def` and `await` in SDK examples — which you will, starting in Part 2 today — you can read them correctly and avoid the one classic mistake.

### 3.1 Intuition — marking the places where waiting happens

Most of what a backend or an agent does is **waiting**: waiting for a database, waiting for a web API, waiting for an LLM to generate tokens (seconds — an eternity in CPU time). During a normal (synchronous) function call, the whole program stands still while it waits. Python's `async` syntax exists to mark, *in the code itself*, the exact points where waiting occurs — so that a scheduler can let *other* work proceed during the wait. Today you only care about the marks, not the scheduler:

- **`async def`** declares a function as a **coroutine function** — a function that is *allowed to pause* partway through and resume later.
- **`await expression`** marks a pause point: "start this (typically slow, I/O-bound) operation; I can be paused here until it finishes." `await` is only legal *inside* an `async def`.
- **`asyncio.run(main())`** is the on-switch at the bottom of the file: it starts the machinery (the *event loop* — black box, Days 19–21) that actually runs coroutines.

Reading rule that covers 95% of async code you'll meet: **`await x` reads like "call x and wait for its result" — like a normal call, but pause-able.** Read past it exactly as you'd read a normal call, and note that the function containing it must say `async def`.

### 3.2 Analogy — the waiter (and where it breaks)

A synchronous program is a **waiter who takes one table's order, walks to the kitchen, and stands there staring at the stove until the dish is ready** before approaching table two. An async program is a normal waiter: take table one's order, hand it to the kitchen (`await` = "kitchen, take over — I'll be back"), and serve other tables while the food cooks. Same one waiter — just never idle. That's why async matters for servers (thousands of tables, most of them "cooking") and agents (several LLM calls "cooking" at once).

**Where the analogy breaks:** (1) it explains *why* async exists, but nothing about the *keywords* — the waiter doesn't tell you that forgetting `await` means the order never even reaches the kitchen (see §3.4); (2) the kitchen implies a second worker doing the cooking — in I/O-bound Python the "kitchen" is the network/OS, and there is still only **one** Python thread; async is one worker scheduling around waits, not parallel workers (the distinction becomes load-bearing on Day 19 — park it); (3) waiters improvise, but a coroutine can only pause at the explicit `await` marks.

### 3.3 Worked example — reading an async file top to bottom

Here is the shape of every async script you'll meet. Read it with the rules from §3.1:

```python
import asyncio

async def fetch_price(symbol: str) -> float:      # coroutine function (async def + hints)
    await asyncio.sleep(1.0)                      # pause point: stands in for a slow network call
    return 101.5

async def main() -> None:
    price = await fetch_price("AAPL")             # "call it and wait" — result is a float
    print(f"AAPL: {price}")

asyncio.run(main())                               # the on-switch, exactly once, at the bottom
```

Trace: `asyncio.run(main())` starts the machinery and runs `main`; `main` hits `await fetch_price("AAPL")`, which runs `fetch_price`; that coroutine pauses one second at its own `await` (in a real program: while bytes travel the network), resumes, returns `101.5`; `main` prints. **Sequentially, this behaves exactly like the same code without the keywords** — the payoff appears only when several coroutines wait *at the same time*, which is Day 19's story.

### 3.4 Runnable example — the coroutine-object trap (the one bug you must recognize today)

Calling an `async def` function **does not run it.** It returns a *coroutine object* — a paused, not-yet-started computation. This is the classic silent failure:

```python
# coroutine_trap.py — no install needed
import asyncio

async def main() -> int:
    print("running!")
    return 42

result = main()          # WRONG: no await, no asyncio.run — nothing runs
print(result)
```

```bash
python coroutine_trap.py
# -> <coroutine object main at 0x7f2a1c3e0d40>
# -> sys:1: RuntimeWarning: coroutine 'main' was never awaited
# -> RuntimeWarning: Enable tracemalloc to get the object allocation traceback
```

Note what's missing: **"running!" never printed.** The body never executed. Now the fix:

```python
result = asyncio.run(main())     # at top level, hand the coroutine to the machinery
print(result)
# -> running!
# -> 42
```

**Why this works, line by line.** `main()` with plain parentheses *creates* a coroutine object and stops — for an `async def`, call syntax means "package the computation," not "execute it" (the first output line is that package's `repr`, complete with a memory address). Python notices at exit that a coroutine was created but never driven, and emits `RuntimeWarning: coroutine 'main' was never awaited` — a warning, **not** an error, which is exactly what makes this bug dangerous: in a bigger program the message scrolls past, nothing crashes, and the work silently never happens (Part 3 shows this as "the agent that does nothing"). The fixed line, `asyncio.run(main())`, hands the coroutine object to the event-loop machinery, which drives it to completion and returns its value. Two rules to carry forward: **inside** an `async def`, drive coroutines with `await`; **at top level**, use `asyncio.run(...)` exactly once. If you see a `<coroutine object ...>` where you expected a value — you forgot one of those.

### 3.5 Runnable example — a taste of *why* (preview; trust the timing, mechanics on Day 19)

Marked clearly as a **preview — taken on faith until Days 19–21.** Two 1-second waits, run two ways:

```python
# taste_of_async.py — no install needed
import asyncio
import time

async def slow_task(name: str) -> None:
    await asyncio.sleep(1.0)          # stands in for a 1-second network call
    print(f"{name} done")

async def main() -> None:
    t0 = time.perf_counter()
    await slow_task("first")          # one after the other: waits add up
    await slow_task("second")
    print(f"sequential: {time.perf_counter() - t0:.1f}s")

    t0 = time.perf_counter()
    await asyncio.gather(slow_task("first"), slow_task("second"))   # preview: both at once
    print(f"concurrent: {time.perf_counter() - t0:.1f}s")

asyncio.run(main())
```

```bash
python taste_of_async.py
# -> first done
# -> second done
# -> sequential: 2.0s
# -> first done
# -> second done
# -> concurrent: 1.0s
```

**Why this works (preview-level only).** The first block awaits the tasks one at a time — the waits stack: 2.0s, no better than synchronous code. The second block hands *both* coroutines to `asyncio.gather`, which lets them wait **simultaneously** — while `first` is paused at its `await`, `second` runs up to *its* `await`, and both 1-second waits overlap: total 1.0s. Two LLM calls, ten API fetches — same shape, same win. *How* the loop interleaves them is precisely the black box we are leaving sealed until Day 19; today's takeaway is only that the `async`/`await` marks are what make this possible, which is why every serious SDK (including Anthropic's — Part 2 §4) ships an async client.

### 3.6 What to deliberately NOT learn today

Sealed boxes, listed so you know they're sealed **on purpose**: the event loop and how it schedules coroutines (Day 19); `async with` / `async for` beyond "the async-flavored `with`/`for` — same meaning, pause-able" (you'll *use* one in Part 2 on that basis); threads vs. async vs. multiprocessing (Day 20); when async actually helps and the blocking-call trap (Day 21). If you can read §3.3 aloud correctly and explain the §3.4 trap, today's async goal is met.

### 3.7 Common mistakes (recognition-level)

- **Forgetting `await`** → coroutine object instead of a result; work silently skipped; `RuntimeWarning` at exit (§3.4).
- **`await` outside `async def`** → `SyntaxError: 'await' outside async function` — at least this one fails loudly.
- **Sprinkling `async` on everything** — `async def` on a function that never awaits anything adds ceremony for zero benefit. Until Day 19, only write async where an SDK/example calls for it.

---

## 4. Everyday tools — f-strings, `enumerate`, `zip`, `with` **[WORKING]**

Four small features that appear in virtually every Python file in this course. Each gets a compact ladder; `with` gets the deepest treatment because it returns as a load-bearing pattern in Part 2 (streaming).

### 4.1 f-strings — readable string building **[WORKING]**

**What came before (term necessity):** building strings from values used `%` formatting (`"Hello, %s, you are %d" % (name, age)` — cryptic symbols, arguments matched to slots *by position*, easy to mismatch) and later `.format()` (`"Hello, {}, you are {}".format(name, age)` — better, but the value still lives far from its slot; with several values you play matching games). Both put distance between *where the value appears* and *what the value is*. **PEP 498** (f-strings, Python 3.6, 2015) removed the distance: put the expression *in the slot*.

**Intuition:** an f-string is a string with **live holes** — anything inside `{}` is evaluated as normal Python and stitched into the text.

**Worked example — the three generations on one line:**

```python
name, price = "dosa", 60.0
"Item: %s costs %.2f" % (name, price)          # 1990s: what maps to what?
"Item: {} costs {:.2f}".format(name, price)    # 2008: closer, still displaced
f"Item: {name} costs {price:.2f}"              # 2015: the value sits in its slot
```

### Runnable example — f-string features you'll actually use

```python
# fstrings.py — no install needed
name = "Vinay"
price = 1234.5678
ratio = 0.8731

print(f"Hello, {name}!")                  # plain interpolation
print(f"total: {price:.2f}")              # format spec after ':' — 2 decimal places
print(f"total: {price:,.2f}")             # thousands separator + 2 dp
print(f"progress: {ratio:.1%}")           # render a fraction as a percentage
print(f"2 + 3 = {2 + 3}")                 # any expression works
print(f"{name=}")                         # '=' debug form: prints name AND value (3.8+)
```

```bash
python fstrings.py
# -> Hello, Vinay!
# -> total: 1234.57
# -> total: 1,234.57
# -> progress: 87.3%
# -> 2 + 3 = 5
# -> name='Vinay'
```

**Why this works, line by line.** The `f` prefix tells Python to scan the literal for `{...}` holes and evaluate each as an ordinary expression in the current scope — `{name}` is a variable lookup, `{2 + 3}` is arithmetic, anything goes. A colon inside the hole starts a **format spec**, a mini-language applied to the value after evaluation: `.2f` = fixed-point with 2 decimals (note it *rounds*: `…5678` → `.57`), `,` inserts thousands separators, `.1%` multiplies by 100 and appends `%`. The `{name=}` form prints the *expression text*, an `=`, then the value — the fastest debugging print in Python. **Common confusion:** forgetting the `f` prefix — `"Hello, {name}"` prints the braces literally, no error, which is a classic silent bug.

### 4.2 `enumerate` — index and value together **[WORKING]**

**What came before (term necessity):** looping with a counter meant `for i in range(len(items)):` then `items[i]` inside — an anti-pattern that's noisier, easy to off-by-one, and only works on things with `len()`. `enumerate` exists because "I'm iterating AND I need the position" is so common it deserved one word.

**Intuition:** `enumerate(items)` yields `(index, value)` pairs — the loop you were writing, minus the bookkeeping.

### Runnable example — numbering lines the clean way

```python
# enum_demo.py — no install needed
steps = ["create venv", "activate", "pip install requests"]

for i in range(len(steps)):            # the old way — works, but noisy
    print(i, steps[i])

for i, step in enumerate(steps, start=1):   # the modern way, counting from 1
    print(f"{i}. {step}")
```

```bash
python enum_demo.py
# -> 0 create venv
# -> 1 activate
# -> 2 pip install requests
# -> 1. create venv
# -> 2. activate
# -> 3. pip install requests
```

**Why this works.** `enumerate(steps, start=1)` produces the pairs `(1, "create venv")`, `(2, "activate")`, `(3, "pip install requests")`, and the loop head `for i, step in ...` unpacks each pair into two variables in one motion (the tuple-unpacking you met on P2). `start=` only changes the *numbering*, not which elements you get — indexing for humans (1-based) without touching the data. Notice the old way needed three references to `steps` and manual indexing; the new way needs one and can't be off by one.

### 4.3 `zip` — iterate two (or more) sequences in lockstep **[WORKING]**

**What came before:** the same `range(len(...))` dance, but indexing *two* lists — twice the noise, twice the off-by-one surface. `zip` pairs sequences element-by-element, like the two sides of a zipper meshing tooth by tooth (that's the name).

### Runnable example — pairing, dict-building, and the shortest-wins trap

```python
# zip_demo.py — no install needed
names = ["idli", "dosa", "vada"]
prices = [40.0, 60.0, 30.0]

for name, price in zip(names, prices):
    print(f"{name}: ₹{price:.0f}")

menu: dict[str, float] = dict(zip(names, prices))     # pairs → dict in one call
print(menu)

print(list(zip(names, [40.0, 60.0])))                 # TRAP: stops at the SHORTEST input
list(zip(names, [40.0, 60.0], strict=True))           # 3.10+: make the mismatch loud
```

```bash
python zip_demo.py
# -> idli: ₹40
# -> dosa: ₹60
# -> vada: ₹30
# -> {'idli': 40.0, 'dosa': 60.0, 'vada': 30.0}
# -> [('idli', 40.0), ('dosa', 60.0)]
# -> Traceback (most recent call last):
# ->   File "zip_demo.py", line 12, in <module>
# -> ValueError: zip() argument 2 is shorter than argument 1
```

**Why this works, line by line.** `zip(names, prices)` yields `("idli", 40.0)`, `("dosa", 60.0)`, `("vada", 30.0)` — one tuple per step, unpacked in the loop head exactly like `enumerate`'s pairs. `dict(zip(...))` works because `dict()` accepts any sequence of 2-tuples as key–value pairs — the standard idiom for welding two parallel lists into a mapping. The trap line: with a 3-item and a 2-item input, plain `zip` **silently drops** `"vada"` — it stops at the shortest input, and silent data loss is the worst kind. `strict=True` (Python 3.10+) converts that silence into an immediate `ValueError` naming which argument fell short — prefer it whenever the sequences are *supposed* to be equal length.

### 4.4 `with` — context managers: guaranteed cleanup **[WORKING]**

**Depth: [WORKING]**, but with the protocol opened one level, because this construct carries real weight later today (Part 2: streaming) and all course long (files, DB connections/transactions, HTTP clients, locks).

**The problem (term necessity).** Some objects are **resources**: they hold something outside your process — an open file handle from the OS, a network socket, a database connection — and *must be given back*. Before `with` (PEP 343, 2005), correct code required the `try`/`finally` ritual:

```python
f = open("report.txt", "w")
try:
    f.write("hello")          # if THIS raises...
finally:
    f.close()                 # ...this still runs. Forget the finally → leaked handle.
```

**Worked example — what forgetting cleanup actually costs.** A script that opens files in a loop without closing them: each `open()` consumes an OS file handle; handles are a finite per-process resource (often 1024 on Linux); at iteration 1025 → `OSError: [Errno 24] Too many open files` — a crash whose cause (the missing `close()` at the top) is nowhere near the symptom. Worse, on some platforms unclosed writes aren't flushed, so the file is silently *incomplete*. The failure category: **cleanup that depends on a human remembering, under all exit paths (normal, `return`, exception), will eventually be forgotten.**

**Intuition:** `with` moves the guarantee from the human to the language. "For this indented block, this resource is open; when the block exits — *however* it exits — Python performs the cleanup."

**Analogy — the hotel keycard.** Checking in gives you a keycard (acquire); checkout takes it back (release); and the hotel's system auto-expires it at checkout time *even if you storm out mid-stay without visiting the desk* (exception path covered). **Where it breaks:** a keycard expires on a *clock*; `with` releases on *block exit*, deterministic and immediate — and unlike a hotel, `with` can't be bypassed by "staying an extra night": there is no way to exit the block with the resource still held.

### Runnable example — files with `with`, and proof the guarantee holds under exceptions

```python
# with_demo.py — no install needed
with open("report.txt", "w") as f:      # acquire: open the file, bind it to f
    f.write("day P3: venv + types + async\n")
print(f.closed)                          # after the block: is it really closed?

with open("report.txt") as f:
    print(f.read(), end="")

try:                                     # the guarantee under fire:
    with open("report.txt") as f:
        raise ValueError("boom mid-block")
except ValueError:
    print(f"closed even after the exception? {f.closed}")
```

```bash
python with_demo.py
# -> True
# -> day P3: venv + types + async
# -> closed even after the exception? True
```

**Why this works, line by line.** `with open(...) as f:` acquires the resource and binds it to `f`; when the first block ends, Python closes the file *for you* — the first `True` proves it (`f` still exists as a variable; only the underlying handle is released, which is also why writes are guaranteed flushed before anyone reads). The third block is the part `try`/`finally` was invented for: an exception detonates *mid-block*, control flies toward the `except` — and on the way out, `with` **still runs the cleanup**. `f.closed` is `True` even on the exception path. That is the whole contract: cleanup on every exit, no human memory involved. Note `with` does **not** swallow the exception — it cleans up and lets the error propagate (hence our `except` catching it).

**Under the hood — the protocol (one level open, then sealed).** `with EXPR as name:` is not file-specific magic; it drives two dunder methods on the object: `EXPR.__enter__()` runs at block entry (its return value is bound to `name`), and `EXPR.__exit__(...)` runs at block exit — *always*, exception or not, receiving the exception details if there was one. Any object implementing that pair is a **context manager**, which is why the same syntax later manages DB transactions (`__exit__` commits or rolls back), locks (`__exit__` releases), and LLM streams (`__exit__` closes the network connection — Part 2 §5). Ten-line proof you can run:

```python
# timer_cm.py — a minimal custom context manager
import time

class Timer:
    def __enter__(self) -> "Timer":
        self.t0 = time.perf_counter()
        return self                                   # ← bound to the 'as' name
    def __exit__(self, exc_type, exc, tb) -> None:    # ← ALWAYS runs on block exit
        print(f"block took {time.perf_counter() - self.t0:.2f}s")

with Timer():
    total = sum(range(10_000_000))
```

```bash
python timer_cm.py
# -> block took 0.16s
```

**Why this works.** `with Timer():` calls `__enter__` (clock starts), runs the block, then calls `__exit__` (clock stops, prints) — the acquire/release pattern with *your* code in the two slots. The three `__exit__` parameters carry exception info when one occurred (all `None` otherwise); handling them properly, and the `@contextmanager` decorator shortcut, are refinements you can pick up when a project forces you — the sealed edge of [WORKING].

---

## Part 1 — interview & practice questions

**Interview-style (answer aloud, from memory):**

1. What does Python do with type hints at runtime? Then where does their value come from? Name the two *kinds* of consumers (static vs. runtime) and one example of each.
2. `400` question of P-days: a colleague says "I installed requests but `import requests` fails." Give the two most likely causes and the one-line diagnostic for each.
3. What is the difference between `mypy` catching a bug and the interpreter catching it? Where does each report the error relative to the *cause*?
4. What does calling an `async def` function *without* `await` return, and what tells you it happened?
5. Why is `with` more than "shorter try/finally"? What guarantee does it give on the exception path, and which two methods implement it?

**Practice — Easy:** annotate `def area(w, h): return w * h` fully; predict what `list(zip("ab", [1, 2, 3]))` returns. · **Medium:** write `parse_price(text: str) -> float | None` that returns `None` on bad input, and explain why the return hint forces callers to be safer. · **Hard:** your teammate's script printed `<coroutine object fetch at 0x...>` into the database instead of the fetched value, and nothing crashed. Explain the exact chain of events and the two possible one-line fixes.

---

# PART 2 — AGENTIC AI: The Same Four Constructs in Real LLM Code

> **What this Part is — and isn't.** No new fundamentals here. Part 2's only job is to show you that everything you just learned is *load-bearing* in the agentic code you'll write for the rest of this course: the Anthropic SDK **lives in a venv** and gets **pinned**; SDK calls are **fully type-hinted** (that's why your editor autocompletes them); a **tool schema** is the same contract idea as a type hint; `AsyncAnthropic` uses **exactly the `async def`/`await` syntax** from §3; and **`with`** manages streaming connections. Backend mechanics (HTTP, servers) stay a black box — cross-reference Part 1 and later days.
>
> **Faith notes.** Anything about the API itself — what a model ID is, what `max_tokens` bounds, what a `tool_use` stop reason means — is **preview material, taken on faith until Days 19/22** when the SDK and tool-use loop are taught properly. Today you're reading this code for its *Python*, not its *AI*. Also: these examples call a paid API — you need an API key (`console.anthropic.com`) exported as `ANTHROPIC_API_KEY`, each run costs a fraction of a cent, and model IDs/SDK details drift fast — *verify against the official docs before relying on them*.

## Overview — your first look at the code you'll write for 100 days

Here is the point of Part 2 in one observation: **an "AI agent" is an ordinary Python program.** It lives in a venv, imports a pip-installed package, calls typed functions, awaits network responses, and manages connections with `with`. There is no separate "AI Python." So the four P3 constructs aren't prerequisites *for* the agentic work — they're the material it's *made of*. Each section below takes one Part 1 concept and shows it operating, unchanged, in real Anthropic SDK code.

## 1. The SDK lives in a venv — isolation & pinning applied (from §2)

**Intuition.** `anthropic` is just a package on PyPI — exactly like `requests`. Everything from §2 applies verbatim, with one amplifier: **LLM SDKs move fast.** Parameters get added and *removed* between versions (real example you'll meet on Day 22: sampling parameters like `temperature` that older tutorials show are **rejected with an error** on current Claude models). An unpinned SDK version means code that worked in January can throw `TypeError: unexpected keyword argument` in March — dependency hell across time, precisely the §2.6 disease.

### Runnable example — installing and pinning the Anthropic SDK

```bash
# inside your project, venv active — the (.venv) prefix is your seatbelt check
source .venv/bin/activate

pip install anthropic
# -> Collecting anthropic
# ->   Downloading anthropic-0.116.0-py3-none-any.whl (1.2 MB)
# -> Collecting httpx<1,>=0.25.0 (from anthropic)
# -> Collecting pydantic<3,>=1.9.0 (from anthropic)
# -> Collecting typing-extensions<5,>=4.10 (from anthropic)
# -> ...
# -> Successfully installed anthropic-0.116.0 httpx-0.27.2 pydantic-2.9.2 ...
#    (versions drift — yours will be newer)

pip show anthropic
# -> Name: anthropic
# -> Version: 0.116.0
# -> Location: /home/vinay/p3/.venv/lib/python3.12/site-packages
# -> Requires: anyio, distro, httpx, jiter, pydantic, sniffio, typing-extensions

pip freeze > requirements.txt        # pin it — future installs get THIS version
grep anthropic requirements.txt
# -> anthropic==0.116.0
```

**Why this works, line by line.** `pip install anthropic` resolves the SDK's dependency closure into *this venv's* `site-packages` — and look at what rode in: **`pydantic`**. The SDK itself depends on the validation library this course teaches on Day 8; the response objects you'll touch in §2 are Pydantic models built from type hints — Part 1 §1 and §2 are literally in the wheel you just installed. `pip show`'s `Location:` line confirms the install landed in the venv (your forgot-to-activate detector from §2.7). `pip freeze > requirements.txt` pins `anthropic==0.116.0`, so when this project is rebuilt in six months, it gets the SDK surface your code was written against — not whatever the newest release renamed. For fast-moving SDKs, the pin isn't hygiene; it's what keeps your agent runnable.

## 2. SDK calls are fully type-hinted — contracts applied (from §1)

**Intuition.** The Anthropic SDK is a *typed* library: every function you call and every object you get back carries hints (`py.typed` — the marker that tells checkers "this package's hints are official"). Consequence you can feel immediately: type `response.` in your editor and it lists `content`, `usage`, `stop_reason`, ... — the editor isn't guessing; it's **reading the SDK's type hints**, the same `__annotations__` machinery from §1.8. Your own code should hold up its end of the contract.

### Runnable example — a fully type-hinted first Claude call (+ mypy pass)

```python
# ask_claude.py
# pip install anthropic
# export ANTHROPIC_API_KEY=sk-ant-...      (from console.anthropic.com; costs pennies)
import anthropic
from anthropic.types import Message

client = anthropic.Anthropic()      # reads ANTHROPIC_API_KEY from the environment


def ask_claude(question: str) -> str:
    """Send one question to Claude; return the text of its answer."""
    response: Message = client.messages.create(
        model="claude-opus-4-8",    # preview/faith: current model ID — verify, these drift
        max_tokens=1024,            # preview/faith: cap on the answer's length in tokens
        messages=[{"role": "user", "content": question}],
    )
    # response.content is a LIST of typed blocks; keep only the text ones (§1: contracts!)
    parts: list[str] = [block.text for block in response.content if block.type == "text"]
    return "".join(parts)


if __name__ == "__main__":
    print(ask_claude("In one sentence: what is a virtual environment in Python?"))
```

```bash
python ask_claude.py
# -> A virtual environment is an isolated directory containing its own Python interpreter
#    and packages, so each project's dependencies stay separate from every other project's.
#    (an LLM generates fresh text — your wording WILL differ)

mypy ask_claude.py
# -> Success: no issues found in 1 source file
```

**Why this works, line by line.** `anthropic.Anthropic()` builds a client, pulling the API key from the environment — keys never belong in source code (they'd end up in git; Day P4 makes this concrete). The function signature `ask_claude(question: str) -> str` is a §1 contract: callers know its shape without reading the body. The `response: Message` annotation names the SDK's typed response object — hover it in your editor and the full structure unfolds, courtesy of the SDK's hints. The list comprehension is a *typed-contract habit in action*: `response.content` is a list of blocks of **several possible types** (text, tool-use, thinking, ...), so we check `block.type == "text"` before touching `block.text` — exactly the §1.4 "maybe-shaped data: handle the cases" discipline (touching `.text` on a non-text block would be the runtime error hints exist to prevent, and a type checker flags the unguarded version). Finally, `mypy` passes over the whole file — *including the SDK calls* — because both sides of the boundary are typed: your contract meets theirs, and the checker verifies the handshake. **Honesty caveat:** without a valid key the script raises `anthropic.AuthenticationError` — a clean, typed exception, not a mystery; and the output line shown is illustrative — LLMs don't produce byte-identical answers twice.

## 3. Tool schemas — the same "contract" idea, one layer up (from §1, preview of Pydantic)

**Intuition.** On Day 22 you'll teach Claude to *call functions you wrote* ("tool use" — the mechanical heart of every agent). How do you tell a model what a function accepts? **You send it a contract.** Look at the two artifacts side by side:

```
your Python (for mypy/humans):            the tool schema you send the API (for the model):
                                          {
def get_weather(                            "name": "get_weather",
    city: str,                              "input_schema": {
) -> str:                                     "type": "object",
    ...                                       "properties": {
                                                "city": {"type": "string"}
                                              },
                                              "required": ["city"]
                                          } }
```

Same information, two notations: `city: str` *is* `"city": {"type": "string"}`; a parameter without a default *is* `"required"`. Type hints are contracts read by mypy; **input schemas are contracts read by a model** (JSON Schema — a language-neutral way to describe data shapes). And the bridge between the notations is exactly **Pydantic** (Day 8): define the shape *once* as a type-hinted class, and it can both validate data at runtime *and* emit the JSON Schema. One contract, three consumers — checker, validator, model. That's why this course hammers type hints on day P3.

### Runnable example — sending a tool contract and watching the model honor it (preview)

**Preview — taken on faith until Day 22.** We send one tool definition and *only look at what comes back*; actually executing tools and looping is Day 22's manual tool loop.

```python
# tool_contract_preview.py
# pip install anthropic     (ANTHROPIC_API_KEY must be set)
import anthropic

client = anthropic.Anthropic()

TOOLS = [{
    "name": "get_weather",
    "description": "Get the current weather for a city. Use for any question "
                   "about temperature, rain, or what to wear.",
    "input_schema": {                                  # ← the contract, as JSON Schema
        "type": "object",
        "properties": {
            "city": {"type": "string", "description": "City name, e.g. 'Hyderabad'"},
        },
        "required": ["city"],
    },
}]

response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    tools=TOOLS,
    messages=[{"role": "user", "content": "Do I need an umbrella in Hyderabad today?"}],
)

print(response.stop_reason)                 # why did the model stop?
for block in response.content:
    if block.type == "tool_use":            # same typed-block discipline as §2
        print(block.name, block.input)
```

```bash
python tool_contract_preview.py
# -> tool_use
# -> get_weather {'city': 'Hyderabad'}
```

**Why this works, line by line.** `TOOLS` carries one tool with the three parts of the contract: a `name` (the handle), a `description` (when to use it), and the `input_schema` — the JSON-Schema twin of `def get_weather(city: str)`. The model reads the question, decides the weather tool applies, and *stops mid-answer* to ask for it: `stop_reason == "tool_use"` means "I'm not answering yet — run this function and give me the result." The `tool_use` block it emits contains `input` = `{'city': 'Hyderabad'}` — **arguments that conform to the schema you declared**: a JSON object, a string `city`, required field present. The model filled in your function's parameters the way a well-behaved caller satisfies a type-hinted signature. What we *don't* do today — execute the function, send back the result, loop until done — is exactly the Day-22 material; note only that in a real agent your code should still *validate* `block.input` before executing (models can produce well-typed but wrong values — a real city name isn't something a schema can check), and the validator will be Pydantic: hints, enforced at runtime. The circle closes.

## 4. `AsyncAnthropic` — the §3 syntax, verbatim (from §3)

**Intuition.** The SDK ships two clients: `Anthropic` (synchronous — every call blocks until the answer arrives, like the waiter staring at the stove) and `AsyncAnthropic` (the async twin). The async client's methods are awaitable — and the code is *character-for-character* the pattern from §3.3: `async def`, `await`, `asyncio.run`. Why an agent will care (preview, Days 19–21): agents wait on LLM calls that take *seconds*; the §3.5 timing demo with `asyncio.sleep(1)` becomes real money and real minutes when the sleep is a model generating tokens — five research queries concurrently instead of serially is the same `gather` shape.

### Runnable example — the same call, async

```python
# ask_claude_async.py
# pip install anthropic     (ANTHROPIC_API_KEY must be set)
import asyncio

import anthropic

client = anthropic.AsyncAnthropic()          # the async twin of Anthropic()


async def ask(question: str) -> str:         # §3: async def → coroutine function
    response = await client.messages.create( # §3: await = "call it and wait, pause-ably"
        model="claude-opus-4-8",
        max_tokens=1024,
        messages=[{"role": "user", "content": question}],
    )
    return "".join(block.text for block in response.content if block.type == "text")


async def main() -> None:
    answer = await ask("In one sentence: what does 'await' do in Python?")
    print(answer)


asyncio.run(main())                           # §3: the on-switch, once, at the bottom
```

```bash
python ask_claude_async.py
# -> 'await' pauses the current coroutine until the awaited operation completes, letting
#    other tasks run in the meantime.    (your wording will differ)
```

**Why this works, line by line.** Diff this against §2's `ask_claude.py` and only four tokens changed: `AsyncAnthropic` for `Anthropic`, `async def` on the functions, `await` before the SDK call, `asyncio.run` at the bottom. Same arguments, same typed response, same block filtering. That's the entire lesson: **async agentic code is not a different language — it's the same calls with the §3 pause-marks added.** And the §3.4 trap applies with teeth: drop the `await` before `client.messages.create(...)` and you get a coroutine object instead of a response — no crash, a `RuntimeWarning` at exit, and an agent that *silently never called the model*. Recognizing that failure shape is precisely why async syntax is on today's menu.

## 5. `with` manages streaming — context managers applied (from §4.4)

**Intuition.** By default the API behaves like a letter: the full answer arrives at once, after the whole generation finishes. **Streaming** delivers it like a phone call — text flows in as it's generated (this is how every chat UI shows the answer "typing itself"). A stream is an **open network connection**: a resource, exactly in the §4.4 sense — it must be closed whether your code finishes cleanly or blows up mid-read. So the SDK hands it to you as a **context manager**: `with client.messages.stream(...) as stream:`. Files taught you the pattern this morning; here is the pattern earning its keep tonight.

### Runnable example — streaming inside `with` (sync)

```python
# stream_claude.py
# pip install anthropic     (ANTHROPIC_API_KEY must be set)
import anthropic

client = anthropic.Anthropic()

with client.messages.stream(                      # §4.4: acquire the connection...
    model="claude-opus-4-8",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Count from 1 to 5, one number per line."}],
) as stream:
    for text in stream.text_stream:               # ...consume chunks as they arrive...
        print(text, end="", flush=True)
print()                                            # ...connection closed by __exit__ here
```

```bash
python stream_claude.py
# -> 1
# -> 2
# -> 3
# -> 4
# -> 5
#    (the lines APPEAR ONE BY ONE as the model generates them — run it to feel it)
```

**Why this works, line by line.** `client.messages.stream(...)` opens the request and returns a context manager; `with ... as stream:` runs its `__enter__` (connection live) and binds the stream object. `stream.text_stream` is an iterator of text *fragments* — the loop prints each the instant it arrives, and `flush=True` (from §4.1's print family) forces it onto the screen immediately instead of sitting in an output buffer, which is what produces the typing effect. When the block exits — the stream ends, *or* you `Ctrl-C`, *or* an exception fires mid-generation — `__exit__` closes the network connection, every path, guaranteed: the §4.4 contract, verbatim, with a socket where the file used to be. **Honesty caveat:** streaming isn't cosmetic — for long answers the SDK *expects* you to stream (very large responses over a plain blocking call can hit HTTP timeouts); the mechanics of why live in later days.

### Runnable example — `async with`: sections 3 and 4.4 in one line (the course's signature pattern)

```python
# stream_claude_async.py
# pip install anthropic     (ANTHROPIC_API_KEY must be set)
import asyncio

import anthropic

client = anthropic.AsyncAnthropic()


async def stream_answer(question: str) -> None:
    async with client.messages.stream(            # async-flavored with: same guarantee,
        model="claude-opus-4-8",                  # pause-able acquire/release (§3.6)
        max_tokens=1024,
        messages=[{"role": "user", "content": question}],
    ) as stream:
        async for text in stream.text_stream:     # async-flavored for: pause-able loop
            print(text, end="", flush=True)
    print()


asyncio.run(stream_answer("In two short lines, why do Python projects use virtual environments?"))
```

```bash
python stream_claude_async.py
# -> Each project gets its own isolated set of packages,
# -> so upgrading one project can never break another.
#    (streams in live; your wording will differ)
```

**Why this works, line by line.** Read it with the two P3 decoding rules: `async with` **is** `with` — acquire, guaranteed release on every exit — just allowed to *pause* at the acquire/release points instead of blocking; `async for` **is** `for` — just allowed to pause while waiting for the next chunk to arrive over the network (that waiting is precisely where other coroutines would get to run — Day 19). Everything else is the sync version unchanged. This one function is the densest artifact of Day P3: a **venv-installed, pinned** SDK (§1), a **type-hinted contract** (§2), the **async syntax** (§4), and a **context-managed stream** (§5) in eleven lines. When you can read it aloud and say what every keyword is doing, Part 2 is complete.

## Part 2 — interview & practice questions

1. Why must the `anthropic` package version be *pinned*, and what §2 file records the pin? What failure appears when it isn't?
2. Your editor autocompletes `response.usage.` — what mechanism, from Part 1, makes that possible?
3. Explain in two sentences how a tool's `input_schema` and a type-hinted function signature are "the same idea," and name the Day-8 library that converts between them.
4. In `ask_claude_async.py`, exactly which tokens differ from the sync version? What happens if the `await` before `client.messages.create` is deleted — what prints, what warns, what *doesn't* happen?
5. Why does the SDK expose streaming as a context manager rather than a plain function returning chunks? What guarantee would be lost otherwise, and on which exit path?

---

# PART 3 — THE BRIDGE: one discipline, two layers

> Everything here references only Parts 1 and 2 — no new concepts. The payoff of the framing note: **contracts + isolation are one discipline that both layers share.** Part 1 taught the discipline in plain Python; Part 2 showed it wearing agentic clothes; this Part draws the dependency map between them and shows how they fail *together*.

## The dependency map — what sits on what

Your Day-P3 stack, bottom to top, with the P3 construct that secures each seam:

```
┌────────────────────────────────────────────────────────────────┐
│  YOUR SCRIPT           ask_claude("...")                       │
│    · type-hinted signatures  → mypy checks YOUR contracts (§1) │
│    · async def / await       → marks the waiting points (§3)   │
│    · with ... stream         → guarantees connection cleanup (§4.4)
├────────────────────────────────────────────────────────────────┤
│  THE SDK               anthropic==0.116.0                      │
│    · installed & pinned in .venv/…/site-packages (§2, P2 §1)   │
│    · fully typed → your editor/mypy check the HANDSHAKE (P2 §2)│
│    · itself built on pydantic → hints enforced at runtime      │
├────────────────────────────────────────────────────────────────┤
│  THE NETWORK / API      (black box until Phase 1)              │
│    · tool input_schema → the contract the MODEL reads (P2 §3)  │
│    · streaming = long-lived connection → the resource `with`   │
│      was releasing (P2 §5)                                     │
└────────────────────────────────────────────────────────────────┘
```

Read the seams, not the boxes: **every layer boundary is protected by either a contract or an isolation mechanism you learned today.** Your code ↔ SDK: type hints on both sides, checked by mypy. Project ↔ machine: the venv wall plus the `requirements.txt` pin. Code ↔ network resource: the `with` guarantee. Code ↔ model: the JSON-Schema contract. Same discipline, four dress codes.

## The construct-by-construct bridge table

| P3 construct | In backend code (Part 1) | In agentic code (Part 2) | What breaks without it |
|---|---|---|---|
| Type hints **[CORE]** | function contracts; mypy pre-run checks (§1.7) | typed SDK → autocomplete + checked calls (P2 §2); the *idea* behind tool schemas (P2 §3) | arg bugs surface at runtime, deep inside a paid multi-step run instead of in the editor |
| venv + pip **[CORE]** | per-project isolation; `requirements.txt` reproducibility (§2.5–2.6) | `anthropic` pinned per project; SDK drift contained (P2 §1) | "works on my machine"; SDK update silently breaks agent code that worked last month |
| `async`/`await` **[WORKING]** | syntax recognition; the coroutine-object trap (§3.4) | `AsyncAnthropic` — same syntax verbatim (P2 §4) | forgotten `await` → the agent *silently never calls the model* |
| `with` **[WORKING]** | files closed on every exit path (§4.4) | streaming connections closed on every exit path (P2 §5) | leaked handles/connections; crashes far from their cause |
| f-strings / `enumerate` / `zip` **[WORKING]** | everyday string-building & iteration (§4.1–4.3) | prompt construction is *string-building* (f-strings interpolating context into prompts); numbering steps, pairing questions↔answers | clumsy, off-by-one prompt/plumbing code everywhere |

## How they fail together — three coupled-failure traces

Failures at one layer surface as mysteries at the other. Three miniatures, each traced with the P3 vocabulary:

**Trace 1 — the unpinned SDK (venv discipline ⊗ agent layer).**
`t0` Agent works; `requirements.txt` was never written. `t1` Months later, new laptop: `pip install anthropic` grabs the newest release, whose surface changed. `t2` `python agent.py` → `TypeError: Messages.create() got an unexpected keyword argument '...'`. `t3` The *agent* looks broken; the *environment* is what changed. The §2.6 pin is the fix, and `pip show anthropic` (§2.7) is the diagnostic. *Lesson: reproducibility failures at the bottom layer masquerade as code bugs at the top.*

**Trace 2 — the agent that does nothing (async trap ⊗ agent layer).**
`t0` Refactor to `AsyncAnthropic`; one call site keeps its old form: `response = client.messages.create(...)` — no `await`. `t1` Run: no crash; the log shows `<coroutine object create at 0x...>` where a response should be; at exit, one line scrolls past: `RuntimeWarning: coroutine ... was never awaited`. `t2` The model was **never called** — no request, no tokens billed, no answer, and nothing red. This is §3.4 wearing production clothes, and it's why P3 insists you *recognize* the coroutine-object shape on sight. *Lesson: the async syntax's failure mode is silence — the most expensive kind in an autonomous system.*

**Trace 3 — the typo that waited twenty steps (contracts ⊗ agent layer).**
`t0` An untyped helper passes around plain dicts: `result["punchlin"]` — typo'd key, written on step 1 of an agent's plan, only *read* on step 20. `t1` Nineteen paid LLM calls succeed; step 20 raises `KeyError` and the whole run is garbage. `t2` With the §1 discipline — typed structures instead of loose dicts (and, from Day 8, Pydantic models) — mypy flags the typo *in the editor*, before any run and any spend. *Lesson: the earlier a contract is checked, the cheaper its violation — and in agentic systems, "runtime" costs actual money.*

## Capstone runnable — all four constructs in one 25-line artifact

Everything Day P3 taught, in one file you can run and then explain line-by-line as your teach-back (see wrap-up):

```python
# p3_capstone.py — venv-pinned (§2), type-hinted (§1), async (§3), context-managed (§4.4)
# pip install anthropic     (ANTHROPIC_API_KEY must be set)
import asyncio

import anthropic

client = anthropic.AsyncAnthropic()


async def stream_and_measure(question: str) -> int:
    """Stream Claude's answer to the terminal; return how many output tokens it used."""
    async with client.messages.stream(
        model="claude-opus-4-8",
        max_tokens=1024,
        messages=[{"role": "user", "content": question}],
    ) as stream:
        async for text in stream.text_stream:
            print(text, end="", flush=True)
        final = await stream.get_final_message()      # the complete typed response object
    print()
    return final.usage.output_tokens                  # typed all the way down (§1 ↔ P2 §2)


async def main() -> None:
    questions: list[str] = [
        "One sentence: what does a type hint promise?",
        "One sentence: what does a venv isolate?",
    ]
    for i, q in enumerate(questions, start=1):        # §4.2
        print(f"--- Q{i}: {q}")                       # §4.1
        tokens = await stream_and_measure(q)          # §3
        print(f"(answer used {tokens} output tokens)")


asyncio.run(main())
```

```bash
python p3_capstone.py
# -> --- Q1: One sentence: what does a type hint promise?
# -> A type hint declares what types a function accepts and returns so tools and
#    readers can rely on that contract.
# -> (answer used 24 output tokens)
# -> --- Q2: One sentence: what does a venv isolate?
# -> A virtual environment isolates a project's installed packages and interpreter
#    from every other project on the machine.
# -> (answer used 26 output tokens)
#    (wording and token counts will differ run to run)
```

**Why this works, line by line.** The client is the async twin (P2 §4); `stream_and_measure` is a §1 contract (`str -> int` — callers know it returns a token count without reading the body); `async with` acquires the streaming connection with the §4.4 guarantee, `async for` drains it chunk-by-chunk, and `await stream.get_final_message()` collects the complete **typed** response after streaming ends — so `final.usage.output_tokens` autocompletes in your editor and type-checks under mypy (P2 §2), giving you the day's first taste of *cost observability* (tokens are what you pay for). The driver loop is pure §4: `enumerate(..., start=1)` numbers the questions, f-strings assemble the output. Every construct on today's menu is load-bearing in these 25 lines — and this exact shape (typed async function, streamed model call, usage accounting) is the skeleton that Phase 1's agents grow around.

---

# Wrap-up — the whole day in your pocket

## Cheat Sheet

```
TYPE HINTS (§1) — contracts; IGNORED at runtime; checked by tools
  def f(x: int, y: float = 1.0) -> str:          params, default, return
  n: int = 3                                     variable
  list[int]  dict[str, float]  tuple[int, str]   collections
  X | None   (older: Optional[X])                maybe-missing → callers must handle
  Any                                            deliberate hole (boundaries only)
  mypy file.py                                   check without running
  f.__annotations__                              where hints are stored
  static check = mypy/editor (pre-run) · runtime check = Pydantic (Day 8)

VENV + PIP (§2) — one project, one site-packages
  python3 -m venv .venv                          create (once)
  source .venv/bin/activate                      activate (each shell)   [Win: .venv\Scripts\Activate.ps1]
  which python · pip show PKG (Location:)        am-I-in-the-venv checks
  pip install PKG / PKG==1.2.3 / -r requirements.txt
  pip freeze > requirements.txt                  pin → commit; .venv/ → gitignore
  deactivate · delete .venv/ = clean uninstall
  activation = PATH edit; interpreter finds site-packages via its own location
  uv = faster all-in-one replacement — Day 7 [AWARE]

ASYNC SYNTAX (§3) — recognition only; mechanics Days 19–21
  async def f(): ...        can pause; CALLING it returns a coroutine object
  await expr                "call and wait, pause-ably" — only inside async def
  asyncio.run(main())       the on-switch, once, at the bottom
  <coroutine object ...> + "never awaited" warning  →  you forgot await
  async with / async for    = with / for, pause-able

EVERYDAY (§4)
  f"{x:.2f} {n:,} {r:.1%} {var=}"                f-string + format specs
  for i, v in enumerate(xs, start=1):            index + value
  zip(a, b)  · dict(zip(a, b)) · strict=True     lockstep; shortest wins unless strict
  with open(p) as f: ...                         cleanup on EVERY exit (__enter__/__exit__)

AGENTIC MIRROR (Part 2)
  pip install anthropic → pin it                 §2 applied (SDKs drift fast)
  typed SDK → autocomplete + mypy on YOUR calls  §1 applied
  input_schema ≙ type-hinted signature           contract for the model (Pydantic bridges)
  AsyncAnthropic + await client.messages.create  §3 applied, verbatim
  (async) with client.messages.stream(...)       §4.4 applied — stream = resource
```

## Build This — Day P3 (the study plan's build task)

**Task:** set up a virtual environment, `pip install requests`, and write a **fully type-hinted** script that fetches a public API and prints a field from the JSON. Run `mypy` on it and fix every complaint — your first taste of the Day 7 toolchain.

**Step-by-step with the target script:**

```bash
mkdir p3-build && cd p3-build
python3 -m venv .venv && source .venv/bin/activate
pip install requests mypy
pip freeze > requirements.txt
```

```python
# p3_fetch.py — fetch a random joke and print its fields. Fully type-hinted.
from typing import Any

import requests

JOKE_URL = "https://official-joke-api.appspot.com/random_joke"


def fetch_json(url: str, timeout_seconds: float = 10.0) -> dict[str, Any]:
    """GET a URL and return the parsed JSON body as a dict."""
    response = requests.get(url, timeout=timeout_seconds)
    response.raise_for_status()          # turn HTTP errors (404, 500, ...) into exceptions
    data: dict[str, Any] = response.json()
    return data


def format_joke(joke: dict[str, Any]) -> str:
    setup: str = joke["setup"]
    punchline: str = joke["punchline"]
    return f"{setup}\n... {punchline}"


def main() -> None:
    joke = fetch_json(JOKE_URL)
    print(format_joke(joke))


if __name__ == "__main__":
    main()
```

```bash
python p3_fetch.py
# -> Why did the scarecrow win an award?
# -> ... Because he was outstanding in his field.
#    (random joke — yours will differ; the JSON has keys: type, setup, punchline, id)
```

**Now the mypy part — including the real first complaint everyone hits, and its fix:**

```bash
mypy p3_fetch.py
# -> p3_fetch.py:4: error: Library stubs not installed for "requests"  [import-untyped]
# -> p3_fetch.py:4: note: Hint: "python3 -m pip install types-requests"
# -> p3_fetch.py:4: note: (or run "mypy --install-types" to install all missing stub packages)
# -> Found 1 error in 1 file (checked 1 source file)
```

This is a *real* error you must understand, not a nuisance: `requests` predates type hints and doesn't ship its own — the community publishes its contracts separately as a **stub package** (hints-only, no code). Install the contracts, and mypy can check your usage of the library:

```bash
pip install types-requests
# -> Successfully installed types-requests-2.32.0.20241016 ...
mypy p3_fetch.py
# -> Success: no issues found in 1 source file
```

**Then prove the checker earns its keep — plant a bug, watch it get caught, fix it:**

```python
# temporarily change main() to:
    joke = fetch_json(JOKE_URL)
    print(format_joke(joke["setup"]))     # BUG: passing a str where the contract says dict
```

```bash
mypy p3_fetch.py
# -> p3_fetch.py:24: error: Argument 1 to "format_joke" has incompatible type "Any";
#    expected "dict[str, Any]"  [arg-type]
# -> Found 1 error in 1 file (checked 1 source file)
```

Revert the line; `mypy` → `Success` again.

**Definition of done (all runnable, all checkable):**

- [ ] `which python` prints a path *inside* `.venv/` while activated
- [ ] `pip list` outside the venv does **not** show `requests` (isolation proven)
- [ ] `requirements.txt` exists with pinned versions; `.venv/` is not in it
- [ ] `python p3_fetch.py` prints a joke's setup and punchline
- [ ] every function in `p3_fetch.py` has parameter **and** return hints
- [ ] `mypy p3_fetch.py` prints `Success: no issues found in 1 source file`
- [ ] you deliberately broke a call, saw mypy's `[arg-type]` error **before running**, and fixed it
- [ ] **stretch:** rewrite the script with `anthropic` instead of `requests` — ask Claude to *rate the joke* (Part 2 §2 shape), still mypy-clean

## Active Recall & Self-Test (answer from memory — retrieval builds the memory)

1. What are the two distinct payoffs of a type hint, and which of them does the Python interpreter itself provide? (Trick question — answer precisely.)
2. Write from memory the signature of a function taking a URL string and optional float timeout, returning a dict of string keys and unknown values.
3. Name the exact two mechanisms that make a venv work (one about the interpreter, one about the shell).
4. Your script says `ModuleNotFoundError: No module named 'requests'` but you *know* you installed it. Give the two diagnostic commands and what each tells you.
5. What does `main()` return when `main` is declared `async def`, and what are the two correct ways to actually run it (one inside async code, one at top level)?
6. What does `zip` do when its inputs have different lengths, and how do you make that a loud error?
7. Which two dunder methods does `with` drive, when does each run, and what is guaranteed on the exception path?
8. In Part 2: which P3 concept explains editor autocomplete on `response.`? Which explains why `input_schema` felt familiar? Which explains `async with client.messages.stream(...)` — and why is it *two* of today's concepts at once?

**Teach-back prompt (60 seconds, aloud, no notes):** *"Explain to a new teammate why we never `pip install` globally and why every function we write has type hints — using one concrete disaster story for each."* If you can't produce both disaster stories (the two-projects version conflict; the 2 a.m. `TypeError`/silent-wrong-value), re-read §§1.1 and 2.2 — you haven't learned it yet.

## Spaced-Repetition Flashcards

| Q | A |
|---|---|
| What does Python do with type hints at runtime? | Stores them in `__annotations__` and otherwise **ignores** them — enforcement comes from external tools (mypy, editors) or runtime validators (Pydantic) |
| Syntax: parameter, default, return hint in one signature? | `def f(x: int, y: float = 1.0) -> str:` |
| Modern spelling of "an int or None"? | `int \| None` (PEP 604; older: `Optional[int]`) |
| What is `Any` for? | A deliberate no-contract hole — use only at boundaries where data shape is genuinely unknown, then convert to real types |
| mypy vs. Pydantic in one line? | mypy checks *your code against itself* before running; Pydantic checks *outside data against your types* at runtime |
| What is a venv, mechanically? | A folder with `pyvenv.cfg`, its own interpreter entry point, and a **private `site-packages`**; activation just edits `PATH` |
| Why did venvs become necessary? | One global `site-packages` can hold only one version of each package → two projects with conflicting needs cannot coexist (dependency hell) |
| Reproduce an environment on another machine? | Commit `requirements.txt` (`pip freeze > requirements.txt`), then `python3 -m venv .venv` + `pip install -r requirements.txt`; never commit `.venv/` |
| `pip show requests` — which line is the misconfiguration detector? | `Location:` — tells you *which* `site-packages` (venv or global) the package actually sits in |
| What does calling an `async def` function return? | A **coroutine object** — the body has NOT run; drive it with `await` (inside async) or `asyncio.run()` (top level) |
| The telltale signs of a forgotten `await`? | A `<coroutine object ...>` where a value should be + `RuntimeWarning: coroutine '...' was never awaited` at exit — and no crash |
| `async with` / `async for` at P3 level? | Same meaning as `with`/`for`, but allowed to pause at the wait points |
| `zip` on unequal lengths? | Silently stops at the shortest; `strict=True` raises `ValueError` instead |
| The `with` guarantee, precisely? | `__exit__` (cleanup) runs on **every** exit path — normal, `return`, or exception — and the exception still propagates |
| f-string decimal + debug forms? | `f"{price:.2f}"` (2 dp), `f"{var=}"` (prints `var='value'`) |
| In agentic code: what mirrors a type-hinted signature? | A tool's `input_schema` (JSON Schema) — the same contract, written for the model; Pydantic bridges the two notations |
| Why pin `anthropic==X.Y.Z`? | LLM SDKs change surface fast; the pin makes the environment reproducible so working agent code keeps working |

## Primary Sources (verify against these; go deeper here)

- **PEP 484 — Type Hints** (peps.python.org/pep-0484/) — the founding document; also PEP 3107 (annotation syntax), **PEP 526** (variable annotations), **PEP 604** (`X | None`), PEP 585 (built-in generics `list[int]`)
- **`typing` module docs** — docs.python.org/3/library/typing.html — and the **mypy docs** (mypy.readthedocs.io), esp. "Running mypy → missing imports" for the stub-package story
- **`venv` docs** — docs.python.org/3/library/venv.html (the "How venvs work" section confirms §2.4) — plus the **pip docs** (pip.pypa.io) and **Python Packaging User Guide** (packaging.python.org)
- **PEP 492 — Coroutines with async and await syntax** (peps.python.org/pep-0492/); `asyncio` docs for `run`/`gather` signatures
- **PEP 498 — f-strings**; **PEP 343 — the `with` statement**; built-ins `enumerate`/`zip` — docs.python.org/3/library/functions.html
- **Anthropic SDK & API docs** — the official Python SDK README and platform docs for current model IDs, streaming, and tool use — *everything SDK-side drifts fast: verify before relying on it*
- **uv docs** — docs.astral.sh/uv — for Day 7

## Key Takeaways & Summary

**10-second version:** Type hints are contracts checked by tools, not by Python; a venv is a private `site-packages` per project; `async def`/`await` mark pause-able waiting (mechanics Day 19); `with` guarantees cleanup on every exit. The Anthropic SDK exercises all four, unchanged.

**1-minute version:** Modern Python is P1/P2 Python plus explicitness. **Type hints** (`def f(x: int) -> str:`) declare what functions accept and return; the interpreter ignores them, but editors and mypy catch violations before anything runs, and Pydantic/FastAPI later *generate machinery* from them. **Virtual environments** end dependency hell by giving each project its own package folder — create, activate, `pip install`, `pip freeze > requirements.txt`; never install globally. **`async`/`await`** you only need to *read* today: `async def` makes a pause-able coroutine function, calling one *without* `await` yields an inert coroutine object (the classic silent bug), `asyncio.run()` is the on-switch. **`with`** guarantees resource cleanup on every exit path via `__enter__`/`__exit__`. In agentic code these are not analogies but the literal material: the SDK is pip-pinned in a venv, its calls are type-checked, tool schemas are contracts for the model, `AsyncAnthropic` is the async syntax verbatim, and streams live inside `with`.

**5-minute version:** Start from the failure modes. Pre-2014 Python functions were unlabeled boxes — wrong-type calls crashed at runtime (or worse, silently computed nonsense), and the "contracts" of the era (docstrings, naming conventions, `isinstance` boilerplate) lived outside the language where no tool could check them. PEP 484's gradual type hints fixed this: annotations are stored metadata (`__annotations__`), *ignored by the interpreter* — a fact you proved by running `add("ab","cd")` against an `int` contract — but consumed by three kinds of readers: humans (signatures read as sentences), static checkers (mypy flags the bad call at its *cause*, in-editor, pre-run — versus the interpreter's crash at the *symptom*, mid-run), and runtime validators (Pydantic, which turns the same hints into actual data validation; mypy checks your code against itself, Pydantic checks the world against your code). Meanwhile, the packaging failure mode: one global `site-packages` can hold one version of each package, so two projects with conflicting needs break each other — dependency hell. A venv is the disarmingly simple fix: a per-project folder with `pyvenv.cfg`, an interpreter entry point, and a private `site-packages`; activation merely edits `PATH`, the interpreter finds its packages relative to its own executable, and `pip freeze > requirements.txt` makes the whole thing reproducible across machines and months. Third, the waiting problem: programs that talk to networks spend most of their time blocked; `async def`/`await` (PEP 492) mark, in syntax, where pausing may happen so a scheduler can overlap waits — today you learn just the reading rules (`await x` ≈ "call and wait"; calling an `async def` without `await` produces a coroutine object and a `RuntimeWarning`, the silent do-nothing bug) and defer the event loop to Days 19–21, having glimpsed the payoff (two 1-second waits finishing in 1 second under `gather`). Finally `with` (PEP 343): resources demand cleanup on *every* exit path, humans forget, so the language guarantees it via `__enter__`/`__exit__`. Part 2 then demonstrates that agentic code is these same constructs in production dress — the pinned `anthropic` package (whose own dependency list includes Pydantic), fully typed SDK calls your editor autocompletes, JSON-Schema tool contracts that mirror type-hinted signatures, `AsyncAnthropic` calls differing from sync by four tokens, and streaming connections managed by `(async) with`. The bridge: contracts and isolation are one engineering discipline; today you learned Python's native forms, and every later layer (Pydantic Day 8, FastAPI Day 9, uv Day 7, the tool loop Day 22, the event loop Day 19) is a more powerful consumer of exactly these forms.

**Expert summary:** P3 installs the two invariants the rest of the stack presumes: (1) *machine-checkable interface contracts* — PEP 484 gradual typing as stored annotations with a three-consumer ecosystem (static: mypy/pyright at zero runtime cost; runtime: Pydantic-class validators generating checks from `__annotations__`; generative: FastAPI deriving parsing/serialization/docs), with `Any` as the deliberate untyped boundary and stub packages covering untyped third-party surfaces; and (2) *hermetic, reproducible dependency scopes* — venv's `pyvenv.cfg` + interpreter-relative `site-packages` resolution and PATH-based activation, with pinned requirements as the temporal reproducibility layer (superseded ergonomically, not conceptually, by uv's locked projects). Async is introduced strictly as surface grammar — coroutine functions as deferred computations, `await` as suspension points, the never-awaited-coroutine failure signature — with the scheduler deliberately opaque until the concurrency unit. The context-manager protocol is taught as the general acquire/release guarantee (deterministic `__exit__` on all exit paths) precisely because it reappears as the resource-safety idiom for SDK streaming. The agentic mapping is literal rather than metaphorical: typed SDK surfaces make model calls statically checkable; JSON-Schema tool definitions are the cross-process projection of the same contracts (with Pydantic as the isomorphism); and the dominant failure modes of naive agent code — SDK-surface drift, silent unawaited calls, leaked stream connections, late-detected shape errors mid-run — are each preempted by one of the day's four constructs.

## Mental Models

- **The shipping label (type hints):** labels let every handler treat the box correctly without opening it — but Python's courier never weighs the box; only inspectors you hire (mypy) or unpacking clerks (Pydantic) do.
- **One toolbox per job site (venvs):** duplicate cheap tools rather than share one wrench across sites; `requirements.txt` is the toolbox's packing list, so any site can be re-provisioned identically.
- **The waiter, not more waiters (async):** one worker who never stands staring at the stove; `await` is "kitchen, take over"; the payoff is overlapped waiting, not parallel cooking.
- **The hotel keycard (`with`):** access granted on entry, revoked on exit — including when you storm out mid-stay.
- **Contracts down, isolation around (the bridge):** every seam in the stack is guarded by a contract (hints, schemas) or a wall (venv, `with`-scoped resources). When debugging, ask first: *which seam's guard is missing?*

## Mnemonics

- **"Hints are for readers, not the runner."** — the interpreter ignores annotations; tools and humans consume them.
- **"No `(.venv)` in the prompt → don't type `pip`."** — the activation seatbelt check.
- **F-A-R: Freeze After Requirements-change.** — every `pip install` should be followed by re-freezing `requirements.txt`.
- **"`async def` makes it, `await` runs it, `asyncio.run` starts it."** — the three-part async grammar.
- **"See `<coroutine object>`? You forgot `await`."** — the one async bug of P3, on sight.
- **"`with` = cleanup, whatever happens."** — every exit path, guaranteed.
- **CIVA** — today's menu: **C**ontracts (hints), **I**solation (venvs), **V**erbs-that-pause (async), **A**lways-cleanup (`with`).

## Common Confusions

| Confusion | Resolution |
|---|---|
| "Type hints make Python check types at runtime." | No — the interpreter stores and ignores them (§1.6). Checking is mypy/editor (pre-run) or Pydantic (runtime, Day 8). |
| "`Optional[int]` means the argument is optional." | It means *maybe-None* (`int \| None`). "Optional argument" is about having a *default value* — unrelated. |
| "mypy passed, so my program can't fail on types." | mypy checks your code against *itself*. Data from outside (JSON, user input) arrives as `Any`-shaped — runtime validation (Pydantic) guards that border. |
| "A venv is a copy of Python / a VM / a container." | It's a folder: config file + interpreter entry point + private `site-packages`. Activation just edits `PATH`. Delete the folder = clean uninstall. |
| "`pip install` puts the package wherever my project is." | It installs into the `site-packages` of **whichever pip ran** — venv's if activated, global if not. `pip show PKG` → `Location:` settles it. |
| "`async def` makes my function faster." | Alone it does nothing but change the calling convention. Speedups come from *overlapping waits* across several coroutines (Days 19–21). |
| "Calling `main()` runs it — it's a function." | Not when it's `async def`: calling *creates a coroutine object*; only `await`/`asyncio.run` execute it (§3.4). |
| "`with` swallows exceptions." | It runs cleanup and then lets the exception propagate. (A DB-transaction `__exit__` may *react* to it — rollback — but still re-raises unless designed otherwise.) |
| "`zip` errors on unequal lengths." | It silently truncates to the shortest — pass `strict=True` to make it error (§4.3). |
| "The tool `input_schema` is some AI-specific format." | It's JSON Schema — the language-neutral rendering of the same contract a type-hinted signature declares; Pydantic converts between them (P2 §3). |
| "I'll learn venv now, then relearn everything for uv." | uv drives the *same* venvs and `site-packages` — faster driver, same road (§2.8, Day 7). |

---

*Next: [Day P4 — Git & the shell](p4-git-and-shell.md), where the venv/`requirements.txt` conventions from today become commits — and the P-phase checkpoint decides whether you're ready for Day 1.*
