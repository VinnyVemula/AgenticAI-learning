# The 60-Day Plan: From Zero → Independent Low-Level Design (LLD) in Python

> **Who this is for:** someone with **zero** programming background — never written a loop, never declared a variable. Nothing is assumed. Every day builds strictly on the previous one.
>
> **The thesis (memorize it):** *Low-Level Design is the discipline of deciding which object owns which state and which behaviour, so that the thing most likely to change is the cheapest thing to change.* Every principle, pattern, and problem in this plan is one more answer to that single question. Syntax is the entry fee; judgment is the skill.

---

# Read this first: the honest answer about 30 days

You asked for a 30-day plan and for the truth if 30 days is too short. **It is too short.** Here is why, concretely:

1. **Fluency comes before design.** A person who has never written code needs roughly 10–12 days of daily practice just to write loops, functions, dictionaries, and error handling *without thinking about syntax*. You cannot reason about *where a responsibility belongs* while you are still fighting a colon or an indentation error. That alone consumes a third of a 30-day plan.
2. **OOP is a second language on top of the first.** Classes, dunder methods, inheritance vs composition, abstract interfaces, type hints, and the Python data model take another 8–10 days to become natural.
3. **LLD is judgment, and judgment comes from reps.** The target — *"design any system independently, reasoning about tradeoffs, not copying patterns"* — is what people with 2–3 years of experience are tested on in interviews. Realistically it requires ~15 solved design problems *after* patterns are learned. Fitting fundamentals + OOP + SOLID + 20 patterns + 15 systems into 30 days would mean one new major idea every ~4 hours with no time to make mistakes, which is the only way design intuition forms.

**So this is a 60-day plan** at ~3–4 focused hours a day (~210 hours). **Day 30 is a marked checkpoint** so you can see exactly where a 30-day effort would leave you:

| By the end of… | You will be able to… | You will *not yet* be able to… |
|---|---|---|
| **Day 30** (checkpoint) | write clean, tested, type-hinted Python; model a domain with classes, dataclasses, enums and interfaces; explain and apply SOLID; use the creational patterns; design *simple* systems (Parking Lot, Library, Vending Machine) *with a method to follow* | design a concurrent system (booking, rate limiter, cache) safely; pick between 20 patterns under pressure; design an unfamiliar system from a blank page in 45 minutes without prompts |
| **Day 60** (goal) | do all of the above independently: take any unfamiliar problem statement, run the LLD method end to end, defend every decision, and ship production-quality code with tests | (this is where "full LLD capability" honestly begins — HLD/distributed systems is a separate journey; see *What comes next*) |

If you can only afford 30 days, follow Days 1–30 as written and treat the checkpoint table as your real outcome. Do **not** compress the plan to force everything into 30 days — the compression destroys the reps that create the skill.

**Time budget per day:** ~60 min concept + internals (read, then type every example), ~90 min design problem (paper first, then code), ~30 min build/tests, ~20 min recall + teach-back. If a day overflows, let it — day numbers are a sequence, not a deadline.

---

# Environment setup (once, before Day 1)

LLD is pure Python — unlike backend/systems work you do **not** need WSL2 or Linux for this plan. Native Windows is fine.

- **Python 3.12 or newer** from python.org (tick "Add to PATH"). Verify: `python --version`.
- **VS Code** with the official Python extension (Microsoft) and Pylance. Turn on "Format on save."
- **A project folder** `lld-journey/` under git (`git init`). Every day's code is a commit — that history is your proof of skill.
- **Tools you will meet on Day 8 and Day 10:** `python -m venv .venv` (isolated environment), `pip install pytest mypy ruff` — don't worry about what they are yet; the plan introduces each when needed.
- **Paper and a pen.** Every design problem starts on paper. Drawing boxes and arrows *before* typing is the single habit that separates people who design from people who type.

---

# How to run every single day (the method IS the plan)

Reading builds *recognition*; engineers run on *recall + judgment*. Run each day through these six moves:

1. **Read the concept, then type every code example yourself** — never paste. Run it. Then **break it on purpose**: change a value, delete a line, feed it garbage, read the error. Bugs you cause are lessons you keep.
2. **Do the design problem on paper first.** Every day has one. Write the requirements, draw the boxes, write the approaches and their tradeoffs *before* you look at the plan's decision. Then compare. Where you disagree with the plan, argue it out in writing — you are often allowed to be right.
3. **Run the LLD method** (below) on every design problem from Day 11 onward, even the tiny ones. Method becomes reflex only through repetition.
4. **Write the code and at least three tests.** From Day 10 onward, no design is "done" without tests. Tests are how you prove a design is actually decoupled — if it's hard to test, it's badly designed.
5. **Study the Python internals block.** Each day ties the concept to how CPython actually implements it. This is what turns "I use classes" into "I know what a class *is*," and it is what lets you predict how a pattern will behave in Python specifically.
6. **Recall + teach back.** Answer the day's recall questions from memory; explain the day's core idea aloud in 60 seconds as if to a colleague. Add flashcards to spaced repetition (Anki).

**Rule of thumb:** if a day produced only reading — no typed code, no paper design, no tests, no teach-back — the day didn't land. Redo before moving on.

---

# The LLD method (learn it once on Day 25, use it every day after)

Every design problem in Phases 3–5 runs through these six steps. Time-box them (interview pacing in brackets):

1. **Clarify requirements & scope** (5 min): functional requirements as a numbered list; non-functional (concurrency? persistence? scale?); explicitly *out of scope*. Ask: "What is the one operation this system exists to do?"
2. **Identify core entities** (5 min): nouns in the requirements → candidate classes. Split them into **entities** (have identity and lifecycle: `Order`, `Vehicle`), **value objects** (defined by their values: `Money`, `Address`), **enums** (closed sets of states/kinds), and **services** (behaviour that doesn't belong to one entity).
3. **Define relationships & ownership** (5 min): is-a vs has-a; cardinality (one `ParkingLot` has many `Floor`s); *who owns which state*. The rule: state lives with the object that must keep it consistent.
4. **Define behaviours / APIs** (10 min): verbs → methods; decide which class each method belongs to (**Tell, don't ask**: the object with the data does the work). Write method signatures with type hints before any bodies.
5. **Apply principles & patterns only where a force demands it** (10 min): *what varies?* → Strategy/Factory; *state-dependent behaviour?* → State; *many interested parties?* → Observer; *undo/queue/log actions?* → Command; *treelike?* → Composite. No force, no pattern.
6. **Walk through scenarios & stress the design** (10 min): the happy path, two edge cases, one failure, one concurrency scenario, and **the extension test** — "what changes if we add X?" A good design localizes X to one class.

**The design-decision rubric** (ask these when torn between approaches): Which parts are most likely to change? Who is the single owner of this state? What must never be observed in an inconsistent state? What does it cost to be wrong (money, safety, or just a refactor)? What is the *simplest* thing that satisfies today's requirements without blocking tomorrow's? Prefer the approach that answers those best, not the one with the most patterns.

---

# Trusted references (verify the plan against these, not against random blogs)

- **Python language:** the official tutorial and docs (docs.python.org); *Fluent Python*, 2nd ed. (Ramalho) for internals and the data model; *Python Distilled* (Beazley) for a compact reference.
- **Design principles & patterns:** *Head First Design Patterns* (Freeman & Robson — Java, but the clearest explanations); refactoring.guru (patterns with Python examples); **python-patterns.guide** (Brandon Rhodes — how each GoF pattern actually looks in idiomatic Python; read this over the Java-flavoured versions); *Design Patterns* (Gamma et al., the "GoF" book) as the reference of record.
- **Design judgment:** *Refactoring*, 2nd ed. (Fowler) — the code-smell catalog; *Clean Architecture* / *Agile Software Development: Principles, Patterns, and Practices* (Martin) for SOLID as originally stated; *A Philosophy of Software Design* (Ousterhout) for the deep/shallow module idea.
- **Python-specific architecture:** *Architecture Patterns with Python* (Percival & Gregory, free at cosmicpython.com) — Repository, Unit of Work, Service Layer, events, in real Python.
- **LLD problem practice:** the problem bank at the end of this plan; the *Grokking the Low Level Design Interview* style catalogs for problem statements (use them for *statements*, not solutions — solve first, compare after).

---

# The 60 days at a glance

| Phase | Days | Theme | You can now… |
|---|---|---|---|
| 0 | 1–10 | Programming from zero | write real Python: values, control flow, functions, collections, errors, files, generators, tests |
| 1 | 11–20 | Object-oriented Python & the data model | model a domain with classes, dunders, dataclasses, enums, inheritance, ABCs/Protocols, type hints, decorators |
| 2 | 21–27 | Design principles | smell bad design, apply SOLID/GRASP, run the LLD method, reason about thread safety |
| 3 | 28–41 | Design patterns, the Python way | know all 23 GoF patterns + the enterprise and concurrency patterns, *and when not to use them* |
| 4 | 42–57 | LLD problems, end to end | design and code 20+ classic systems: parking lot → booking → cache → rate limiter → editor → job scheduler |
| 5 | 58–60 | Independence & capstone | design an unseen system from a blank page under time; ship a capstone you can defend line by line |

**Checkpoints:** Day 10 (fundamentals), Day 20 (OOP), **Day 30 (the 30-day line)**, Day 41 (patterns), Day 57 (problems), Day 60 (capstone).

---

# PHASE 0 — Programming From Zero (Days 1–10)

You cannot design what you cannot express. This phase gives you the *vocabulary* of Python — but every day already plants a design idea, because even "which type holds money?" is a design decision. Days 1–2 use straight-line code on purpose; from Day 3 onward every example uses the full production structure (functions, validation, a `main()`).

### Day 1 — What a program is: values, names, and types

- **Concept & why it matters:** A program is a list of instructions the computer runs top to bottom. A **value** is a piece of data (`42`, `3.14`, `"hello"`, `True`, `None`). A **type** is the *kind* of value, and it decides what you can do with it (you can add two numbers, you can't add a number and a word). A **variable** is a *name attached to a value* — in Python, think of a name tag hung on an object, not a box that holds it (this one mental model prevents a dozen future bugs). The five core types: `int` (whole numbers), `float` (decimals — approximate!), `str` (text), `bool` (`True`/`False`), `None` (the deliberate absence of a value). Arithmetic (`+ - * / // % **`), comparison (`== != < > <= >=`), and `print()` / f-strings to show results. **Why it matters for LLD:** choosing a type is your first design decision — the wrong one (float for money) silently corrupts a system.
- **Real-world case study:** **The Vancouver Stock Exchange index (1982–83).** The index was recomputed thousands of times a day and truncated to three decimals each time; after 22 months the index read ~520 when it should have read ~1,100. Nobody chose "truncate" as a design; it was a *type* decision nobody made consciously. Same family: the 1991 Patriot missile timing drift caused by accumulated floating-point error in a 24-bit fixed-point time counter. Lesson: the representation of a value *is* part of the design.
- **Design problem — represent a price in a shop system.** Approaches: **(A) `float`** — natural, but `0.1 + 0.2 == 0.30000000000000004`; rounding errors accumulate across thousands of transactions. **(B) `int` in the smallest unit (paise/cents)** — exact, fast, but every display needs `/100` and every developer must remember the unit. **(C) `decimal.Decimal`** — exact decimal arithmetic, explicit rounding rules, slower, slightly more verbose. Tradeoffs: A is fine for a physics simulation and wrong for money; B is what most payment systems store in databases; C is what most Python business code computes with.
- **Thought process → decision:** Ask *"what does it cost to be wrong?"* For money, being off by one paisa per transaction across a million transactions is a real financial and legal problem, so exactness beats convenience. Between B and C: B for storage and network (compact, unambiguous), C for computation (explicit rounding). You will wrap this in a `Money` class on Day 12 — today, just internalize that `float` is *not* the default for anything that must be exact.
- **Code (type it, run it, then change values and watch the output):**

```python
"""Day 1 — a receipt, straight-line style. No loops or functions yet (those are Days 2–3)."""

from decimal import Decimal, ROUND_HALF_UP

# --- inputs. UPPER_CASE names signal "a constant — set once, never reassigned" ---
ITEM_NAME = "Notebook"
UNIT_PRICE = Decimal("45.50")   # Decimal, built from a *string* so no float error sneaks in
QUANTITY = 3                    # int: you cannot buy 2.5 notebooks
GST_RATE = Decimal("0.18")      # 18 %
IS_MEMBER = True                # bool
COUPON_CODE = None              # None means "no coupon" — deliberately different from "" or 0

PAISA = Decimal("0.01")         # the rounding unit

# --- computation ---
subtotal = UNIT_PRICE * QUANTITY
tax = (subtotal * GST_RATE).quantize(PAISA, rounding=ROUND_HALF_UP)
member_discount = (subtotal * Decimal("0.05")).quantize(PAISA) if IS_MEMBER else Decimal("0")
total = subtotal + tax - member_discount

# --- output (f-strings: put expressions inside {} and format them after the colon) ---
print(f"Item      : {ITEM_NAME} x {QUANTITY}")
print(f"Subtotal  : ₹{subtotal:,.2f}")
print(f"GST (18%) : ₹{tax:,.2f}")
print(f"Discount  : -₹{member_discount:,.2f}")
print(f"Total     : ₹{total:,.2f}")
print(f"Coupon    : {'none' if COUPON_CODE is None else COUPON_CODE}")

# --- look behind the names: every value is an object with a type and an identity ---
print(type(UNIT_PRICE), type(QUANTITY), type(IS_MEMBER), type(COUPON_CODE))
print(0.1 + 0.2 == 0.3)                       # False — the float trap, live
print(Decimal("0.1") + Decimal("0.2") == Decimal("0.3"))   # True
```

- **Python internals:** Every value in Python is a **PyObject** on the heap: a small header holding a *reference count* (how many names point at it) and a pointer to its *type*, followed by the data. That is why `type(x)` works on anything and why a Python `int` costs ~28 bytes, not 8 — it carries its own metadata. A variable is an entry in a namespace *dictionary* mapping a name to a pointer; `x = 5` doesn't "store 5 in x," it binds the name `x` to the object `5`. `id(x)` shows the object's address in CPython. Small ints (−5..256) are pre-created and shared, which is why `id(5)` is the same everywhere — an optimization, never something to rely on. `is` compares identity; `==` compares value; you'll use `is` only for `None`.
- **Build & drill:** Modify the receipt for three items (three sets of names — yes, it's clumsy; that pain is why lists exist on Day 5). Then write a second script that prints the type of ten different literals you invent, and predict each type *before* running.
- **Recall:** What is the difference between a value, a type, and a name? Why is `0.1 + 0.2 != 0.3`? Why is a name a *tag*, not a *box*, and when will that matter?

### Day 2 — Control flow: decisions and repetition

- **Concept & why it matters:** Programs make decisions (`if` / `elif` / `else`) and repeat (`while` as long as a condition holds; `for` over each item of a sequence, most often `range(n)`). Boolean logic (`and`, `or`, `not`), comparison chaining (`0 <= x < 10`), and **truthiness** — Python treats `0`, `""`, `None`, and empty containers as false in a condition. `break` exits a loop early; `continue` skips to the next iteration; `else` on a loop runs if it did *not* break. Indentation is not decoration — it *is* the block structure. **Why it matters for LLD:** long `if/elif` chains keyed on a "type" or "state" are the #1 smell that later becomes the State, Strategy, and Factory patterns. Learn to write them today so you can recognize them on Day 31.
- **Real-world case study:** **Airline fare rules.** Every airline once encoded fare logic as thousands of nested conditions ("if Saturday-night stay and booked >14 days and not December…"). It became impossible to change one rule without breaking another. Modern systems express rules as *data* (tables of conditions) evaluated by a small, generic engine — the difference between *branching in code* and *branching on data* is exactly today's design problem.
- **Design problem — a shipping-fee calculator:** free above ₹999, ₹49 for orders ≥ ₹500, ₹99 below that, plus a ₹200 surcharge for remote pincodes. Approaches: **(A) nested `if/elif`** — obvious, readable for three tiers, but adding a fourth tier means editing logic, and the remote surcharge doubles every branch. **(B) an ordered list of `(threshold, fee)` tiers scanned with a loop** — the *rules become data*; adding a tier is adding one row; surcharge stays a single separate step. **(C) a dictionary keyed by tier name** — fine for named categories but awkward for numeric ranges. Tradeoffs: A is fastest to write and fine when rules are truly fixed; B separates *policy* (the table) from *mechanism* (the loop); C confuses ranges with categories.
- **Thought process → decision:** Apply the rubric: *what's most likely to change?* The thresholds and fees (marketing changes them every festival); *what's stable?* "Find the first tier the amount qualifies for." Put the changing thing in data and the stable thing in code → **B**. Keep the surcharge as its own step so the two rules stay independent (they vary for different reasons — a preview of the Single Responsibility Principle on Day 22).
- **Code:**

```python
"""Day 2 — shipping fee: rules as data, evaluated by a small loop."""

from decimal import Decimal

# Ordered from the highest threshold down; the first matching row wins.
FEE_TIERS = [
    (Decimal("999"), Decimal("0")),    # (minimum order amount, fee)
    (Decimal("500"), Decimal("49")),
    (Decimal("0"),   Decimal("99")),
]
REMOTE_SURCHARGE = Decimal("200")
REMOTE_PINCODE_PREFIXES = ("79", "19", "74")   # e.g. parts of the North-East, Ladakh, Andaman

order_amount = Decimal("650.00")
pincode = "793001"

if order_amount < 0:
    print("Order amount cannot be negative.")
else:
    fee = None
    for minimum, tier_fee in FEE_TIERS:       # loop over rows; unpack each tuple into two names
        if order_amount >= minimum:
            fee = tier_fee
            break                              # first match wins; stop scanning
    else:                                      # runs only if the loop never hit `break`
        print("Configuration error: no tier matched — the table must end with a 0 threshold.")

    if fee is not None:
        is_remote = pincode.startswith(REMOTE_PINCODE_PREFIXES)
        if is_remote:
            fee += REMOTE_SURCHARGE
        label = "remote" if is_remote else "standard"
        print(f"Order ₹{order_amount:,.2f} to {pincode} ({label}): shipping ₹{fee:,.2f}")

# A `while` loop: count down until a condition fails
attempts_left = 3
while attempts_left > 0:
    print(f"{attempts_left} attempt(s) left")
    attempts_left -= 1                        # same as attempts_left = attempts_left - 1
print("Done.")
```

- **Python internals:** Run `import dis; dis.dis("if x > 3: y = 1")` — you'll see `COMPARE_OP` then `POP_JUMP_IF_FALSE`: an `if` compiles to a *conditional jump* in bytecode; `elif` chains become sequential jumps, so a 40-branch `elif` is 40 comparisons worst-case (a dict lookup, Day 6, is one). Truthiness is a protocol: `if obj:` calls `obj.__bool__()`, falling back to `obj.__len__() != 0`, falling back to `True` — you'll implement `__bool__` on your own classes on Day 13. `for` does not count indexes; it asks the object for an *iterator* and calls `next()` on it until it raises `StopIteration` (Day 9 opens this fully). `range` is lazy: `range(10**9)` allocates nothing until iterated.
- **Build & drill:** FizzBuzz (1–100: multiples of 3 print "Fizz", of 5 "Buzz", both "FizzBuzz"). Then a number-guessing game: pick a secret, loop until the user guesses it with `input()`, give higher/lower hints, cap at 7 guesses. Then rewrite the shipping tiers as nested `if` and *feel* the difference when you add a fourth tier to both versions.
- **Recall:** What does the `else` on a `for` loop mean? Name five falsy values. Why did we put the tiers in a list instead of in `if` branches — and what changes if the *shape* of a rule changes (e.g., a tier that depends on weight too)?

### Day 3 — Functions: naming a piece of behaviour

- **Concept & why it matters:** A **function** packages a block of code under a name so it can be *called* with inputs (**parameters**) and hand back a result (`return`). Positional vs keyword arguments, default values, `*args`/`**kwargs` (accept any number), and **scope** — names created inside a function live only inside it (the LEGB rule: Local → Enclosing → Global → Builtins). **Pure functions** (same input → same output, no side effects) are the easiest things in software to test and reuse. Docstrings say *what* a function does; type hints (`def f(x: int) -> str`) say what goes in and out — start using them today; they are how you write contracts before code, which is the whole game in LLD. **Why it matters for LLD:** a method is a function that belongs to an object. Every design decision about "which class owns this behaviour" is a decision about where a function lives. Learn to write small, single-purpose, well-named functions now and classes will be easy.
- **Real-world case study:** **The "God function."** Almost every legacy codebase has a 1,500-line `process_order()` that validates, prices, taxes, discounts, saves, emails, and logs. Nobody dares touch it; a bug fix in tax breaks emails. The fix is always the same: extract small functions with clear names and contracts, then compose them. Google's and Meta's style guides cap function length for this reason — not aesthetics, but *change isolation*.
- **Design problem — a discount engine** for a cart: 10% for members, ₹100 off with coupon `SAVE100` on orders above ₹1,000, free shipping over ₹999, discounts must not push a total below zero. Approaches: **(A) one function `compute_total(...)`** with all rules inline — quick, but each rule change touches the same function and its tests. **(B) one small pure function per rule** (`member_discount(subtotal, is_member)`, `coupon_discount(subtotal, code)`) composed by a thin `compute_total`** — each rule tested alone; adding a rule adds a function. **(C) a list of rule functions looped over** — rules become data (Day 2's idea applied to *behaviour*); most flexible, slightly more abstract for Day 3.
- **Thought process → decision:** Rules vary independently (marketing owns coupons, finance owns membership) → they must be changeable independently → **B** today, and you'll notice B is one step from C (a list of functions), which is one step from the **Strategy pattern** (Day 31). Choosing the *smallest* abstraction that separates the things that change separately is the core LLD move — you just made it.
- **Code:**

```python
"""Day 3 — discount engine built from small, pure, tested functions."""

from decimal import Decimal

ZERO = Decimal("0")
MEMBER_RATE = Decimal("0.10")
COUPONS: dict[str, tuple[Decimal, Decimal]] = {    # code -> (minimum order, flat discount)
    "SAVE100": (Decimal("1000"), Decimal("100")),
    "FIRST50": (Decimal("0"), Decimal("50")),
}


def member_discount(subtotal: Decimal, is_member: bool) -> Decimal:
    """Members get a percentage off the subtotal."""
    return subtotal * MEMBER_RATE if is_member else ZERO


def coupon_discount(subtotal: Decimal, code: str | None) -> Decimal:
    """Flat discount if the code exists and the order qualifies; otherwise zero."""
    if code is None:
        return ZERO
    rule = COUPONS.get(code.strip().upper())     # .get returns None instead of crashing on a bad key
    if rule is None:
        return ZERO
    minimum, amount = rule
    return amount if subtotal >= minimum else ZERO


def compute_total(subtotal: Decimal, *, is_member: bool = False, coupon: str | None = None) -> Decimal:
    """Apply all discounts; the total never drops below zero.

    Parameters after `*` are keyword-only: callers must write is_member=True,
    which prevents the classic bug of passing booleans in the wrong order.
    """
    if subtotal < ZERO:
        raise ValueError(f"subtotal cannot be negative: {subtotal}")
    total = subtotal - member_discount(subtotal, is_member) - coupon_discount(subtotal, coupon)
    return max(total, ZERO)


def main() -> None:
    print(compute_total(Decimal("1200"), is_member=True, coupon="save100"))   # 980
    print(compute_total(Decimal("800"), coupon="SAVE100"))                    # 800 (doesn't qualify)
    print(compute_total(Decimal("30"), coupon="FIRST50"))                     # 0, not -20


if __name__ == "__main__":     # run main() only when this file is executed, not when imported (Day 8)
    main()
```

- **Python internals:** `def` is an executable statement that creates a **function object** at runtime — you can put functions in lists, pass them as arguments, and return them (Day 19 builds on this). The function object holds a **code object** (compiled bytecode), its `__defaults__`, `__name__`, `__doc__`, and `__annotations__` (your type hints — stored as metadata, *not* enforced; that's what `mypy` is for on Day 10). **Default values are evaluated once, at `def` time**, so `def f(items=[])` shares one list across every call — the most famous Python bug; use `None` and create inside. Each call pushes a **frame** (local namespace + instruction pointer) onto the call stack; `return` pops it. Python resolves a name at runtime by walking LEGB, which is why a local variable lookup is faster than a global one.
- **Build & drill:** Write `is_valid_pincode(s: str) -> bool` (6 digits, first digit not 0), `clamp(x, low, high)`, and `format_inr(amount: Decimal) -> str` producing `₹1,23,456.00` with Indian grouping (that one is a real puzzle — do it with a loop, not a library). Refactor yesterday's shipping script into functions with a `main()`. Then reproduce the mutable-default bug on purpose and fix it.
- **Recall:** What are the LEGB rules? Why are default arguments evaluated only once, and what's the safe idiom? What makes a function *pure*, and why are pure functions easier to test?

### Day 4 — Strings and text: the data type you'll parse forever

- **Concept & why it matters:** Text is a `str` — an **immutable** sequence of Unicode characters. Indexing and slicing (`s[0]`, `s[-1]`, `s[2:5]`, `s[::-1]`), the workhorse methods (`strip`, `split`, `join`, `replace`, `startswith`, `find`, `lower`, `isdigit`), f-string formatting (`{x:>10}`, `{x:.2f}`, `{x!r}`), multi-line strings, escape sequences, and the difference between **`str` (text) and `bytes` (raw data)** with `.encode()`/`.decode()` in UTF-8. Regular expressions (`re`) for pattern matching — just `match`, `search`, `findall`, groups. **Why it matters for LLD:** every system boundary (files, HTTP, logs, user input) is text; parsing it robustly, and *validating it at the boundary* so the inside of your system deals with real types, is a design principle you'll apply on every problem.
- **Real-world case study:** **Log parsing at scale.** Every observability company (Splunk, Datadog, Elastic) is at its core a robust log-line parser. Their hard-won rule: *never parse with string position assumptions* — a log format changes, a field contains a space, and position-based parsing silently produces wrong data. Structured parsing (named groups, or better, structured logging in JSON) is why those systems survive format changes. Also: the 2010s "Unicode sandwich" — decode bytes at the edges, work in `str` inside, encode on the way out — is how Python 3 ended a decade of mojibake bugs.
- **Design problem — parse an application log line** like `2026-09-03 10:15:32 ERROR payment: card declined user=42 amount=1200`. Approaches: **(A) `split()` by spaces and index into the list** — trivial, breaks the moment a message contains extra spaces or a field is missing. **(B) `partition`/`split(maxsplit=)` step by step** — more robust than A, still positional, gets long. **(C) one regular expression with named groups** — declares the *shape* of the line in one place; unparseable lines are detected, not silently garbled; slightly opaque to read. **(D) a hand-written state machine character by character** — maximal control, way too much code for this problem (but it's exactly how real lexers work; you'll meet the State pattern on Day 34).
- **Thought process → decision:** The question is *"what happens when the input is slightly wrong?"* A and B produce *wrong data silently* — the worst failure mode in software. C fails *loudly* on a malformed line, which is what you want at a boundary. Choose **C**, return a structured result (a tuple today, a dataclass on Day 14), and keep the free-form `key=value` tail as a separate small parse so the two concerns stay independent.
- **Code:**

```python
"""Day 4 — parsing a log line robustly with a named-group regex."""

import re

LOG_LINE = re.compile(
    r"^(?P<date>\d{4}-\d{2}-\d{2}) "
    r"(?P<time>\d{2}:\d{2}:\d{2}) "
    r"(?P<level>DEBUG|INFO|WARNING|ERROR|CRITICAL) "
    r"(?P<component>[a-z_]+): "
    r"(?P<message>.*)$"
)
KEY_VALUE = re.compile(r"(\w+)=(\S+)")


def parse_log_line(line: str) -> tuple[str, str, str, str, dict[str, str]]:
    """Return (timestamp, level, component, message, fields) or raise ValueError.

    Failing loudly on a malformed line beats producing wrong data silently.
    """
    match = LOG_LINE.match(line.strip())
    if match is None:
        raise ValueError(f"unrecognised log line: {line!r}")
    parts = match.groupdict()
    fields = dict(KEY_VALUE.findall(parts["message"]))          # "user=42 amount=1200" -> {...}
    plain_message = KEY_VALUE.sub("", parts["message"]).strip()  # message without the key=value tail
    timestamp = f"{parts['date']}T{parts['time']}"
    return timestamp, parts["level"], parts["component"], plain_message, fields


def main() -> None:
    sample = "2026-09-03 10:15:32 ERROR payment: card declined user=42 amount=1200"
    timestamp, level, component, message, fields = parse_log_line(sample)
    print(f"{timestamp} [{level:<8}] {component:>10} | {message} | {fields}")

    for bad in ("garbage", "2026-09-03 10:15 INFO x: short time"):
        try:
            parse_log_line(bad)
        except ValueError as err:
            print("rejected:", err)

    # str vs bytes: the boundary between text and the outside world
    text = "₹1,200 café"
    raw = text.encode("utf-8")
    print(len(text), "characters;", len(raw), "bytes;", raw.decode("utf-8") == text)


if __name__ == "__main__":
    main()
```

- **Python internals:** Strings are **immutable**: every "modification" builds a new object, so `s += piece` in a loop is O(n²) in principle — build a list and `"".join(parts)` instead (CPython has an in-place optimization for the common case, but never rely on it). Because they're immutable they're **hashable** and usable as dict keys (Day 6), and short identifier-like strings are **interned** (one shared object) — the reason `"a" is "a"` is usually `True` and also why you never use `is` for string comparison. CPython stores strings with a *flexible representation* (1, 2, or 4 bytes per character depending on the widest character present), so `len()` is O(1) and indexing is O(1). `re.compile` at module level caches the compiled pattern once instead of per call.
- **Build & drill:** Write `slugify("Hello, World! 2026") -> "hello-world-2026"`, `mask_card("4111111111111111") -> "**** **** **** 1111"`, and a Caesar-cipher encode/decode pair. Then write a validator for an Indian mobile number (10 digits starting 6–9, optional `+91`) with regex, and list five inputs that *should* fail.
- **Recall:** Why is repeated `+=` on strings a problem, and what's the idiom? What is the difference between `str` and `bytes`? Why do we prefer failing loudly on a malformed line?

### Day 5 — Lists and tuples: ordered collections, and your first Big-O

- **Concept & why it matters:** A **list** is an ordered, mutable sequence (`append`, `insert`, `pop`, `remove`, `sort`, slicing, `in`, `len`). A **tuple** is an ordered, *immutable* sequence — use it for fixed records (`(lat, lon)`, `(name, price)`) and for anything that must be hashable. **Unpacking** (`a, b = pair`, `first, *rest = items`), `enumerate`, `zip`, `sorted(key=...)`, `reversed`, and **list comprehensions** (`[x * 2 for x in nums if x > 0]`) as the Pythonic way to transform. Introduce **Big-O**: how the cost of an operation grows with the size of the data — indexing a list is O(1), searching it is O(n), sorting is O(n log n), `insert(0, x)` is O(n). **Why it matters for LLD:** every "which collection should this class hold?" decision is a Big-O decision, and interviewers probe it ("what's the cost of `find_vehicle`?").
- **Real-world case study:** **Leaderboards and top-K.** A gaming leaderboard with 10 million players must show the top 100. Re-sorting the whole list on every score update is O(n log n) per update — thousands of times a second, that melts a server. Using a bounded heap (`heapq.nlargest`) or a sorted structure with O(log n) updates is the difference between a working game and an outage. The *data structure choice* is the design.
- **Design problem — store a to-do list** with title, priority, and done flag, supporting "show highest-priority pending items." Approaches: **(A) three parallel lists** (`titles`, `priorities`, `done_flags`) — every operation must keep three lists in sync; deleting index 3 from one and forgetting another corrupts everything. **(B) one list of tuples** `(priority, title, done)` — one item, one record; immutable records mean "mark done" must *replace* the tuple. **(C) one list of dicts** — named fields, mutable, but nothing stops a typo like `item["dnoe"]`. Tradeoffs: A is the classic beginner bug factory; B is safe and simple until items need to change; C is flexible but unchecked. (Day 11's answer, a class, is C with rules.)
- **Thought process → decision:** *What must never be inconsistent?* The three attributes of one task. That rules out A instantly. Between B and C: today's tasks mostly *don't* change except the done flag, and immutability means no aliasing surprises → **B**, replacing the tuple when marking done. Note the itch you feel writing `item[1]` for "title" — that itch is the motivation for `NamedTuple`/`dataclass` on Day 14.
- **Code:**

```python
"""Day 5 — a to-do list as a list of immutable records; sorting and top-k."""

import heapq

Task = tuple[int, str, bool]          # (priority: 1 = highest, title, done) — a *type alias*


def add_task(tasks: list[Task], title: str, priority: int) -> None:
    if not title.strip():
        raise ValueError("title cannot be empty")
    if not 1 <= priority <= 5:
        raise ValueError(f"priority must be 1..5, got {priority}")
    tasks.append((priority, title.strip(), False))


def mark_done(tasks: list[Task], title: str) -> None:
    for index, (priority, task_title, done) in enumerate(tasks):
        if task_title == title:
            tasks[index] = (priority, task_title, True)   # tuples are immutable: replace, don't mutate
            return
    raise LookupError(f"no task titled {title!r}")


def pending_by_priority(tasks: list[Task]) -> list[Task]:
    """All pending tasks, most urgent first. sorted() is O(n log n) and stable."""
    return sorted((t for t in tasks if not t[2]), key=lambda t: t[0])


def top_pending(tasks: list[Task], k: int) -> list[Task]:
    """Only the k most urgent — O(n log k), better than sorting everything when k << n."""
    return heapq.nsmallest(k, (t for t in tasks if not t[2]), key=lambda t: t[0])


def main() -> None:
    tasks: list[Task] = []
    add_task(tasks, "Pay electricity bill", 1)
    add_task(tasks, "Read Day 5 notes", 2)
    add_task(tasks, "Buy milk", 3)
    mark_done(tasks, "Buy milk")

    for priority, title, _ in pending_by_priority(tasks):    # `_` = "I don't need this value"
        print(f"P{priority}  {title}")
    print("Top 1:", top_pending(tasks, 1))

    squares = [n * n for n in range(1, 6)]                    # comprehension
    first, *rest = squares                                    # star-unpacking
    print(first, rest, list(zip(squares, "abcde")))


if __name__ == "__main__":
    main()
```

- **Python internals:** A CPython list is a **dynamic array of pointers** to objects (never the objects themselves — that's why a list can hold mixed types and why assigning `b = a` gives two names for *one* list; mutate through one, see it through the other). `append` is amortized O(1) because the array over-allocates by ~12.5% each time it grows; `insert(0, x)` and `pop(0)` shift every pointer (O(n)) — use `collections.deque` for queues (Day 39). Tuples are immutable and *may* be hashable (only if their contents are), which is why `(1, 2)` can be a dict key but `([1], 2)` cannot. `sorted` uses **Timsort**, stable and adaptive — stability is why sorting by priority then by date can be done in two passes. Slicing copies (shallow): `a[:]` gives a new list of the same object pointers.
- **Build & drill:** Implement `top_k_students(scores, k)`; a `moving_average(values, window)`; matrix transpose with `zip(*rows)`; and demonstrate the aliasing bug (`b = a; b.append(1)`) then fix it with `list(a)`. Time `insert(0, x)` vs `append` on a 100k list with `time.perf_counter()` and write one sentence explaining the difference.
- **Recall:** Why is `append` O(1) but `insert(0, ...)` O(n)? When do you choose a tuple over a list? What does *stable* sort mean and why does it matter?

### Day 6 — Dictionaries and sets: the O(1) lookup that runs everything

- **Concept & why it matters:** A **dict** maps keys to values with ~O(1) lookup, insertion, and deletion: `d[key]`, `d.get(key, default)`, `in`, `.items()`, `.setdefault`, `.pop`, `.update`, dict comprehensions, and merging with `|`. Keys must be **hashable** (immutable-ish: `str`, `int`, `tuple` of hashables — not `list`). A **set** is an unordered collection of unique hashables with O(1) membership and the algebra `| & - ^` (union, intersection, difference, symmetric difference). `collections.Counter` and `defaultdict` remove boilerplate. **Why it matters for LLD:** "find the vehicle by plate," "is this seat taken," "all users subscribed to this topic" — every LLD class you'll ever write holds a dict or a set as its index. Knowing when a list scan (O(n)) becomes a dict lookup (O(1)) is what interviewers mean by "reason about complexity."
- **Real-world case study:** **The N+1 lookup.** A very common production performance bug: code loops over 10,000 orders and, for each, scans a list of 50,000 customers to find the owner — 500 million comparisons. Building one `dict` from customer id → customer first (50k operations) makes the whole thing ~60k operations. The same shape appears as the N+1 query problem in databases. Recognizing "a lookup inside a loop → build an index" is a pattern you'll use on nearly every LLD problem.
- **Design problem — an inventory** supporting "add stock," "remove stock (never below zero)," "quantity of SKU," and "which SKUs are low." Approaches: **(A) a list of `(sku, qty)` tuples** — every lookup is a scan; duplicates can sneak in. **(B) a dict `sku -> qty`** — O(1) everything; uniqueness of SKU is structural, not a check you might forget. **(C) two structures: the dict plus a set of low-stock SKUs kept in sync** — O(1) "which are low" but two things to keep consistent. Tradeoffs: A is only fine for a dozen items; B is the correct default; C is premature until "which are low" is measured to be hot.
- **Thought process → decision:** The rubric's *"single owner of state"* question decides it: B has one structure that is the truth; C has two that can drift. Start with **B** and compute low-stock on demand with a comprehension. Only add the derived set if profiling shows it matters — *"you aren't gonna need it"* (YAGNI, Day 21) is a design principle, not laziness.
- **Code:**

```python
"""Day 6 — inventory as a dict index; sets for membership; Counter for tallies."""

from collections import Counter, defaultdict

LOW_STOCK_THRESHOLD = 5


def add_stock(inventory: dict[str, int], sku: str, quantity: int) -> None:
    if quantity <= 0:
        raise ValueError(f"quantity must be positive, got {quantity}")
    inventory[sku] = inventory.get(sku, 0) + quantity     # .get avoids KeyError on a new SKU


def remove_stock(inventory: dict[str, int], sku: str, quantity: int) -> None:
    available = inventory.get(sku, 0)
    if quantity > available:
        raise ValueError(f"cannot remove {quantity} of {sku}: only {available} in stock")
    remaining = available - quantity
    if remaining == 0:
        del inventory[sku]                                # keep the index clean: no zero rows
    else:
        inventory[sku] = remaining


def low_stock(inventory: dict[str, int]) -> set[str]:
    return {sku for sku, qty in inventory.items() if qty < LOW_STOCK_THRESHOLD}


def main() -> None:
    inventory: dict[str, int] = {}
    add_stock(inventory, "PEN-BLUE", 12)
    add_stock(inventory, "PEN-RED", 3)
    remove_stock(inventory, "PEN-BLUE", 10)
    print(inventory, "low:", low_stock(inventory))

    # sets: uniqueness and algebra
    visitors_monday = {"asha", "ravi", "meera"}
    visitors_tuesday = {"ravi", "kiran"}
    print("both days:", visitors_monday & visitors_tuesday, "| only monday:", visitors_monday - visitors_tuesday)

    # Counter and defaultdict remove boilerplate
    words = "the cat and the hat and the bat".split()
    print(Counter(words).most_common(2))
    by_length: defaultdict[int, list[str]] = defaultdict(list)
    for word in words:
        by_length[len(word)].append(word)
    print(dict(by_length))

    try:
        {}[["a", "list"]] = 1                              # lists aren't hashable
    except TypeError as err:
        print("unhashable:", err)


if __name__ == "__main__":
    main()
```

- **Python internals:** A dict is a **hash table**: `hash(key)` gives an integer; the table index is derived from it; collisions are resolved by *open addressing* with probing. Since 3.7, dicts preserve **insertion order** (guaranteed by the language) via a compact layout: a dense array of entries plus a sparse index array — this made dicts smaller *and* ordered. The **hash/eq contract**: if `a == b` then `hash(a) == hash(b)` must hold — you'll implement `__eq__` and `__hash__` together on Day 13 and this contract is why. Mutable objects (lists) aren't hashable because their hash would change while they sit in the table, making them unfindable. Attribute access on every object (`obj.name`) is *itself* a dict lookup into `obj.__dict__` (Day 11) — the whole language runs on this structure. A set is a hash table with keys only.
- **Build & drill:** Word-frequency counter for a text file (top 10); "first non-repeating character"; two-sum with a dict in O(n); group anagrams with a `defaultdict(list)` keyed by `tuple(sorted(word))`. Then rewrite the Day 5 to-do list with a dict keyed by title and note which operations became O(1).
- **Recall:** What makes an object hashable, and why can't a list be a key? Why do we prefer one authoritative structure over two synced ones? Describe the N+1 lookup and its fix.

### Day 7 — Errors and exceptions: designing how things fail

- **Concept & why it matters:** When something goes wrong, Python **raises an exception** — an object describing the failure — which unwinds the call stack until something **catches** it with `try`/`except`. `else` runs if nothing was raised; `finally` always runs (cleanup). The built-in hierarchy (`ValueError`, `KeyError`, `TypeError`, `FileNotFoundError` … all under `Exception`); **custom exceptions** via subclassing (a preview of classes); `raise ... from err` to chain causes; and reading a **traceback** bottom-up. Two philosophies: **LBYL** (look before you leap — check, then act) vs **EAFP** (easier to ask forgiveness — act, catch failure), and Python leans EAFP. **Why it matters for LLD:** *how a system fails* is part of its design. Which class raises what, which layer catches it, and what the caller can do about it is a decision you'll make on every problem — and a poor one (catching everything, silently) is how systems corrupt data quietly.
- **Real-world case study:** **Knight Capital, August 1, 2012.** A deployment left old code active on one server; a repurposed flag triggered it; the system began firing millions of unintended orders. Error signals existed but were routed to a mailbox nobody watched, and there was no *fail-stop* design — no exception path that said "this must halt." In 45 minutes the firm lost ~$440 million. Lesson: an error that is caught and ignored is worse than a crash. Design errors to be **loud, specific, and actionable**.
- **Design problem — validating a user registration** (age 18–120, email format, non-empty name). How should the validator report problems? Approaches: **(A) return `True`/`False`** — the caller learns *that* it failed, not *why*; can't show the user a message. **(B) return `None` on success or an error string** — works, but easy to forget to check, and strings aren't machine-readable. **(C) raise a custom `ValidationError` carrying the field and message** — impossible to ignore; carries structured data; the caller decides what to do (show, log, retry). **(D) collect *all* errors into a list and return it (a Result object)** — best for forms (show every problem at once), slightly more code. Tradeoffs: A/B are fine for internal helpers; C is right when one failure should stop processing; D is right for user-facing forms.
- **Thought process → decision:** Ask *"who handles this and what can they do?"* For a form, the user wants every problem at once → collect (D) at the form level, but each field's *individual* check should raise a specific exception (C) so it can be reused anywhere. Design an exception hierarchy with a base `RegistrationError` so callers can catch broadly or narrowly. Never `except Exception: pass`.
- **Code:**

```python
"""Day 7 — an exception hierarchy plus a collect-all-errors validator."""

import re

EMAIL = re.compile(r"^[^@\s]+@[^@\s]+\.[a-z]{2,}$", re.IGNORECASE)


class RegistrationError(Exception):
    """Base class: lets callers catch every registration problem at once."""

    def __init__(self, field: str, message: str) -> None:
        super().__init__(f"{field}: {message}")
        self.field = field
        self.message = message


class InvalidAgeError(RegistrationError):
    pass


class InvalidEmailError(RegistrationError):
    pass


def parse_age(raw: str) -> int:
    """EAFP: try to convert; translate the low-level error into a domain error."""
    try:
        age = int(raw)
    except ValueError as err:
        raise InvalidAgeError("age", f"{raw!r} is not a whole number") from err
    if not 18 <= age <= 120:
        raise InvalidAgeError("age", f"must be between 18 and 120, got {age}")
    return age


def parse_email(raw: str) -> str:
    email = raw.strip().lower()
    if not EMAIL.match(email):
        raise InvalidEmailError("email", f"{raw!r} is not a valid address")
    return email


def validate_registration(form: dict[str, str]) -> list[RegistrationError]:
    """Run every check and return *all* problems so a form can show them together."""
    errors: list[RegistrationError] = []
    if not form.get("name", "").strip():
        errors.append(RegistrationError("name", "cannot be empty"))
    for field, parser in (("age", parse_age), ("email", parse_email)):
        try:
            parser(form.get(field, ""))
        except RegistrationError as err:          # catch only what we expect — never bare `except:`
            errors.append(err)
    return errors


def main() -> None:
    form = {"name": "  ", "age": "17", "email": "not-an-email"}
    for err in validate_registration(form):
        print(f"- {err}")

    try:
        parse_age("abc")
    except InvalidAgeError as err:
        print("caught:", err, "| caused by:", repr(err.__cause__))
    finally:
        print("finally always runs — use it for cleanup (closing files, releasing locks)")


if __name__ == "__main__":
    main()
```

- **Python internals:** An exception is an ordinary object; `raise` attaches a traceback and starts **unwinding** frames until a matching `except` is found (matching = `isinstance` against the listed classes, which is why hierarchies work). Since Python 3.11, `try` blocks are **zero-cost** when no exception occurs (exception tables replace setup instructions), so EAFP is not slower on the happy path. `raise X from err` sets `__cause__` (explicit chaining); an exception raised *inside* an `except` block sets `__context__` automatically — both appear in tracebacks. `finally` runs even if you `return` inside `try`. `StopIteration` (Day 9) and `KeyboardInterrupt` are exceptions too — control flow in Python is exception-based more than most languages.
- **Build & drill:** Write `safe_divide` that raises a custom `DivisionByZeroError` with a helpful message; a `read_int(prompt)` that loops until valid; and rewrite Day 6's `remove_stock` to raise an `InsufficientStockError(sku, requested, available)` carrying attributes. Deliberately write `except Exception: pass` around a bug and observe how the program continues with wrong data — then never do it again.
- **Recall:** Why is a base exception class useful? What's the difference between `__cause__` and `__context__`? Why is "catch and ignore" worse than a crash?

### Day 8 — Modules, files, JSON, and running real programs

- **Concept & why it matters:** A **module** is a `.py` file; a **package** is a folder of modules with an `__init__.py`; `import` brings names in. `if __name__ == "__main__":` separates "this file as a script" from "this file as a library." Reading and writing **files** with `open()` inside a `with` block (a context manager that guarantees closing), `pathlib.Path` for paths, **JSON** (`json.dumps`/`loads`, `dump`/`load`) as the universal data-interchange format, and reading **environment variables** (`os.environ`). **Virtual environments** (`python -m venv .venv`) and `pip` — never install packages globally. A sensible **project layout**: `src/` or a package folder, `tests/`, `pyproject.toml`. **Why it matters for LLD:** where code lives is design too — module boundaries are the coarsest form of encapsulation, and "which module may import which" is how you keep a domain model independent of storage and UI (Day 38).
- **Real-world case study:** **The Twelve-Factor App, factor III: config.** Heroku's engineers codified what every deployed system learns painfully: configuration (database URLs, API keys, feature flags) must live *outside* the code, in the environment, so the same code runs in dev, test, and prod without edits — and so secrets never get committed to git (GitHub's secret-scanning exists because this happens thousands of times a day). Your config-loading design today is that lesson in miniature.
- **Design problem — load application settings** (database path, log level, max retries) for a program. Approaches: **(A) hard-code constants at the top of the file** — simplest, but changing the log level means editing and redeploying code. **(B) a JSON config file** — editable without touching code; but secrets in a file get committed, and the file must exist. **(C) environment variables** — the deployment standard; awkward for nested or many settings; everything arrives as a string. **(D) layered: defaults in code, overridden by a JSON file, overridden by environment variables** — the most flexible, requires clear precedence rules and type conversion.
- **Thought process → decision:** Ask *"who changes each setting and how often?"* Developers set defaults (code); operators tune per-environment (file); deployment injects secrets (env). Different owners, different lifetimes → **D**, with explicit precedence (env > file > defaults) and conversion at the boundary so the rest of the program sees `int`, not `"3"`. Missing file is *not* an error (defaults apply); a *malformed* file is (fail loudly — Day 7).
- **Code:**

```python
"""Day 8 — layered configuration: defaults <- JSON file <- environment variables.

Layout this day introduces:
    lld_journey/
      config.py      (this file)
      main.py        (imports and uses load_settings)
      settings.json  (optional, created on demand)
"""

import json
import os
from pathlib import Path

DEFAULTS: dict[str, object] = {
    "database_path": "data/app.db",
    "log_level": "INFO",
    "max_retries": 3,
}
ENV_PREFIX = "APP_"


class ConfigError(Exception):
    pass


def _read_json_file(path: Path) -> dict[str, object]:
    if not path.exists():
        return {}                                            # absent file is fine: defaults apply
    try:
        with path.open(encoding="utf-8") as handle:          # `with` closes the file even on error
            data = json.load(handle)
    except json.JSONDecodeError as err:
        raise ConfigError(f"{path} is not valid JSON: {err}") from err
    if not isinstance(data, dict):
        raise ConfigError(f"{path} must contain a JSON object at the top level")
    return data


def _read_environment(keys: list[str]) -> dict[str, object]:
    found: dict[str, object] = {}
    for key in keys:
        raw = os.environ.get(f"{ENV_PREFIX}{key.upper()}")   # e.g. APP_MAX_RETRIES=5
        if raw is not None:
            found[key] = raw
    return found


def load_settings(config_file: Path = Path("settings.json")) -> dict[str, object]:
    """Merge the three layers; convert types at the boundary; validate."""
    settings = {**DEFAULTS, **_read_json_file(config_file), **_read_environment(list(DEFAULTS))}
    try:
        settings["max_retries"] = int(settings["max_retries"])   # env vars arrive as strings
    except (TypeError, ValueError) as err:
        raise ConfigError(f"max_retries must be an integer, got {settings['max_retries']!r}") from err
    if settings["log_level"] not in {"DEBUG", "INFO", "WARNING", "ERROR"}:
        raise ConfigError(f"unknown log_level {settings['log_level']!r}")
    return settings


def save_example(config_file: Path = Path("settings.json")) -> None:
    config_file.write_text(json.dumps({"log_level": "DEBUG"}, indent=2), encoding="utf-8")


if __name__ == "__main__":
    save_example()
    os.environ["APP_MAX_RETRIES"] = "5"
    print(load_settings())        # {'database_path': 'data/app.db', 'log_level': 'DEBUG', 'max_retries': 5}
```

- **Python internals:** `import x` runs `x.py` **once**, stores the resulting module object in `sys.modules`, and every later import returns that cached object — this is why module-level state acts as a natural singleton (Day 30) and why circular imports are painful. The interpreter compiles modules to bytecode and caches it in `__pycache__/*.pyc` to skip recompiling. `__name__` is `"__main__"` only for the file you ran directly. `with open(...)` uses the **context-manager protocol** (`__enter__`/`__exit__`, Day 13) — the file is closed in `__exit__` even if an exception flies through. `json.load` produces plain `dict`/`list`/`str`/`int`/`float`/`bool`/`None` — nothing else — which is why converting to real domain types at the boundary matters.
- **Build & drill:** Create a venv, activate it, `pip install rich` and print a coloured table. Split Day 7's validator into a package `registration/` with `validators.py` and `errors.py`, plus a `main.py` that imports them. Write a script that appends every validation failure to `errors.log` with a timestamp and reads it back. Commit with a `.gitignore` that excludes `.venv/` and `__pycache__/`.
- **Recall:** What does `if __name__ == "__main__":` do? Why is `sys.modules` relevant to singletons? Why do we convert types at the boundary instead of throughout the program?

### Day 9 — Iteration in depth: iterators, generators, and lazy pipelines

- **Concept & why it matters:** Under every `for` loop is the **iterator protocol**: `iter(obj)` returns an iterator; `next(it)` returns items until `StopIteration`. You can make anything iterable by implementing `__iter__` (Day 13) — or, far more simply, by writing a **generator function** (`yield`) that pauses and resumes, producing items one at a time *lazily*. Generator expressions `(x for x in ...)`, `itertools` (`islice`, `chain`, `groupby`, `batched`), `any`/`all`, and `zip`/`enumerate` as lazy tools. **Why it matters for LLD:** lazy pipelines let a system process 100 GB with 1 MB of memory; a custom iterable is how a `ParkingLot` exposes its spots without exposing its internal list (encapsulation, Day 12); and the **Iterator pattern** (Day 36) is built into the language.
- **Real-world case study:** **Streaming vs loading.** A data team's nightly job loaded a 40 GB CSV into a list of dicts and crashed the 32 GB machine at 3 a.m. The fix was a generator pipeline (read → parse → filter → aggregate) that never held more than one line in memory — same logic, ten lines changed, memory went from 40 GB to kilobytes. Python's own `csv.reader`, file objects, and database cursors are all iterators for exactly this reason.
- **Design problem — count ERROR lines per component in a huge log file** (the Day 4 format). Approaches: **(A) `lines = f.readlines()` then loop** — simplest; memory = file size. **(B) loop directly over the file object** — lazy, but parsing/filtering/counting are tangled into one loop body. **(C) a pipeline of small generators** (`read_lines` → `parse` → `only_errors` → `count_by_component`) — each stage independently testable and reusable; still lazy end to end. Tradeoffs: A dies on big inputs; B works but grows into a God loop; C costs a little indirection for composability.
- **Thought process → decision:** The same reasoning as Day 3 (small composable functions) applied to *data flow*: each stage varies independently (the format, the filter, the aggregation), so **C**. Make each stage accept and return an *iterable*, so you can plug a list in for tests and a file in for production — that's dependency inversion (Day 24) sneaking in early.
- **Code:**

```python
"""Day 9 — a lazy generator pipeline over a log file."""

import re
from collections import Counter
from collections.abc import Iterable, Iterator
from pathlib import Path

LOG_LINE = re.compile(
    r"^(?P<date>\S+) (?P<time>\S+) (?P<level>[A-Z]+) (?P<component>[a-z_]+): (?P<message>.*)$"
)


def read_lines(path: Path) -> Iterator[str]:
    """Yield lines one at a time; the file is never fully in memory."""
    with path.open(encoding="utf-8") as handle:
        for line in handle:
            yield line.rstrip("\n")


def parse(lines: Iterable[str]) -> Iterator[dict[str, str]]:
    """Skip malformed lines but count them, instead of crashing the whole batch."""
    skipped = 0
    for line in lines:
        match = LOG_LINE.match(line)
        if match is None:
            skipped += 1
            continue
        yield match.groupdict()
    if skipped:
        print(f"[warn] skipped {skipped} malformed line(s)")


def only_level(records: Iterable[dict[str, str]], level: str) -> Iterator[dict[str, str]]:
    return (record for record in records if record["level"] == level)       # generator expression


def count_by_component(records: Iterable[dict[str, str]]) -> Counter[str]:
    return Counter(record["component"] for record in records)


def main() -> None:
    sample = Path("sample.log")
    sample.write_text(
        "2026-09-03 10:00:00 INFO auth: login user=1\n"
        "2026-09-03 10:00:01 ERROR payment: card declined\n"
        "this line is broken\n"
        "2026-09-03 10:00:02 ERROR payment: timeout\n"
        "2026-09-03 10:00:03 ERROR auth: bad token\n",
        encoding="utf-8",
    )
    pipeline = count_by_component(only_level(parse(read_lines(sample)), "ERROR"))
    print(pipeline.most_common())          # [('payment', 2), ('auth', 1)]

    # the protocol, by hand
    it = iter([10, 20])
    print(next(it), next(it))
    try:
        next(it)
    except StopIteration:
        print("exhausted — this is exactly what `for` catches for you")


if __name__ == "__main__":
    main()
```

- **Python internals:** Calling a generator function does *not* run its body; it returns a **generator object** whose frame is kept alive in a suspended state. Each `next()` resumes the frame at the last `yield` and runs to the next one; `return` raises `StopIteration`. Frames are heap-allocated in CPython, which is what makes suspension possible — and it's the same machinery `async`/`await` runs on (coroutines are generators with a different API). Generators are *single-pass*: once exhausted, they're done (call the function again for a fresh one) — a common bug is iterating a generator twice and getting nothing the second time. `Iterable` (has `__iter__`) vs `Iterator` (has `__next__` and returns itself from `__iter__`): a list is iterable, not an iterator; `iter(list)` gives a fresh iterator each time.
- **Build & drill:** Write `fibonacci()` as an infinite generator and take the first 20 with `itertools.islice`; `chunked(iterable, size)` yielding lists of `size`; and `read_csv_rows(path)` yielding dicts. Reproduce the "exhausted generator" bug and explain it in one sentence. Time the memory of `sum(range(10**7))` vs `sum(list(range(10**7)))` with `tracemalloc`.
- **Recall:** What happens when you call a generator function? Why is a generator single-pass? Iterable vs iterator — which one is a list?

### Day 10 — Consolidation I: type hints, testing, and your first real program

- **Concept & why it matters:** Two professional habits that make design possible. **Type hints** (`list[int]`, `dict[str, Decimal]`, `Optional`/`| None`, function signatures) are *executable documentation* — they let `mypy` catch a whole class of bugs before running and, more importantly for LLD, they force you to decide *what* flows between objects before deciding *how*. **Tests** with `pytest`: a test is a function named `test_*` with `assert`; `pytest.raises` for expected exceptions; `@pytest.mark.parametrize` to run one test over many cases. **Formalize Big-O**: O(1), O(log n), O(n), O(n log n), O(n²) with one example each from Days 5–6. **Why it matters for LLD:** you will hear "testability" as a design criterion forty times in this plan. Today you learn what it means physically: a unit test can only exist if the unit can be created and exercised alone. If a class is hard to test, its design is coupled — the test suite is a *design instrument*.
- **Real-world case study:** **Dropbox's 4-million-line type-checking migration.** Dropbox added type hints to its Python monolith over several years and reported that the hints found real bugs, made refactors safe, and turned "what does this function return?" from archaeology into a glance. Their lesson: types are most valuable at *module boundaries* — exactly where LLD decisions live. And on tests: SQLite's test suite is ~600 times the size of its source, which is why a 20-year-old C library is trusted in every phone on earth.
- **Design problem — an expense tracker CLI:** add expense (amount, category, date), list by category, monthly total, persist to JSON. Approaches for the *code structure*: **(A) one file, one big `main()`** — quick, untestable (everything talks to the user via `input`/`print`). **(B) split into pure domain functions (add/filter/total on a list of records), a storage module (load/save JSON), and a thin CLI layer that only parses arguments and prints** — each layer testable without the others. **(C) a class-based design** — correct, but classes arrive tomorrow; today, B proves layering works even with plain functions.
- **Thought process → decision:** *What must be testable?* The arithmetic and filtering — they hold the business rules. *What is hard to test?* I/O (files, terminal). Separate them → **B**: the domain layer never touches a file or `print`, so tests pass it plain lists. This "pure core, thin shell" shape is the seed of every architecture you'll meet later (Repository/Service Layer on Day 38).
- **Code (the domain layer plus its tests — write the storage and CLI layers yourself):**

```python
# expenses/domain.py
"""Pure business logic: no files, no printing, no input. Fully unit-testable."""

from decimal import Decimal
from datetime import date

Expense = tuple[date, str, Decimal]          # (when, category, amount) — a class on Day 14
VALID_CATEGORIES = frozenset({"food", "travel", "rent", "fun", "other"})


def make_expense(when: date, category: str, amount: Decimal) -> Expense:
    category = category.strip().lower()
    if category not in VALID_CATEGORIES:
        raise ValueError(f"unknown category {category!r}; choose from {sorted(VALID_CATEGORIES)}")
    if amount <= 0:
        raise ValueError(f"amount must be positive, got {amount}")
    return (when, category, amount)


def by_category(expenses: list[Expense], category: str) -> list[Expense]:
    return [e for e in expenses if e[1] == category]


def monthly_total(expenses: list[Expense], year: int, month: int) -> Decimal:
    return sum((e[2] for e in expenses if e[0].year == year and e[0].month == month), Decimal("0"))
```

```python
# tests/test_domain.py
import pytest
from datetime import date
from decimal import Decimal

from expenses.domain import by_category, make_expense, monthly_total


@pytest.fixture
def sample() -> list:
    return [
        make_expense(date(2026, 9, 1), "food", Decimal("250")),
        make_expense(date(2026, 9, 2), "travel", Decimal("1200")),
        make_expense(date(2026, 8, 30), "food", Decimal("99.50")),
    ]


def test_monthly_total_sums_only_that_month(sample: list) -> None:
    assert monthly_total(sample, 2026, 9) == Decimal("1450")


def test_by_category_filters(sample: list) -> None:
    assert len(by_category(sample, "food")) == 2


@pytest.mark.parametrize("bad_amount", [Decimal("0"), Decimal("-5")])
def test_rejects_non_positive_amount(bad_amount: Decimal) -> None:
    with pytest.raises(ValueError, match="positive"):
        make_expense(date(2026, 9, 1), "food", bad_amount)


def test_rejects_unknown_category() -> None:
    with pytest.raises(ValueError, match="unknown category"):
        make_expense(date(2026, 9, 1), "gadgets", Decimal("10"))
```

Run with `pytest -q` and `mypy expenses/`. Both should be green before you build the storage and CLI layers.

- **Python internals:** Type hints are stored in `__annotations__` and are **ignored at runtime** by the interpreter — `def f(x: int)` happily accepts a string; only `mypy`/Pylance check them. This is why Python hints are cheap to add and why libraries like `dataclasses` (Day 14) and Pydantic can *read* them to generate behaviour. `pytest` discovers `test_*.py` files, imports them, and rewrites `assert` statements' bytecode to produce rich failure messages — which is why a plain `assert a == b` shows both values. A fixture is just a function whose return value pytest injects by parameter *name* — your first taste of dependency injection (Day 24).
- **Checkpoint build (prove Phase 0 landed):** Finish the expense tracker with `storage.py` (load/save JSON, dates as ISO strings, `Decimal` as strings) and `cli.py` (using `argparse`: `add`, `list --category`, `total --month 2026-09`). ≥ 8 passing tests, `mypy` clean, committed. Then, from a blank file and without looking anything up, write: a function that reads a JSON file of records, validates each with a custom exception, groups them with a dict, and prints a sorted summary. If you can do that in under 30 minutes, you're ready for Phase 1. If not, repeat the weakest day — do not push on.
- **Recall:** Why are type hints "free" at runtime? What makes code *testable*, physically? Give one O(1), O(n), O(n log n), and O(n²) operation from this phase.

---

# PHASE 1 — Object-Oriented Python & the Data Model (Days 11–20)

Objects are how you *bundle state with the behaviour that keeps it consistent*. This phase teaches classes as Python actually implements them — not the Java-shaped version — because LLD in Python is done with dataclasses, protocols, dunders, and first-class functions as much as with class hierarchies. From here on, every design problem runs through the LLD method's steps 2–4 (entities → ownership → behaviours), even before the full method arrives on Day 25.

### Day 11 — Classes and objects: bundling state with behaviour

- **Concept & why it matters:** A **class** is a blueprint; an **object** (instance) is one concrete thing built from it. `__init__` sets up initial state; `self` is the instance the method was called on; **attributes** hold state; **methods** are functions that act on that state. Instance attributes (per object) vs class attributes (shared). **Why OOP exists:** on Day 6 the inventory was a dict plus free functions that *any* code could bypass (`inventory["PEN"] = -50` was legal). A class puts the data and the rules that protect it in one place, so the invariant "stock is never negative" has one owner. **Why it matters for LLD:** every box on an LLD diagram is a class; every arrow is one object holding a reference to another. Deciding what becomes a class is step 2 of the method.
- **Real-world case study:** **Simula and the bank-account tradition.** OOP was invented (Simula 67, Norway) to *simulate* real systems — ships in a harbour, customers in a bank — where each thing has its own state and rules. Fifty years on, banking software is still the clearest illustration: an account's balance must never change except through deposit/withdraw with rules attached. Every core-banking system (and every interview) starts here because it's the smallest system where "who may change this state?" has a life-or-death answer.
- **Design problem — a bank account** supporting deposit, withdraw (no overdraft), and balance. Approaches: **(A) a dict `{"owner": ..., "balance": ...}` plus functions** — nothing prevents `acct["balance"] = -1_000_000` anywhere in the codebase. **(B) a tuple** — immutable, so every operation returns a new tuple; rules live in functions again, callers can still construct an invalid tuple. **(C) a class with the balance as state and deposit/withdraw as the only doors** — the invariant has a single owner; misuse becomes a method call that raises. Tradeoffs: A/B are fine for throwaway scripts; C costs a few lines and buys a guarantee.
- **Thought process → decision:** The rubric's *"what must never be observed in an inconsistent state?"* → the balance. Anything with an invariant deserves a class → **C**. Then apply *Tell, don't ask*: callers say `account.withdraw(amount)`, they never read the balance, check it, and subtract it themselves (that "ask" version is a race condition waiting to happen — Day 27).
- **Code:**

```python
"""Day 11 — a class as the single owner of an invariant."""

from decimal import Decimal
from itertools import count


class InsufficientFundsError(Exception):
    def __init__(self, requested: Decimal, available: Decimal) -> None:
        super().__init__(f"requested {requested}, available {available}")
        self.requested = requested
        self.available = available


class BankAccount:
    """A current account that can never be overdrawn."""

    _ids = count(1)                       # class attribute: shared by all instances, used to mint ids
    MINIMUM_DEPOSIT = Decimal("1")        # class attribute: a shared constant

    def __init__(self, owner: str, opening_balance: Decimal = Decimal("0")) -> None:
        if not owner.strip():
            raise ValueError("owner name cannot be empty")
        if opening_balance < 0:
            raise ValueError("opening balance cannot be negative")
        self.account_id = next(BankAccount._ids)     # instance attributes: this object's own state
        self.owner = owner.strip()
        self._balance = opening_balance              # leading underscore: "internal — don't touch directly"

    def deposit(self, amount: Decimal) -> None:
        if amount < self.MINIMUM_DEPOSIT:
            raise ValueError(f"deposit must be at least {self.MINIMUM_DEPOSIT}, got {amount}")
        self._balance += amount

    def withdraw(self, amount: Decimal) -> None:
        if amount <= 0:
            raise ValueError(f"withdrawal must be positive, got {amount}")
        if amount > self._balance:
            raise InsufficientFundsError(amount, self._balance)
        self._balance -= amount

    def balance(self) -> Decimal:
        return self._balance

    def __repr__(self) -> str:            # what you see when you print(account) — Day 13 goes deeper
        return f"BankAccount(id={self.account_id}, owner={self.owner!r}, balance={self._balance})"


def main() -> None:
    asha = BankAccount("Asha", Decimal("500"))
    ravi = BankAccount("Ravi")
    asha.withdraw(Decimal("120"))
    ravi.deposit(Decimal("120"))
    print(asha, ravi, sep="\n")
    try:
        ravi.withdraw(Decimal("1000"))
    except InsufficientFundsError as err:
        print("refused:", err)
    print(type(asha), type(BankAccount), asha.__dict__)     # look inside


if __name__ == "__main__":
    main()
```

- **Python internals:** A class is itself an object (an instance of `type`), created when the `class` statement runs. Each instance has a `__dict__` — a plain dict of its attributes — so `asha.owner` is literally `asha.__dict__["owner"]` after a lookup miss on the class. **Attribute lookup order:** data descriptors on the type → instance `__dict__` → class `__dict__` (walking the MRO, Day 15) → `__getattr__` if defined → `AttributeError`. A method is a plain function stored on the class; `asha.withdraw` creates a **bound method** object that pre-fills `self` — which is why `self` is explicit in the definition. Class attributes are shared: mutate a *mutable* class attribute through one instance and every instance sees it (a real bug source) — reassigning through an instance creates a *new* instance attribute that shadows it. There is no `private` keyword: `_name` is a convention; `__name` triggers name mangling (Day 12).
- **Build & drill:** Add `transfer_to(other, amount)` that is all-or-nothing (if the deposit fails, the withdrawal must not have happened — think about the order of checks). Model a `Student` with grades and a `Course` with enrolled students. Then demonstrate the shared-mutable-class-attribute bug (`class A: items = []`) and fix it.
- **Recall:** What is `self`, and why must it be written? Where does `obj.attr` actually look, in order? Why is a class the right tool when there's an invariant?

### Day 12 — Encapsulation, properties, invariants, and a real `Money` type

- **Concept & why it matters:** **Encapsulation** means an object's state can only change through operations that keep it valid. Python's tools: the `_single_underscore` convention (internal), `__double_underscore` **name mangling** (accidental-override protection, not secrecy), `@property` to expose a computed or validated attribute *as if* it were a plain field, setters for validated writes, and `__slots__` / frozen dataclasses for immutability. **Value objects** — things defined by their value (money, a date range, a coordinate) — should be *immutable*: you never change ₹100, you compute a new amount. **Why it matters for LLD:** interviewers probe "how do you stop someone setting `seat.booked = True` directly?" and "what stops two threads corrupting this?" — encapsulation is the answer to the first and the precondition for the second.
- **Real-world case study:** **The Money pattern (Fowler, *Patterns of Enterprise Application Architecture*).** Nearly every finance codebase eventually writes a `Money` class after being burned three ways: float rounding (Day 1), adding rupees to dollars, and rounding a split so the parts don't sum to the whole (allocate ₹100 across 3 people → 33.33 × 3 = 99.99; one paisa vanishes). Stripe, Shopify, and every ledger system encode currency + minor units together so mixing them is impossible by construction. Design lesson: make the *illegal state unrepresentable*.
- **Design problem — represent money** for an e-commerce system. Approaches: **(A) bare `Decimal`** — exact, but ₹ and $ mix silently and rounding is ad hoc at every call site. **(B) a mutable class with `amount` and `currency` attributes** — currency-safe, but `price.amount = -5` is legal and two names for one object cause aliasing bugs when one is mutated. **(C) an immutable value object** — constructor validates once; operations return new objects; equality by value; allocation implemented once, correctly. Tradeoffs: A is fine inside a single-currency calculator; B is a false sense of safety; C is more upfront code and the standard answer.
- **Thought process → decision:** Ask *"is this thing defined by its identity or its value?"* Two ₹100 notes are interchangeable → value object → **immutable**. Immutability also makes it safely hashable and thread-safe for free. Put *every* money rule (currency match, rounding, allocation) inside the type so no call site can get it wrong → **C**. (Today with `__slots__` and read-only properties; on Day 14 you'll see `@dataclass(frozen=True)` do most of this in one line.)
- **Code:**

```python
"""Day 12 — Money as an immutable value object with validated construction."""

from __future__ import annotations

from decimal import Decimal, ROUND_HALF_EVEN


class CurrencyMismatchError(ValueError):
    pass


class Money:
    __slots__ = ("_minor", "_currency")           # fixed attribute set: less memory, no accidental new attrs
    _MINOR_UNITS = {"INR": 100, "USD": 100, "JPY": 1}

    def __init__(self, amount: Decimal | str | int, currency: str) -> None:
        currency = currency.upper()
        if currency not in self._MINOR_UNITS:
            raise ValueError(f"unsupported currency {currency!r}")
        units = self._MINOR_UNITS[currency]
        minor = (Decimal(str(amount)) * units).quantize(Decimal("1"), rounding=ROUND_HALF_EVEN)
        object.__setattr__(self, "_minor", int(minor))      # bypass our own __setattr__ guard below
        object.__setattr__(self, "_currency", currency)

    def __setattr__(self, name: str, value: object) -> None:
        raise AttributeError(f"{type(self).__name__} is immutable")

    @property
    def amount(self) -> Decimal:
        """Read-only computed view; there is deliberately no setter."""
        return Decimal(self._minor) / self._MINOR_UNITS[self._currency]

    @property
    def currency(self) -> str:
        return self._currency

    def _check_same_currency(self, other: Money) -> None:
        if self._currency != other._currency:
            raise CurrencyMismatchError(f"cannot combine {self._currency} with {other._currency}")

    def __add__(self, other: Money) -> Money:
        self._check_same_currency(other)
        return Money._from_minor(self._minor + other._minor, self._currency)

    def __sub__(self, other: Money) -> Money:
        self._check_same_currency(other)
        return Money._from_minor(self._minor - other._minor, self._currency)

    def __mul__(self, factor: int | Decimal) -> Money:
        minor = (Decimal(self._minor) * Decimal(str(factor))).quantize(Decimal("1"), rounding=ROUND_HALF_EVEN)
        return Money._from_minor(int(minor), self._currency)

    def allocate(self, shares: int) -> list[Money]:
        """Split evenly; distribute the remainder one minor unit at a time so the parts sum exactly."""
        if shares <= 0:
            raise ValueError("shares must be positive")
        base, remainder = divmod(self._minor, shares)
        return [Money._from_minor(base + (1 if i < remainder else 0), self._currency) for i in range(shares)]

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Money):
            return NotImplemented
        return (self._minor, self._currency) == (other._minor, other._currency)

    def __hash__(self) -> int:
        return hash((self._minor, self._currency))

    def __repr__(self) -> str:
        return f"Money('{self.amount}', '{self._currency}')"

    @classmethod
    def _from_minor(cls, minor: int, currency: str) -> Money:
        money = cls.__new__(cls)                       # allocate without running __init__ (see internals)
        object.__setattr__(money, "_minor", minor)
        object.__setattr__(money, "_currency", currency)
        return money


if __name__ == "__main__":
    price = Money("199.99", "INR")
    print(price * 3, price + Money("0.01", "INR"))
    print(Money("100", "INR").allocate(3))                  # [33.34, 33.33, 33.33] — sums to 100.00
    try:
        price + Money("1", "USD")
    except CurrencyMismatchError as err:
        print("refused:", err)
    try:
        price.amount = Decimal("1")  # type: ignore[misc]
    except AttributeError as err:
        print("refused:", err)
```

- **Python internals:** `@property` is a **descriptor** — an object on the *class* with `__get__`/`__set__`; because it's a *data descriptor*, it wins over the instance `__dict__` in attribute lookup, which is how `money.amount` runs code instead of reading a field. `__slots__` replaces the per-instance `__dict__` with fixed C-level slots: less memory (~40–50% for small objects), faster access, and no new attributes can be added — but subclasses without `__slots__` regain a `__dict__`. Name mangling rewrites `__x` to `_ClassName__x` at compile time; it exists to stop *subclass* name collisions, not to hide data. `cls.__new__(cls)` allocates the object *without* calling `__init__` — the low-level hook `__init__` sits on top of (Day 30's Singleton uses it). `object.__setattr__` is how you write to an instance whose own `__setattr__` is locked.
- **Build & drill:** Add `__lt__` so `sorted(prices)` works (same-currency only). Write a `DateRange` value object with `overlaps(other)` and `contains(day)` — the building block of every booking system (Day 53). Add a `Percentage` value object that refuses values outside 0–100. Test all three with pytest, including that mutation attempts raise.
- **Recall:** Why is a *property* found before the instance `__dict__`? Entity vs value object — give two examples of each. Why does immutability make an object safe to share?

### Day 13 — The Python data model: dunder methods make objects native

- **Concept & why it matters:** Python's operators, built-ins, and syntax all dispatch to **special ("dunder") methods**: `__repr__`/`__str__` (display), `__eq__`/`__hash__` (equality — the contract from Day 6), `__lt__` and friends via `functools.total_ordering`, `__len__`/`__getitem__`/`__contains__`/`__iter__` (containers), `__add__`/`__radd__` (arithmetic, with `NotImplemented` for "I don't know this type"), `__bool__`, `__call__` (objects that act like functions), and `__enter__`/`__exit__` (context managers — the `with` statement). Implementing them makes your objects behave like built-ins: sortable, printable, usable in `for`, `in`, `with`. **Why it matters for LLD:** a `ParkingLot` you can `len()`, a `Seat` you can compare, a `Transaction` you can use as `with`, an `Inventory` that supports `sku in inventory` — that is the difference between Java-in-Python and *Pythonic* design, and interviewers who know Python notice.
- **Real-world case study:** **pathlib and the `/` operator.** `Path("data") / "app.db"` reads like a path because `Path` implements `__truediv__`. `datetime - datetime` gives a `timedelta`; `Decimal` interoperates with `int` via `__radd__`; `requests.Session` and `open()` are context managers so resources close even on error. Python's standard library is a catalog of the data model done right — and `collections.abc` codifies which dunders make a *Sequence*, a *Mapping*, a *Container*.
- **Design problem — a `Deck` of playing cards** for a card-game engine (this is the standard warm-up in *Fluent Python*). Requirements: know how many cards, get the nth, iterate, shuffle, check membership, compare cards by rank. Approaches: **(A) a class with explicit methods** `size()`, `card_at(i)`, `cards()` — works; every caller must learn your API. **(B) implement `__len__`, `__getitem__`, `__contains__`** — the class now works with `len`, indexing, slicing, `for`, `in`, `random.choice`, `reversed`, and `sorted` for free. **(C) subclass `list`** — inherits everything including operations that break your invariants (`deck.append("banana")`). Tradeoffs: A is verbose and unfamiliar; B is small and native; C leaks.
- **Thought process → decision:** Ask *"does this object behave like something Python already knows?"* A deck is a sequence → implement the sequence protocol (**B**), keep the list *inside* (composition, Day 17) so only your methods can change it. Make `Card` an immutable, ordered value object so `sorted(deck)` and `card in deck` are correct. That is the pattern: *protocols for the interface, composition for the guts*.
- **Code:**

```python
"""Day 13 — a Deck that behaves like a native sequence via the data model."""

from __future__ import annotations

import random
from collections.abc import Iterator
from functools import total_ordering

RANKS = ("2", "3", "4", "5", "6", "7", "8", "9", "10", "J", "Q", "K", "A")
SUITS = ("clubs", "diamonds", "hearts", "spades")


@total_ordering                                # generates the other comparisons from __eq__ + __lt__
class Card:
    __slots__ = ("rank", "suit")

    def __init__(self, rank: str, suit: str) -> None:
        if rank not in RANKS or suit not in SUITS:
            raise ValueError(f"invalid card {rank!r} of {suit!r}")
        object.__setattr__(self, "rank", rank)
        object.__setattr__(self, "suit", suit)

    def __setattr__(self, name: str, value: object) -> None:
        raise AttributeError("Card is immutable")

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Card):
            return NotImplemented              # let Python try the other operand or fall back to identity
        return (self.rank, self.suit) == (other.rank, other.suit)

    def __hash__(self) -> int:
        return hash((self.rank, self.suit))

    def __lt__(self, other: Card) -> bool:      # rank first, then suit — one place defines "order"
        if not isinstance(other, Card):
            return NotImplemented
        return (RANKS.index(self.rank), SUITS.index(self.suit)) < (RANKS.index(other.rank), SUITS.index(other.suit))

    def __repr__(self) -> str:
        return f"Card({self.rank!r}, {self.suit!r})"

    def __str__(self) -> str:                   # human-facing; repr is developer-facing
        return f"{self.rank} of {self.suit}"


class Deck:
    """A sequence of cards. The internal list is private; the protocol is the public API."""

    def __init__(self, rng: random.Random | None = None) -> None:
        self._cards = [Card(rank, suit) for suit in SUITS for rank in RANKS]
        self._rng = rng or random.Random()      # injectable randomness makes shuffling testable

    def __len__(self) -> int:
        return len(self._cards)

    def __getitem__(self, position: int | slice) -> Card | list[Card]:
        return self._cards[position]           # delegating gives us indexing AND slicing

    def __contains__(self, card: object) -> bool:
        return card in self._cards

    def __iter__(self) -> Iterator[Card]:
        return iter(self._cards)

    def __bool__(self) -> bool:
        return bool(self._cards)

    def shuffle(self) -> None:
        self._rng.shuffle(self._cards)

    def deal(self, count: int = 1) -> list[Card]:
        if count > len(self._cards):
            raise ValueError(f"cannot deal {count} from {len(self._cards)} cards")
        dealt, self._cards = self._cards[:count], self._cards[count:]
        return dealt

    def __repr__(self) -> str:
        return f"Deck({len(self._cards)} cards)"


if __name__ == "__main__":
    deck = Deck(random.Random(42))            # seeded: reproducible for tests
    deck.shuffle()
    print(deck, deck[0], deck[-3:], Card("A", "spades") in deck)
    print(sorted(deck.deal(5)))
    print(max(deck), len(deck), "empty" if not deck else "has cards")
    for card in deck[:3]:
        print(str(card))
```

- **Python internals:** `a + b` calls `type(a).__add__(a, b)`; if that returns `NotImplemented`, Python tries `type(b).__radd__(b, a)`; if both fail → `TypeError`. Special methods are looked up on the **type**, not the instance (`obj.__len__ = ...` does nothing for `len(obj)`). `len()` calls `__len__` but also has a fast path for built-ins (reads the C struct directly) — one reason built-ins are fast. If you define `__eq__` without `__hash__`, Python sets `__hash__ = None` and your objects become unhashable — the interpreter enforcing the Day 6 contract. `__bool__` falls back to `__len__`. `__getitem__` with `int` indexes was enough to make old-style iteration work; `__iter__` is the modern protocol. `with x as y:` calls `x.__enter__()` (result bound to `y`), then `x.__exit__(exc_type, exc, tb)` — returning `True` from `__exit__` *suppresses* the exception; `contextlib.contextmanager` builds one from a generator.
- **Build & drill:** Write a `Timer` context manager (`with Timer() as t: ... ; t.elapsed`) two ways — class and `@contextmanager`. Give the Day 12 `Money` class `__radd__` so `sum(prices, Money("0","INR"))` works, and `__bool__` (zero is falsy). Build a `Matrix` with `__matmul__` (`@`) and `__eq__`. Make the Day 5 to-do list a class with `__len__`, `__iter__`, and `__contains__`.
- **Recall:** What does returning `NotImplemented` do? Why are dunders looked up on the type? What must you define alongside `__eq__`, and why?

### Day 14 — Dataclasses, enums, and the vocabulary of a domain model

- **Concept & why it matters:** `@dataclass` generates `__init__`, `__repr__`, `__eq__` (and optionally `__hash__`, ordering) from type-annotated fields; `frozen=True` gives immutability; `field(default_factory=list)` handles mutable defaults; `__post_init__` for validation; `dataclasses.replace` for "copy with changes." `Enum` (and `StrEnum`/`IntEnum`/`Flag`) models a **closed set** of options — states, kinds, roles — so `"actve"` can never sneak in. `NamedTuple` for tiny immutable records; `TypedDict` for typed JSON-shaped dicts at boundaries. **Why it matters for LLD:** these are the *nouns* of every design you'll draw: entities (dataclass with an id), value objects (frozen dataclass), enums for states and types. Choosing the right one for each concept is LLD method step 2, made concrete.
- **Real-world case study:** **Stringly-typed status fields.** A 2019 incident write-up from a logistics company: shipment status was a free string; over years the database accumulated `"delivered"`, `"Delivered"`, `"DELIVERED "`, and `"deliverd"`, each treated differently by different services. Reports were wrong for months. The fix — an enum at every boundary — is a *type-level* design decision. Equally, Python's own `http.HTTPStatus` and `logging` levels are enums for the same reason.
- **Design problem — model an `Order`** with line items, a status, a customer, and a total. Approaches: **(A) a dict** — flexible, unchecked, typos silent. **(B) a hand-written class with `__init__`, `__repr__`, `__eq__`** — correct but 40 lines of boilerplate per class that drift out of sync when a field is added. **(C) dataclasses + an enum for status** — one line per field, generated dunders that never drift, explicit mutability choice per type. Tradeoffs: A for prototypes only; B when you need full control (rare); C the default for domain models.
- **Thought process → decision:** Separate the kinds: `OrderStatus` is a closed set → **Enum**; `LineItem` is defined by its values and never changes → **frozen dataclass**; `Order` has identity and a lifecycle → **mutable dataclass with an id**, but keep mutation behind methods (`add_item`, `mark_paid`) so status transitions stay legal (the full State pattern arrives on Day 34; today a transition table is enough). **C.**
- **Code:**

```python
"""Day 14 — the domain-model vocabulary: Enum, frozen dataclass, entity dataclass."""

from __future__ import annotations

from dataclasses import dataclass, field
from datetime import datetime, timezone
from decimal import Decimal
from enum import Enum, auto
from uuid import UUID, uuid4


class OrderStatus(Enum):
    CREATED = auto()
    PAID = auto()
    SHIPPED = auto()
    CANCELLED = auto()


# Which transitions are legal. Everything not listed is forbidden.
_TRANSITIONS: dict[OrderStatus, frozenset[OrderStatus]] = {
    OrderStatus.CREATED: frozenset({OrderStatus.PAID, OrderStatus.CANCELLED}),
    OrderStatus.PAID: frozenset({OrderStatus.SHIPPED, OrderStatus.CANCELLED}),
    OrderStatus.SHIPPED: frozenset(),
    OrderStatus.CANCELLED: frozenset(),
}


class IllegalTransitionError(Exception):
    pass


@dataclass(frozen=True, slots=True)
class LineItem:
    """Value object: two identical line items are interchangeable."""
    sku: str
    unit_price: Decimal
    quantity: int

    def __post_init__(self) -> None:
        if self.quantity <= 0:
            raise ValueError(f"quantity must be positive, got {self.quantity}")
        if self.unit_price < 0:
            raise ValueError("unit price cannot be negative")

    @property
    def subtotal(self) -> Decimal:
        return self.unit_price * self.quantity


@dataclass(eq=False)                    # entities compare by identity, not by field values
class Order:
    customer_id: UUID
    order_id: UUID = field(default_factory=uuid4)
    created_at: datetime = field(default_factory=lambda: datetime.now(timezone.utc))
    status: OrderStatus = OrderStatus.CREATED
    _items: list[LineItem] = field(default_factory=list, repr=False)

    def add_item(self, item: LineItem) -> None:
        if self.status is not OrderStatus.CREATED:
            raise IllegalTransitionError(f"cannot modify an order in status {self.status.name}")
        self._items.append(item)

    @property
    def items(self) -> tuple[LineItem, ...]:      # expose a read-only view, never the list itself
        return tuple(self._items)

    @property
    def total(self) -> Decimal:
        return sum((item.subtotal for item in self._items), Decimal("0"))

    def _transition(self, target: OrderStatus) -> None:
        if target not in _TRANSITIONS[self.status]:
            raise IllegalTransitionError(f"{self.status.name} -> {target.name} is not allowed")
        self.status = target

    def mark_paid(self) -> None:
        if not self._items:
            raise ValueError("cannot pay for an empty order")
        self._transition(OrderStatus.PAID)

    def ship(self) -> None:
        self._transition(OrderStatus.SHIPPED)

    def cancel(self) -> None:
        self._transition(OrderStatus.CANCELLED)


if __name__ == "__main__":
    order = Order(customer_id=uuid4())
    order.add_item(LineItem("PEN-BLUE", Decimal("45.50"), 2))
    order.add_item(LineItem("NOTEBOOK", Decimal("120"), 1))
    print(order.total, order.status, order.items[0] == LineItem("PEN-BLUE", Decimal("45.50"), 2))
    order.mark_paid()
    order.ship()
    try:
        order.cancel()
    except IllegalTransitionError as err:
        print("refused:", err)
    print(order)
    print([s.name for s in OrderStatus], OrderStatus["PAID"], OrderStatus.PAID.value)
```

- **Python internals:** `@dataclass` is a plain decorator that *reads `__annotations__`* (Day 10's "hints are metadata" paying off) and **generates source code** for the dunders with `exec` at class-creation time — inspect with `dataclasses.fields(Order)` or read the generated `__init__` via `inspect.getsource` (it's real code). `frozen=True` installs a `__setattr__` that raises — same trick as Day 12 — and `slots=True` (3.10+) rebuilds the class with `__slots__`. Field defaults must be immutable or use `default_factory` because the class body runs once (the Day 3 mutable-default rule again). `Enum` uses a **metaclass** (`EnumMeta`) to turn class attributes into singleton members: `OrderStatus.PAID is OrderStatus.PAID` is always `True`, members are hashable, iteration order is definition order, and `OrderStatus("x")`/`OrderStatus["X"]` do value/name lookup. `eq=False` on `Order` keeps `object.__eq__` (identity) and thus keeps the object hashable by id.
- **Build & drill:** Model `Address` (frozen), `Customer` (entity), and `PaymentMethod` as a `StrEnum`. Rebuild Day 10's `Expense` tuple as a frozen dataclass and update the tests. Add a `Flag` enum `Permission` (READ | WRITE | ADMIN) and check membership with `&`. Serialize an `Order` to JSON with `dataclasses.asdict` plus a custom encoder for `Decimal`, `UUID`, `datetime`, and `Enum` — and note every place a plain dict would have let a bad value through.
- **Recall:** Entity vs value object → which dataclass options for each? Why must `Enum` be used instead of strings for status? What does `@dataclass` actually do at class creation?

### Day 15 — Inheritance, polymorphism, and the MRO

- **Concept & why it matters:** **Inheritance** lets a class reuse and specialize another (`class Savings(BankAccount)`); `super()` calls the parent's version; **overriding** replaces behaviour; **polymorphism** means calling `account.monthly_close()` on a list of mixed account types and having each do its own thing. Multiple inheritance and **mixins** (small classes that add one capability); the **Method Resolution Order (MRO)** that decides which method wins; `isinstance` vs **duck typing** ("if it quacks…"). **Why it matters for LLD:** "is-a" is the most over-used arrow in beginner designs. Today you learn both *how* to use inheritance and the two questions that decide *whether* to: does the subclass truly satisfy every promise of the parent (Liskov, Day 23)? and is the variation a *kind* of the thing or a *part* of it (composition, Day 17)?
- **Real-world case study:** **Java's `Stack extends Vector`** — a classic design mistake baked into the JDK forever: because a stack *inherits* from a growable array, you can `insertElementAt` into the middle of a "stack," breaking its whole promise. Java's own docs now say "a more complete and consistent set of LIFO operations is provided by the Deque interface." The Square/Rectangle problem is the same bug in miniature (Day 23). The industry conclusion after 30 years: **prefer composition; inherit only for true substitutability.**
- **Design problem — account types**: current (no interest, overdraft up to a limit), savings (interest, max 3 withdrawals/month), fixed deposit (no withdrawals before maturity). Approaches: **(A) one `Account` class with a `kind` field and `if kind == ...` in every method** — every new kind edits every method (the Day 2 smell at class scale). **(B) inheritance: `Account` base with shared deposit/ledger, subclasses override `withdraw` rules and add `apply_interest`** — new kind = new class; polymorphic `for acct in accounts: acct.month_end()`. **(C) composition: one `Account` holding a `WithdrawalPolicy` and an `InterestPolicy` object** — policies mix and match (a savings account with overdraft?); more objects. Tradeoffs: A fails the extension test; B is natural when kinds are few, stable, and truly "is-a"; C wins when behaviours combine independently.
- **Thought process → decision:** Run the extension test: "add a *salary account* = current account rules + interest." In B that forces either duplication or a diamond of inheritance; in C it's two policy objects. But today the requirements list three distinct kinds with no mixing → **B is sufficient and simpler**; note in the design that if policies start combining, refactor to C (Strategy, Day 31). Writing down *the condition under which you'd change the design* is what senior engineers do — the decision isn't "B forever," it's "B until X."
- **Code:**

```python
"""Day 15 — an account hierarchy with polymorphic month-end processing."""

from __future__ import annotations

from datetime import date
from decimal import Decimal


class InsufficientFundsError(Exception):
    pass


class WithdrawalNotAllowedError(Exception):
    pass


class Account:
    """Base: ledger + deposit. Subclasses specialize withdrawal rules and month-end behaviour."""

    def __init__(self, owner: str, opening_balance: Decimal = Decimal("0")) -> None:
        self.owner = owner
        self._balance = opening_balance
        self._withdrawals_this_month = 0

    @property
    def balance(self) -> Decimal:
        return self._balance

    def deposit(self, amount: Decimal) -> None:
        if amount <= 0:
            raise ValueError("deposit must be positive")
        self._balance += amount

    def withdraw(self, amount: Decimal) -> None:
        """Template: shared checks here; the *limit* rule is delegated to the subclass hook."""
        if amount <= 0:
            raise ValueError("withdrawal must be positive")
        self._check_withdrawal_allowed(amount)            # the hook
        self._balance -= amount
        self._withdrawals_this_month += 1

    def _check_withdrawal_allowed(self, amount: Decimal) -> None:
        if amount > self._balance:
            raise InsufficientFundsError(f"balance {self._balance} < {amount}")

    def month_end(self) -> None:
        """Default: just reset counters. Subclasses extend via super()."""
        self._withdrawals_this_month = 0

    def __repr__(self) -> str:
        return f"{type(self).__name__}(owner={self.owner!r}, balance={self._balance})"


class CurrentAccount(Account):
    OVERDRAFT_LIMIT = Decimal("10000")

    def _check_withdrawal_allowed(self, amount: Decimal) -> None:
        if amount > self._balance + self.OVERDRAFT_LIMIT:
            raise InsufficientFundsError(f"exceeds overdraft limit of {self.OVERDRAFT_LIMIT}")


class SavingsAccount(Account):
    MONTHLY_RATE = Decimal("0.005")
    MAX_WITHDRAWALS = 3

    def _check_withdrawal_allowed(self, amount: Decimal) -> None:
        super()._check_withdrawal_allowed(amount)          # keep the parent's rule, add one
        if self._withdrawals_this_month >= self.MAX_WITHDRAWALS:
            raise WithdrawalNotAllowedError(f"only {self.MAX_WITHDRAWALS} withdrawals per month")

    def month_end(self) -> None:
        self._balance += (self._balance * self.MONTHLY_RATE).quantize(Decimal("0.01"))
        super().month_end()


class FixedDeposit(Account):
    def __init__(self, owner: str, principal: Decimal, matures_on: date) -> None:
        super().__init__(owner, principal)
        self.matures_on = matures_on

    def _check_withdrawal_allowed(self, amount: Decimal) -> None:
        if date.today() < self.matures_on:
            raise WithdrawalNotAllowedError(f"locked until {self.matures_on}")
        super()._check_withdrawal_allowed(amount)


class AuditMixin:
    """A mixin adds one capability and expects to be combined with a real Account."""

    def withdraw(self, amount: Decimal) -> None:
        print(f"[audit] {type(self).__name__} withdraw {amount}")
        super().withdraw(amount)  # type: ignore[misc]  # cooperative: continues along the MRO


class AuditedSavings(AuditMixin, SavingsAccount):
    pass


if __name__ == "__main__":
    accounts: list[Account] = [
        CurrentAccount("Asha", Decimal("100")),
        SavingsAccount("Ravi", Decimal("10000")),
        FixedDeposit("Meera", Decimal("50000"), date(2027, 1, 1)),
        AuditedSavings("Kiran", Decimal("500")),
    ]
    accounts[0].withdraw(Decimal("5000"))            # allowed: overdraft
    accounts[3].withdraw(Decimal("100"))             # audited
    for account in accounts:                         # polymorphism: one call, four behaviours
        account.month_end()
        print(account)
    try:
        accounts[2].withdraw(Decimal("1"))
    except WithdrawalNotAllowedError as err:
        print("refused:", err)
    print([cls.__name__ for cls in AuditedSavings.__mro__])
```

- **Python internals:** Attribute lookup walks `type(obj).__mro__`, a tuple computed by the **C3 linearization** algorithm — it guarantees each class appears once, children before parents, and preserves the order you listed bases in. `super()` does *not* mean "my parent": it means "the next class after me in *the instance's* MRO" — that is why mixins that call `super()` chain correctly no matter what they're combined with (cooperative multiple inheritance), and why you must call `super().__init__()` in every class of a cooperative hierarchy. Overriding a method simply places a new entry earlier in the MRO walk. `isinstance(x, Base)` is `True` for any subclass; `type(x) is Base` is not. Duck typing means Python never *needs* inheritance for polymorphism — anything with the right methods works — which is why Day 16's Protocols exist.
- **Build & drill:** Add a `SalaryAccount` and feel the friction — then sketch (on paper) the composition version with `WithdrawalPolicy`/`InterestPolicy` objects. Build a `Shape` hierarchy (`area`, `perimeter`) and a function that totals areas polymorphically; then add a mixin that gives any shape a `describe()` method. Print the MRO of a diamond (`D(B, C)`, both from `A`) and predict which `hello()` wins before running.
- **Recall:** What does `super()` actually refer to? State the C3 guarantees. Give the two questions that decide whether inheritance is appropriate.

### Day 16 — Abstraction: ABCs, Protocols, and interfaces the Python way

- **Concept & why it matters:** An **interface** is a promise of *what* an object can do without saying *how*. Python offers three: **duck typing** (no declaration — just call the method), **Abstract Base Classes** (`abc.ABC` + `@abstractmethod`: a nominal contract; instantiation fails if a method is missing), and **Protocols** (`typing.Protocol`: *structural* typing — any class with matching methods satisfies it, checked by `mypy`, no inheritance needed). `collections.abc` supplies ready protocols (`Iterable`, `Mapping`, `Sequence`). **Why it matters for LLD:** every "pluggable" part of a design — payment gateway, notification channel, storage backend, pricing strategy — is an interface plus implementations. Which of the three Python mechanisms you pick is a real tradeoff about coupling and enforcement, and you must be able to defend it.
- **Real-world case study:** **The file-like object.** Python never defined a `File` interface; anything with `.read()`/`.write()` works — `io.StringIO`, sockets, gzip streams, HTTP response bodies. That duck-typed protocol is why `json.load(f)` accepts hundreds of sources with zero coordination. Contrast: Java's `java.sql.Driver` is a nominal interface every database vendor must implement — enforced, discoverable, but every new capability needs a spec change. Python typing's `Protocol` (PEP 544) exists to give the file-like flexibility *with* static checking.
- **Design problem — a notification sender** supporting email, SMS, and push; new channels must be addable without touching existing code. Approaches: **(A) duck typing** — write `EmailSender` with `send(msg)`, pass any object with `send`; zero ceremony, but a missing method is discovered at 3 a.m. in production, and nothing documents the contract. **(B) an ABC `Notifier`** — the contract is a named, importable thing; instantiating an incomplete implementation fails immediately; implementations must inherit (coupling to your package). **(C) a `Protocol`** — contract documented and statically checked, no inheritance required (third-party classes qualify by shape); no runtime enforcement unless `@runtime_checkable`. Tradeoffs: A for scripts and internal glue; B when you also want shared *helper* code in the base or runtime guarantees; C when implementations may come from code you don't control.
- **Thought process → decision:** Ask *"who writes the implementations, and when do I want to find out they're wrong?"* Channels are written in-house, and a broken channel should fail at *startup* (when the registry loads), not at send time → **B (ABC)** for the runtime guarantee, plus one shared template method for retries in the base. Note the alternative honestly: if channels became plugins from other teams, a Protocol would avoid forcing them to import your base class. Two mechanisms, one decision criterion.
- **Code:**

```python
"""Day 16 — an interface (ABC) with multiple implementations and a registry."""

from __future__ import annotations

from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Protocol, runtime_checkable


@dataclass(frozen=True)
class Message:
    recipient: str
    subject: str
    body: str


class DeliveryError(Exception):
    pass


class Notifier(ABC):
    """Contract: every channel can send a Message. Subclasses fill in _deliver."""

    channel_name: str = "base"
    MAX_ATTEMPTS = 3

    def send(self, message: Message) -> None:
        """Template method (Day 31): shared retry policy, channel-specific delivery."""
        last_error: Exception | None = None
        for attempt in range(1, self.MAX_ATTEMPTS + 1):
            try:
                self._deliver(message)
                return
            except DeliveryError as err:
                last_error = err
                print(f"[{self.channel_name}] attempt {attempt} failed: {err}")
        raise DeliveryError(f"{self.channel_name}: gave up after {self.MAX_ATTEMPTS} attempts") from last_error

    @abstractmethod
    def _deliver(self, message: Message) -> None:
        """Perform one delivery attempt or raise DeliveryError."""


class EmailNotifier(Notifier):
    channel_name = "email"

    def __init__(self, smtp_host: str) -> None:
        self._smtp_host = smtp_host

    def _deliver(self, message: Message) -> None:
        if "@" not in message.recipient:
            raise DeliveryError(f"{message.recipient!r} is not an email address")
        print(f"[email via {self._smtp_host}] to={message.recipient} subject={message.subject!r}")


class SmsNotifier(Notifier):
    channel_name = "sms"

    def _deliver(self, message: Message) -> None:
        if not message.recipient.startswith("+"):
            raise DeliveryError("SMS needs an international number")
        print(f"[sms] to={message.recipient} body={message.body[:40]!r}")


class NotifierRegistry:
    """Fails at registration time if a channel is incomplete — not at send time."""

    def __init__(self) -> None:
        self._channels: dict[str, Notifier] = {}

    def register(self, notifier: Notifier) -> None:
        if not isinstance(notifier, Notifier):
            raise TypeError(f"{type(notifier).__name__} does not implement Notifier")
        self._channels[notifier.channel_name] = notifier

    def send(self, channel: str, message: Message) -> None:
        try:
            notifier = self._channels[channel]
        except KeyError:
            raise LookupError(f"no channel named {channel!r}; have {sorted(self._channels)}") from None
        notifier.send(message)


# The Protocol alternative, side by side: structural — no inheritance needed.
@runtime_checkable
class SupportsSend(Protocol):
    def send(self, message: Message) -> None: ...


class SlackWebhook:                    # never heard of Notifier, yet satisfies SupportsSend by shape
    def send(self, message: Message) -> None:
        print(f"[slack] {message.subject}")


def broadcast(targets: list[SupportsSend], message: Message) -> None:
    for target in targets:
        target.send(message)


if __name__ == "__main__":
    registry = NotifierRegistry()
    registry.register(EmailNotifier("smtp.example.com"))
    registry.register(SmsNotifier())
    msg = Message("asha@example.com", "Order shipped", "Your order #42 is on its way.")
    registry.send("email", msg)
    try:
        registry.send("sms", msg)                      # wrong recipient shape → retries → gives up
    except DeliveryError as err:
        print("final:", err)
    try:
        Notifier()  # type: ignore[abstract]
    except TypeError as err:
        print("cannot instantiate:", err)
    broadcast([SlackWebhook(), EmailNotifier("smtp")], msg)
    print(isinstance(SlackWebhook(), SupportsSend))    # True — structural check at runtime
```

- **Python internals:** `ABC` uses the `ABCMeta` metaclass, which records abstract method names in `__abstractmethods__`; `object.__new__` refuses to create an instance whose class still has any — that's the runtime guarantee, and it's checked at *instantiation*, not at class definition. ABCs also support **virtual subclasses** via `Notifier.register(SomeClass)` and `__subclasshook__` — `isinstance(x, Iterable)` works for any class with `__iter__` because `collections.abc.Iterable` defines a subclasshook; that is how the standard library gets structural behaviour out of nominal ABCs. `Protocol` is a typing-time construct: `mypy` compares method signatures; at runtime it does nothing unless `@runtime_checkable`, and even then `isinstance` only checks that the *names* exist, not signatures. `abstractmethod` on a `@property` or `@classmethod` works too — the decorator just sets `__isabstractmethod__ = True`.
- **Build & drill:** Define a `Storage` ABC (`save`, `load`, `delete`) with `InMemoryStorage` and `JsonFileStorage`; write tests that run against *both* via `pytest.mark.parametrize` over the implementations — the same test suite proving the contract. Then convert `Storage` to a Protocol and note what changed in the tests (nothing) and in the type checker (it now checks shape). Make `Notifier.send` a `Protocol` your Day 10 expense tracker's storage could satisfy.
- **Recall:** Nominal vs structural typing — which is which in Python? When does an ABC's guarantee fire? Give the one question that picks ABC vs Protocol vs duck typing.

### Day 17 — Composition, aggregation, and object relationships (with UML)

- **Concept & why it matters:** Objects hold references to other objects. **Composition** ("owns"; the part dies with the whole: `Order` → `LineItem`s), **aggregation** ("has, but doesn't own": `Course` ↔ `Student`s), **association** ("uses": `PaymentService` calls a `Gateway`), and **dependency injection** — passing collaborators *in* rather than constructing them inside (so they can be swapped and tested). Reading and drawing **UML class diagrams**: boxes with attributes/methods; arrows for inheritance (hollow triangle), composition (filled diamond), aggregation (hollow diamond), association (plain line with multiplicity `1..*`). Copying: `copy.copy` vs `copy.deepcopy` and why aliasing matters in object graphs. **Why it matters for LLD:** the arrows between your classes *are* the design. "Prefer composition over inheritance" is the most quoted LLD advice, and today you learn what it means in code, not as a slogan.
- **Real-world case study:** **Game-engine entity systems.** Early game engines used deep inheritance (`GameObject → Character → Enemy → FlyingEnemy → FlyingShootingEnemy`) and collapsed under combinatorial explosion — what about a swimming, shooting, flying enemy? Modern engines (Unity, Unreal, every ECS) compose an entity from *components* (`Physics`, `Render`, `AI`), each swappable. The same shift happened in UI toolkits and in Django's class-based views (mixins over hierarchies). Composition scales with the number of *combinations*; inheritance scales with the number of *kinds*.
- **Design problem — a `Car`** with an engine, wheels, and a GPS, where engines can be petrol/electric and GPS is optional. Approaches: **(A) inheritance** — `PetrolCar`, `ElectricCar`, `PetrolCarWithGps`, … (4 classes now, 8 with one more option). **(B) composition** — `Car` *has an* `Engine` (an interface with two implementations) and an optional `Navigator`; behaviour delegated. **(C) composition + dependency injection** — B, but the `Car` receives its `Engine` and `Navigator` in the constructor rather than building them, so tests can pass a `FakeEngine` and the composition root decides the real wiring. Tradeoffs: A is fine with one axis of variation and dies with two; B is right; C is B done in a way that survives testing and configuration.
- **Thought process → decision:** Count the axes of variation: engine type × GPS presence = two independent axes → inheritance would multiply, composition adds → **C**. The DI part is the design step people skip: *who creates the parts?* If `Car.__init__` does `self.engine = PetrolEngine()`, the car is glued to petrol and untestable without a real engine. Move creation outward to a *composition root* (`main()` today; a Factory on Day 28).
- **Code:**

```python
"""Day 17 — composition + dependency injection; the object graph is the design.

UML (text form):
    Car ◆── 1 Engine            (composition: the car owns its engine)
    Car ◇── 0..1 Navigator      (aggregation: optional, may be shared/replaced)
    Engine ◁── PetrolEngine, ElectricEngine   (implementations of the Engine interface)
"""

from __future__ import annotations

import copy
from abc import ABC, abstractmethod
from dataclasses import dataclass, field


class Engine(ABC):
    @abstractmethod
    def start(self) -> str: ...

    @abstractmethod
    def consume(self, km: float) -> float:
        """Return energy used for `km` in the engine's own units (litres or kWh)."""


class PetrolEngine(Engine):
    def __init__(self, litres_per_100km: float) -> None:
        self._rate = litres_per_100km

    def start(self) -> str:
        return "vroom"

    def consume(self, km: float) -> float:
        return km * self._rate / 100


class ElectricEngine(Engine):
    def __init__(self, kwh_per_100km: float) -> None:
        self._rate = kwh_per_100km

    def start(self) -> str:
        return "hum"

    def consume(self, km: float) -> float:
        return km * self._rate / 100


@dataclass
class Navigator:
    waypoints: list[str] = field(default_factory=list)

    def route(self, destination: str) -> list[str]:
        return [*self.waypoints, destination]


class Car:
    def __init__(self, plate: str, engine: Engine, navigator: Navigator | None = None) -> None:
        if not plate.strip():
            raise ValueError("plate is required")
        self.plate = plate
        self._engine = engine               # injected: the car doesn't know or care which engine
        self._navigator = navigator         # optional collaborator
        self.odometer_km = 0.0

    def drive(self, km: float, destination: str | None = None) -> str:
        if km <= 0:
            raise ValueError("distance must be positive")
        sound = self._engine.start()
        used = self._engine.consume(km)
        self.odometer_km += km
        path = self._navigator.route(destination) if (self._navigator and destination) else []
        return f"{self.plate}: {sound}, {km} km, used {used:.2f}, route={path or 'n/a'}"


class FakeEngine(Engine):                 # a test double — possible only because Engine is injected
    def start(self) -> str:
        return "(silent)"

    def consume(self, km: float) -> float:
        return 0.0


def main() -> None:                       # the composition root: the one place that knows concrete classes
    shared_nav = Navigator(["Hebbal", "Airport Rd"])
    petrol = Car("KA01AB1234", PetrolEngine(6.5), shared_nav)
    electric = Car("KA05EV9999", ElectricEngine(15.0))
    print(petrol.drive(120, "Airport"))
    print(electric.drive(40))
    print(Car("TEST", FakeEngine()).drive(10))

    # aliasing vs copying in an object graph
    shallow = copy.copy(petrol)           # new Car, SAME engine and navigator objects
    deep = copy.deepcopy(petrol)          # new Car with its own copies of the parts
    print(shallow._engine is petrol._engine, deep._engine is petrol._engine)   # True False


if __name__ == "__main__":
    main()
```

- **Python internals:** All object relationships are **references** (pointers) — `car._engine = engine` stores an address; no copying happens. That makes composition cheap and makes *aliasing* the thing to watch: two cars given the same `Navigator` share its `waypoints` list. `copy.copy` builds a new outer object pointing at the *same* inner objects; `copy.deepcopy` recursively copies, using a `memo` dict to handle cycles and shared sub-objects; both consult `__copy__`/`__deepcopy__` if defined (Prototype pattern, Day 29). Reference **cycles** (parent ↔ child pointing at each other) are not freed by reference counting alone — CPython's generational **garbage collector** finds them; `weakref` lets a child point back at a parent without keeping it alive (Day 32's Observer uses this). An `Optional` collaborator is just a reference that may be `None`; the type hint forces you to handle it.
- **Build & drill:** Draw the UML (on paper) for the Day 14 `Order` model and for a `University` (departments, courses, professors, students) marking each arrow's kind and multiplicity. Then code the university with composition and *no* inheritance, injecting a `Clock` object into anything that needs the current date so tests can freeze time. Deliberately share a mutable part between two objects and observe the aliasing bug.
- **Recall:** Composition vs aggregation — which one implies lifetime ownership? Why does DI make code testable? What's the difference between `copy` and `deepcopy`, and when does a cycle matter?

### Day 18 — Type hints as a design tool: generics, Protocols, and typed contracts

- **Concept & why it matters:** Beyond basic hints: `Optional`/`X | None`, `Union`, `Literal`, `TypeAlias`, `Callable[[int], str]`, **generics** with `TypeVar` and `Generic[T]` (or the 3.12 `class Box[T]:` syntax), bounded type variables, `Protocol` with generics, `Final`, `ClassVar`, `TypedDict` at JSON boundaries, `overload`, `cast`, and `assert_never` for exhaustive `match` on enums. **Why it matters for LLD:** hints let you design the *contract* between two classes before writing either body — the signature *is* the design — and `mypy --strict` turns "I think this works" into "the checker agrees." A generic `Repository[T]` written once is the backbone of every persistence layer you'll build (Day 38).
- **Real-world case study:** **Instagram/Meta's Pyre and Dropbox's mypy** run over tens of millions of lines; both teams report that the highest-value hints are on *module boundaries and public interfaces* — the same places LLD decisions live — and that generic containers (`Repository[T]`, `Result[T, E]`) eliminated whole categories of "wrong object in the wrong place" bugs. Typed contracts are also how FastAPI and Pydantic generate validation and docs from signatures alone — hints as *executable design*.
- **Design problem — a repository** (save / get by id / list) for `Customer`, `Order`, and `Product`. Approaches: **(A) one repository class per entity** — three near-identical classes; a bug fix touches all three. **(B) one untyped `Repository` storing `object`** — one class, but `repo.get(id)` returns `Any` and the checker can't stop you saving an `Order` in the customer repo. **(C) a generic `Repository[T]`** — one implementation, full type safety per instantiation; `CustomerRepo = Repository[Customer]` catches misuse at check time. Tradeoffs: A is duplication; B trades safety for brevity; C is the standard answer and costs one `TypeVar`.
- **Thought process → decision:** The variation is *the entity type*, and nothing else changes → that is precisely what a type parameter expresses → **C**. Bound the type variable to a `HasId` Protocol so the repository can read `.id` without knowing the concrete class — the Protocol says the *minimum* it needs (Interface Segregation, Day 23). Return `T | None` from `get` and make the caller handle absence explicitly — no silent `None`s.
- **Code:**

```python
"""Day 18 — a generic, protocol-bounded in-memory repository."""

from __future__ import annotations

from collections.abc import Callable, Iterator
from dataclasses import dataclass, field
from typing import Literal, Protocol, TypeVar, Generic, assert_never
from uuid import UUID, uuid4
from enum import Enum


class HasId(Protocol):
    @property
    def id(self) -> UUID: ...


T = TypeVar("T", bound=HasId)              # T can be *any* type that has a UUID `id`


class DuplicateIdError(Exception):
    pass


class Repository(Generic[T]):
    """In-memory store. Same code serves every entity type, with full type checking."""

    def __init__(self) -> None:
        self._items: dict[UUID, T] = {}

    def add(self, item: T) -> None:
        if item.id in self._items:
            raise DuplicateIdError(f"id {item.id} already exists")
        self._items[item.id] = item

    def get(self, item_id: UUID) -> T | None:
        return self._items.get(item_id)

    def find(self, predicate: Callable[[T], bool]) -> list[T]:
        return [item for item in self._items.values() if predicate(item)]

    def __len__(self) -> int:
        return len(self._items)

    def __iter__(self) -> Iterator[T]:
        return iter(list(self._items.values()))    # a snapshot: safe even if mutated during iteration


class Tier(Enum):
    FREE = "free"
    PRO = "pro"


@dataclass(frozen=True)
class Customer:
    name: str
    tier: Tier
    id: UUID = field(default_factory=uuid4)


@dataclass(frozen=True)
class Product:
    title: str
    price_paise: int
    id: UUID = field(default_factory=uuid4)


def monthly_fee_paise(tier: Tier) -> int:
    match tier:                                 # exhaustive: adding a Tier member fails type-checking here
        case Tier.FREE:
            return 0
        case Tier.PRO:
            return 49_900
        case _:
            assert_never(tier)


SortOrder = Literal["asc", "desc"]


def sorted_by_price(repo: Repository[Product], order: SortOrder = "asc") -> list[Product]:
    return sorted(repo, key=lambda p: p.price_paise, reverse=(order == "desc"))


if __name__ == "__main__":
    customers: Repository[Customer] = Repository()
    products: Repository[Product] = Repository()
    customers.add(Customer("Asha", Tier.PRO))
    products.add(Product("Notebook", 12_000))
    products.add(Product("Pen", 4_550))
    print(customers.find(lambda c: c.tier is Tier.PRO))
    print([p.title for p in sorted_by_price(products, "desc")])
    print(monthly_fee_paise(Tier.PRO))
    # customers.add(Product("x", 1))   # <- uncomment: mypy reports the error; Python itself would not
```

- **Python internals:** Generics are **erased at runtime**: `Repository[Customer]` evaluates to a `_GenericAlias` that, when called, just constructs a plain `Repository` — the `[Customer]` exists for the checker only (you can read it via `__orig_class__` after construction, but don't build behaviour on it). `Generic[T]` gives the class a `__class_getitem__` so the subscript syntax works. `from __future__ import annotations` makes all annotations lazy strings (so forward references and `T | None` on older versions work); `typing.get_type_hints()` evaluates them when a library (dataclasses, Pydantic) needs to. `match` compiles to a sequence of `isinstance`/equality checks — no faster than `if`, but *exhaustiveness* is a static property `mypy` verifies via `assert_never`, which is typed as taking `Never`. `Protocol` members declared as `@property` accept both properties and plain attributes on the implementing class.
- **Build & drill:** Add `update(item)` and `remove(id)` to the repository with proper errors; write a `Result[T, E]` generic (either `Ok(value)` or `Err(error)`) and use it in Day 7's validator instead of exceptions — then write one paragraph on when each approach is better. Run `mypy --strict` on everything from Days 11–18 and fix every complaint (there will be some; each is a lesson).
- **Recall:** What does `Repository[Customer]` become at runtime? Why bound `T` to a Protocol instead of a base class? What does `assert_never` buy you?

### Day 19 — Functions as objects: closures, decorators, and functional alternatives to classes

- **Concept & why it matters:** Functions are objects (Day 3) — so they can be stored, passed, returned, and *made* by other functions. **Closures** capture variables from the enclosing scope; **decorators** are functions that take a function and return a replacement (`@timer`, `@retry`, `@lru_cache`), with `functools.wraps` to preserve metadata; decorators with arguments (a factory returning a decorator); class decorators; `functools.partial`; `lambda`. **Why it matters for LLD:** in Python, a "strategy" can be a function, a "command" can be a closure, and cross-cutting concerns (logging, retry, caching, auth) are decorators rather than subclasses. Knowing when a *function* is the right unit — instead of a one-method class — is what makes Python LLD idiomatic rather than translated from Java.
- **Real-world case study:** **Flask and FastAPI routing.** `@app.get("/users")` is a decorator that *registers* the function in a routing table — the Registry/Command patterns as three characters. Django's `@login_required`, `@cache_page`, and pytest's fixtures are the same move: behaviour layered around a function without touching its body. This is the Decorator pattern (Day 35) implemented at the language level, and it's why Python codebases have far fewer `AbstractHandlerBase` classes than Java ones.
- **Design problem — add retry, timing, and logging** to the `Notifier.send` and `Repository` operations without cluttering their bodies. Approaches: **(A) inline the logic in every method** — duplication; a change to the retry policy touches dozens of methods. **(B) subclasses / mixins** (`RetryingNotifier`, `LoggingRepository`) — works for one class at a time; combining three concerns means three mixins per class, and mixins only apply to classes. **(C) decorators** `@retry(attempts=3)`, `@timed`, `@logged` applied to any function or method — compose by stacking, reusable across unrelated classes, testable in isolation. Tradeoffs: A never; B when the concern is genuinely part of the type's identity; C for cross-cutting concerns that apply everywhere.
- **Thought process → decision:** Retry/timing/logging are *orthogonal* to what a notifier or repository *is* → they don't belong in the type hierarchy → **C**. Design the decorators to be *configurable* (a factory), *transparent* (`@wraps`), and *honest* about exceptions (re-raise after the last attempt; never swallow). Choose which exceptions are retryable explicitly — retrying a `ValueError` is a bug, retrying a `DeliveryError` is a feature.
- **Code:**

```python
"""Day 19 — cross-cutting concerns as composable decorators."""

from __future__ import annotations

import functools
import logging
import time
from collections.abc import Callable
from typing import ParamSpec, TypeVar

P = ParamSpec("P")          # preserves the decorated function's parameter types for the checker
R = TypeVar("R")

logger = logging.getLogger(__name__)


def retry(
    *, attempts: int = 3, on: tuple[type[Exception], ...] = (Exception,), backoff_seconds: float = 0.0
) -> Callable[[Callable[P, R]], Callable[P, R]]:
    """Decorator factory: retry(...) returns the actual decorator."""
    if attempts < 1:
        raise ValueError("attempts must be >= 1")

    def decorator(func: Callable[P, R]) -> Callable[P, R]:
        @functools.wraps(func)                       # keeps func.__name__, __doc__, __annotations__
        def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
            last: Exception | None = None
            for attempt in range(1, attempts + 1):
                try:
                    return func(*args, **kwargs)
                except on as err:                    # only the *retryable* exceptions
                    last = err
                    logger.warning("%s failed (attempt %d/%d): %s", func.__name__, attempt, attempts, err)
                    if attempt < attempts and backoff_seconds:
                        time.sleep(backoff_seconds * attempt)
            assert last is not None
            raise last                               # never swallow the final failure
        return wrapper
    return decorator


def timed(func: Callable[P, R]) -> Callable[P, R]:
    @functools.wraps(func)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        start = time.perf_counter()
        try:
            return func(*args, **kwargs)
        finally:                                     # timing is recorded even if the call raises
            logger.info("%s took %.3f ms", func.__name__, (time.perf_counter() - start) * 1000)
    return wrapper


def make_counter() -> Callable[[], int]:
    """A closure: `count` lives on after make_counter returns, private to this function."""
    count = 0

    def increment() -> int:
        nonlocal count                               # without this, `count += 1` would create a new local
        count += 1
        return count

    return increment


class TransientError(Exception):
    pass


class FlakyGateway:
    def __init__(self, fail_times: int) -> None:
        self._remaining_failures = fail_times

    @timed
    @retry(attempts=3, on=(TransientError,))         # applied bottom-up: retry wraps charge, timed wraps retry
    def charge(self, amount_paise: int) -> str:
        if self._remaining_failures > 0:
            self._remaining_failures -= 1
            raise TransientError("gateway timeout")
        return f"charged {amount_paise}"


if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO, format="%(levelname)s %(message)s")
    print(FlakyGateway(fail_times=2).charge(50_000))       # two warnings, then success
    print(FlakyGateway.charge.__name__, FlakyGateway.charge.__doc__)   # preserved by wraps
    tick = make_counter()
    print(tick(), tick(), tick())
    cached_square = functools.lru_cache(maxsize=None)(lambda n: n * n)   # memoization in one line
    print(cached_square(12), cached_square.cache_info())
```

- **Python internals:** A closure works because each function object carries `__closure__`, a tuple of **cell** objects shared with the enclosing frame; `nonlocal` tells the compiler to write to the cell instead of creating a new local. Decoration happens *once, at definition time*: `@d def f` is exactly `f = d(f)`, so the name `f` is rebound to whatever `d` returned — a decorator that forgets `@wraps` makes every function look like `wrapper` in tracebacks and breaks tools that read signatures. Methods are functions found on the class; the **descriptor protocol** (`function.__get__`) is what turns `obj.method` into a bound method — decorated functions are still functions, so they bind normally, which is why plain decorators work on methods. `lru_cache` stores results in a dict keyed by the arguments (which must be hashable) — a thread-safe memo you'll re-implement as a caching Proxy on Day 35. `ParamSpec` lets the checker carry the wrapped signature through.
- **Build & drill:** Write `@validate_positive` for numeric arguments, `@singleton_per_key` that caches objects by first argument, and a `@rate_limited(calls, per_seconds)` decorator (a token bucket — you'll design the full thing on Day 47). Then re-implement Day 3's discount rules as a *list of functions* and note that you have just written the Strategy pattern without a class.
- **Recall:** What does `@decorator` literally expand to? Why does `nonlocal` exist? When is a function a better "strategy" than a one-method class?

### Day 20 — Consolidation II: testing objects, test doubles, and the Library System v1

- **Concept & why it matters:** Testing object-oriented code properly: **arrange–act–assert**, one behaviour per test, fixtures for shared setup, **test doubles** (a *stub* returns canned data; a *fake* is a working lightweight implementation, like `InMemoryStorage`; a *mock* records calls, via `unittest.mock`/`pytest-mock`; a *spy* observes); `monkeypatch` for environment and time; testing exceptions and state transitions; property-based hints with `hypothesis` (optional). The design feedback loop: **if you need a mock to test a class, that class has a dependency — is it injected?** **Why it matters for LLD:** you're about to enter the design phases. Every design you produce from now on ends with tests, and the tests will tell you when a design is coupled long before an interviewer does.
- **Real-world case study:** **Testing pyramid and the "London vs Detroit" schools.** Google's testing blog ("Test Sizes") and the wider industry converged on: many fast unit tests on pure logic, fewer integration tests through real collaborators, very few end-to-end. Over-mocking (asserting *how* a class talked to its collaborators) makes tests brittle — refactor the internals and 200 tests break though behaviour is unchanged. Prefer **fakes for state, mocks only for side effects you can't observe otherwise** (emails sent, payments charged).
- **Design problem — Library Management System v1** (books, members, borrow, return, due dates, late fees; no concurrency, no persistence). This is the first full mini-LLD: run the method's steps 2–4 on paper before reading on. Approaches for the *core question — who owns "which member has which book?"*: **(A) `Book` holds `borrowed_by`** — simple, but "list a member's loans" scans every book. **(B) `Member` holds `borrowed_books`** — the mirror problem: "who has this book?" scans every member. **(C) a `Loan` entity (book, member, due date) owned by the `Library`, with indexes by book id and member id** — both queries O(1); the loan is the natural place for due-date and fee logic; return is "close the loan." Tradeoffs: A/B each answer one question well and make the other ugly and easy to desync; C adds a class and answers both.
- **Thought process → decision:** The rubric: *"what must never be inconsistent?"* — a book being borrowed by two members, or a member's list disagreeing with the book's state. One owner of that truth (**C**: the `Library`'s loan index) removes the whole class of bugs. The `Loan` also turns out to be *where the money is* (late fees), which is a hint you found the right entity: the domain's important rules land on it naturally. Inject a `Clock` so due dates are testable.
- **Code (the core plus its key tests — complete the rest yourself):**

```python
# library/core.py
"""Day 20 — Library Management v1: the Loan entity owns the borrow relationship."""

from __future__ import annotations

from collections.abc import Callable
from dataclasses import dataclass, field
from datetime import date, timedelta
from decimal import Decimal
from uuid import UUID, uuid4


class LibraryError(Exception): ...
class BookUnavailableError(LibraryError): ...
class LoanLimitReachedError(LibraryError): ...
class NoSuchLoanError(LibraryError): ...


@dataclass(frozen=True)
class Book:
    isbn: str
    title: str
    author: str


@dataclass(eq=False)
class Member:
    name: str
    id: UUID = field(default_factory=uuid4)


@dataclass(eq=False)
class Loan:
    book: Book
    member: Member
    due_on: date
    returned_on: date | None = None
    id: UUID = field(default_factory=uuid4)

    def days_late(self, today: date) -> int:
        end = self.returned_on or today
        return max((end - self.due_on).days, 0)


class Library:
    LOAN_PERIOD = timedelta(days=14)
    MAX_LOANS_PER_MEMBER = 3
    LATE_FEE_PER_DAY = Decimal("5")

    def __init__(self, clock: Callable[[], date] = date.today) -> None:
        self._clock = clock                                  # injected time → deterministic tests
        self._copies: dict[str, int] = {}                    # isbn -> copies on shelf
        self._catalogue: dict[str, Book] = {}
        self._open_loans: dict[UUID, Loan] = {}              # loan id -> loan
        self._loans_by_member: dict[UUID, set[UUID]] = {}    # member id -> loan ids

    def add_copies(self, book: Book, count: int = 1) -> None:
        if count <= 0:
            raise ValueError("count must be positive")
        self._catalogue[book.isbn] = book
        self._copies[book.isbn] = self._copies.get(book.isbn, 0) + count

    def available_copies(self, isbn: str) -> int:
        return self._copies.get(isbn, 0)

    def borrow(self, member: Member, isbn: str) -> Loan:
        if self._copies.get(isbn, 0) == 0:
            raise BookUnavailableError(f"no copies of {isbn} available")
        member_loans = self._loans_by_member.setdefault(member.id, set())
        if len(member_loans) >= self.MAX_LOANS_PER_MEMBER:
            raise LoanLimitReachedError(f"{member.name} already has {len(member_loans)} books")
        loan = Loan(self._catalogue[isbn], member, due_on=self._clock() + self.LOAN_PERIOD)
        self._copies[isbn] -= 1                      # every change to shared truth happens in one place
        self._open_loans[loan.id] = loan
        member_loans.add(loan.id)
        return loan

    def return_book(self, loan_id: UUID) -> Decimal:
        """Close the loan and return the late fee owed (zero if on time)."""
        loan = self._open_loans.pop(loan_id, None)
        if loan is None:
            raise NoSuchLoanError(f"no open loan {loan_id}")
        loan.returned_on = self._clock()
        self._loans_by_member[loan.member.id].discard(loan_id)
        self._copies[loan.book.isbn] += 1
        return self.LATE_FEE_PER_DAY * loan.days_late(loan.returned_on)

    def loans_for(self, member: Member) -> list[Loan]:
        return [self._open_loans[i] for i in self._loans_by_member.get(member.id, ())]
```

```python
# tests/test_library.py
from datetime import date, timedelta
from decimal import Decimal

import pytest

from library.core import Book, BookUnavailableError, Library, LoanLimitReachedError, Member


class FakeClock:
    """A fake: real behaviour, controllable state."""
    def __init__(self, today: date) -> None:
        self.today = today
    def __call__(self) -> date:
        return self.today
    def advance(self, days: int) -> None:
        self.today += timedelta(days=days)


@pytest.fixture
def clock() -> FakeClock:
    return FakeClock(date(2026, 9, 1))


@pytest.fixture
def library(clock: FakeClock) -> Library:
    lib = Library(clock)
    lib.add_copies(Book("978-1", "Fluent Python", "Ramalho"), count=1)
    return lib


def test_borrow_reduces_available_copies(library: Library) -> None:
    library.borrow(Member("Asha"), "978-1")
    assert library.available_copies("978-1") == 0


def test_second_borrower_is_refused(library: Library) -> None:
    library.borrow(Member("Asha"), "978-1")
    with pytest.raises(BookUnavailableError):
        library.borrow(Member("Ravi"), "978-1")


def test_late_return_charges_per_day(library: Library, clock: FakeClock) -> None:
    loan = library.borrow(Member("Asha"), "978-1")
    clock.advance(17)                                 # 14-day loan → 3 days late
    assert library.return_book(loan.id) == Decimal("15")
    assert library.available_copies("978-1") == 1


def test_loan_limit(library: Library) -> None:
    for i in range(3):
        library.add_copies(Book(f"isbn-{i}", f"Book {i}", "X"))
    member = Member("Kiran")
    for i in range(3):
        library.borrow(member, f"isbn-{i}")
    with pytest.raises(LoanLimitReachedError):
        library.borrow(member, "978-1")
```

- **Python internals:** `unittest.mock.Mock` works by implementing `__getattr__` to create child mocks on demand and `__call__` to record calls — which is why a typo'd attribute on a mock silently "works"; use `spec=RealClass` or `autospec` so mocks reject attributes the real class lacks. `monkeypatch.setattr` swaps an attribute on a module or class for the test's duration and restores it afterwards (it's a context-managed assignment). Fixtures are resolved by name through introspection of the test function's signature (`inspect.signature`) — dependency injection by parameter name. A callable `FakeClock` works wherever `date.today` does because both are callables returning a `date` — duck typing making the fake trivial.
- **Checkpoint build (prove Phase 1 landed):** Finish Library v1: reservations (queue per ISBN), a `search(title|author)` using indexes, and JSON persistence through a `Storage` interface with an in-memory fake for tests. ≥ 15 passing tests, `mypy --strict` clean. Then the *from-memory* test: with a blank file and no references, design and code a **Vending Machine v0** (products with stock and price, insert coins, select, dispense, return change) using dataclasses, an enum for state, a custom exception hierarchy, and tests — 45 minutes. If your design has a single owner for inventory and money, you're ready for Phase 2. If either is scattered, redo Days 11–14.
- **Recall:** Stub vs fake vs mock — when each? Why is an injected clock a *design* improvement, not just a testing trick? Why did the `Loan` entity make late fees easy?

---

# PHASE 2 — Design Principles (Days 21–27)

You can now build anything. This phase teaches you to build it *well* — and, more importantly, to *see* when it isn't. Principles are not rules to obey; they are named forces you weigh. Every day here refactors real code from Phase 1, because principles only mean something when applied to code you wrote and can feel the friction in.

### Day 21 — Why design matters: coupling, cohesion, smells, and the three humble rules

- **Concept & why it matters:** **Coupling** is how much one part must know about another to work; **cohesion** is how much the things inside one part belong together. Good design = low coupling, high cohesion. Fowler's **code smells**: long method, large class ("God object"), long parameter list, feature envy (a method more interested in another class's data), data clumps (the same 3 parameters travelling together → a value object), primitive obsession (strings for status, floats for money — Days 1, 14), divergent change (one class edited for many reasons), shotgun surgery (one change hits many classes), speculative generality. The humble rules: **DRY** (one source of truth for each *fact* — not "no similar-looking code"), **KISS**, **YAGNI** (build for today's requirements with tomorrow's *in mind*, not tomorrow's *in code*), and Ousterhout's **deep modules** (small interface, lots of functionality). **Why it matters for LLD:** interviewers grade your *reasoning*; naming a smell and the force behind it ("this class changes for two reasons, so I split it") is what reasoning sounds like.
- **Real-world case study:** **Healthcare.gov, 2013.** The U.S. health-insurance marketplace launched with 55 contractors' components so tightly coupled that a slow identity service froze the entire site; a change anywhere required coordinated redeploys everywhere. The rescue team's first act was drawing the dependency graph and cutting coupling (caching, queues, isolating the identity path). Compare: **Amazon's 2002 API mandate** — every team exposes data only through interfaces — which is coupling discipline enforced organizationally. Both are LLD at civilizational scale: the *shape of dependencies* decided the outcome.
- **Design problem — refactor a God object.** You're given (write it yourself first — 80 lines, one class `ReportManager` that loads CSV sales data, computes totals per region, formats a text report, emails it, and logs to a file, all in one class with a 60-line `run()` method). Approaches: **(A) leave it, add comments** — cheapest today; every future change is expensive and risky. **(B) extract methods within the class** — readable, still one class that changes for five reasons. **(C) extract classes by *reason to change*: `SalesLoader`, `RegionalSummary` (pure computation), `TextReportFormatter`, `EmailSender`, and a thin `ReportJob` that wires them** — each testable alone; the formatter can change without touching the loader. Tradeoffs: A accrues interest; B is a good first step; C is the target when the class genuinely has multiple change drivers (it does — data source, business rule, presentation, delivery are owned by different people).
- **Thought process → decision:** Ask *"who asks for changes to this, and how often?"* Five different stakeholders → five reasons to change → **C**, but *incrementally*: B first (extract methods, tests still pass), then move method groups into classes one at a time, running tests after each step. Refactoring is a sequence of behaviour-preserving steps, never a rewrite. Keep the pure computation (`RegionalSummary`) free of I/O — that's the piece whose tests are cheapest and most valuable.
- **Code (before → after, the computational core shown; do the extraction of the other classes yourself):**

```python
"""Day 21 — the smell and the extraction. BEFORE: everything in one method."""

# BEFORE (do not write code like this; type it once to feel it)
class ReportManager:
    def run(self, path, recipient):
        import csv, smtplib
        rows = list(csv.DictReader(open(path)))                   # loading
        totals = {}
        for r in rows:                                             # computing
            totals[r["region"]] = totals.get(r["region"], 0) + float(r["amount"])
        text = "Sales report\n" + "\n".join(f"{k}: {v:.2f}" for k, v in totals.items())   # formatting
        smtplib.SMTP("localhost").sendmail("me", recipient, text)  # delivering
        open("report.log", "a").write("sent\n")                    # logging


# AFTER: one reason to change per unit. The pure core:
from __future__ import annotations

from collections.abc import Iterable
from dataclasses import dataclass
from decimal import Decimal
from typing import Protocol


@dataclass(frozen=True)
class SaleRecord:
    region: str
    amount: Decimal


@dataclass(frozen=True)
class RegionalSummary:
    totals: dict[str, Decimal]

    @classmethod
    def from_sales(cls, sales: Iterable[SaleRecord]) -> RegionalSummary:
        totals: dict[str, Decimal] = {}
        for sale in sales:
            totals[sale.region] = totals.get(sale.region, Decimal("0")) + sale.amount
        return cls(totals)

    @property
    def grand_total(self) -> Decimal:
        return sum(self.totals.values(), Decimal("0"))


class ReportFormatter(Protocol):
    def format(self, summary: RegionalSummary) -> str: ...


class TextReportFormatter:
    def format(self, summary: RegionalSummary) -> str:
        lines = [f"{region:<12} {amount:>12,.2f}" for region, amount in sorted(summary.totals.items())]
        lines.append(f"{'TOTAL':<12} {summary.grand_total:>12,.2f}")
        return "Sales report\n" + "\n".join(lines)


class ReportSender(Protocol):
    def send(self, recipient: str, body: str) -> None: ...


class ReportJob:
    """Orchestrates; knows nothing about CSV, SMTP, or formatting details."""

    def __init__(self, formatter: ReportFormatter, sender: ReportSender) -> None:
        self._formatter = formatter
        self._sender = sender

    def run(self, sales: Iterable[SaleRecord], recipient: str) -> None:
        summary = RegionalSummary.from_sales(sales)
        self._sender.send(recipient, self._formatter.format(summary))
```

- **Python internals:** Coupling has a physical footprint in Python: every `import` is a dependency edge, and `python -X importtime` shows the cost; a God module drags its whole dependency tree into every process that touches it. Circular imports are the runtime symptom of two modules that should have been one or should have had a third extracted. A class's cohesion can be estimated mechanically (LCOM metrics — `radon` and `wily` compute them) by how many of its methods touch how many of its attributes. Protocols (Day 16) are the lowest-coupling interface Python has: the `ReportJob` above imports *nothing* from the formatter's or sender's modules.
- **Build & drill:** Complete the extraction (`CsvSalesLoader`, `SmtpReportSender`, a `FakeSender` for tests) and get to ≥ 6 tests without touching the network or filesystem. Then run a **smell hunt** over your own Phase 1 code: find one instance each of primitive obsession, data clump, and feature envy; fix each and describe the force in one line in the commit message.
- **Recall:** Define coupling and cohesion in one sentence each. Name five smells and the refactor for each. What does DRY actually forbid?

### Day 22 — SOLID I: Single Responsibility and Open/Closed

- **Concept & why it matters:** **SRP** — a class should have *one reason to change* (one stakeholder, one axis of variation), which is a statement about *change*, not size. **OCP** — software should be open for *extension* but closed for *modification*: adding a new case should mean adding code (a new class/function/table row), not editing existing tested code. In Python OCP is achieved through polymorphism (ABC/Protocol implementations), **registries** (a dict from key to handler that new code registers into), and data-driven tables (Day 2). **Why it matters for LLD:** the *extension test* in LLD method step 6 is OCP applied: "add a new vehicle type / payment method / notification channel — what do you touch?" The right answer is always "one new class and one registration line."
- **Real-world case study:** **Payment-method sprawl.** Every e-commerce codebase starts with `if method == "card": … elif method == "upi": … elif method == "netbanking": …` inside `checkout()`. Two years later there are 14 methods, the `if` chain is 600 lines, and adding "buy now, pay later" requires touching checkout, refunds, reporting, and reconciliation — four places, four regressions. Companies like Shopify and Razorpay describe the same fix: a `PaymentMethod` interface, one class per method, a registry, and a checkout that never changes again. That's OCP with a business case attached.
- **Design problem — an invoice exporter** that can produce PDF, CSV, and JSON, and will need XML next quarter. Approaches: **(A) `export(invoice, fmt)` with an `if/elif` on `fmt`** — every new format modifies and re-tests the function; violates OCP. **(B) subclasses of `Invoice` (`PdfInvoice`, `CsvInvoice`)** — mixes what an invoice *is* with how it's *rendered*; an invoice would need 4 classes and can't be exported two ways; violates SRP. **(C) an `Exporter` interface, one class per format, a registry mapping format name → exporter** — new format = new class + one line; `Invoice` stays a pure domain object. Tradeoffs: A is fine for a script with two formats that will never change; B is a common mistake; C is the standard shape (and is the Strategy pattern, Day 31, before it has a name).
- **Thought process → decision:** Two axes are entangled in A/B: the *invoice's data rules* (finance owns) and the *output formats* (integration team owns). Different owners → SRP says split → **C**. Make the registry a small class with `register`/`get` and clear errors, rather than a bare module dict, so a typo'd format fails loudly. Use a decorator to register (Day 19) so adding a format is genuinely one file.
- **Code:**

```python
"""Day 22 — SRP + OCP: invoice rendering as pluggable exporters behind a registry."""

from __future__ import annotations

import csv
import io
import json
from collections.abc import Callable
from dataclasses import dataclass
from decimal import Decimal
from typing import Protocol


@dataclass(frozen=True)
class InvoiceLine:
    description: str
    amount: Decimal


@dataclass(frozen=True)
class Invoice:                                   # SRP: only invoice data and invoice rules live here
    number: str
    customer: str
    lines: tuple[InvoiceLine, ...]

    def __post_init__(self) -> None:
        if not self.lines:
            raise ValueError("an invoice needs at least one line")

    @property
    def total(self) -> Decimal:
        return sum((line.amount for line in self.lines), Decimal("0"))


class Exporter(Protocol):
    def export(self, invoice: Invoice) -> bytes: ...


class ExporterRegistry:
    """OCP: adding a format never edits this class or the exporters that exist."""

    def __init__(self) -> None:
        self._exporters: dict[str, Exporter] = {}

    def register(self, fmt: str) -> Callable[[type[Exporter]], type[Exporter]]:
        def decorator(cls: type[Exporter]) -> type[Exporter]:
            key = fmt.lower()
            if key in self._exporters:
                raise ValueError(f"format {key!r} already registered")
            self._exporters[key] = cls()
            return cls
        return decorator

    def export(self, invoice: Invoice, fmt: str) -> bytes:
        try:
            exporter = self._exporters[fmt.lower()]
        except KeyError:
            raise LookupError(f"unknown format {fmt!r}; available: {sorted(self._exporters)}") from None
        return exporter.export(invoice)


exporters = ExporterRegistry()


@exporters.register("json")
class JsonExporter:
    def export(self, invoice: Invoice) -> bytes:
        payload = {
            "number": invoice.number,
            "customer": invoice.customer,
            "lines": [{"description": l.description, "amount": str(l.amount)} for l in invoice.lines],
            "total": str(invoice.total),
        }
        return json.dumps(payload, indent=2).encode("utf-8")


@exporters.register("csv")
class CsvExporter:
    def export(self, invoice: Invoice) -> bytes:
        buffer = io.StringIO()
        writer = csv.writer(buffer)
        writer.writerow(["number", "customer", "description", "amount"])
        for line in invoice.lines:
            writer.writerow([invoice.number, invoice.customer, line.description, f"{line.amount:.2f}"])
        return buffer.getvalue().encode("utf-8")


# Next quarter's XML exporter is a new file containing only a class with @exporters.register("xml").

if __name__ == "__main__":
    invoice = Invoice("INV-001", "Asha", (InvoiceLine("Consulting", Decimal("12000")), InvoiceLine("Travel", Decimal("850.50"))))
    print(exporters.export(invoice, "json").decode())
    print(exporters.export(invoice, "CSV").decode())
    try:
        exporters.export(invoice, "xml")
    except LookupError as err:
        print(err)
```

- **Python internals:** A class-decorator registry works because the `class` statement is executed code: the decorator runs at *import time*, so a format is available the moment its module is imported — which means your composition root must import the plugin modules (or use `importlib.import_module`/entry points for true plugins). Python's `functools.singledispatch` is a built-in OCP mechanism for functions: register a new implementation per type without editing the original. `__init_subclass__` is another: a base class can auto-register every subclass at definition time — see Day 28.
- **Build & drill:** Add the XML exporter *without editing any existing file* except an import in `main`. Apply SRP to Day 20's `Library`: is fee calculation a separate responsibility (`FeePolicy`)? Argue both sides in five lines, then decide. Find one OCP violation in your Phase 1 code (any `if isinstance(...)` chain or `if kind == ...`) and refactor it to polymorphism or a registry.
- **Recall:** SRP is about *what*, not size? OCP: extension vs modification — give the three Python mechanisms. Why does a registry beat a bare module-level dict?

### Day 23 — SOLID II: Liskov Substitution and Interface Segregation

- **Concept & why it matters:** **LSP** — a subclass must be usable anywhere its parent is, *without the caller knowing*: it may not strengthen preconditions (reject inputs the parent accepts), weaken postconditions (return less than promised), throw new exception types, or break invariants. Violations look like `isinstance` checks in callers and `raise NotImplementedError` in subclasses. **ISP** — clients shouldn't depend on methods they don't use: prefer several small interfaces (`Readable`, `Writable`) over one fat one (`Storage` with 12 methods every implementation must stub). In Python ISP is nearly free with Protocols (Day 16). **Why it matters for LLD:** LSP is the test for whether inheritance is legitimate at all (Day 15's question, formalized); ISP is why your Day 18 repository bound `T` to a tiny `HasId` and not to a 10-method base class.
- **Real-world case study:** **Square/Rectangle, and Python's own `collections.abc`.** The textbook LSP failure: `Square(Rectangle)` with `set_width` that also changes height breaks any code that sets width then asserts height unchanged. The real-world echo is Java's `Stack extends Vector` (Day 15) and Python's early `UserDict` woes. Python's answer is a *segregated* hierarchy: `Sized`, `Iterable`, `Container` → `Collection` → `Sequence` → `MutableSequence`. A read-only view implements `Sequence` and simply *isn't* a `MutableSequence` — no `raise NotImplementedError("read only")`, no LSP breach. Interface segregation *prevents* Liskov violations.
- **Design problem — a document storage abstraction** used by three clients: an uploader (writes), a viewer (reads), and a cleanup job (deletes old files), with S3, local disk, and a read-only archive backend. Approaches: **(A) one `Storage` ABC with `read/write/delete/list/copy/move`** — the read-only archive must stub `write`/`delete` with `NotImplementedError` (LSP breach); every client depends on the fat interface. **(B) segregated protocols `Readable`, `Writable`, `Deletable`, `Listable`** — each client declares only what it needs; the archive implements `Readable` + `Listable` and is *correctly* rejected by the type checker if passed to the uploader. **(C) capability flags (`storage.can_write`) with runtime checks** — works, but every caller must remember to check; failures at runtime instead of check time. Tradeoffs: A is the common mistake; C is a workaround; B is the principle applied.
- **Thought process → decision:** Ask *"does every implementation honestly support every method?"* No → the interface is too fat → split by *client need*, not by *implementation convenience* → **B**. Combine with intersection where a client needs two (`class ReadWrite(Readable, Writable, Protocol)`). The result: a type-level guarantee that a read-only backend can never reach code that writes — the illegal state is unrepresentable (Day 12's principle at the interface level).
- **Code:**

```python
"""Day 23 — LSP + ISP: segregated storage protocols; a read-only backend that never lies."""

from __future__ import annotations

from collections.abc import Iterator
from pathlib import Path
from typing import Protocol


class Readable(Protocol):
    def read(self, key: str) -> bytes: ...


class Writable(Protocol):
    def write(self, key: str, data: bytes) -> None: ...


class Deletable(Protocol):
    def delete(self, key: str) -> None: ...


class Listable(Protocol):
    def keys(self) -> Iterator[str]: ...


class ReadWrite(Readable, Writable, Protocol):        # intersection for clients that need both
    ...


class LocalDiskStorage:                               # satisfies all four protocols by shape
    def __init__(self, root: Path) -> None:
        self._root = root
        self._root.mkdir(parents=True, exist_ok=True)

    def _path(self, key: str) -> Path:
        if "/" in key or key.startswith("."):
            raise ValueError(f"invalid key {key!r}")   # a precondition the *interface* documents for all impls
        return self._root / key

    def read(self, key: str) -> bytes:
        try:
            return self._path(key).read_bytes()
        except FileNotFoundError:
            raise KeyError(key) from None              # same exception contract as every other backend

    def write(self, key: str, data: bytes) -> None:
        self._path(key).write_bytes(data)

    def delete(self, key: str) -> None:
        path = self._path(key)
        if not path.exists():
            raise KeyError(key)
        path.unlink()

    def keys(self) -> Iterator[str]:
        return (p.name for p in self._root.iterdir() if p.is_file())


class ReadOnlyArchive:                                # Readable + Listable only. Not Writable — honestly.
    def __init__(self, snapshot: dict[str, bytes]) -> None:
        self._snapshot = dict(snapshot)

    def read(self, key: str) -> bytes:
        return self._snapshot[key]                    # KeyError on miss: same contract

    def keys(self) -> Iterator[str]:
        return iter(sorted(self._snapshot))


class Uploader:
    def __init__(self, storage: Writable) -> None:    # depends on the smallest capability it needs
        self._storage = storage

    def upload(self, key: str, payload: bytes) -> None:
        if not payload:
            raise ValueError("refusing to upload an empty payload")
        self._storage.write(key, payload)


class Viewer:
    def __init__(self, storage: Readable) -> None:
        self._storage = storage

    def show(self, key: str) -> str:
        return self._storage.read(key).decode("utf-8", errors="replace")


class CleanupJob:
    def __init__(self, storage: Listable, remover: Deletable) -> None:   # two roles, possibly two objects
        self._storage, self._remover = storage, remover

    def purge(self, prefix: str) -> int:
        doomed = [k for k in self._storage.keys() if k.startswith(prefix)]
        for key in doomed:
            self._remover.delete(key)
        return len(doomed)


if __name__ == "__main__":
    disk = LocalDiskStorage(Path("storage_demo"))
    Uploader(disk).upload("tmp-report.txt", b"hello")
    print(Viewer(disk).show("tmp-report.txt"))
    print(CleanupJob(disk, disk).purge("tmp-"))
    archive = ReadOnlyArchive({"2025-audit.txt": b"frozen"})
    print(Viewer(archive).show("2025-audit.txt"))
    # Uploader(archive)   # <- mypy: "ReadOnlyArchive" has no attribute "write". Caught before it runs.
```

- **Python internals:** `mypy` checks Protocol conformance *structurally* by comparing each member's signature, including parameter names and return types — an implementation whose `read` takes `(self, key: int)` fails, which is LSP's "no strengthened preconditions" enforced mechanically. The `collections.abc` hierarchy is the standard library's own ISP: `Sequence` requires only `__getitem__` and `__len__`, then *derives* `__contains__`, `__iter__`, `index`, `count` as mixin methods — implement two, get six. When you subclass an ABC, the mixin methods come along; when you satisfy a Protocol, they don't — one real reason to sometimes prefer ABCs (Day 16). Contravariance/covariance: a method that *accepts* a broader type or *returns* a narrower type than the parent is LSP-safe, and `mypy` understands this for overrides.
- **Build & drill:** Write the Square/Rectangle violation, a test that exposes it, and two fixes (immutable shapes; separate types with a shared `Shape` protocol). Go back to Day 15's accounts: does `FixedDeposit.withdraw` raising `WithdrawalNotAllowedError` break LSP for a caller that only expects `InsufficientFundsError`? Fix it (hint: a common base exception is part of the contract). Split Day 20's `Storage` ABC into segregated protocols.
- **Recall:** List the four LSP rules. Why does ISP *prevent* LSP violations? What does `mypy` check when it checks a Protocol?

### Day 24 — SOLID III: Dependency Inversion, injection, and the composition root

- **Concept & why it matters:** **DIP** — high-level policy (what the business does) must not depend on low-level detail (how bytes get stored or sent); *both* depend on abstractions, and the abstraction is *owned by the high-level side*. The mechanism is **Dependency Injection**: constructor injection (default), setter/parameter injection where a dependency varies per call, and a single **composition root** (`main()`, an app factory) where concrete classes are chosen and wired — nowhere else. DI *containers* exist in Python (`dependency-injector`, `punq`) but are rarely needed: a `main()` that builds the graph is enough. **Why it matters for LLD:** DIP is the principle behind "how would you test this?", "how would you swap the database?", and "how would you run this without a network?" — three questions asked in every LLD interview. If your `OrderService` does `PaymentGateway()` inside `__init__`, all three answers are "I can't."
- **Real-world case study:** **The Clean/Hexagonal architecture story.** Cockburn's "ports and adapters" and Martin's "Clean Architecture" both formalize what teams at companies like Netflix and Uber describe when they migrated payment providers or databases: the domain defines *ports* (interfaces it needs); infrastructure supplies *adapters*; swapping Stripe for Adyen touches one adapter and the composition root. Teams *without* this shape describe multi-quarter migrations because provider-specific code was smeared across the business logic. Percival & Gregory's *Architecture Patterns with Python* is this story in Python.
- **Design problem — an order-checkout service** that reserves inventory, charges a payment gateway, and sends a confirmation, testable without any real external system. Approaches: **(A) `CheckoutService` constructs `StripeGateway()`, `SmtpMailer()`, `PostgresInventory()` internally** — zero flexibility; tests need real services or monkeypatching internals. **(B) module-level globals/singletons that the service imports** — hidden dependencies; tests fight the global state; order of imports matters. **(C) the service declares Protocols for what it *needs* (`PaymentGateway`, `InventoryPort`, `Notifier`) and receives implementations via the constructor; `main()` wires the real ones, tests wire fakes** — explicit, swappable, testable. Tradeoffs: A is the default beginner shape; B is A with extra steps; C costs a few Protocols and a `main()` and is the industry norm.
- **Thought process → decision:** Ask *"which direction do the imports point?"* In A, the domain imports Stripe (high depends on low — the arrow is backwards). Invert it: the domain *defines* `PaymentGateway`; the Stripe adapter imports the domain's protocol → **C**. Also decide the *failure design*: if payment succeeds and notification fails, the order is still paid — so notification is best-effort and logged, while inventory release on payment failure is mandatory. Compensation logic belongs in the service, not the adapters.
- **Code:**

```python
"""Day 24 — Dependency Inversion: the service owns the ports; adapters plug in at the root."""

from __future__ import annotations

import logging
from dataclasses import dataclass
from decimal import Decimal
from typing import Protocol
from uuid import UUID, uuid4

logger = logging.getLogger(__name__)


# ---- the domain owns these abstractions (ports) --------------------------------------------
class PaymentGateway(Protocol):
    def charge(self, customer_id: UUID, amount: Decimal) -> str:
        """Return a payment reference or raise PaymentDeclined."""


class InventoryPort(Protocol):
    def reserve(self, sku: str, quantity: int) -> None: ...
    def release(self, sku: str, quantity: int) -> None: ...


class Notifier(Protocol):
    def notify(self, customer_id: UUID, text: str) -> None: ...


class PaymentDeclined(Exception): ...
class OutOfStock(Exception): ...


@dataclass(frozen=True)
class CheckoutRequest:
    customer_id: UUID
    sku: str
    quantity: int
    unit_price: Decimal

    @property
    def total(self) -> Decimal:
        return self.unit_price * self.quantity


@dataclass(frozen=True)
class Receipt:
    order_id: UUID
    payment_ref: str
    total: Decimal


class CheckoutService:
    """High-level policy. Imports nothing concrete; testable with fakes."""

    def __init__(self, payments: PaymentGateway, inventory: InventoryPort, notifier: Notifier) -> None:
        self._payments = payments
        self._inventory = inventory
        self._notifier = notifier

    def checkout(self, request: CheckoutRequest) -> Receipt:
        if request.quantity <= 0:
            raise ValueError("quantity must be positive")
        self._inventory.reserve(request.sku, request.quantity)          # may raise OutOfStock
        try:
            payment_ref = self._payments.charge(request.customer_id, request.total)
        except PaymentDeclined:
            self._inventory.release(request.sku, request.quantity)      # compensation: never strand stock
            raise
        receipt = Receipt(uuid4(), payment_ref, request.total)
        try:                                                            # best-effort: a failed email must not undo a sale
            self._notifier.notify(request.customer_id, f"Order {receipt.order_id} confirmed: {receipt.total}")
        except Exception:                                               # noqa: BLE001 — logged, deliberately swallowed
            logger.exception("confirmation failed for order %s", receipt.order_id)
        return receipt


# ---- adapters (low level) depend on the domain's ports, never the reverse --------------------
class InMemoryInventory:
    def __init__(self, stock: dict[str, int]) -> None:
        self._stock = dict(stock)

    def reserve(self, sku: str, quantity: int) -> None:
        if self._stock.get(sku, 0) < quantity:
            raise OutOfStock(sku)
        self._stock[sku] -= quantity

    def release(self, sku: str, quantity: int) -> None:
        self._stock[sku] = self._stock.get(sku, 0) + quantity

    def on_hand(self, sku: str) -> int:
        return self._stock.get(sku, 0)


class AlwaysDeclines:                          # a fake for tests
    def charge(self, customer_id: UUID, amount: Decimal) -> str:
        raise PaymentDeclined("test decline")


class ConsoleNotifier:
    def notify(self, customer_id: UUID, text: str) -> None:
        print(f"[notify {customer_id}] {text}")


class FakeGateway:
    def charge(self, customer_id: UUID, amount: Decimal) -> str:
        return f"pay_{customer_id.hex[:8]}_{amount}"


def main() -> None:                            # the composition root: the ONLY place concrete classes meet
    logging.basicConfig(level=logging.INFO)
    inventory = InMemoryInventory({"PEN": 5})
    service = CheckoutService(FakeGateway(), inventory, ConsoleNotifier())
    print(service.checkout(CheckoutRequest(uuid4(), "PEN", 2, Decimal("45.50"))))

    declining = CheckoutService(AlwaysDeclines(), inventory, ConsoleNotifier())
    try:
        declining.checkout(CheckoutRequest(uuid4(), "PEN", 3, Decimal("45.50")))
    except PaymentDeclined:
        print("declined; stock released back to", inventory.on_hand("PEN"))    # 3, not 0


if __name__ == "__main__":
    main()
```

- **Python internals:** DIP in Python is about *import direction*: `from stripe_adapter import StripeGateway` inside the domain module makes the domain depend on Stripe *at import time* — even if unused — and drags its dependency tree into every test. Protocols make the dependency point the other way without an ABC import. The type checker enforces the arrow: the adapter module must import the domain to annotate against its Protocol (or just match by shape). `logger.exception` records the traceback at ERROR level from inside an `except` — the right tool when you deliberately swallow. Note `# noqa: BLE001`: linters (ruff) flag blind `except Exception` for good reason (Day 7); when you do it *on purpose*, you say so in code.
- **Build & drill:** Write the tests: successful checkout reduces stock; declined payment restores stock; notifier failure still returns a receipt (use a notifier that raises). Then refactor Day 20's `Library` to receive a `LoanRepository` port and provide in-memory and JSON adapters. Draw the dependency arrows before and after: every arrow must point *toward* the domain.
- **Recall:** What does "inversion" invert? Where do concrete classes get chosen, and why only there? Why is notification best-effort but inventory release mandatory?

### Day 25 — The LLD method, end to end: from requirements to classes (Vending Machine)

- **Concept & why it matters:** Today you learn the *procedure* you'll run on every remaining problem (it's stated at the top of this plan; today it becomes muscle). **Object-oriented analysis:** requirements → **use cases** (actor + goal) → **nouns** (candidate entities/value objects/enums) → **verbs** (candidate behaviours) → responsibility assignment (**GRASP**: *Information Expert* — give the job to the object with the data; *Creator* — the object that aggregates creates; *Controller* — a coordinating facade for a use case; *Low Coupling / High Cohesion*; *Polymorphism* over type-switching; *Pure Fabrication* — invent a class like `Loan` when no real-world noun fits). Drawing **class diagrams** and **sequence diagrams** (who calls whom, in order) for the main scenario. **Why it matters for LLD:** interviews are 45 minutes; without a procedure you freeze or code too early. With one, you always have a next step.
- **Real-world case study:** **Vending machines as the canonical state machine.** Real machine controllers (and every LLD book) use them because they exhibit every design force in miniature: state-dependent behaviour (idle / has-money / dispensing), a closed set of actions per state, money with exactness rules, inventory with consistency rules, and a hardware boundary. The 1990s embedded-systems literature (and the Gang of Four's State pattern chapter) draws on them for exactly this reason.
- **Design problem — the full Vending Machine**, run through all six steps (do it on paper for 40 minutes *before* reading this walkthrough):
  1. **Requirements:** products with price & stock; accept coins/notes of fixed denominations; select product; dispense if paid enough; return change with fewest coins; cancel returns money; admin restocks. Out of scope: multiple currencies, card payments, networking.
  2. **Entities/values/enums:** `Product` (value), `Slot` (entity: product + stock), `Denomination` (enum), `Money` (Day 12), `Transaction` (entity: inserted money + selection), `MachineState` (enum or State classes), `VendingMachine` (controller/aggregate).
  3. **Ownership:** `Inventory` owns slots and stock consistency; `CashBox` owns coins on hand and the *change-making* algorithm (Information Expert: it has the coin counts); `VendingMachine` owns the current transaction and state.
  4. **Behaviours:** `insert(denomination)`, `select(code)`, `dispense()`, `cancel()`, `restock(code, qty)`; each valid only in some states.
  5. **Forces → patterns:** state-dependent behaviour → *State* (today: an enum + transition guards; Day 34 upgrades to State classes so you can compare); change-making strategy might vary → keep it in `CashBox` behind a method; product creation → simple factory later.
  6. **Scenarios:** exact payment; overpayment with change; insufficient funds then more coins; out of stock after selection; cannot make exact change (refuse sale and refund — a real edge case); cancel mid-transaction. Extension test: "add card payment" → a `PaymentSource` abstraction wraps `CashBox`, machine unchanged.
- **Thought process → decision:** The interesting fork is *where change-making lives*. Approaches: **(A) in `VendingMachine`** — it becomes a God class (it already coordinates). **(B) in `CashBox`** — the object that knows the coin counts computes change (Information Expert), and its algorithm can be greedy today, DP tomorrow, with no caller change. **(C) a separate `ChangeMaker` strategy** — justified only when multiple algorithms are required now. Choose **B**; note C as the refactor if a second algorithm appears. Second fork: *enum state with guards vs State classes* — with 4 states and 5 actions, the enum table is smaller and readable; State classes win when per-state behaviour grows (Day 34 revisits with the same machine so you can feel the crossover).
- **Code (the coordinator and cash box; the inventory is yours to write from the Day 6/11 patterns):**

```python
"""Day 25 — Vending Machine: the LLD method applied. Enum state + guards; CashBox owns change-making."""

from __future__ import annotations

from collections import Counter
from dataclasses import dataclass
from enum import Enum, IntEnum


class Denomination(IntEnum):        # value = paise, so arithmetic and sorting just work
    ONE = 100
    TWO = 200
    FIVE = 500
    TEN = 1_000
    TWENTY = 2_000
    FIFTY = 5_000


class State(Enum):
    IDLE = "idle"
    COLLECTING = "collecting"     # money inserted, no product yet
    READY = "ready"               # product selected and fully paid
    OUT_OF_SERVICE = "out_of_service"


class VendingError(Exception): ...
class InvalidActionError(VendingError): ...
class OutOfStockError(VendingError): ...
class CannotMakeChangeError(VendingError): ...


@dataclass(frozen=True)
class Product:
    code: str
    name: str
    price_paise: int


class Inventory:
    """Owns slot stock. You write this: add(product, qty), available(code) -> int, take(code), product(code)."""


class CashBox:
    """Information Expert: it holds the coins, so it computes change."""

    def __init__(self, float_coins: dict[Denomination, int] | None = None) -> None:
        self._coins: Counter[Denomination] = Counter(float_coins or {})

    def accept(self, coin: Denomination) -> None:
        self._coins[coin] += 1

    def make_change(self, amount_paise: int) -> list[Denomination]:
        """Greedy, largest first. Simple and usually right; with limited coin counts it can miss a
        feasible combination (a DP is the upgrade — see the drill). Raises if impossible."""
        if amount_paise < 0:
            raise ValueError("change cannot be negative")
        plan: list[Denomination] = []
        remaining = amount_paise
        for coin in sorted(Denomination, reverse=True):
            while remaining >= coin and self._coins[coin] - plan.count(coin) > 0:
                plan.append(coin)
                remaining -= coin
        if remaining:
            raise CannotMakeChangeError(f"cannot return {amount_paise} paise with coins on hand")
        for coin in plan:
            self._coins[coin] -= 1
        return plan

    def total_paise(self) -> int:
        return sum(coin * count for coin, count in self._coins.items())


class VendingMachine:
    def __init__(self, inventory: Inventory, cashbox: CashBox) -> None:
        self._inventory = inventory
        self._cashbox = cashbox
        self._state = State.IDLE
        self._inserted: list[Denomination] = []
        self._selected: Product | None = None

    @property
    def state(self) -> State:
        return self._state

    @property
    def inserted_paise(self) -> int:
        return sum(self._inserted)

    def _require(self, *allowed: State) -> None:
        if self._state not in allowed:
            raise InvalidActionError(f"not allowed while {self._state.value}")

    def insert(self, coin: Denomination) -> None:
        self._require(State.IDLE, State.COLLECTING, State.READY)
        self._inserted.append(coin)
        self._state = State.COLLECTING
        self._refresh_readiness()

    def select(self, code: str) -> None:
        self._require(State.IDLE, State.COLLECTING)
        if self._inventory.available(code) == 0:
            raise OutOfStockError(code)
        self._selected = self._inventory.product(code)
        self._state = State.COLLECTING
        self._refresh_readiness()

    def _refresh_readiness(self) -> None:
        if self._selected is not None and self.inserted_paise >= self._selected.price_paise:
            self._state = State.READY

    def dispense(self) -> tuple[Product, list[Denomination]]:
        self._require(State.READY)
        assert self._selected is not None
        change_due = self.inserted_paise - self._selected.price_paise
        for coin in self._inserted:                      # money becomes the machine's before change is computed
            self._cashbox.accept(coin)
        try:
            change = self._cashbox.make_change(change_due)
        except CannotMakeChangeError:
            self._cashbox.make_change(self.inserted_paise)            # undo: give everything back
            self._reset()
            raise VendingError("exact change unavailable; refunded") from None
        product = self._inventory.take(self._selected.code)
        self._reset()
        return product, change

    def cancel(self) -> list[Denomination]:
        self._require(State.COLLECTING, State.READY)
        refund = list(self._inserted)                    # return the same coins: no change-making needed
        self._reset()
        return refund

    def _reset(self) -> None:
        self._inserted.clear()
        self._selected = None
        self._state = State.IDLE
```

- **Python internals:** `IntEnum` members are real `int`s, so `sum(self._inserted)` and `remaining >= coin` work without conversions and sorting is numeric — a small representation choice that removes a whole layer of helper code. `Counter` returns `0` for missing keys, which keeps the change loop free of `.get(..., 0)`. The `assert self._selected is not None` after `_require(State.READY)` is a *type-narrowing* assertion: the state guard guarantees it, and the checker needs to be told. The transaction's atomicity here is *single-threaded by construction* — Day 27 shows what breaks when two threads press buttons, and Day 34 shows the State-class version of the same machine.
- **Build & drill:** Write `Inventory`, then tests for all six scenarios in step 6 (the "cannot make change" one needs a cashbox with no small coins). Draw the sequence diagram for "insert ₹10, select ₹7 product, dispense." Then run the *entire six-step method on paper* for an **ATM** (card, PIN, balance, withdraw, deposit, receipts) in 40 minutes — no code — and compare its state machine to the vending machine's.
- **Recall:** Recite the six steps. Name four GRASP patterns and what each assigns. Why does change-making belong in `CashBox`?

### Day 26 — Designing for correctness: invariants, immutability, errors as contracts, and Demeter

- **Concept & why it matters:** Beyond SOLID, the principles that keep systems *correct*: **invariants** (statements that must always be true of an object — document them, enforce them in one place, check them in `__post_init__` or the mutating methods); **Tell, don't ask** (don't fetch state and decide outside the object; tell the object what you want); the **Law of Demeter** (talk only to your immediate collaborators — `order.customer.address.city` couples you to three classes; ask `order.shipping_city()`); **immutability by default** for anything shared or passed around (frozen dataclasses, tuples, `frozenset`); **error design as contract** — an exception hierarchy per module, domain errors distinct from programmer errors, never expose low-level exceptions across a boundary (translate `sqlite3.IntegrityError` into `DuplicateOrderError`); **fail fast** (validate at construction, not at first use); **Result types vs exceptions** — when expected outcomes ("insufficient funds") should be return values and when they're truly exceptional. **Why it matters for LLD:** interviewers ask "what happens if…?" and the strongest answer is "that state is impossible because the constructor/transition refuses it."
- **Real-world case study:** **The Mars Climate Orbiter (1999)** was lost because one module produced pound-force·seconds and another consumed newton·seconds — a *primitive obsession* failure: both were `float`. A `Force` value object with a unit would have made the bug a type error. In software finance, **double-entry ledgers** (Day 49) exist so the invariant "debits equal credits" is checked on every write rather than discovered at month end. Both are the same principle: encode invariants in the types and enforce them at the boundary.
- **Design problem — a hotel-room booking core** (rooms, bookings with date ranges, no double-booking). Approaches for *where the "no overlap" invariant lives*: **(A) in the API layer** — check before calling `room.add_booking()`; any other caller can bypass it. **(B) in `Booking.__post_init__`** — a booking can't know about other bookings; wrong owner. **(C) in `Room.book(date_range)`** — the room owns its bookings, checks overlap against them, and refuses; `DateRange` is a frozen value object with `overlaps()`; callers *tell* the room to book and get a `Booking` or an exception. Tradeoffs: A is the classic "validation in the controller" leak; B misplaces the expert; C puts the invariant with the data that decides it.
- **Thought process → decision:** Information Expert again → **C**. Then decide the *error contract*: is "room already booked for those dates" exceptional? It is an *expected* business outcome, so give it a specific domain exception (`RoomUnavailableError`) callers are meant to catch — or return a `Result`. Choose exceptions here because the operation has one success shape and callers almost always want to stop; note the Result alternative for a search-many-rooms flow where partial failure is normal. Demeter: the service asks `room.is_available(range)`, never `room.bookings[i].date_range.start`.
- **Code:**

```python
"""Day 26 — invariants enforced by their owner; Tell-don't-ask; Demeter-friendly APIs."""

from __future__ import annotations

from dataclasses import dataclass, field
from datetime import date, timedelta
from uuid import UUID, uuid4


class BookingError(Exception): ...
class RoomUnavailableError(BookingError): ...
class InvalidStayError(BookingError): ...


@dataclass(frozen=True, slots=True)
class DateRange:
    """Half-open [start, end): checkout day is not a night. Invariant: start < end."""
    start: date
    end: date

    def __post_init__(self) -> None:
        if self.start >= self.end:
            raise InvalidStayError(f"stay must be at least one night: {self.start} -> {self.end}")

    @property
    def nights(self) -> int:
        return (self.end - self.start).days

    def overlaps(self, other: DateRange) -> bool:
        return self.start < other.end and other.start < self.end       # the one place this rule lives


@dataclass(frozen=True)
class Booking:
    room_number: str
    guest_id: UUID
    stay: DateRange
    id: UUID = field(default_factory=uuid4)


class Room:
    """Invariant: no two bookings overlap. Enforced here and nowhere else."""

    MAX_NIGHTS = 30

    def __init__(self, number: str, capacity: int) -> None:
        if capacity <= 0:
            raise ValueError("capacity must be positive")
        self.number = number
        self.capacity = capacity
        self._bookings: dict[UUID, Booking] = {}

    def is_available(self, stay: DateRange) -> bool:                 # Tell/ask boundary: a yes/no question
        return not any(stay.overlaps(b.stay) for b in self._bookings.values())

    def book(self, guest_id: UUID, stay: DateRange) -> Booking:      # Tell: "book this"; the room decides
        if stay.nights > self.MAX_NIGHTS:
            raise InvalidStayError(f"max stay is {self.MAX_NIGHTS} nights")
        if not self.is_available(stay):
            raise RoomUnavailableError(f"room {self.number} is taken for {stay.start}..{stay.end}")
        booking = Booking(self.number, guest_id, stay)
        self._bookings[booking.id] = booking
        return booking

    def cancel(self, booking_id: UUID) -> None:
        if self._bookings.pop(booking_id, None) is None:
            raise BookingError(f"no booking {booking_id} in room {self.number}")

    def occupancy_nights(self, within: DateRange) -> int:            # Demeter: callers never touch _bookings
        total = 0
        for booking in self._bookings.values():
            start = max(booking.stay.start, within.start)
            end = min(booking.stay.end, within.end)
            total += max((end - start).days, 0)
        return total


if __name__ == "__main__":
    room = Room("101", capacity=2)
    guest = uuid4()
    stay = DateRange(date(2026, 12, 24), date(2026, 12, 27))
    booking = room.book(guest, stay)
    print(booking.stay.nights, room.is_available(DateRange(date(2026, 12, 27), date(2026, 12, 29))))   # 3 True
    try:
        room.book(uuid4(), DateRange(date(2026, 12, 26), date(2026, 12, 28)))
    except RoomUnavailableError as err:
        print("refused:", err)
    try:
        DateRange(date(2026, 1, 5), date(2026, 1, 5))
    except InvalidStayError as err:
        print("refused:", err)
    print(room.occupancy_nights(DateRange(date(2026, 12, 1), date(2027, 1, 1))))
```

- **Python internals:** Frozen dataclasses give you immutability *and* a correct `__hash__` from the fields, so `DateRange` can be a set member or dict key — needed the moment you index bookings by date. `__post_init__` runs after the generated `__init__` assigns fields, so it's the standard spot for invariants; with `frozen=True`, use `object.__setattr__` if you must normalize a field there. Exceptions as a *contract* means the module's public exceptions are part of its API: put them in one place (`errors.py`) and never let `KeyError` or `sqlite3.Error` escape a domain method — callers should only need to know your hierarchy. `slots=True` on frozen dataclasses gives the smallest possible value objects, important when a system holds millions of them (Day 37's Flyweight).
- **Build & drill:** Add a `Hotel` that owns rooms and offers `find_available(stay, capacity)`; write tests for adjacent stays (checkout day = next check-in — must be allowed), overlapping, and 31-night refusal. Go back through Days 20–25 code and write the *invariants* of each class as a comment at the top — then check each is enforced in exactly one method. Convert one exception-based flow to a `Result` type (Day 18 drill) and write three sentences on which read better.
- **Recall:** Where must an invariant be enforced, and why exactly once? State the Law of Demeter in one sentence. When is an exception the wrong tool for an expected outcome?

### Day 27 — Concurrency fundamentals for LLD: threads, the GIL, locks, and thread-safe objects

- **Concept & why it matters:** Many LLD problems say "must be thread-safe" (booking, cache, rate limiter, ID generator). A **thread** is a second stream of execution sharing the same memory; a **race condition** is two threads interleaving reads and writes so the result depends on timing (`balance -= amount` is three operations: read, subtract, write). The **GIL** lets only one thread run Python bytecode at a time, which makes *individual bytecodes* atomic — but a method is many bytecodes, so the GIL does *not* make your objects thread-safe. Tools: `threading.Lock` (mutual exclusion), `RLock` (re-entrant, for methods that call other locked methods), `Condition` (wait for a state change), `Semaphore` (limit concurrency), `queue.Queue` (thread-safe hand-off), `concurrent.futures.ThreadPoolExecutor`. Rules: keep critical sections *small*; acquire locks in a *consistent order* (deadlock prevention); never hold a lock while calling out to unknown code; prefer *immutable data* and *message passing* over shared mutable state. **Why it matters for LLD:** "how do you prevent two users booking the same seat?" has a precise answer — and "the GIL" is the wrong one.
- **Real-world case study:** **The Therac-25 (1985–87)** radiation-therapy machine delivered lethal overdoses because a race between the operator's fast keyboard input and the machine's setup routine let treatment start before safety settings were applied — a concurrency bug that killed people. In web systems the same class of bug appears as **double-spend / double-booking**: two requests read "1 seat left" simultaneously and both succeed. Every ticketing system (BookMyShow, Ticketmaster) and every bank ledger is a fight against this interleaving; you'll design the booking version on Day 50.
- **Design problem — a thread-safe bank account and a unique ID generator.** Approaches for making `withdraw` safe: **(A) rely on the GIL** — wrong: `if amount <= balance: balance -= amount` is many bytecodes; two threads can both pass the check. **(B) one `Lock` per account guarding every method that reads or writes balance** — correct; contention limited to that account; transfers between two accounts need *two* locks in a consistent order (by account id) to avoid deadlock. **(C) a single global lock for all accounts** — correct and simple; serializes the whole bank; fine for a demo, unacceptable at scale. **(D) lock-free with an atomic compare-and-swap** — Python has no CAS on ints; not available. Tradeoffs: B is the right default; C when contention is irrelevant; A is a bug.
- **Thought process → decision:** *"What is the unit of consistency?"* One account's balance → one lock per account (**B**). For transfers, the unit is *two* accounts → acquire both locks, always in the same global order (sort by id) so two opposite transfers can't deadlock. Encapsulate the lock *inside* the class (callers must not be able to forget it) and never expose the raw balance for external check-then-act — Tell, don't ask is *also* a concurrency principle. For IDs, a lock-guarded counter is simplest; `itertools.count` is *not* guaranteed atomic across threads by the language spec, so lock it.
- **Code:**

```python
"""Day 27 — race conditions made visible, then fixed with per-object locks and ordered acquisition."""

from __future__ import annotations

import threading
from concurrent.futures import ThreadPoolExecutor
from decimal import Decimal


class InsufficientFundsError(Exception): ...


class UnsafeAccount:
    def __init__(self, balance: int) -> None:
        self.balance = balance

    def withdraw(self, amount: int) -> None:
        if amount <= self.balance:            # check ...
            self.balance -= amount            # ... then act — another thread can slip in between


class Account:
    """Thread-safe: the lock lives inside; the critical section is tiny; no check-then-act leaks out."""

    _next_id = 0
    _id_lock = threading.Lock()

    def __init__(self, balance: Decimal) -> None:
        with Account._id_lock:                # a class-level lock guards the shared counter
            Account._next_id += 1
            self.id = Account._next_id
        self._balance = balance
        self._lock = threading.RLock()        # re-entrant: balance() can be called while holding it

    def balance(self) -> Decimal:
        with self._lock:
            return self._balance

    def deposit(self, amount: Decimal) -> None:
        if amount <= 0:
            raise ValueError("deposit must be positive")
        with self._lock:
            self._balance += amount

    def withdraw(self, amount: Decimal) -> None:
        if amount <= 0:
            raise ValueError("withdrawal must be positive")
        with self._lock:                      # check and act inside ONE critical section
            if amount > self._balance:
                raise InsufficientFundsError(f"{self._balance} < {amount}")
            self._balance -= amount


def transfer(source: Account, target: Account, amount: Decimal) -> None:
    """Acquire both locks in a global order (by id) so A->B and B->A can never deadlock."""
    if source is target:
        raise ValueError("cannot transfer to the same account")
    first, second = sorted((source, target), key=lambda a: a.id)
    with first._lock, second._lock:           # noqa: SLF001 — transfer is a friend of Account, same module
        if amount > source._balance:
            raise InsufficientFundsError(f"{source._balance} < {amount}")
        source._balance -= amount
        target._balance += amount


def demonstrate_race() -> None:
    unsafe = UnsafeAccount(balance=100)
    barrier = threading.Barrier(50)

    def take_one() -> None:
        barrier.wait()                        # release all threads at once to maximize interleaving
        for _ in range(100):
            unsafe.withdraw(1)

    with ThreadPoolExecutor(max_workers=50) as pool:
        for _ in range(50):
            pool.submit(take_one)
    print("unsafe final balance (should never be negative):", unsafe.balance)


def demonstrate_safe() -> None:
    a, b = Account(Decimal("1000")), Account(Decimal("1000"))

    def churn(src: Account, dst: Account) -> None:
        for _ in range(500):
            try:
                transfer(src, dst, Decimal("3"))
            except InsufficientFundsError:
                pass

    with ThreadPoolExecutor(max_workers=8) as pool:
        for i in range(8):
            pool.submit(churn, *((a, b) if i % 2 else (b, a)))
    print("safe total (must be exactly 2000):", a.balance() + b.balance())


if __name__ == "__main__":
    demonstrate_race()
    demonstrate_safe()
```

- **Python internals:** The **GIL** is a mutex around the interpreter; a thread releases it every ~5 ms (`sys.getswitchinterval()`) or when it blocks on I/O, so switches can happen *between any two bytecodes* — `dis` your `withdraw` and you'll see the `COMPARE_OP` and the `STORE_ATTR` are several instructions apart. Some operations *look* atomic and are in CPython (`list.append`, `dict[k] = v`) because they are a single bytecode calling C — but that's an implementation detail, never a design guarantee, and `x += 1` is definitely not atomic. `threading.Lock` is a raw OS mutex; `RLock` tracks the owning thread and a recursion count; `with lock:` is `acquire()`/`release()` in a `try/finally`, so exceptions can't leave a lock held. Free-threaded CPython (3.13+, experimental `--disable-gil` builds) removes the GIL — every design in this plan that locks correctly keeps working; designs that leaned on the GIL break. Threads are the *concurrency* tool; for *parallel CPU* work use processes; for many-waiting-I/O use `asyncio` (Day 39 touches it).
- **Build & drill:** Run `demonstrate_race` until you see a negative balance (add `time.sleep(0)` between check and act to make it frequent). Make Day 25's `VendingMachine` thread-safe with one lock and explain why per-method locks would *not* be enough (hint: `select` then `dispense` is a check-then-act across calls — the state guard handles it, but only if both are under the same lock). Write a thread-safe `IdGenerator` and a `BoundedCounter` with a `Condition` that blocks `increment` when full. Then implement the same transfer with a *single* global lock and measure throughput with 8 threads vs the per-account version.
- **Recall:** Why doesn't the GIL make your class thread-safe? What is check-then-act and how do you fix it? How do you prevent deadlock with two locks? Why does the lock live inside the object?

---

# PHASE 3 — Design Patterns, the Python Way (Days 28–41)

A pattern is a *named solution to a recurring force*. Learn the force first, the name second, and the Python idiom third — because in Python half the GoF patterns shrink to a function, a dict, or a dataclass, and knowing *which half* is what separates Pythonic design from Java translated. Every day: the force, the classic form, the Pythonic form, and when *not* to use it. Phase 4 will use every pattern here inside a full system.

### Day 28 — Creational I: Simple Factory, Factory Method, Abstract Factory

- **Concept & why it matters:** The force: *code that needs an object shouldn't need to know which concrete class to build* — otherwise every consumer changes when a new kind appears (OCP, Day 22). **Simple Factory**: one function/class that maps a key to a concrete class. **Factory Method**: a base class defers "which product to create" to subclasses via an overridable method. **Abstract Factory**: an interface for creating *families* of related objects (a `UIFactory` producing matching `Button` + `Checkbox`), so families can't be mixed. Pythonic forms: a **dict registry** `{key: cls}`, classes as first-class callables passed as parameters, `@classmethod` alternative constructors (`Money.from_minor`, `datetime.fromtimestamp`), and `__init_subclass__` for self-registering hierarchies. **Why it matters for LLD:** nearly every LLD problem has a "create the right kind" moment — vehicles in a parking lot, pieces in chess, payment methods, notification channels — and the interviewer watches whether creation logic leaks into the coordinator.
- **Real-world case study:** **`logging.getLogger(name)`** and **Django's `get_user_model()`** are factories that hide *which* class and *whether one already exists*; **SQLAlchemy's dialect loading** (`create_engine("postgresql://…")`) picks a driver family from a URL scheme — an abstract factory keyed on a string. On the anti-pattern side, Java codebases with `FooFactoryFactory` show what happens when creation abstraction is applied without a force — the Python community's allergy to that is healthy.
- **Design problem — vehicle creation for a parking lot** from a ticket-machine input (`"car"`, `"bike"`, `"truck"`), where each type has a different size and fee class, and new types arrive yearly. Approaches: **(A) `if kind == "car": return Car(plate) elif …` inside `ParkingLot.enter()`** — the coordinator knows every class; each new type edits it. **(B) a `VehicleFactory` class with the same `if` chain moved out** — better locality; still edit-to-extend. **(C) a registry: `Vehicle` subclasses self-register by `kind` via `__init_subclass__`; `Vehicle.create(kind, plate)` looks the class up** — adding a truck is adding a class, nothing else; unknown kinds fail with a clear list of valid ones. **(D) Abstract Factory** — overkill unless vehicles come in *families* (e.g. EV vs ICE with matching chargers/spots). Tradeoffs: A/B violate OCP; C is the Pythonic factory; D is justified only by the family force.
- **Thought process → decision:** The force is single-axis variation (kind) with frequent additions → **C**. Keep *validation of the plate* in the factory path (fail fast) and keep *what a vehicle is* (size, fee class) as class attributes so the lot never asks `isinstance`. Note the Abstract Factory trigger for the extension test: "add EV spots with chargers" → families → D becomes the right upgrade.
- **Code:**

```python
"""Day 28 — factories: a self-registering hierarchy (Pythonic Factory Method) and an Abstract Factory."""

from __future__ import annotations

import re
from abc import ABC, abstractmethod
from enum import Enum
from typing import ClassVar

PLATE = re.compile(r"^[A-Z]{2}\d{2}[A-Z]{1,2}\d{4}$")


class SpotSize(Enum):
    SMALL = 1
    MEDIUM = 2
    LARGE = 3


class Vehicle(ABC):
    """Base class doubles as the factory. Subclasses register themselves by `kind`."""

    kind: ClassVar[str]
    size: ClassVar[SpotSize]
    _registry: ClassVar[dict[str, type[Vehicle]]] = {}

    def __init_subclass__(cls, **kwargs: object) -> None:      # runs once per subclass definition
        super().__init_subclass__(**kwargs)
        if not hasattr(cls, "kind"):
            raise TypeError(f"{cls.__name__} must declare a `kind`")
        if cls.kind in Vehicle._registry:
            raise TypeError(f"duplicate vehicle kind {cls.kind!r}")
        Vehicle._registry[cls.kind] = cls

    def __init__(self, plate: str) -> None:
        plate = plate.strip().upper().replace(" ", "")
        if not PLATE.match(plate):
            raise ValueError(f"invalid plate {plate!r}")
        self.plate = plate

    @classmethod
    def create(cls, kind: str, plate: str) -> Vehicle:          # the factory entry point
        try:
            concrete = cls._registry[kind.lower()]
        except KeyError:
            raise ValueError(f"unknown vehicle kind {kind!r}; known: {sorted(cls._registry)}") from None
        return concrete(plate)

    def __repr__(self) -> str:
        return f"{type(self).__name__}({self.plate!r})"


class Bike(Vehicle):
    kind = "bike"
    size = SpotSize.SMALL


class Car(Vehicle):
    kind = "car"
    size = SpotSize.MEDIUM


class Truck(Vehicle):
    kind = "truck"
    size = SpotSize.LARGE


# --- Abstract Factory: families of related products that must match each other ---------------
class Spot(ABC):
    @abstractmethod
    def describe(self) -> str: ...


class Charger(ABC):
    @abstractmethod
    def plug(self) -> str: ...


class SpotFamily(ABC):
    """One factory per family guarantees a spot and its charger are compatible."""
    @abstractmethod
    def make_spot(self) -> Spot: ...
    @abstractmethod
    def make_charger(self) -> Charger: ...


class StandardSpot(Spot):
    def describe(self) -> str: return "standard spot"

class NoCharger(Charger):
    def plug(self) -> str: return "no charging"

class EvSpot(Spot):
    def describe(self) -> str: return "EV spot with cable bay"

class Type2Charger(Charger):
    def plug(self) -> str: return "Type-2 7kW"


class StandardFamily(SpotFamily):
    def make_spot(self) -> Spot: return StandardSpot()
    def make_charger(self) -> Charger: return NoCharger()

class EvFamily(SpotFamily):
    def make_spot(self) -> Spot: return EvSpot()
    def make_charger(self) -> Charger: return Type2Charger()


if __name__ == "__main__":
    for kind, plate in (("car", "KA01AB1234"), ("truck", "mh 12 zz 9999")):
        vehicle = Vehicle.create(kind, plate)
        print(vehicle, vehicle.size)
    try:
        Vehicle.create("boat", "KA01AB1234")
    except ValueError as err:
        print(err)
    for family in (StandardFamily(), EvFamily()):
        print(family.make_spot().describe(), "+", family.make_charger().plug())
```

- **Python internals:** Classes are **callables**: `Car(plate)` is `type.__call__(Car, plate)`, which runs `Car.__new__` then `Car.__init__` — so a class stored in a dict *is* a factory with no wrapper. `__init_subclass__` (PEP 487) is a hook on the *parent* invoked whenever a subclass body finishes executing — the lightweight replacement for a registration metaclass. `ClassVar` tells the type checker the attribute lives on the class, not instances (and `dataclasses` skips it). `type(name, bases, namespace)` can build classes at runtime — the ultimate factory — used by ORMs and `namedtuple`. A `@classmethod` receives the *actual* subclass as `cls`, so `create` on a subclass returns instances of that subclass — polymorphic construction for free.
- **Build & drill:** Add `ElectricCar` with `size = SpotSize.MEDIUM` and a `needs_charger` flag without touching any existing class. Write a `NotificationFactory` (Day 16 channels) as a plain dict registry and compare the two idioms in three sentences. Implement `Shape.from_json(dict)` as a classmethod factory dispatching on a `"type"` key. Then write down the *force* for each factory flavour in one line each.
- **Recall:** What force does a factory answer? Why does a dict of classes work as a factory in Python? When is Abstract Factory justified over a registry?

### Day 29 — Creational II: Builder and Prototype

- **Concept & why it matters:** **Builder** separates constructing a complex object (many optional parts, ordering constraints, validation only possible at the end) from its representation, usually via a fluent chain ending in `build()`. **Prototype** creates new objects by *copying* a configured instance instead of constructing from scratch (`copy.deepcopy`, `dataclasses.replace`, a `clone()` method). Pythonic reality: **keyword arguments with defaults kill 80% of Builder use cases** — `Request(url, method="GET", headers=None, timeout=30)` needs no builder. Builder earns its place when construction is *multi-step* (a query that accumulates clauses), when an *invariant spans fields* (validated in `build()`), or when you need *test-data builders* (`an_order().with_items(3).paid().build()`). **Why it matters for LLD:** interviewers ask "how do you construct a `Pizza`/`House`/`Query` with 12 options?" — the strong Python answer names Builder *and* says when kwargs are enough.
- **Real-world case study:** **SQLAlchemy's query API** (`select(User).where(...).order_by(...).limit(10)`) and **`requests.Request(...).prepare()`** are builders: each call returns a new or updated builder; execution happens at the end. **Java's `StringBuilder`** exists because strings are immutable and concatenation is O(n²) — Python's `"".join(list)` fills that role. Prototype: **Django's `QuerySet.clone()`** and game engines cloning a configured enemy template thousands of times.
- **Design problem — construct an HTTP request** for a client library: method, URL, headers, query params, JSON or form body (mutually exclusive), timeout, retries; must be immutable once built. Approaches: **(A) a constructor with 9 parameters** — callers pass `None` for most; the "JSON xor form" rule lives in a long `__init__`; still fine if most calls set 2–3 fields. **(B) a mutable `Request` object whose attributes you set, then `send()`** — easy to leave half-configured; not thread-safe to share. **(C) a `RequestBuilder` with a fluent API and a `build()` that validates cross-field rules and returns a frozen `Request`** — the invariant is checked once, at the end; the built object is immutable and shareable; construction reads like prose. Tradeoffs: A is right for simple objects — do *not* reach past it reflexively; B is the common bug factory; C when steps accumulate or invariants span fields.
- **Thought process → decision:** Two forces are present: *multi-step accumulation* (headers/params added over several calls) and *cross-field invariants* (one body type, timeout > 0 when retries > 0) → **C**, with the frozen dataclass from Day 14 as the product. Make the builder return `self` (mutable builder, cheap) — or a new builder each step (immutable builder, safe to fork configurations); choose mutable + `copy()` for forking, since that's the common client-library pattern.
- **Code:**

```python
"""Day 29 — Builder producing an immutable product; Prototype via copy/replace."""

from __future__ import annotations

import copy
from dataclasses import dataclass, field, replace
from types import MappingProxyType
from typing import Any, Literal
from urllib.parse import urlencode

Method = Literal["GET", "POST", "PUT", "PATCH", "DELETE"]


@dataclass(frozen=True)
class Request:                                  # the product: immutable, safe to share, no half-built state
    method: Method
    url: str
    headers: MappingProxyType[str, str]
    params: MappingProxyType[str, str]
    json_body: Any | None
    form_body: MappingProxyType[str, str] | None
    timeout_seconds: float
    retries: int

    @property
    def full_url(self) -> str:
        return f"{self.url}?{urlencode(dict(self.params))}" if self.params else self.url


class RequestBuilder:
    def __init__(self, method: Method, url: str) -> None:
        if not url.startswith(("http://", "https://")):
            raise ValueError(f"url must be absolute: {url!r}")
        self._method: Method = method
        self._url = url
        self._headers: dict[str, str] = {}
        self._params: dict[str, str] = {}
        self._json: Any | None = None
        self._form: dict[str, str] | None = None
        self._timeout = 30.0
        self._retries = 0

    # each step returns self so calls chain; fluent, cheap, readable
    def header(self, name: str, value: str) -> RequestBuilder:
        self._headers[name.title()] = value
        return self

    def param(self, name: str, value: str | int) -> RequestBuilder:
        self._params[name] = str(value)
        return self

    def json(self, payload: Any) -> RequestBuilder:
        self._json = payload
        return self

    def form(self, fields: dict[str, str]) -> RequestBuilder:
        self._form = dict(fields)
        return self

    def timeout(self, seconds: float) -> RequestBuilder:
        self._timeout = seconds
        return self

    def retries(self, count: int) -> RequestBuilder:
        self._retries = count
        return self

    def copy(self) -> RequestBuilder:            # Prototype: fork a half-configured builder
        return copy.deepcopy(self)

    def build(self) -> Request:
        """Cross-field validation happens exactly once, here."""
        if self._json is not None and self._form is not None:
            raise ValueError("a request cannot have both a JSON body and a form body")
        if self._method == "GET" and (self._json is not None or self._form is not None):
            raise ValueError("GET requests cannot carry a body")
        if self._timeout <= 0:
            raise ValueError("timeout must be positive")
        if self._retries < 0 or self._retries > 10:
            raise ValueError("retries must be between 0 and 10")
        headers = dict(self._headers)
        if self._json is not None:
            headers.setdefault("Content-Type", "application/json")
        return Request(
            method=self._method, url=self._url,
            headers=MappingProxyType(headers), params=MappingProxyType(dict(self._params)),
            json_body=copy.deepcopy(self._json),
            form_body=MappingProxyType(self._form) if self._form is not None else None,
            timeout_seconds=self._timeout, retries=self._retries,
        )


@dataclass(frozen=True)
class EnemyTemplate:                              # Prototype with dataclasses.replace: copy-with-changes
    name: str
    hp: int
    speed: float
    loot: tuple[str, ...] = field(default=())


if __name__ == "__main__":
    base = RequestBuilder("POST", "https://api.example.com/orders").header("authorization", "Bearer x").retries(2)
    fast = base.copy().timeout(2.0).json({"sku": "PEN", "qty": 3}).build()
    slow = base.copy().timeout(60.0).json({"sku": "DESK", "qty": 1}).param("priority", "low").build()
    print(fast.headers, fast.timeout_seconds, slow.full_url)
    try:
        RequestBuilder("GET", "https://x.io").json({"a": 1}).build()
    except ValueError as err:
        print("rejected:", err)

    goblin = EnemyTemplate("goblin", hp=30, speed=1.2, loot=("coin",))
    goblin_chief = replace(goblin, name="goblin chief", hp=90, loot=(*goblin.loot, "key"))
    print(goblin_chief)
```

- **Python internals:** `MappingProxyType` is a read-only *view* over a dict (the same type as `SomeClass.__dict__`) — it makes the product truly immutable without copying on every read. `copy.deepcopy` uses `__reduce_ex__` under the hood and a memo dict; override `__deepcopy__(self, memo)` when an object holds something uncopyable (a socket, a lock) — Prototype in Python is mostly about deciding *what not to copy*. `dataclasses.replace` calls the class constructor with the changed fields, so `__post_init__` validation re-runs — a validated copy for free. Returning `self` from builder methods is why chaining works; returning a *new* builder each time gives a persistent (immutable) builder — the approach `select()` takes in SQLAlchemy 2.0 (generative API) so a base query can be shared.
- **Build & drill:** Write a `QueryBuilder` for a tiny SQL subset (`select(...).from_(...).where(...).order_by(...).limit(...)`) that renders parameterized SQL and refuses `limit` without `order_by`. Write a test-data builder `an_order()` for Day 14's `Order` and rewrite three tests with it. Implement `Prototype` for Day 17's `Car` via `__deepcopy__` that shares the `Navigator` but copies the engine, and explain why in a comment.
- **Recall:** Name the two forces that justify Builder over kwargs. Why is the product frozen? How does `dataclasses.replace` implement Prototype?

### Day 30 — Creational III: Singleton (and its better alternatives), Object Pool — and the 30-day checkpoint

- **Concept & why it matters:** **Singleton** guarantees one instance and a global access point. The force is real (one config, one connection pool, one logger registry) but the classic implementation is widely considered a *code smell*: it is hidden global state, it makes tests interfere, and it hard-wires "one" even when you'll later want "one per tenant." Python's honest alternatives, in order of preference: **a module** (imported once — `sys.modules` caches it — so module-level objects are already singletons), **dependency injection of a single instance from the composition root** (Day 24: one instance *by construction*, not by class), then, if you must, `__new__`-based or metaclass-based singletons with thread-safe construction. **Object Pool** reuses expensive objects (DB connections, threads) with `acquire`/`release`, bounded size, and health checks. **Why it matters for LLD:** "make the `ParkingLot` a singleton" is a common interview line — the strong answer explains the force, names the cost, and prefers injection.
- **Real-world case study:** **The `logging` module**: `getLogger("x")` always returns the same `Logger` — a *registry of named singletons* implemented as a module-level dict, not as a singleton class. **SQLAlchemy's `QueuePool`** and **psycopg's connection pools** are object pools: connections cost ~ms to open and a database allows a fixed number, so reuse is mandatory. Pytest's fixture scopes (`session`) are singletons-by-DI. The Singleton *class* is the thing experienced Python teams almost never write.
- **Design problem — a database connection manager** for an application with many services that need connections, tests that need isolation, and a hard cap on open connections. Approaches: **(A) a Singleton `Database` class** — everyone calls `Database.instance()`; tests share state; can't run two databases (prod + analytics) later. **(B) a module-level `pool = ConnectionPool(...)` created at import** — simplest true singleton in Python; but configuration happens at import time (hard to test, import order issues). **(C) a `ConnectionPool` class with *no* singleton logic, one instance created in the composition root and injected into services** — tests build their own tiny pool; two databases = two instances; "one" is a wiring decision, not a type decision. **(D) a thread-safe `__new__` singleton** — correct when a *library* must guarantee one instance without controlling the caller's wiring (rare in app code). Tradeoffs: C is the default; B for genuinely process-wide constants; D when you truly can't rely on callers.
- **Thought process → decision:** Ask *"is 'exactly one' a property of the class, or of this deployment?"* It's the deployment's → **C**: a plain, testable pool; the composition root makes one. Implement the pool correctly (bounded, blocking acquire with timeout, context-manager release so a connection can't leak past an exception). Also implement **D** once today so you understand it, thread-safety included — you'll recognize it in codebases.
- **Code:**

```python
"""Day 30 — a bounded Object Pool (injected, not a singleton) and, for reference, a correct Singleton."""

from __future__ import annotations

import queue
import threading
from collections.abc import Callable, Iterator
from contextlib import contextmanager
from typing import Generic, TypeVar

T = TypeVar("T")


class PoolExhaustedError(Exception): ...


class ObjectPool(Generic[T]):
    """Reuse expensive objects; never hand out more than `size`; never leak one on exception."""

    def __init__(self, factory: Callable[[], T], size: int, *, is_healthy: Callable[[T], bool] | None = None) -> None:
        if size <= 0:
            raise ValueError("size must be positive")
        self._factory = factory
        self._is_healthy = is_healthy or (lambda _: True)
        self._available: queue.LifoQueue[T] = queue.LifoQueue(maxsize=size)   # LIFO keeps hot objects hot
        self._created = 0
        self._size = size
        self._lock = threading.Lock()

    def _acquire(self, timeout: float | None) -> T:
        try:
            return self._available.get_nowait()
        except queue.Empty:
            pass
        with self._lock:                                   # decide whether we may create one more
            if self._created < self._size:
                self._created += 1
                return self._factory()
        try:
            return self._available.get(timeout=timeout)   # block until someone releases
        except queue.Empty:
            raise PoolExhaustedError(f"no object available within {timeout}s") from None

    def _release(self, obj: T) -> None:
        if self._is_healthy(obj):
            self._available.put(obj)
        else:
            with self._lock:
                self._created -= 1                         # broken object: drop it, allow a replacement

    @contextmanager
    def lease(self, timeout: float | None = 5.0) -> Iterator[T]:
        obj = self._acquire(timeout)
        try:
            yield obj
        finally:
            self._release(obj)                             # runs even if the caller raised

    @property
    def stats(self) -> dict[str, int]:
        return {"created": self._created, "idle": self._available.qsize(), "size": self._size}


class SingletonMeta(type):
    """Reference implementation. Thread-safe. Use only when you cannot control the wiring."""

    _instances: dict[type, object] = {}
    _lock = threading.Lock()

    def __call__(cls, *args: object, **kwargs: object) -> object:
        if cls not in cls._instances:                      # first check without the lock (fast path)
            with SingletonMeta._lock:
                if cls not in cls._instances:              # second check under the lock (double-checked locking)
                    cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]


class AppClock(metaclass=SingletonMeta):
    def __init__(self) -> None:
        self.ticks = 0


class FakeConnection:
    _counter = 0

    def __init__(self) -> None:
        FakeConnection._counter += 1
        self.id = FakeConnection._counter
        self.closed = False

    def query(self, sql: str) -> str:
        if self.closed:
            raise RuntimeError("connection closed")
        return f"conn{self.id}: {sql}"


if __name__ == "__main__":
    pool: ObjectPool[FakeConnection] = ObjectPool(FakeConnection, size=2, is_healthy=lambda c: not c.closed)
    with pool.lease() as a, pool.lease() as b:
        print(a.query("select 1"), b.query("select 2"), pool.stats)
        try:
            with pool.lease(timeout=0.1):
                pass
        except PoolExhaustedError as err:
            print("exhausted:", err)
    with pool.lease() as c:
        print(c.query("select 3"), "(reused:", c.id, ")")
    print(AppClock() is AppClock())      # True
```

- **Python internals:** A module is a singleton because `import` consults `sys.modules` first (Day 8) — `from config import settings` in fifty files yields one object. `__new__` is the *allocation* hook: a singleton via `__new__` returns the cached instance, but **`__init__` still runs on every call** (a classic bug: state gets reset), which is why the metaclass version intercepts `__call__` instead. Metaclasses customize class *creation and calling*: `type.__call__` is what runs `__new__` + `__init__`, and `SingletonMeta.__call__` wraps it. Double-checked locking is safe in Python because the GIL makes the dict lookup atomic; the lock exists for the create-once race. `queue.LifoQueue` is a thread-safe stack; `contextmanager` turns the acquire/release pair into a `with` statement whose `finally` guarantees release. `threading.local()` gives per-thread singletons (e.g. one DB session per thread), a common middle ground.
- **Build & drill:** Replace the singleton `AppClock` with an injected clock in one of your earlier designs and note what became easier to test. Add pool *validation on acquire* (re-check health before handing out) and a `close_all()`. Write a `Registry` of named singletons (like `logging`). Then write, in five sentences, your answer to "would you make `ParkingLot` a singleton?"

> ## ✅ THE 30-DAY CHECKPOINT
> If you have followed the plan for 30 days, this is honestly where you stand:
> - **You can:** write clean, typed, tested Python; model a domain with entities, value objects, enums, interfaces; apply SOLID and name the smell you're fixing; run the six-step LLD method on simple systems (vending machine, library, hotel rooms); use creational patterns idiomatically; explain thread-safety basics.
> - **You cannot yet:** choose confidently among the 15+ behavioural/structural patterns under time pressure; design concurrent systems (booking, cache, rate limiter) that are actually correct; design an *unseen* mid-complexity system (elevator, ride-matching, job scheduler) from a blank page in 45 minutes.
> - **Self-test (60 min, no references):** design and code a **Parking Lot v0** — multiple floors, spot sizes, vehicle kinds via a factory, entry/exit with tickets, hourly fees, tests. Grade yourself: does one object own spot allocation? Is fee calculation swappable? Could you add motorcycles without editing the lot? If yes to all three, Days 31–60 will turn method into fluency. If no, spend three more days repeating Days 22–28 before continuing — the remaining 30 days assume this base.

### Day 31 — Behavioural I: Strategy and Template Method

- **Concept & why it matters:** **Strategy**: a family of interchangeable algorithms behind one interface, chosen at runtime, so the client (context) never branches on *which* — the OCP fix for `if kind ==` chains. **Template Method**: a base class defines the *skeleton* of an algorithm and lets subclasses fill in *steps* (hooks) — inheritance-based variation. They solve the same force (varying steps) from opposite directions: Strategy *composes* (has-a algorithm), Template Method *inherits* (is-a algorithm with holes). Pythonic Strategy is frequently **a plain function** (`sorted(key=...)`, `map(f, …)`, a dict of callables); Template Method appears as **hook methods** in frameworks. **Why it matters for LLD:** pricing, fee calculation, allocation, matching, and scheduling are all "which algorithm today?" questions; Strategy is the most-used pattern in LLD interviews, and *knowing when a lambda is enough* is the Python-specific mark of judgment.
- **Real-world case study:** **Comparators vs key functions.** Java sorts with `Comparator` objects (Strategy as a class); Python sorts with `key=` (Strategy as a function) — a smaller, composable idiom that shows how much a first-class function buys. **Django's class-based views** are Template Method: `dispatch()` is the skeleton, `get()`/`post()`/`get_context_data()` are hooks. **Uber's surge pricing** and **e-commerce promotion engines** are pricing strategies selected by context (time, demand, customer tier).
- **Design problem — a parking-fee engine**: hourly for cars, flat daily cap, free first 15 minutes, weekend rates, EV discount; new rules arrive monthly and can combine. Approaches: **(A) `if vehicle.kind == … and is_weekend …` in `compute_fee`** — grows without bound; every rule tests every other. **(B) a `FeeStrategy` Protocol with one class per pricing scheme, chosen by a factory; the `ParkingLot` only calls `strategy.fee(ticket)`** — each scheme is testable alone; new scheme = new class. **(C) *composable* strategies: a pipeline of small `FeeRule` functions (base rate → cap → discounts) folded over the ticket** — rules combine without a class per combination; order becomes a design question. **(D) Template Method: `BaseFee.compute()` skeleton with `hourly_rate()`, `cap()`, `discount()` hooks overridden per subclass** — good when the *skeleton* is truly fixed and only steps vary; breaks when rules need to combine across the hierarchy. Tradeoffs: A never; B for a small set of whole schemes; C when rules compose (they do here); D when the algorithm's *shape* is stable and subclasses are natural.
- **Thought process → decision:** The requirement "can combine" is the decider: whole-scheme strategies (B) multiply, and inheritance (D) can't mix branches → **C** for the rules, wrapped in a tiny **B**-style `FeePolicy` object so the lot has a single collaborator. Use Template Method for a *different* force present here: the *ticket-processing skeleton* (validate → compute → round → record) is fixed while rounding differs by lot — that's a legitimate hook. Two patterns, each on the force it fits.
- **Code:**

```python
"""Day 31 — Strategy as composable functions + a Template Method skeleton around them."""

from __future__ import annotations

import math
from abc import ABC, abstractmethod
from collections.abc import Callable, Sequence
from dataclasses import dataclass
from datetime import datetime, timedelta
from decimal import Decimal, ROUND_CEILING, ROUND_HALF_UP


@dataclass(frozen=True)
class Ticket:
    vehicle_kind: str
    entered_at: datetime
    exited_at: datetime
    is_electric: bool = False

    @property
    def duration(self) -> timedelta:
        if self.exited_at < self.entered_at:
            raise ValueError("exit before entry")
        return self.exited_at - self.entered_at


FeeRule = Callable[[Ticket, Decimal], Decimal]       # Strategy as a function: (ticket, running fee) -> new fee

HOURLY = {"bike": Decimal("10"), "car": Decimal("30"), "truck": Decimal("60")}


def base_hourly(ticket: Ticket, fee: Decimal) -> Decimal:
    hours = math.ceil(ticket.duration.total_seconds() / 3600)
    return HOURLY[ticket.vehicle_kind] * hours


def free_first_minutes(minutes: int) -> FeeRule:      # a rule factory: parameterized strategy via closure
    def rule(ticket: Ticket, fee: Decimal) -> Decimal:
        return Decimal("0") if ticket.duration <= timedelta(minutes=minutes) else fee
    return rule


def daily_cap(cap_per_day: Decimal) -> FeeRule:
    def rule(ticket: Ticket, fee: Decimal) -> Decimal:
        days = max(1, math.ceil(ticket.duration.total_seconds() / 86400))
        return min(fee, cap_per_day * days)
    return rule


def weekend_multiplier(factor: Decimal) -> FeeRule:
    def rule(ticket: Ticket, fee: Decimal) -> Decimal:
        return fee * factor if ticket.entered_at.weekday() >= 5 else fee
    return rule


def ev_discount(ticket: Ticket, fee: Decimal) -> Decimal:
    return fee * Decimal("0.8") if ticket.is_electric else fee


class FeePolicy:
    """Context: folds ordered rules. Order is a design decision — cap before discount? Document it."""

    def __init__(self, rules: Sequence[FeeRule]) -> None:
        if not rules:
            raise ValueError("a fee policy needs at least one rule")
        self._rules = tuple(rules)

    def fee(self, ticket: Ticket) -> Decimal:
        total = Decimal("0")
        for rule in self._rules:
            total = rule(ticket, total)
        return max(total, Decimal("0"))


class TicketProcessor(ABC):
    """Template Method: the skeleton is fixed; rounding and recording are hooks."""

    def __init__(self, policy: FeePolicy) -> None:
        self._policy = policy

    def process(self, ticket: Ticket) -> Decimal:       # the template — final in spirit; don't override
        self.validate(ticket)
        raw = self._policy.fee(ticket)
        rounded = self.round(raw)
        self.record(ticket, rounded)
        return rounded

    def validate(self, ticket: Ticket) -> None:         # default step: Ticket.duration raises on exit-before-entry
        _ = ticket.duration

    @abstractmethod
    def round(self, amount: Decimal) -> Decimal: ...   # required hook

    def record(self, ticket: Ticket, amount: Decimal) -> None:   # optional hook
        pass


class MallProcessor(TicketProcessor):
    def round(self, amount: Decimal) -> Decimal:
        return amount.quantize(Decimal("1"), rounding=ROUND_CEILING)     # whole rupees, always up


class AirportProcessor(TicketProcessor):
    def __init__(self, policy: FeePolicy, ledger: list[tuple[Ticket, Decimal]]) -> None:
        super().__init__(policy)
        self._ledger = ledger

    def round(self, amount: Decimal) -> Decimal:
        return amount.quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)

    def record(self, ticket: Ticket, amount: Decimal) -> None:
        self._ledger.append((ticket, amount))


if __name__ == "__main__":
    policy = FeePolicy([base_hourly, weekend_multiplier(Decimal("1.5")), daily_cap(Decimal("300")), ev_discount, free_first_minutes(15)])
    sat = datetime(2026, 9, 5, 10, 0)
    ticket = Ticket("car", sat, sat + timedelta(hours=3, minutes=10), is_electric=True)
    print(MallProcessor(policy).process(ticket))          # 4h*30=120 *1.5=180 cap→180 *0.8=144 → 144
    ledger: list[tuple[Ticket, Decimal]] = []
    print(AirportProcessor(policy, ledger).process(Ticket("bike", sat, sat + timedelta(minutes=10))), len(ledger))
```

- **Python internals:** A `Callable` type alias plus closures gives you *parameterized strategies* without classes — the closure cell holds the parameter (Day 19). A Strategy *class* is still right when the algorithm has state, needs several methods, or must be introspected/configured (e.g. `__repr__` for logging which rules applied) — the honest rule: *function until you need identity or more than one method*. Template Method relies on the MRO (Day 15): `process` calls `self.round`, which resolves to the subclass's version — dynamic dispatch on `self`. Python has no `final`, but `typing.final` marks a method for the type checker so subclasses that override `process` are flagged. `abstractmethod` on a hook makes "you forgot to implement rounding" an instantiation-time error, not a 2 a.m. one.
- **Build & drill:** Add a `loyalty_discount(tier)` rule and a *monthly pass* rule that makes everything free — where in the order does it go, and why? Convert Day 15's account hierarchy to Strategy (`WithdrawalPolicy`, `InterestPolicy`) as promised, and write the paragraph comparing the two. Implement `sorted`-style key strategies for a leaderboard (by score, by name, by recency) and a *composite* key.
- **Recall:** Strategy vs Template Method: same force, which mechanism each? When is a function a sufficient strategy? Why is rule *order* a design decision?

### Day 32 — Behavioural II: Observer, Pub/Sub, and event-driven objects

- **Concept & why it matters:** **Observer**: a *subject* keeps a list of *observers* and notifies them on state change, so the subject needs no knowledge of who cares — the fix for `order.mark_paid()` calling `email.send()`, `sms.send()`, `analytics.track()`, `inventory.reserve()` directly (coupling to four modules, edited on every new reaction). **Pub/Sub** decouples one step further via a broker keyed by *topic*: publishers and subscribers don't know each other at all. Design decisions that matter: **push** (pass the event) vs **pull** (observers query the subject); **synchronous** (simple, but one slow or failing observer stalls or breaks the subject) vs **queued/asynchronous**; **error isolation** (one observer's exception must not stop the others); **ordering** guarantees; **weak references** so forgotten subscriptions don't leak memory; **unsubscribe** ergonomics. **Why it matters for LLD:** "notify users when…", "update the dashboard when…", "log/audit every…" appear in most LLD problems; Observer is how the core domain stays ignorant of them.
- **Real-world case study:** **Django signals** (`post_save`) and **Qt's signals/slots** are Observer in frameworks; Django's docs warn about exactly today's pitfalls — hidden control flow and receivers that raise. **Kafka/RabbitMQ** are Pub/Sub industrialized (topics, durable subscriptions, replay). The classic outage shape: a synchronous "on order placed" hook calls a slow analytics API; the API degrades; checkout latency triples — the argument for *queued* observers with a bounded buffer for anything not required for correctness.
- **Design problem — order-status notifications**: when an order is paid/shipped/cancelled, email the customer, alert the warehouse, update analytics; more reactions coming. Approaches: **(A) call each collaborator from `Order` methods** — domain object imports infrastructure; each new reaction edits `Order`; DIP inverted the wrong way. **(B) Observer on `Order`: observers register on each order** — decoupled, but registering on *every* order instance is awkward, and orders are many. **(C) a domain `EventBus`: `Order` records events (`OrderPaid`) and the service publishes them; handlers subscribe by event type; delivery is synchronous with per-handler error isolation; optional async worker** — one subscription point, typed events, testable by asserting on the published events. Tradeoffs: A never; B when the subject is long-lived and few (a stock ticker); C for domain events in application services.
- **Thought process → decision:** Subjects are many and short-lived → per-instance Observer is clumsy → **C**. Decide *synchronous with isolation*: correctness-critical reactions (warehouse reservation) happen in-process; best-effort ones (email, analytics) are queued to a background worker so they can never slow or break the command. Use frozen dataclass events (immutable messages — Day 27's advice), weak references for handler methods so a dead object drops out, and a `Sequence` of handlers per type so order is deterministic.
- **Code:**

```python
"""Day 32 — typed domain events, a bus with error isolation, sync + queued delivery."""

from __future__ import annotations

import logging
import queue
import threading
import weakref
from collections import defaultdict
from collections.abc import Callable
from dataclasses import dataclass, field
from datetime import datetime, timezone
from typing import TypeVar
from uuid import UUID, uuid4

logger = logging.getLogger(__name__)


@dataclass(frozen=True)
class Event:
    occurred_at: datetime = field(default_factory=lambda: datetime.now(timezone.utc), kw_only=True)


@dataclass(frozen=True)
class OrderPaid(Event):
    order_id: UUID
    amount_paise: int


@dataclass(frozen=True)
class OrderShipped(Event):
    order_id: UUID
    tracking_code: str


E = TypeVar("E", bound=Event)
Handler = Callable[[E], None]


class EventBus:
    """Sync handlers run inline (correctness-critical); async handlers are queued to a worker."""

    def __init__(self) -> None:
        self._sync: dict[type[Event], list[Callable[[Event], None]]] = defaultdict(list)
        self._async: dict[type[Event], list[Callable[[Event], None]]] = defaultdict(list)
        self._queue: queue.Queue[tuple[Callable[[Event], None], Event] | None] = queue.Queue(maxsize=10_000)
        self._worker = threading.Thread(target=self._drain, daemon=True, name="eventbus-worker")
        self._worker.start()

    def subscribe(self, event_type: type[E], handler: Handler[E], *, background: bool = False) -> Callable[[], None]:
        """Returns an unsubscribe function — no bookkeeping for the caller."""
        ref = _weak_callable(handler)
        bucket = self._async if background else self._sync
        bucket[event_type].append(ref)              # type: ignore[arg-type]

        def unsubscribe() -> None:
            bucket[event_type].remove(ref)          # type: ignore[arg-type]
        return unsubscribe

    def publish(self, event: Event) -> None:
        for event_type in type(event).__mro__:         # handlers of base classes also fire (Event → everything)
            for handler in list(self._sync.get(event_type, ())):
                try:
                    handler(event)
                except Exception:                      # noqa: BLE001 — isolate: one bad handler can't stop the rest
                    logger.exception("sync handler %r failed for %r", handler, event)
            for handler in self._async.get(event_type, ()):
                try:
                    self._queue.put_nowait((handler, event))
                except queue.Full:
                    logger.error("event queue full; dropping %r for %r", event, handler)   # shed load, don't block

    def _drain(self) -> None:
        while (item := self._queue.get()) is not None:
            handler, event = item
            try:
                handler(event)
            except Exception:                          # noqa: BLE001
                logger.exception("background handler %r failed for %r", handler, event)
            finally:
                self._queue.task_done()

    def flush(self, timeout: float = 2.0) -> None:     # for tests and shutdown
        self._queue.join()


def _weak_callable(handler: Callable[[E], None]) -> Callable[[E], None]:
    """Hold bound methods weakly so a forgotten subscription doesn't keep its object alive."""
    if hasattr(handler, "__self__"):
        weak = weakref.WeakMethod(handler)              # type: ignore[arg-type]

        def call(event: E) -> None:
            method = weak()
            if method is not None:
                method(event)
        return call
    return handler


class Warehouse:
    def __init__(self) -> None:
        self.reserved: list[UUID] = []

    def on_paid(self, event: OrderPaid) -> None:
        self.reserved.append(event.order_id)


class Mailer:
    def on_shipped(self, event: OrderShipped) -> None:
        print(f"[mail] order {event.order_id} shipped, tracking {event.tracking_code}")


def audit(event: Event) -> None:
    print(f"[audit] {type(event).__name__} at {event.occurred_at:%H:%M:%S}")


if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO)
    bus = EventBus()
    warehouse, mailer = Warehouse(), Mailer()
    bus.subscribe(OrderPaid, warehouse.on_paid)                       # critical: inline
    bus.subscribe(OrderShipped, mailer.on_shipped, background=True)   # best-effort: queued
    bus.subscribe(Event, audit)                                       # base type: sees every event
    bus.subscribe(OrderPaid, lambda e: 1 / 0)                         # a broken handler — isolated

    order_id = uuid4()
    bus.publish(OrderPaid(order_id, 4550))
    bus.publish(OrderShipped(order_id, "TRK123"))
    bus.flush()
    print("warehouse reserved:", warehouse.reserved)
```

- **Python internals:** A bound method is created *fresh* on every attribute access (`obj.method` → new object), so a plain `weakref.ref(obj.method)` dies instantly — `weakref.WeakMethod` exists precisely to hold "this method of that object" weakly. `weakref.WeakSet`/`WeakKeyDictionary` are the usual observer collections. Publishing along `type(event).__mro__` gives subtype subscriptions for free — the MRO (Day 15) used as an event hierarchy. `queue.Queue` is the thread-safe hand-off: `put_nowait` + `Full` implements *backpressure by shedding*, the alternative being *blocking* (`put(timeout=)`) — a conscious choice you should be able to defend. A `daemon` thread dies with the process; for graceful shutdown, `put(None)` as a sentinel and `join()` the worker. Handlers that raise are logged with `logger.exception` — the isolation rule in practice.
- **Build & drill:** Add `unsubscribe` tests and a test proving a dead observer (deleted object) no longer receives events (`gc.collect()` may be needed). Make Day 14's `Order` collect events in a `self.events` list during `mark_paid()`/`ship()` and have a service publish them after the state change succeeds. Then write a classic per-instance `StockTicker` Observer (subject with `attach`/`detach`/`notify`) and compare it to the bus in five lines: which is right for which force?
- **Recall:** Push vs pull, sync vs queued — the tradeoffs. Why `WeakMethod` and not `weakref.ref`? Why must handler errors be isolated?

### Day 33 — Behavioural III: Command and Memento (undo, redo, queues, and logs)

- **Concept & why it matters:** **Command** turns a request into an object (or closure) with `execute()` — and often `undo()` — so requests can be queued, logged, retried, scheduled, batched, or reversed, and so the invoker (a button, a scheduler, an API) is decoupled from the receiver (the domain object). **Memento** captures an object's internal state as an opaque snapshot that only the originator can restore, enabling undo without breaking encapsulation. Together they give two undo strategies: **compensating commands** (each command knows how to reverse itself — cheap, but every command must be perfectly invertible) vs **snapshots** (store state before each change — simple and always correct, but memory-heavy; mitigate with diffs or periodic snapshots). **Why it matters for LLD:** text editors, transaction logs, task queues, remote controls, and "undo last booking" all show up in interviews; the Command object is also the unit you hand to a thread pool or serialize into a job queue.
- **Real-world case study:** **Git** stores snapshots (commits are trees, not diffs — Memento at scale) with commands layered on top; **Photoshop's History panel** mixes snapshots and command replay; **Celery/RQ tasks** are serialized Command objects executed by workers; **event-sourced ledgers** store the *commands/events* and derive state by replay (Day 49). Every one decided *snapshot vs compensation* based on state size and invertibility — the exact fork in today's problem.
- **Design problem — a text editor core** with insert/delete/replace, unlimited undo/redo, and macros (a recorded sequence replayed as one command). Approaches: **(A) methods on `Document` plus a list of previous full copies** — undo is trivial (restore copy); memory grows with document size × edits; redo needs a second stack. **(B) Command objects with `execute()`/`undo()` each computing the inverse (`Insert` ↔ `Delete`)** — memory is O(edit size); macros are a `CompositeCommand`; but `Replace.undo` must remember the replaced text, and every new command needs a correct inverse (bugs hide here). **(C) hybrid: commands for the operations, Memento snapshots at checkpoints (every N commands) for fast rollback** — best for very large histories; more machinery. Tradeoffs: A fine for small documents; B the standard, efficient answer; C when history is long and state large.
- **Thought process → decision:** Documents are small-to-medium, edits are many → memory matters more than simplicity → **B**, with the receiver (`Document`) exposing *only* the primitive operations and the commands capturing what they need for the inverse *at execute time* (a `Replace` learns what it replaced when it runs). Macros: a `MacroCommand` holding children, undone in reverse order (Composite, Day 36, arriving early). Redo stack is cleared on any new command — the standard editor semantics; write it down as a requirement rather than discovering it.
- **Code:**

```python
"""Day 33 — Command with undo/redo and macros; Memento for checkpoints."""

from __future__ import annotations

from abc import ABC, abstractmethod
from dataclasses import dataclass


class Document:
    """The receiver. Only primitive mutations; no knowledge of history."""

    def __init__(self, text: str = "") -> None:
        self._text = text

    @property
    def text(self) -> str:
        return self._text

    def insert(self, position: int, fragment: str) -> None:
        self._check(position)
        self._text = self._text[:position] + fragment + self._text[position:]

    def delete(self, position: int, length: int) -> str:
        self._check(position)
        if length < 0 or position + length > len(self._text):
            raise ValueError("delete range out of bounds")
        removed = self._text[position:position + length]
        self._text = self._text[:position] + self._text[position + length:]
        return removed

    def _check(self, position: int) -> None:
        if not 0 <= position <= len(self._text):
            raise ValueError(f"position {position} outside 0..{len(self._text)}")

    # Memento: opaque snapshot only Document knows how to read
    def snapshot(self) -> DocumentMemento:
        return DocumentMemento(self._text)

    def restore(self, memento: DocumentMemento) -> None:
        self._text = memento._state


@dataclass(frozen=True)
class DocumentMemento:
    _state: str          # "private" by convention: callers hold it, never inspect it


class Command(ABC):
    @abstractmethod
    def execute(self, doc: Document) -> None: ...

    @abstractmethod
    def undo(self, doc: Document) -> None: ...


class Insert(Command):
    def __init__(self, position: int, fragment: str) -> None:
        self._position, self._fragment = position, fragment

    def execute(self, doc: Document) -> None:
        doc.insert(self._position, self._fragment)

    def undo(self, doc: Document) -> None:
        doc.delete(self._position, len(self._fragment))


class Delete(Command):
    def __init__(self, position: int, length: int) -> None:
        self._position, self._length = position, length
        self._removed: str | None = None                  # learned at execute time; needed for the inverse

    def execute(self, doc: Document) -> None:
        self._removed = doc.delete(self._position, self._length)

    def undo(self, doc: Document) -> None:
        assert self._removed is not None, "undo before execute"
        doc.insert(self._position, self._removed)


class Macro(Command):
    """Composite command: executes children in order, undoes them in reverse."""

    def __init__(self, commands: list[Command]) -> None:
        if not commands:
            raise ValueError("a macro needs at least one command")
        self._commands = list(commands)

    def execute(self, doc: Document) -> None:
        done: list[Command] = []
        try:
            for command in self._commands:
                command.execute(doc)
                done.append(command)
        except Exception:
            for command in reversed(done):               # all-or-nothing: roll back the partial macro
                command.undo(doc)
            raise

    def undo(self, doc: Document) -> None:
        for command in reversed(self._commands):
            command.undo(doc)


class Editor:
    """The invoker: owns history. Knows commands, not how the document works."""

    def __init__(self, doc: Document) -> None:
        self._doc = doc
        self._undo: list[Command] = []
        self._redo: list[Command] = []

    def run(self, command: Command) -> None:
        command.execute(self._doc)                       # if this raises, history is untouched
        self._undo.append(command)
        self._redo.clear()                               # a new edit invalidates the redo branch

    def undo(self) -> None:
        if not self._undo:
            raise IndexError("nothing to undo")
        command = self._undo.pop()
        command.undo(self._doc)
        self._redo.append(command)

    def redo(self) -> None:
        if not self._redo:
            raise IndexError("nothing to redo")
        command = self._redo.pop()
        command.execute(self._doc)
        self._undo.append(command)

    @property
    def text(self) -> str:
        return self._doc.text


if __name__ == "__main__":
    editor = Editor(Document("hello world"))
    editor.run(Insert(5, ","))
    editor.run(Macro([Delete(0, 1), Insert(0, "H")]))      # capitalize as one undoable step
    print(editor.text)                                   # Hello, world
    editor.undo(); print(editor.text)                    # hello, world
    editor.undo(); print(editor.text)                    # hello world
    editor.redo(); editor.redo(); print(editor.text)     # Hello, world
    checkpoint = editor._doc.snapshot()                  # Memento: an opaque save point
    editor.run(Delete(0, 6)); print(editor.text)         # world
    editor._doc.restore(checkpoint); print(editor.text)  # Hello, world
```

- **Python internals:** A Command with no state or inverse can just be a **closure** (`lambda: doc.insert(3, "x")`) — Python's "Command pattern is a function" observation; use a class as soon as you need `undo`, introspection, or serialization. For queues across processes, commands must be **picklable**: `pickle` serializes instance `__dict__` by default; implement `__getstate__`/`__setstate__` to drop unpicklable members (locks, file handles) — Celery, RQ and multiprocessing all rely on this. Memento's "opaque" state is enforced by convention in Python (no `friend` classes) — a leading-underscore field and a nested/module-private class is the idiom. The `try/except` rollback in `Macro.execute` is the in-memory version of a transaction (Day 38's Unit of Work) and of the Saga pattern in distributed systems.
- **Build & drill:** Add `Replace(position, length, fragment)` (its inverse is learned at execute time), a *cursor* to the document with commands that move it, and a `HistoryLimit` (drop oldest when > N). Then design a bank transaction log where every operation is a Command persisted as JSON and the balance is rebuilt by replay — the seed of event sourcing on Day 49. Implement a remote-control (`Button` → `Command`) with a `NoOpCommand` (Null Object) for unassigned buttons.
- **Recall:** Compensating commands vs snapshots — the tradeoff. Why is the redo stack cleared on a new command? Why does `Delete` learn its inverse at execute time?

### Day 34 — Behavioural IV: State pattern and finite state machines (Vending Machine, revisited)

- **Concept & why it matters:** When an object's behaviour depends on its **state** and the set of states is closed, you have a **finite state machine (FSM)**: states, events/actions, transitions, guards, and entry/exit actions. Two implementations: **(1) enum + transition table + guards** (Day 14/25) — compact, data-driven, best when per-state behaviour is *thin* (mostly "allowed or not"); **(2) the State pattern** — one class per state, each implementing every action, the context delegates and swaps its current state object — best when per-state behaviour is *rich* (different logic, not just permission) and when states carry their own data. Anti-pattern: an `if self.state == …` block inside every method (the Day 2 smell at its worst). Also: hierarchical states, *illegal transition* as a first-class error, persisting state safely, and making transitions atomic under concurrency (Day 27). **Why it matters for LLD:** orders, payments, tickets, elevators, traffic lights, TCP connections, vending machines, document workflows — *most entities have a lifecycle*, and drawing the state diagram is often the single highest-value five minutes in an interview.
- **Real-world case study:** **TCP's connection state machine** (RFC 793's LISTEN/SYN-SENT/ESTABLISHED/…/TIME-WAIT diagram) is the most-studied FSM in software; every network stack implements it, and bugs are found by checking implementations against the diagram. **Stripe's `PaymentIntent` statuses** (`requires_payment_method → requires_confirmation → processing → succeeded/canceled`) are a public FSM that thousands of integrations depend on; Stripe documents the legal transitions precisely because ambiguous lifecycles cause double charges. **Airflow/Temporal task states** are the same idea for workflows.
- **Design problem — the Vending Machine from Day 25, with richer behaviour**: display messages per state, a `maintenance` mode where only admin actions work, a timeout that refunds after 60 s of inactivity in `COLLECTING`, and per-state handling of a coin jam. Approaches: **(A) keep the enum + `_require` guards and add `if self._state == …` branches for the new behaviour** — the guards were enough for permissions; now each method grows a branch per state. **(B) State pattern: `IdleState`, `CollectingState`, `ReadyState`, `MaintenanceState`, each with `insert/select/dispense/cancel/tick/display`; the machine holds `self._state: State` and delegates** — each state's logic is in one place; adding a state adds a class; but shared data (inserted coins) must live in the context and be passed around. **(C) a table-driven FSM library** (`transitions`, `python-statemachine`) — declarative, with diagrams for free; another dependency and a learning curve; excellent for workflow entities in real applications. Tradeoffs: A now fails the "behaviour per state" test; B is the classic; C is what many production teams choose for domain lifecycles.
- **Thought process → decision:** The new requirements make behaviour *rich per state* (display text, timeout handling, jam handling differ by state) → the enum table no longer localizes change → **B**. Keep the transition *authority* in the states (each state returns the next state) so there is exactly one owner of "what happens on this event in this state," and keep the *data* (coins, selection) in the context so states can be stateless singletons or fresh instances. Compare with Day 25 side by side — the crossover point is *"does each state do different things, or just permit different things?"*
- **Code:**

```python
"""Day 34 — State pattern: each state owns its behaviour and decides the next state."""

from __future__ import annotations

from abc import ABC, abstractmethod
from dataclasses import dataclass, field

INACTIVITY_LIMIT_SECONDS = 60


class VendingError(Exception): ...
class InvalidActionError(VendingError): ...


@dataclass
class Session:                      # context data shared by states; the machine owns it
    inserted_paise: int = 0
    selected_price_paise: int | None = None
    idle_seconds: int = 0
    coins: list[int] = field(default_factory=list)

    def reset(self) -> None:
        self.inserted_paise = 0
        self.selected_price_paise = None
        self.idle_seconds = 0
        self.coins.clear()


class State(ABC):
    """Every action returns the next State. Default = not allowed here."""

    name = "state"

    def insert(self, m: Machine, paise: int) -> State:
        raise InvalidActionError(f"cannot insert money while {self.name}")

    def select(self, m: Machine, price_paise: int) -> State:
        raise InvalidActionError(f"cannot select while {self.name}")

    def dispense(self, m: Machine) -> State:
        raise InvalidActionError(f"cannot dispense while {self.name}")

    def cancel(self, m: Machine) -> State:
        raise InvalidActionError(f"nothing to cancel while {self.name}")

    def tick(self, m: Machine, seconds: int) -> State:      # time passes; default: nothing happens
        return self

    def enter_maintenance(self, m: Machine) -> State:
        raise InvalidActionError(f"cannot enter maintenance while {self.name}")

    @abstractmethod
    def display(self, m: Machine) -> str: ...


class Idle(State):
    name = "idle"

    def insert(self, m: Machine, paise: int) -> State:
        m.session.inserted_paise += paise
        m.session.coins.append(paise)
        return Collecting()

    def select(self, m: Machine, price_paise: int) -> State:
        m.session.selected_price_paise = price_paise
        return Collecting()

    def enter_maintenance(self, m: Machine) -> State:
        return Maintenance()

    def display(self, m: Machine) -> str:
        return "Insert coins or select a product"


class Collecting(State):
    name = "collecting"

    def insert(self, m: Machine, paise: int) -> State:
        m.session.inserted_paise += paise
        m.session.coins.append(paise)
        m.session.idle_seconds = 0
        return self._maybe_ready(m)

    def select(self, m: Machine, price_paise: int) -> State:
        m.session.selected_price_paise = price_paise
        m.session.idle_seconds = 0
        return self._maybe_ready(m)

    def cancel(self, m: Machine) -> State:
        m.refund(list(m.session.coins))
        m.session.reset()
        return Idle()

    def tick(self, m: Machine, seconds: int) -> State:
        m.session.idle_seconds += seconds
        if m.session.idle_seconds >= INACTIVITY_LIMIT_SECONDS:      # entry/exit logic lives with the state
            return self.cancel(m)
        return self

    def _maybe_ready(self, m: Machine) -> State:
        price = m.session.selected_price_paise
        return Ready() if price is not None and m.session.inserted_paise >= price else self

    def display(self, m: Machine) -> str:
        need = (m.session.selected_price_paise or 0) - m.session.inserted_paise
        return f"Inserted {m.session.inserted_paise}; " + (f"need {need} more" if need > 0 else "select a product")


class Ready(State):
    name = "ready"

    def insert(self, m: Machine, paise: int) -> State:
        m.session.inserted_paise += paise
        m.session.coins.append(paise)
        return self

    def dispense(self, m: Machine) -> State:
        assert m.session.selected_price_paise is not None
        change = m.session.inserted_paise - m.session.selected_price_paise
        m.deliver(change_paise=change)
        m.session.reset()
        return Idle()

    def cancel(self, m: Machine) -> State:
        return Collecting().cancel(m)                      # reuse: same refund semantics

    def display(self, m: Machine) -> str:
        return "Press dispense"


class Maintenance(State):
    name = "maintenance"

    def enter_maintenance(self, m: Machine) -> State:
        return Idle()                                       # toggles back

    def display(self, m: Machine) -> str:
        return "Out of service"


class Machine:
    """Context: holds data + current state, delegates every action, records transitions."""

    def __init__(self) -> None:
        self.session = Session()
        self._state: State = Idle()
        self.log: list[str] = []

    def _apply(self, action: str, result: State) -> None:
        if result is not self._state:
            self.log.append(f"{self._state.name} --{action}--> {result.name}")
        self._state = result

    def insert(self, paise: int) -> None:        self._apply("insert", self._state.insert(self, paise))
    def select(self, price: int) -> None:        self._apply("select", self._state.select(self, price))
    def dispense(self) -> None:                  self._apply("dispense", self._state.dispense(self))
    def cancel(self) -> None:                    self._apply("cancel", self._state.cancel(self))
    def tick(self, seconds: int) -> None:        self._apply("tick", self._state.tick(self, seconds))
    def toggle_maintenance(self) -> None:        self._apply("maint", self._state.enter_maintenance(self))

    @property
    def display(self) -> str:
        return self._state.display(self)

    # hardware hooks (would be injected ports in production)
    def refund(self, coins: list[int]) -> None:
        self.log.append(f"refund {coins}")

    def deliver(self, change_paise: int) -> None:
        self.log.append(f"deliver product, change {change_paise}")


if __name__ == "__main__":
    m = Machine()
    print(m.display)
    m.insert(500); m.select(700); print(m.display)
    m.tick(59); m.insert(500); print(m.display)
    m.dispense(); print(m.display)
    m.insert(100); m.tick(60); print(m.display)          # auto-refund on inactivity
    try:
        m.dispense()
    except InvalidActionError as err:
        print("refused:", err)
    print(*m.log, sep="\n")
```

- **Python internals:** Delegation `self._state.insert(self, paise)` is plain dynamic dispatch — the state object's *type* selects the behaviour, replacing every `if state ==` chain with the MRO lookup. State classes with no data can be **shared singletons** (module-level instances) to avoid allocation; here fresh instances keep it simple. Python permits a striking alternative: **swapping `self.__class__`** at runtime (`self.__class__ = Ready`) so the object *becomes* its state class — legal (same `__slots__` layout required), used in some stdlib/async code, and a great interview aside about how dynamic Python's object model is (use sparingly; it confuses readers and type checkers). Returning the next state instead of mutating the context from inside the state keeps transitions *visible in one place* (`_apply`), which makes logging, persistence, and thread-safety (wrap `_apply` in a lock) straightforward.
- **Build & drill:** Draw the state diagram for this machine *and* for Day 14's `Order`; convert `Order` to the State pattern and decide, in writing, whether it was worth it (hint: probably not — its behaviour is thin). Add a `Jammed` state entered from `dispense` on a hardware error, exited only by maintenance. Model a traffic light with timed transitions and a pedestrian-request guard. Then implement Day 14's transition *table* and Day 34's *classes* for an elevator door (open/closed/opening/closing/obstructed) and write the one-paragraph rule for choosing between them.
- **Recall:** Table-driven FSM vs State classes — the crossover criterion. Why do states return the next state? What is `self.__class__` swapping and when would you avoid it?

### Day 35 — Structural I: Decorator, Proxy, Adapter, Facade (the wrapping patterns)

- **Concept & why it matters:** Four patterns that all *wrap* an object, distinguished by intent: **Decorator** adds behaviour while keeping the same interface, stackable (`Compressed(Encrypted(FileStream))`); **Proxy** controls *access* with the same interface — lazy (create on first use), caching, protection (auth checks), remote (network stub), or logging; **Adapter** converts one interface into another the client expects (the legacy or third-party wrapper); **Facade** gives a simple entry point over a complex subsystem (many objects behind one method). The same interface? Decorator/Proxy. Different interface? Adapter. Many objects, one door? Facade. In Python, function decorators (Day 19) cover the *function* case; today is the *object* case, where `__getattr__` delegation makes transparent wrappers tiny. **Why it matters for LLD:** "add caching/logging/auth without touching the service," "integrate a third-party payment SDK," "give the UI one method for a 6-step checkout" — three interview staples, three patterns.
- **Real-world case study:** **Java's `java.io` streams** (`BufferedReader(new InputStreamReader(new FileInputStream(f)))`) are the textbook Decorator stack; Python's `io` module does the same (`TextIOWrapper(BufferedReader(FileIO))`) and `open()` is the Facade that builds it. **ORM lazy loading** (`user.orders` triggering a query on first access) is a virtual Proxy; **CDNs** are caching proxies for the whole internet (Day 15 of the backend plan). **The `requests` library** is a Facade over `urllib3`, sockets, and TLS. **Adapters** are every `XyzGatewayAdapter` written to plug Stripe/Razorpay/PayPal into one `PaymentGateway` port (Day 24).
- **Design problem — integrate a legacy payment SDK** (methods `makePayment(amountInRupees: float, cardNo: str) -> dict`) into the Day 24 `PaymentGateway` port, add response caching for idempotent lookups, add call logging, and give the checkout UI one `place_order()` call over inventory + payment + notification. Approaches: **(A) modify the service to call the SDK directly and add `if cache:`/`log` lines** — the domain now depends on a vendor API; every cross-cutting concern bloats the service. **(B) one big `PaymentWrapper` doing adaptation + caching + logging** — works, but three reasons to change in one class (SRP), and you can't reuse the caching for the notifier. **(C) separate wrappers by intent: `LegacyPaymentAdapter` (interface translation + unit conversion), `CachingProxy` (idempotent reads), `LoggingDecorator` (generic, via `__getattr__`), composed in the root; `CheckoutFacade` for the UI** — each is 20 lines, reusable, individually testable. Tradeoffs: A never; B when there's truly one concern; C when concerns are independent (they are).
- **Thought process → decision:** Count the *intents*: convert interface (Adapter), control access (Proxy), add behaviour (Decorator), simplify usage (Facade) — four different forces → four small objects → **C**. Put *unit conversion* (paise ↔ rupees float) in the adapter and make it exact (`Decimal`) — the Day 1 lesson is where adapters usually go wrong. Cache only *idempotent* calls (status lookups), never `charge` — a cached charge is a lost sale or a double charge.
- **Code:**

```python
"""Day 35 — Adapter, Proxy, Decorator and Facade around a legacy payment SDK."""

from __future__ import annotations

import logging
import time
from decimal import Decimal
from typing import Any, Protocol
from uuid import UUID, uuid4

logger = logging.getLogger(__name__)


# --- the third-party thing we cannot change ----------------------------------------------------
class LegacyPaySDK:
    def makePayment(self, amountInRupees: float, cardNo: str) -> dict[str, Any]:       # noqa: N802,N803
        return {"status": "OK", "ref": f"LP{int(time.time() * 1000) % 10_000_000}"}

    def fetchStatus(self, ref: str) -> dict[str, Any]:                                   # noqa: N802
        time.sleep(0.2)                                            # slow remote call
        return {"ref": ref, "state": "SETTLED"}


# --- the port our domain owns (Day 24) -------------------------------------------------------
class PaymentGateway(Protocol):
    def charge(self, customer_id: UUID, amount_paise: int, card_token: str) -> str: ...
    def status(self, reference: str) -> str: ...


class PaymentDeclined(Exception): ...


# --- Adapter: translate interface + units; hide vendor quirks --------------------------------
class LegacyPaymentAdapter:
    def __init__(self, sdk: LegacyPaySDK) -> None:
        self._sdk = sdk

    def charge(self, customer_id: UUID, amount_paise: int, card_token: str) -> str:
        rupees = float(Decimal(amount_paise) / 100)                # exact until the SDK forces a float
        response = self._sdk.makePayment(rupees, card_token)
        if response.get("status") != "OK":
            raise PaymentDeclined(response.get("reason", "declined"))
        return str(response["ref"])

    def status(self, reference: str) -> str:
        return str(self._sdk.fetchStatus(reference)["state"]).lower()


# --- Proxy: same interface, controls access (caching the idempotent call only) -----------------
class CachingPaymentProxy:
    def __init__(self, inner: PaymentGateway, ttl_seconds: float = 30.0) -> None:
        self._inner = inner
        self._ttl = ttl_seconds
        self._cache: dict[str, tuple[float, str]] = {}

    def charge(self, customer_id: UUID, amount_paise: int, card_token: str) -> str:
        return self._inner.charge(customer_id, amount_paise, card_token)     # never cached: not idempotent

    def status(self, reference: str) -> str:
        now = time.monotonic()
        hit = self._cache.get(reference)
        if hit and now - hit[0] < self._ttl:
            return hit[1]
        value = self._inner.status(reference)
        self._cache[reference] = (now, value)
        return value


# --- Decorator: adds behaviour transparently to ANY object via __getattr__ ---------------------
class LoggingDecorator:
    def __init__(self, inner: object, name: str) -> None:
        self._inner = inner
        self._name = name

    def __getattr__(self, attribute: str) -> Any:              # called only when normal lookup fails
        target = getattr(self._inner, attribute)
        if not callable(target):
            return target

        def logged(*args: Any, **kwargs: Any) -> Any:
            start = time.perf_counter()
            try:
                return target(*args, **kwargs)
            finally:
                logger.info("%s.%s took %.1f ms", self._name, attribute, (time.perf_counter() - start) * 1000)
        return logged


# --- Facade: one door for the UI over several collaborators ---------------------------------
class CheckoutFacade:
    def __init__(self, gateway: PaymentGateway, inventory: dict[str, int], notify: Any) -> None:
        self._gateway, self._inventory, self._notify = gateway, inventory, notify

    def place_order(self, customer_id: UUID, sku: str, qty: int, price_paise: int, card_token: str) -> str:
        if self._inventory.get(sku, 0) < qty:
            raise ValueError(f"insufficient stock for {sku}")
        self._inventory[sku] -= qty
        try:
            reference = self._gateway.charge(customer_id, price_paise * qty, card_token)
        except PaymentDeclined:
            self._inventory[sku] += qty
            raise
        self._notify(f"order confirmed, payment {reference}")
        return reference


if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO, format="%(message)s")
    gateway: PaymentGateway = LoggingDecorator(CachingPaymentProxy(LegacyPaymentAdapter(LegacyPaySDK())), "payments")  # type: ignore[assignment]
    checkout = CheckoutFacade(gateway, {"PEN": 10}, print)
    ref = checkout.place_order(uuid4(), "PEN", 2, 4550, "tok_visa")
    print(gateway.status(ref))       # ~200 ms (miss)
    print(gateway.status(ref))       # ~0 ms (hit)
```

- **Python internals:** `__getattr__` is invoked *only after* normal lookup fails, so a wrapper that defines a few methods and delegates the rest is a natural transparent proxy; `__getattribute__` intercepts *everything* (rarely needed, easy to recurse infinitely). Dunder methods bypass `__getattr__` (looked up on the type, Day 13), so a proxy that must support `len()` or `with` must define those explicitly — the `wrapt` library exists to get this right. `isinstance(proxy, Real)` is `False` for a wrapper — one reason to type against Protocols (structural) rather than concrete classes: the decorated gateway above satisfies `PaymentGateway` by shape. `time.monotonic()` for TTLs (never `time.time()`, which can jump). The `# noqa: N802` comments show a real-world adapter chore: the vendor's naming violates PEP 8, and the adapter is where that ugliness stops.
- **Build & drill:** Write a `RetryingProxy` (retry only `status`, never `charge`) and a `RateLimitedProxy`; stack all four and prove the order matters (logging outside caching shows hits; inside shows misses). Adapt a second SDK with a totally different shape to the same port and swap it in the root — the domain must not change. Write a `LazyProxy[T]` that constructs the wrapped object on first attribute access.
- **Recall:** Same interface or different? Which pattern for each? Why never cache `charge`? Why do dunders bypass `__getattr__`?

### Day 36 — Structural II: Composite, Iterator, and Visitor (working with trees)

- **Concept & why it matters:** **Composite** lets clients treat individual objects and groups of objects uniformly: a `File` and a `Folder` both implement `size()`, and a folder sums its children — recursion in the object structure. **Iterator** (built into Python via `__iter__`/generators, Day 9) lets you traverse a structure without exposing it — for trees, that means depth-first vs breadth-first generators. **Visitor** separates *operations* on a structure from the structure itself, so you can add a new operation (render, price, validate, export) without editing every node class — at the cost of making it hard to add new *node types*. Pythonic Visitor is `functools.singledispatch` or a `match` on type, instead of double-dispatch `accept()`/`visit_X()` boilerplate. **Why it matters for LLD:** file systems, org charts, menus, UI widget trees, expression evaluators, permission hierarchies, and nested comments are all composites — and "compute X over the whole tree" is the Visitor question.
- **Real-world case study:** **Python's `ast` module** is Composite + Visitor in the standard library: nodes form a tree, `ast.NodeVisitor` dispatches `visit_<ClassName>`, and tools like `black`, `ruff`, and `mypy` are visitors over that tree. **The DOM** is the web's composite; **React's reconciliation** is a visitor over two trees. **Filesystem `du`** is Composite's `size()` in one command.
- **Design problem — an in-memory file system** supporting files and folders, total size, find-by-extension, a tree printout, and a JSON export, with more operations expected. Approaches: **(A) `isinstance(node, Folder)` checks in every operation function** — operations live outside the structure (good for adding operations) but each does its own recursion and type-switching; a new node type (symlink) breaks all of them. **(B) Composite with each operation as a method on `Node` (`size()`, `find()`, `render()`, `to_json()`)** — uniform recursion; adding an operation edits every node class. **(C) Composite for the structure (`size()`, children management) + Visitor (`singledispatch`) for the open-ended operations + generator-based traversal for iteration** — structure stable, operations extensible. Tradeoffs: A is the smell; B when operations are few and stable; C when operations grow (they do here) — with the honest caveat that new *node types* then touch every visitor.
- **Thought process → decision:** Ask *"which grows faster: node kinds or operations?"* Operations → **C**. Keep intrinsic, universal behaviour (size, children, path) on the nodes; move everything else to visitors. Provide *one* traversal generator (`walk()`) that every operation reuses, so recursion is written once. Invariants: names unique within a folder; a folder cannot be added to its own subtree (cycle check).
- **Code:**

```python
"""Day 36 — Composite (nodes), Iterator (walk generator), Visitor (singledispatch operations)."""

from __future__ import annotations

from abc import ABC, abstractmethod
from collections.abc import Iterator
from dataclasses import dataclass
from functools import singledispatch
from typing import Any


class FileSystemError(Exception): ...


class Node(ABC):
    def __init__(self, name: str) -> None:
        if not name or "/" in name:
            raise FileSystemError(f"invalid name {name!r}")
        self.name = name
        self.parent: Folder | None = None

    @abstractmethod
    def size(self) -> int: ...                       # intrinsic: every node kind knows its size

    @property
    def path(self) -> str:
        return self.name if self.parent is None else f"{self.parent.path}/{self.name}"


class File(Node):
    def __init__(self, name: str, size_bytes: int) -> None:
        super().__init__(name)
        if size_bytes < 0:
            raise FileSystemError("size cannot be negative")
        self._size = size_bytes

    def size(self) -> int:
        return self._size


class Folder(Node):
    def __init__(self, name: str) -> None:
        super().__init__(name)
        self._children: dict[str, Node] = {}

    def add(self, node: Node) -> Node:
        if node.name in self._children:
            raise FileSystemError(f"{node.name!r} already exists in {self.path}")
        if isinstance(node, Folder) and (node is self or self._is_inside(node)):
            raise FileSystemError("cannot add a folder to its own subtree")
        node.parent = self
        self._children[node.name] = node
        return node

    def remove(self, name: str) -> None:
        node = self._children.pop(name, None)
        if node is None:
            raise FileSystemError(f"no entry {name!r} in {self.path}")
        node.parent = None

    def _is_inside(self, other: Folder) -> bool:
        ancestor = self.parent
        while ancestor is not None:
            if ancestor is other:
                return True
            ancestor = ancestor.parent
        return False

    @property
    def children(self) -> tuple[Node, ...]:
        return tuple(self._children.values())

    def size(self) -> int:                              # Composite: recursion through the structure
        return sum(child.size() for child in self._children.values())

    def walk(self, depth: int = 0) -> Iterator[tuple[int, Node]]:    # Iterator: one traversal, reused everywhere
        yield depth, self
        for child in self._children.values():
            if isinstance(child, Folder):
                yield from child.walk(depth + 1)
            else:
                yield depth + 1, child


# --- Visitor via singledispatch: add operations without touching node classes -------------------
@singledispatch
def render(node: Node, depth: int = 0) -> str:
    raise NotImplementedError(type(node))


@render.register
def _(node: File, depth: int = 0) -> str:
    return f"{'  ' * depth}{node.name} ({node.size()} B)"


@render.register
def _(node: Folder, depth: int = 0) -> str:
    lines = [f"{'  ' * depth}{node.name}/"]
    lines.extend(render(child, depth + 1) for child in node.children)
    return "\n".join(lines)


@singledispatch
def to_json(node: Node) -> dict[str, Any]:
    raise NotImplementedError(type(node))


@to_json.register
def _(node: File) -> dict[str, Any]:
    return {"type": "file", "name": node.name, "size": node.size()}


@to_json.register
def _(node: Folder) -> dict[str, Any]:
    return {"type": "folder", "name": node.name, "children": [to_json(c) for c in node.children]}


def find_by_extension(root: Folder, extension: str) -> list[File]:
    return [node for _, node in root.walk() if isinstance(node, File) and node.name.endswith(extension)]


if __name__ == "__main__":
    root = Folder("root")
    src = root.add(Folder("src"))
    assert isinstance(src, Folder)
    src.add(File("main.py", 1200)); src.add(File("util.py", 800))
    docs = root.add(Folder("docs")); assert isinstance(docs, Folder)
    docs.add(File("guide.md", 5000))
    print(render(root)); print(root.size(), [f.path for f in find_by_extension(root, ".py")])
    print(to_json(root)["children"][0])
    try:
        src.add(root)
    except FileSystemError as err:
        print("refused:", err)
```

- **Python internals:** `yield from` delegates to a sub-generator, making recursive tree generators one line per level; each level is a suspended frame, so very deep trees approach the recursion limit (`sys.getrecursionlimit()`, default 1000) — for pathological depth use an explicit stack. `singledispatch` dispatches on the *type of the first argument* using the MRO, so registering for `Node` gives a fallback and registering for a subclass overrides it — Visitor's double dispatch collapses into a type-keyed dict lookup. Parent back-references create **reference cycles** (parent ↔ child), which reference counting alone can't free; CPython's cycle GC handles them, or use `weakref.proxy` for the parent link in memory-sensitive trees (Day 17). `isinstance` inside `walk` is acceptable because it's *structural* (leaf vs branch), not *behavioural* switching.
- **Build & drill:** Add a `Symlink` node and observe exactly which visitors must change (the honest cost of Visitor). Implement an **expression tree** (`Number`, `Add`, `Multiply`) with `evaluate` and `to_string` visitors, then add `Variable` with an environment. Build an **organization chart** composite with `headcount()` and `total_salary()` and a breadth-first `walk`. Write a classic double-dispatch Visitor (`accept(visitor)`/`visit_file`) once, then explain why `singledispatch` is preferable in Python.
- **Recall:** Composite's core idea in one sentence. Which grows faster question — and which pattern each answer favours? How does `singledispatch` replace double dispatch?

### Day 37 — Behavioural/Structural III: Chain of Responsibility, Mediator, Flyweight

- **Concept & why it matters:** **Chain of Responsibility**: pass a request along a sequence of handlers, each deciding to handle it, transform it, or pass it on — middleware pipelines, approval workflows, logging levels, support escalation. Design choices: *first handler wins* vs *every handler participates*; how the chain is assembled (linked handlers vs a list); what happens when nobody handles. **Mediator**: many objects that would otherwise reference each other (n² coupling — chat participants, form widgets, aircraft) talk only to a central mediator that coordinates. It's a Facade for *interactions* rather than for a subsystem; its risk is becoming a God object. **Flyweight**: share immutable *intrinsic* state among many objects (glyphs in a document, tiles in a map, product attributes in a million cart lines) and pass *extrinsic* state (position) in — a memory pattern. **Why it matters for LLD:** request pipelines and approval flows are LLD staples; "chat room" and "air traffic control" are canonical Mediator problems; and Flyweight is the answer to "how would you store a million of these?"
- **Real-world case study:** **ASGI/WSGI middleware and Express/Koa** are Chain of Responsibility in every web framework: auth → rate limit → logging → routing. **Python's `logging`** propagates records up a logger hierarchy and through handler chains with level filters. **Air-traffic control** is the textbook Mediator: pilots don't coordinate with each other; the tower does. **Flyweight in CPython itself:** small integers (−5..256) and interned strings are shared objects; `sys.intern()` exposes it; `__slots__` shrinks the rest.
- **Design problem — an expense-approval pipeline**: a request passes validation, fraud checks, then manager → director → CFO approval based on amount; any step may reject; every step logs. Approaches: **(A) one `approve(request)` function with sequential `if`s** — fine for three steps; grows into a God function; steps can't be reordered per tenant. **(B) linked handler objects (`handler.set_next(...)`), each calling `next.handle()`** — the GoF form; assembly is fiddly; forgetting to call next silently drops requests. **(C) a `Pipeline` holding an ordered list of `Handler` objects; each returns a `Decision` (`CONTINUE`, `APPROVE`, `REJECT`); the pipeline drives iteration and guarantees a terminal decision** — assembly is a list literal per tenant; no handler can forget to forward; unhandled requests fail loudly. Tradeoffs: A for fixed tiny flows; B is the classic and error-prone; C is the Pythonic chain.
- **Thought process → decision:** Steps vary by tenant and order matters → data-driven assembly → **C**. Model the decision as an enum so *not deciding* is impossible to forget. The Mediator today is small: a `ChatRoom` where participants only know the room — do it alongside to fix the concept. Flyweight: intern the `ExpenseCategory` objects (rules, GL codes) shared by millions of expense lines.
- **Code:**

```python
"""Day 37 — Chain of Responsibility as a pipeline; a Mediator chat room; a Flyweight factory."""

from __future__ import annotations

from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from decimal import Decimal
from enum import Enum, auto


# --- Flyweight: shared intrinsic state -------------------------------------------------------
@dataclass(frozen=True, slots=True)
class ExpenseCategory:
    code: str
    gl_account: str
    requires_receipt: bool


class CategoryPool:
    """One instance per code; millions of expenses share them."""
    _pool: dict[str, ExpenseCategory] = {}

    @classmethod
    def get(cls, code: str, gl_account: str, requires_receipt: bool) -> ExpenseCategory:
        if code not in cls._pool:
            cls._pool[code] = ExpenseCategory(code, gl_account, requires_receipt)
        return cls._pool[code]


@dataclass
class Expense:                                  # extrinsic state lives here; category is shared
    employee: str
    amount: Decimal
    category: ExpenseCategory
    has_receipt: bool
    trail: list[str] = field(default_factory=list)


# --- Chain of Responsibility ----------------------------------------------------------------
class Decision(Enum):
    CONTINUE = auto()
    APPROVE = auto()
    REJECT = auto()


class Handler(ABC):
    @abstractmethod
    def handle(self, expense: Expense) -> Decision: ...


class ValidationHandler(Handler):
    def handle(self, expense: Expense) -> Decision:
        if expense.amount <= 0:
            expense.trail.append("validation: non-positive amount")
            return Decision.REJECT
        if expense.category.requires_receipt and not expense.has_receipt:
            expense.trail.append("validation: receipt missing")
            return Decision.REJECT
        expense.trail.append("validation: ok")
        return Decision.CONTINUE


class ApprovalTier(Handler):
    def __init__(self, role: str, limit: Decimal) -> None:
        self._role, self._limit = role, limit

    def handle(self, expense: Expense) -> Decision:
        if expense.amount <= self._limit:
            expense.trail.append(f"{self._role}: approved")
            return Decision.APPROVE
        expense.trail.append(f"{self._role}: above limit, escalating")
        return Decision.CONTINUE


class Pipeline:
    def __init__(self, handlers: list[Handler]) -> None:
        if not handlers:
            raise ValueError("pipeline needs handlers")
        self._handlers = tuple(handlers)

    def run(self, expense: Expense) -> Decision:
        for handler in self._handlers:
            decision = handler.handle(expense)
            if decision is not Decision.CONTINUE:
                return decision
        expense.trail.append("pipeline: no handler decided")          # loud, not silent
        return Decision.REJECT


# --- Mediator: participants know only the room ------------------------------------------------
class ChatRoom:
    def __init__(self) -> None:
        self._members: dict[str, Participant] = {}

    def join(self, participant: Participant) -> None:
        if participant.name in self._members:
            raise ValueError(f"{participant.name} already in the room")
        self._members[participant.name] = participant
        participant.room = self

    def broadcast(self, sender: Participant, text: str) -> None:
        for member in self._members.values():
            if member is not sender:
                member.receive(sender.name, text)

    def whisper(self, sender: Participant, to: str, text: str) -> None:
        try:
            self._members[to].receive(sender.name, f"(private) {text}")
        except KeyError:
            raise LookupError(f"no member {to!r}") from None


class Participant:
    def __init__(self, name: str) -> None:
        self.name = name
        self.room: ChatRoom | None = None
        self.inbox: list[str] = []

    def say(self, text: str) -> None:
        if self.room is None:
            raise RuntimeError("not in a room")
        self.room.broadcast(self, text)

    def receive(self, sender: str, text: str) -> None:
        self.inbox.append(f"{sender}: {text}")


if __name__ == "__main__":
    travel = CategoryPool.get("TRAVEL", "6100", requires_receipt=True)
    assert travel is CategoryPool.get("TRAVEL", "6100", True)              # shared flyweight
    pipeline = Pipeline([ValidationHandler(), ApprovalTier("manager", Decimal("5000")),
                         ApprovalTier("director", Decimal("50000")), ApprovalTier("cfo", Decimal("10000000"))])
    e = Expense("asha", Decimal("12000"), travel, has_receipt=True)
    print(pipeline.run(e), e.trail)
    e2 = Expense("ravi", Decimal("300"), travel, has_receipt=False)
    print(pipeline.run(e2), e2.trail)

    room = ChatRoom()
    asha, ravi = Participant("asha"), Participant("ravi")
    room.join(asha); room.join(ravi)
    asha.say("hello"); room.whisper(ravi, "asha", "hi back")
    print(ravi.inbox, asha.inbox)
```

- **Python internals:** Flyweight is how CPython economizes: `sys.intern(s)` returns the canonical string object so comparisons become pointer checks and dict lookups skip `__eq__`; `sys.getsizeof` shows a frozen `slots=True` dataclass instance is far smaller than a dict-backed object — measure it for a million categories. `functools.lru_cache` on a constructor-like function is a one-line flyweight factory. A pipeline as a *tuple of handlers* is immutable once built, so it can be shared across threads without locking (only the `Expense` is mutated, and each request has its own). The Mediator's `participant.room = self` back-reference is a cycle — fine for short-lived rooms; use `weakref` for long-lived registries.
- **Build & drill:** Add a `FraudHandler` that rejects duplicate (employee, amount, day) submissions using a set, and a per-tenant pipeline factory (Day 28). Convert Day 32's `EventBus` into a Mediator-style `Dispatcher` and write two sentences on how Observer and Mediator differ. Implement Flyweight for `Card` (Day 13): 52 shared instances, `Card.get(rank, suit)`, and measure memory for a million-card list before/after.
- **Recall:** First-wins vs all-participate chains — when each? What is a Mediator's main risk? Intrinsic vs extrinsic state in Flyweight.

### Day 38 — Enterprise patterns: Repository, Unit of Work, Service Layer, Specification

- **Concept & why it matters:** The patterns that connect a domain model to *persistence* without letting the database leak in: **Repository** — a collection-like interface over storage (`add`, `get`, `list(spec)`), one per aggregate; the domain sees objects, never SQL (Day 18's generic version becomes a port with a SQLite adapter). **Unit of Work** — a context manager that groups changes into one atomic commit/rollback and tracks which repositories participated. **Service Layer (application service)** — the use-case orchestrator: takes a request, opens a UoW, loads aggregates, calls domain methods, commits, publishes events (Day 32) — the code the API/CLI/tests all call. **Specification** — composable query objects (`Overdue() & ByMember(id)`) so filtering logic isn't duplicated across repository methods. **Why it matters for LLD:** interviewers ask "how does this persist?" and "where does the transaction boundary go?" — these patterns are the standard vocabulary, and they are how your Library/Parking Lot moves from in-memory toy to a real service.
- **Real-world case study:** **cosmicpython** (*Architecture Patterns with Python*) documents MADE.com's real migration to Repository + UoW + Service Layer and the payoff: tests that run against fakes in milliseconds, a domain model free of ORM imports, and swappable storage. **Django's ORM** is an *Active Record* (objects that save themselves) — convenient, but it couples the model to the database, which is why large Django codebases grow a "services" layer anyway. Knowing both shapes and their tradeoff is expected of a senior candidate.
- **Design problem — persist the Library (Day 20) to SQLite** with atomic borrow (decrement copies + insert loan must succeed or fail together), testable without a database. Approaches: **(A) put `sqlite3` calls inside `Library.borrow()`** — the domain imports the driver; tests need a DB; transaction boundaries are implicit and scattered. **(B) Active Record: `Loan.save()`, `Book.save()`** — each object commits itself; "borrow" becomes two commits that can half-succeed. **(C) Repository ports (`BookRepository`, `LoanRepository`) with SQLite and in-memory adapters; a `UnitOfWork` context manager owning the connection/transaction; a `LibraryService.borrow()` that does everything inside one `with uow:`** — atomic by construction, domain pure, tests use fakes. Tradeoffs: A/B are faster to type once; C is more files but is the only one where "atomic borrow" is a *guarantee*.
- **Thought process → decision:** The requirement "must succeed or fail together" *is* a transaction boundary → it needs an owner → **UoW** → **C**. Repositories expose only what use cases need (ISP, Day 23). The UoW's `__exit__` rolls back on exception and requires an explicit `commit()` — forgetting to commit loses work *safely* rather than committing half-done work.
- **Code:**

```python
"""Day 38 — Repository + Unit of Work + Service Layer over SQLite, with in-memory fakes for tests."""

from __future__ import annotations

import sqlite3
from abc import ABC, abstractmethod
from dataclasses import dataclass
from datetime import date, timedelta
from types import TracebackType
from uuid import UUID, uuid4


class LibraryError(Exception): ...
class BookUnavailableError(LibraryError): ...


@dataclass
class Book:
    isbn: str
    title: str
    copies_available: int

    def take_copy(self) -> None:                    # domain rule stays on the domain object
        if self.copies_available <= 0:
            raise BookUnavailableError(self.isbn)
        self.copies_available -= 1


@dataclass(frozen=True)
class Loan:
    id: UUID
    isbn: str
    member_id: str
    due_on: date


# --- ports -------------------------------------------------------------------------------------
class BookRepository(ABC):
    @abstractmethod
    def get(self, isbn: str) -> Book | None: ...
    @abstractmethod
    def add(self, book: Book) -> None: ...
    @abstractmethod
    def update(self, book: Book) -> None: ...


class LoanRepository(ABC):
    @abstractmethod
    def add(self, loan: Loan) -> None: ...
    @abstractmethod
    def open_for_member(self, member_id: str) -> list[Loan]: ...


class UnitOfWork(ABC):
    books: BookRepository
    loans: LoanRepository

    def __enter__(self) -> UnitOfWork:
        return self

    def __exit__(self, exc_type: type[BaseException] | None, exc: BaseException | None, tb: TracebackType | None) -> None:
        self.rollback()                              # commit must be explicit; anything else is rolled back

    @abstractmethod
    def commit(self) -> None: ...
    @abstractmethod
    def rollback(self) -> None: ...


# --- SQLite adapters ---------------------------------------------------------------------------
SCHEMA = """
CREATE TABLE IF NOT EXISTS books (isbn TEXT PRIMARY KEY, title TEXT NOT NULL, copies_available INTEGER NOT NULL CHECK (copies_available >= 0));
CREATE TABLE IF NOT EXISTS loans (id TEXT PRIMARY KEY, isbn TEXT NOT NULL REFERENCES books(isbn), member_id TEXT NOT NULL, due_on TEXT NOT NULL, returned_on TEXT);
"""


class SqliteBookRepository(BookRepository):
    def __init__(self, conn: sqlite3.Connection) -> None:
        self._conn = conn

    def get(self, isbn: str) -> Book | None:
        row = self._conn.execute("SELECT isbn, title, copies_available FROM books WHERE isbn = ?", (isbn,)).fetchone()
        return Book(*row) if row else None

    def add(self, book: Book) -> None:
        self._conn.execute("INSERT INTO books VALUES (?, ?, ?)", (book.isbn, book.title, book.copies_available))

    def update(self, book: Book) -> None:
        self._conn.execute("UPDATE books SET copies_available = ? WHERE isbn = ?", (book.copies_available, book.isbn))


class SqliteLoanRepository(LoanRepository):
    def __init__(self, conn: sqlite3.Connection) -> None:
        self._conn = conn

    def add(self, loan: Loan) -> None:
        self._conn.execute("INSERT INTO loans (id, isbn, member_id, due_on) VALUES (?, ?, ?, ?)",
                           (str(loan.id), loan.isbn, loan.member_id, loan.due_on.isoformat()))

    def open_for_member(self, member_id: str) -> list[Loan]:
        rows = self._conn.execute("SELECT id, isbn, member_id, due_on FROM loans WHERE member_id = ? AND returned_on IS NULL", (member_id,))
        return [Loan(UUID(i), isbn, m, date.fromisoformat(d)) for i, isbn, m, d in rows]


class SqliteUnitOfWork(UnitOfWork):
    def __init__(self, path: str = ":memory:") -> None:
        self._conn = sqlite3.connect(path, isolation_level=None)   # we control transactions explicitly
        self._conn.execute("PRAGMA foreign_keys = ON")
        self._conn.executescript(SCHEMA)

    def __enter__(self) -> UnitOfWork:
        self._conn.execute("BEGIN")
        self.books = SqliteBookRepository(self._conn)
        self.loans = SqliteLoanRepository(self._conn)
        return self

    def commit(self) -> None:
        self._conn.execute("COMMIT")
        self._conn.execute("BEGIN")                 # keep the UoW usable until __exit__

    def rollback(self) -> None:
        if self._conn.in_transaction:
            self._conn.execute("ROLLBACK")


# --- Service Layer: the use case -------------------------------------------------------------
class LibraryService:
    LOAN_DAYS = 14
    MAX_LOANS = 3

    def __init__(self, uow: UnitOfWork) -> None:
        self._uow = uow

    def borrow(self, isbn: str, member_id: str, today: date) -> Loan:
        with self._uow as uow:
            if len(uow.loans.open_for_member(member_id)) >= self.MAX_LOANS:
                raise LibraryError(f"{member_id} has reached the loan limit")
            book = uow.books.get(isbn)
            if book is None:
                raise LibraryError(f"unknown isbn {isbn}")
            book.take_copy()                                     # domain rule; raises if none left
            uow.books.update(book)
            loan = Loan(uuid4(), isbn, member_id, today + timedelta(days=self.LOAN_DAYS))
            uow.loans.add(loan)
            uow.commit()                                         # both writes or neither
            return loan


if __name__ == "__main__":
    uow = SqliteUnitOfWork()
    with uow as u:
        u.books.add(Book("978-1", "Fluent Python", copies_available=1)); u.commit()
    service = LibraryService(uow)
    print(service.borrow("978-1", "asha", date(2026, 9, 3)))
    try:
        service.borrow("978-1", "ravi", date(2026, 9, 3))
    except BookUnavailableError as err:
        print("refused:", err)
    with uow as u:
        print(u.books.get("978-1"), len(u.loans.open_for_member("ravi")))     # 0 copies; ravi has 0 loans
```

- **Python internals:** `sqlite3` with `isolation_level=None` disables the module's implicit transaction management so `BEGIN`/`COMMIT`/`ROLLBACK` are exactly where you write them — the UoW *is* the transaction. `__exit__` receives the exception (if any); returning `None`/`False` lets it propagate after rollback — you rarely want a UoW to swallow errors. A `CHECK` constraint in the schema is the database enforcing your domain invariant as a last line of defence — belt and braces with `Book.take_copy()`. `sqlite3` connections are not thread-safe by default (`check_same_thread`) — one UoW per thread/request is the rule, which is why the UoW is created per use case in real apps rather than shared.
- **Build & drill:** Write `InMemoryUnitOfWork` with dict-backed repositories and run the *same* service tests against both (parametrize) — the contract test from Day 16 at scale. Add `return_book` with a late-fee calculation and a `Specification` class (`OverdueSpec(today) & ForMember(id)`) that both repositories can evaluate. Publish `BookBorrowed` on the Day 32 bus *after* commit and explain why after, not before.
- **Recall:** Where is the transaction boundary and who owns it? Repository vs Active Record. Why is commit explicit and rollback the default?

### Day 39 — Concurrency patterns for LLD: producer–consumer, worker pools, read/write locks, futures, and asyncio

- **Concept & why it matters:** Day 27 gave you locks; today gives you the *patterns* that avoid needing many of them. **Producer–Consumer** with a bounded `queue.Queue`: producers block when full (backpressure), consumers block when empty; a *sentinel* or `shutdown()` ends it. **Worker pool** (`ThreadPoolExecutor`): N workers pulling tasks; `Future` as a handle to a result-not-yet-available; `as_completed` and timeouts. **Read/Write lock**: many readers or one writer, for read-heavy shared structures (Python has none built in — you build it from `Condition`). **Active Object**: an object that owns a thread and a queue, executing method calls asynchronously — how a single-threaded component (a printer, a game loop, a ticket allocator) can be driven by many callers safely *without* locks on every method. **Immutable messages** between threads. **asyncio** in one paragraph: single-threaded cooperative concurrency for I/O-bound waiting; `async def`/`await`; not a replacement for threads in CPU-bound or blocking-library code; `asyncio.Lock` exists because `await` points are where interleaving happens. **Why it matters for LLD:** job schedulers, rate limiters, caches, booking systems, and elevator controllers all need *a concurrency model*, and "one lock per method" is rarely it.
- **Real-world case study:** **Nginx and Redis** are single-threaded event loops (Active Object at process scale): one thread, a queue of events, no locks, predictable latency. **Celery/RQ/Sidekiq** are producer–consumer over a broker with worker pools. **Java's `ReadWriteLock`** and **Postgres MVCC** both encode "readers shouldn't block readers." The classic failure: an unbounded in-memory queue in a service that ingests faster than it processes — memory climbs until the OOM killer arrives; bounded queues + backpressure are the fix, and they must be *designed in*.
- **Design problem — a background job scheduler** that accepts jobs from many threads, runs at most N concurrently, supports priorities, retries failed jobs with backoff, exposes status, and shuts down gracefully. Approaches: **(A) spawn a thread per job** — unbounded threads; no priority; no backpressure; 10k jobs = 10k threads. **(B) `ThreadPoolExecutor` fed directly** — bounded workers, futures for status; but no priority and no retry semantics; shutdown drops queued work unpredictably. **(C) a `PriorityQueue` of immutable `Job` records, N worker threads consuming, a `Condition`-guarded status table, retries re-enqueued with a not-before time, graceful shutdown by sentinel + join** — every requirement met; more code; one lock protects the status table only. **(D) asyncio** — excellent if jobs are I/O-bound coroutines; wrong if jobs are CPU-bound or call blocking libraries. Tradeoffs: A never; B when jobs are simple and fire-and-forget; C for a real scheduler; D when everything is async-native.
- **Thought process → decision:** Requirements include priority, retry, and graceful shutdown → **C**. Design the *messages* as frozen dataclasses so workers never share mutable job state; keep the *only* shared mutable structure (status map) behind one lock with tiny critical sections; use `queue.PriorityQueue` (thread-safe) so scheduling needs no extra lock; treat `not_before` by re-queueing with a sleep-free check (a delayed queue) to avoid blocking a worker on `time.sleep`.
- **Code:**

```python
"""Day 39 — a bounded, prioritized, retrying job scheduler built on producer–consumer + worker pool."""

from __future__ import annotations

import heapq
import logging
import threading
import time
from collections.abc import Callable
from dataclasses import dataclass, field
from enum import Enum, auto
from itertools import count
from uuid import UUID, uuid4

logger = logging.getLogger(__name__)


class JobStatus(Enum):
    QUEUED = auto()
    RUNNING = auto()
    SUCCEEDED = auto()
    FAILED = auto()


@dataclass(frozen=True, order=True)
class ScheduledJob:
    """Immutable message. Ordering: priority (lower first), then not_before, then a tiebreaker."""
    priority: int
    not_before: float
    sequence: int
    id: UUID = field(compare=False)
    func: Callable[[], object] = field(compare=False)
    attempts_left: int = field(compare=False)
    max_attempts: int = field(compare=False)


class JobScheduler:
    def __init__(self, workers: int = 4, max_queue: int = 1000, base_backoff: float = 0.5) -> None:
        if workers <= 0:
            raise ValueError("workers must be positive")
        self._heap: list[ScheduledJob] = []
        self._cv = threading.Condition()                 # guards heap + status; also signals workers
        self._status: dict[UUID, JobStatus] = {}
        self._max_queue = max_queue
        self._base_backoff = base_backoff
        self._sequence = count()
        self._stopping = False
        self._threads = [threading.Thread(target=self._worker, name=f"worker-{i}", daemon=True) for i in range(workers)]
        for thread in self._threads:
            thread.start()

    def submit(self, func: Callable[[], object], *, priority: int = 5, retries: int = 2, delay: float = 0.0) -> UUID:
        job_id = uuid4()
        with self._cv:
            if self._stopping:
                raise RuntimeError("scheduler is shutting down")
            if len(self._heap) >= self._max_queue:
                raise RuntimeError("queue full")           # backpressure: refuse, don't grow without bound
            self._push(ScheduledJob(priority, time.monotonic() + delay, next(self._sequence), job_id, func, retries + 1, retries + 1))
            self._status[job_id] = JobStatus.QUEUED
        return job_id

    def _push(self, job: ScheduledJob) -> None:            # caller holds the lock
        heapq.heappush(self._heap, job)
        self._cv.notify()

    def status(self, job_id: UUID) -> JobStatus:
        with self._cv:
            return self._status[job_id]

    def _next_job(self) -> ScheduledJob | None:
        """Block until a job is due or shutdown. Returns None on shutdown."""
        with self._cv:
            while True:
                if self._stopping and not self._heap:
                    return None
                if self._heap:
                    wait = self._heap[0].not_before - time.monotonic()
                    if wait <= 0:
                        job = heapq.heappop(self._heap)
                        self._status[job.id] = JobStatus.RUNNING
                        return job
                    self._cv.wait(timeout=wait)            # sleep only until the earliest job is due
                else:
                    self._cv.wait()

    def _worker(self) -> None:
        while (job := self._next_job()) is not None:
            try:
                job.func()
            except Exception as err:                       # noqa: BLE001 — a job failure must not kill the worker
                self._on_failure(job, err)
            else:
                with self._cv:
                    self._status[job.id] = JobStatus.SUCCEEDED

    def _on_failure(self, job: ScheduledJob, err: Exception) -> None:
        remaining = job.attempts_left - 1
        with self._cv:
            if remaining > 0:
                attempt_number = job.max_attempts - remaining     # 1 for the first retry, 2 for the second …
                backoff = self._base_backoff * (2 ** attempt_number)
                logger.warning("job %s failed (%s); retrying in %.1fs", job.id, err, backoff)
                self._push(ScheduledJob(job.priority, time.monotonic() + backoff, next(self._sequence), job.id, job.func, remaining, job.max_attempts))
                self._status[job.id] = JobStatus.QUEUED
            else:
                logger.error("job %s failed permanently: %s", job.id, err)
                self._status[job.id] = JobStatus.FAILED

    def shutdown(self, *, wait: bool = True) -> None:
        with self._cv:
            self._stopping = True
            self._cv.notify_all()
        if wait:
            for thread in self._threads:
                thread.join()


class ReadWriteLock:
    """Many readers or one writer. Writers are preferred once waiting, to avoid starvation."""

    def __init__(self) -> None:
        self._cv = threading.Condition()
        self._readers = 0
        self._writer = False
        self._waiting_writers = 0

    def acquire_read(self) -> None:
        with self._cv:
            while self._writer or self._waiting_writers:
                self._cv.wait()
            self._readers += 1

    def release_read(self) -> None:
        with self._cv:
            self._readers -= 1
            if self._readers == 0:
                self._cv.notify_all()

    def acquire_write(self) -> None:
        with self._cv:
            self._waiting_writers += 1
            while self._writer or self._readers:
                self._cv.wait()
            self._waiting_writers -= 1
            self._writer = True

    def release_write(self) -> None:
        with self._cv:
            self._writer = False
            self._cv.notify_all()


if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO, format="%(threadName)s %(message)s")
    scheduler = JobScheduler(workers=2)
    flaky_calls = {"n": 0}

    def flaky() -> None:
        flaky_calls["n"] += 1
        if flaky_calls["n"] < 3:
            raise ConnectionError("blip")

    urgent = scheduler.submit(lambda: print("urgent ran"), priority=0)
    later = scheduler.submit(lambda: print("delayed ran"), priority=5, delay=0.3)
    retry_job = scheduler.submit(flaky, priority=3, retries=2)
    time.sleep(2.5)
    print(scheduler.status(urgent), scheduler.status(later), scheduler.status(retry_job))
    scheduler.shutdown()
```

- **Python internals:** `threading.Condition` bundles a lock with `wait()`/`notify()`: `wait()` *releases* the lock while sleeping and re-acquires it before returning — the loop `while not condition: cv.wait()` is mandatory because of spurious wake-ups and because another thread may have consumed the item first. `wait(timeout=)` gives a *delayed queue* without a sleeping worker. `heapq` on a dataclass with `order=True` compares field tuples; `compare=False` excludes the function and id from ordering (functions aren't orderable). `Future` objects (from `concurrent.futures`) are the thread-safe result cells behind `ThreadPoolExecutor`; `asyncio.Future` is their single-threaded cousin. An `asyncio` event loop is a `select`/`epoll`-driven scheduler that runs one coroutine at a time and switches only at `await` — so shared state needs no locks *between awaits*, but *does* across them (`asyncio.Lock`). Daemon threads + a sentinel/flag + `join()` is the graceful-shutdown idiom; never `Thread.stop()` (there isn't one, deliberately).
- **Build & drill:** Add job cancellation (queued only), a `wait(job_id, timeout)` using the Condition, and per-job timeouts (hint: you cannot kill a thread — run the job in a subprocess or make it cooperative). Rewrite the scheduler's I/O-bound variant with `asyncio.PriorityQueue` and compare the code size. Wrap Day 34's `Machine` as an Active Object (its own thread + queue) and observe that its methods no longer need a lock. Benchmark `ReadWriteLock` vs a plain `Lock` on a 95%-read workload.
- **Recall:** Why is `while` (not `if`) mandatory around `Condition.wait()`? What does a bounded queue buy you? When is asyncio the wrong tool? What is an Active Object?

### Day 40 — Anti-patterns, Pythonic replacements, and the pattern-selection matrix

- **Concept & why it matters:** Patterns misapplied are worse than no patterns. **Anti-patterns:** *pattern fever* (a `Factory` for a class with one implementation), *anemic domain model* (dataclasses with no behaviour + a giant service doing everything — the opposite of Information Expert), *God object*, *Singleton for convenience*, *deep inheritance*, *interface for one implementation* (in Python, premature ABCs), *premature abstraction* (YAGNI), *speculative generality*, *boolean parameters that select behaviour* (`send(msg, is_sms=True)` → two methods or Strategy), *stringly-typed everything*, *exceptions as control flow for non-exceptional paths*, *mutable default/shared state*. **Pythonic replacements** (Brandon Rhodes' *Python Patterns* thesis): first-class functions replace Strategy/Command/Template hooks; dict registries replace Factory/Abstract Factory; generators replace Iterator; `__getattr__` replaces Proxy/Decorator boilerplate; modules replace Singleton; dataclasses replace Builders for simple cases; `singledispatch` replaces Visitor; context managers replace try/finally resource patterns; `match` replaces some State/Visitor dispatch. **Why it matters for LLD:** interviewers at Python shops reward *restraint*: knowing the pattern and choosing the smaller idiom is the sign you understand the force, not the name.
- **Real-world case study:** **"FizzBuzz Enterprise Edition"** (a satirical GitHub repo) implements FizzBuzz with factories, strategies, and visitors across dozens of files — a joke that every reviewer has seen for real. Contrast **the Python standard library**: `sorted(key=)` instead of Comparator classes, `contextlib` instead of resource-handler hierarchies, `dict` registries everywhere. Peter Norvig's 1998 talk *Design Patterns in Dynamic Languages* found 16 of 23 GoF patterns are "invisible or simpler" in dynamic languages — that observation is why this day exists.
- **Design problem — you inherit an over-engineered notification module**: `NotificationFactoryProvider` → `AbstractNotificationFactory` → `EmailNotificationFactory` → `EmailNotifierBuilder` → `EmailNotifier` (one implementation each), a `NotificationStrategyContext`, and a `NotificationVisitor` for formatting three message types. Approaches: **(A) leave it — it "follows patterns"** — every change touches six files; new hires can't find the `send` call. **(B) delete everything and write one function** — fast, but if there genuinely are three channels and two formats, you'll re-grow the structure badly. **(C) *identify the real forces*, keep exactly the structure they need, and collapse the rest to Python idioms:** three channels → a `Protocol` + dict registry (Days 16/28); formatting per message type → `singledispatch` (Day 36); retry → a decorator (Day 19); no builder (kwargs suffice); no singleton (inject). Tradeoffs: A is pattern-worship; B is the overcorrection; C is the judgment.
- **Thought process → decision:** For each class, ask *"what force does this answer, and is that force present?"* Write the answer down. Anything with no force goes. That's **C**. The result is ~1/5 the code and *more* extensible where extension is actually expected. This exercise — the *pattern audit* — is something you should be able to perform on any codebase, including your own from Phase 4.
- **Code (the collapsed result — write the bloated "before" yourself first, it's instructive):**

```python
"""Day 40 — after the pattern audit: only the structure the forces justify."""

from __future__ import annotations

import logging
from dataclasses import dataclass
from functools import singledispatch
from typing import Protocol

logger = logging.getLogger(__name__)


# Force: three message kinds with different rendering → small types + singledispatch (not a Visitor hierarchy)
@dataclass(frozen=True)
class OrderConfirmation:
    order_id: str
    total: str


@dataclass(frozen=True)
class PasswordReset:
    link: str


@dataclass(frozen=True)
class PlainText:
    text: str


@singledispatch
def render(message: object) -> str:
    raise TypeError(f"cannot render {type(message).__name__}")


@render.register
def _(message: OrderConfirmation) -> str:
    return f"Your order {message.order_id} ({message.total}) is confirmed."


@render.register
def _(message: PasswordReset) -> str:
    return f"Reset your password: {message.link}"


@render.register
def _(message: PlainText) -> str:
    return message.text


# Force: multiple channels, added over time → Protocol + registry (not AbstractFactory + Builder + Singleton)
class Channel(Protocol):
    def deliver(self, recipient: str, body: str) -> None: ...


class EmailChannel:
    def __init__(self, smtp_host: str) -> None:
        self._host = smtp_host

    def deliver(self, recipient: str, body: str) -> None:
        print(f"[email:{self._host}] {recipient}: {body}")


class SmsChannel:
    def deliver(self, recipient: str, body: str) -> None:
        print(f"[sms] {recipient}: {body[:60]}")


class Notifications:
    """The whole module's public surface. Channels injected; nothing global."""

    def __init__(self, channels: dict[str, Channel]) -> None:
        if not channels:
            raise ValueError("at least one channel is required")
        self._channels = dict(channels)

    def send(self, channel: str, recipient: str, message: object) -> None:
        try:
            target = self._channels[channel]
        except KeyError:
            raise LookupError(f"unknown channel {channel!r}; have {sorted(self._channels)}") from None
        target.deliver(recipient, render(message))


if __name__ == "__main__":
    notifications = Notifications({"email": EmailChannel("smtp.example.com"), "sms": SmsChannel()})
    notifications.send("email", "asha@example.com", OrderConfirmation("42", "₹1,200.00"))
    notifications.send("sms", "+919999999999", PasswordReset("https://x.io/r/abc"))
```

**The pattern-selection matrix (memorize the *force* column):**

| Force you observe | Pattern | Pythonic first choice |
|---|---|---|
| Creating the right concrete class from a key | Factory / Abstract Factory | dict registry, `__init_subclass__`, classmethod constructors |
| Many optional parts, cross-field invariants, multi-step construction | Builder | kwargs + frozen dataclass; Builder only when steps accumulate |
| Exactly one shared instance | Singleton | module or injected instance; never a Singleton class in app code |
| Expensive objects to reuse | Object Pool | `ObjectPool` with context-managed lease |
| Interchangeable algorithms | Strategy | a function / `Callable`, class when stateful |
| Fixed skeleton, varying steps | Template Method | ABC with hooks; or Strategy injected |
| One change, many interested parties | Observer / Pub-Sub | event bus with typed events, `WeakMethod` |
| Requests as objects (queue, undo, log) | Command / Memento | closures for simple; classes when `undo`/serialize |
| Behaviour depends on lifecycle state | State | enum + table when thin; State classes when rich |
| Add behaviour / control access transparently | Decorator / Proxy | function decorators; `__getattr__` delegation |
| Mismatched interface | Adapter | thin class translating and converting units |
| Complex subsystem, simple entry | Facade | one class/function at the boundary |
| Tree of parts and wholes | Composite | recursive classes + generator `walk()` |
| Operations over a structure growing faster than node kinds | Visitor | `singledispatch` / `match` |
| Sequential handlers that may stop the flow | Chain of Responsibility | pipeline over a list of handlers |
| n² object interactions | Mediator | a coordinator object; watch for God-object drift |
| Millions of similar objects | Flyweight | shared frozen `slots` instances via a pool |
| Persistence without leaking storage | Repository / UoW / Service Layer | ABC ports + SQLite/in-memory adapters, UoW as context manager |
| Concurrent producers and consumers | Producer–Consumer / Worker Pool / Active Object | `queue.Queue`, `ThreadPoolExecutor`, `Condition` |

- **Python internals:** Norvig's "invisible patterns" are invisible because of concrete language features: first-class functions and closures (Strategy/Command), duck typing and Protocols (Adapter/Interface), `__getattr__` and the data model (Proxy/Decorator), the import system (Singleton), generators (Iterator), `singledispatch`/MRO (Visitor). Each feature is something you've studied in the internals blocks — which is why you can now *predict* when a pattern collapses in Python.
- **Build & drill:** Perform a **pattern audit** on your Day 20 Library and Day 25 Vending Machine: for each class, one line naming its force (or "none — remove"). Refactor one over-engineered piece of your own code down. Then do the reverse: find one place where you *should* have used a pattern and didn't (a growing `if` chain, a class doing I/O and logic) and fix it. Write your own version of the matrix from memory.
- **Recall:** Name five anti-patterns and the smell each shows. Which GoF patterns are "invisible" in Python and why? What single question kills pattern fever?

### Day 41 — Consolidation III: the patterns checkpoint and a multi-pattern build

- **Do:** teach-backs for Days 28–40 (60 seconds each, aloud): the force, the classic form, the Pythonic form, when not to use it. Rebuild the selection matrix from memory. Flashcard sweep.
- **Synthesis build — a plugin-based "Smart Home" controller** that deliberately exercises ≥ 8 patterns with a *documented force for each*: devices discovered by kind (Factory/registry), each device a State machine (light: off/on/dimmed; lock: locked/unlocked/jammed), scenes as Macros of Commands with undo ("movie night", then revert), an event bus for sensor events (Observer) with automations as handlers, a rules pipeline (Chain of Responsibility: safety checks → schedules → user rules), third-party device SDK adapters (Adapter), a `Home` Facade for the app, a `Room`/`Floor`/`Home` Composite with `power_usage()` via `singledispatch` (Visitor), and a background `JobScheduler` (Day 39) for timed automations. ≥ 25 tests, `mypy --strict` clean, a `README` with a UML diagram and a table mapping *pattern → force → class*.
- **Pattern audit of your own build:** for every class, one line naming its force. Delete anything without one. Interviewers love hearing "I considered X and rejected it because…" — write three such rejections into the README.
- **Checkpoint self-test (60 min, no references):** design and code a **Notification/Alerting Service**: rules that match events, multiple channels, per-user preferences and quiet hours, deduplication, retry with backoff, and an audit log. Grade: is there one registry for channels? Are rules data or a chain? Are cross-cutting concerns decorators/proxies rather than inline? Is anything a Singleton that shouldn't be? If you can name the force behind every structural decision, Phase 4 will feel like applying, not learning.

---

# PHASE 4 — LLD Problems, End to End (Days 42–57)

Every day is a full system, run through the six-step method **on paper first** (40 minutes, no code), then compared to the walkthrough, then coded with tests. Each day names the *new force* the system introduces — that's what you're learning; the patterns are already yours. The code blocks show the *core* (the part where the design decision lives); you complete the rest. From Day 49 onward, every system is concurrent.

### Day 42 — Parking Lot (the canonical problem: allocation, ticketing, fees, and one lock)

- **Concept & why it matters:** The problem every LLD course starts with, because it contains every basic force: a *closed set* of kinds (vehicle/spot sizes), a *resource allocation* decision (which spot?), a *ticket* as the record of a transaction, *pricing* that varies (Day 31), *multiple containers* (floors), and *concurrency* at the gate (two cars, one spot). New force today: **allocation as a swappable strategy over a structure the lot owns**, and **the entry/exit transaction as the unit of consistency**.
- **Real-world case study:** Airport and mall parking systems (e.g. Skidata, Amano) separate exactly these concerns: gate controllers issue tickets, a central allocation service assigns bays by category and occupancy, and a tariff engine prices on exit — the tariff changes monthly, the allocation logic changes yearly, the gate hardware changes never. Their failure mode: two gates assigning the same "last available" bay when the occupancy service isn't the single owner of the count.
- **Design problem (run the method first):** Requirements: multiple floors; spot sizes small/medium/large; vehicles bike/car/truck (a vehicle fits any spot ≥ its size); entry issues a ticket; exit computes a fee by duration and vehicle; "lot full" must be reported; new vehicle types and pricing rules must not edit the lot. Key fork — *who chooses the spot?* **(A) `ParkingLot.enter()` loops floors and spots inline** — allocation policy fused with the lot; changing "nearest to entrance" or "tightest fit" edits the lot. **(B) each `Floor` allocates itself and the lot asks floors in order** — distributes the policy across floors; cross-floor preferences are impossible. **(C) a `SpotAllocator` strategy that receives the floors and returns a spot; the lot owns the structure and the lock, the allocator owns the choice** — policy swappable, lot stable, one critical section. Second fork — *fee*: Strategy as callable (Day 31), injected.
- **Thought process → decision:** Allocation varies (first-fit, tightest-fit, nearest-entrance) and pricing varies; the *structure* (floors→spots) and the *transaction* (ticket in, fee out) are stable → **C** + injected fee policy. The lock lives on the lot around the whole `enter` (choose + park) and `exit` (lookup + free) — a per-spot lock would not stop two threads choosing the same free spot (check-then-act, Day 27). Tickets indexed by id and by plate so both "lost ticket" and "exit by plate" work.
- **Code (core):**

```python
"""Day 42 — Parking Lot: lot owns structure + lock; allocator owns choice; fee policy injected."""

from __future__ import annotations

import math
import threading
from collections.abc import Callable, Iterator, Sequence
from dataclasses import dataclass, field
from datetime import datetime, timezone
from decimal import Decimal
from enum import Enum
from typing import Protocol
from uuid import UUID, uuid4


class Size(Enum):
    SMALL = 1
    MEDIUM = 2
    LARGE = 3


@dataclass(frozen=True)
class Vehicle:
    plate: str
    size: Size


class ParkingError(Exception): ...
class LotFullError(ParkingError): ...
class UnknownTicketError(ParkingError): ...


class Spot:
    def __init__(self, spot_id: str, size: Size) -> None:
        self.id, self.size = spot_id, size
        self._vehicle: Vehicle | None = None

    @property
    def is_free(self) -> bool:
        return self._vehicle is None

    def fits(self, vehicle: Vehicle) -> bool:
        return self.is_free and vehicle.size.value <= self.size.value

    def park(self, vehicle: Vehicle) -> None:
        if not self.fits(vehicle):
            raise ParkingError(f"spot {self.id} cannot take {vehicle.plate}")
        self._vehicle = vehicle

    def release(self) -> Vehicle:
        if self._vehicle is None:
            raise ParkingError(f"spot {self.id} is already empty")
        vehicle, self._vehicle = self._vehicle, None
        return vehicle


class Floor:
    def __init__(self, number: int, spots: Sequence[Spot]) -> None:
        self.number = number
        self._by_size: dict[Size, list[Spot]] = {size: [] for size in Size}
        for spot in spots:
            self._by_size[spot.size].append(spot)

    def free_spots(self, size: Size) -> Iterator[Spot]:
        return (s for s in self._by_size[size] if s.is_free)

    def free_count(self) -> dict[Size, int]:
        return {size: sum(1 for s in spots if s.is_free) for size, spots in self._by_size.items()}


class SpotAllocator(Protocol):
    def choose(self, floors: Sequence[Floor], vehicle: Vehicle) -> Spot | None: ...


class TightestFitAllocator:
    """Smallest size that fits, lowest floor first — keeps big spots free for big vehicles."""
    def choose(self, floors: Sequence[Floor], vehicle: Vehicle) -> Spot | None:
        for size in sorted(Size, key=lambda s: s.value):
            if size.value < vehicle.size.value:
                continue
            for floor in floors:
                spot = next(floor.free_spots(size), None)
                if spot is not None:
                    return spot
        return None


@dataclass(frozen=True)
class Ticket:
    vehicle: Vehicle
    spot_id: str
    floor_number: int
    entered_at: datetime
    id: UUID = field(default_factory=uuid4)


FeePolicy = Callable[[Ticket, datetime], Decimal]

HOURLY_RATE = {Size.SMALL: Decimal("10"), Size.MEDIUM: Decimal("30"), Size.LARGE: Decimal("60")}


def hourly_fee(ticket: Ticket, exited_at: datetime) -> Decimal:
    hours = max(1, math.ceil((exited_at - ticket.entered_at).total_seconds() / 3600))
    return HOURLY_RATE[ticket.vehicle.size] * hours


class ParkingLot:
    def __init__(self, floors: Sequence[Floor], allocator: SpotAllocator, fee_policy: FeePolicy,
                 clock: Callable[[], datetime] = lambda: datetime.now(timezone.utc)) -> None:
        if not floors:
            raise ValueError("a lot needs at least one floor")
        self._floors = tuple(floors)
        self._spots = {spot.id: spot for floor in floors for size in Size for spot in floor._by_size[size]}
        self._allocator, self._fee_policy, self._clock = allocator, fee_policy, clock
        self._tickets: dict[UUID, Ticket] = {}
        self._by_plate: dict[str, UUID] = {}
        self._lock = threading.Lock()

    def enter(self, vehicle: Vehicle) -> Ticket:
        with self._lock:                                       # choose + park is one atomic step
            if vehicle.plate in self._by_plate:
                raise ParkingError(f"{vehicle.plate} is already inside")
            spot = self._allocator.choose(self._floors, vehicle)
            if spot is None:
                raise LotFullError(f"no spot for a {vehicle.size.name.lower()} vehicle")
            spot.park(vehicle)
            floor_number = next(f.number for f in self._floors if spot in f._by_size[spot.size])
            ticket = Ticket(vehicle, spot.id, floor_number, self._clock())
            self._tickets[ticket.id] = ticket
            self._by_plate[vehicle.plate] = ticket.id
            return ticket

    def exit(self, ticket_id: UUID) -> Decimal:
        with self._lock:
            ticket = self._tickets.pop(ticket_id, None)
            if ticket is None:
                raise UnknownTicketError(str(ticket_id))
            del self._by_plate[ticket.vehicle.plate]
            self._spots[ticket.spot_id].release()
            return self._fee_policy(ticket, self._clock())

    def availability(self) -> dict[int, dict[Size, int]]:
        with self._lock:
            return {f.number: f.free_count() for f in self._floors}
```

- **Python internals:** `next(generator, None)` is the idiom for "first match or nothing" without building a list — O(1) extra memory over the lazy `free_spots`. The `_spots` flat index is built once so `exit` is O(1) — a deliberate space-for-time trade; note that `Floor._by_size` is accessed by the lot (same module, "friend" access) — in a multi-module design expose `Floor.spots` instead. A single `threading.Lock` on the lot is *coarse-grained*: correct, simple, and fine for a physical lot (a few entries per second). Say that out loud in an interview: *"coarse lock is correct; I'd shard by floor only if measurements showed contention."*
- **Build & drill:** Add `NearestEntranceAllocator` (spots carry a distance) and an EV-only spot type that only EVs may use (where does that rule live — `Spot.fits` or the allocator? argue it). Add a display board as an Observer of enter/exit. Write the concurrency test: 50 threads entering 30 spots — exactly 30 succeed, 20 raise `LotFullError`, no spot double-booked. Draw the final UML.
- **Recall:** Why one lock around choose + park? Why is the allocator a strategy but the ticket a value object? Where would you put the EV-only rule?

### Day 43 — Elevator System (scheduling strategies, a real-time state machine, and simulation)

- **Concept & why it matters:** New forces: **time-driven simulation** (the system evolves in ticks, not just in response to calls), a **controller/dispatcher** coordinating several state machines, and **scheduling algorithms** as strategies (nearest car, same-direction/LOOK, destination dispatch) whose quality you can *measure*. Requests come in two kinds — hall calls (floor + direction) and car calls (destination inside the car) — and the design must keep them distinct.
- **Real-world case study:** Otis, KONE, and Schindler sell *dispatch algorithms* as premium software: "destination dispatch" (you type your floor in the lobby and are assigned a car) cuts average wait times by grouping passengers with similar destinations. Classic elevator control is literally the same algorithm as disk head scheduling (SCAN/LOOK) — the operating-systems textbooks borrowed the elevator's name. The lesson: when the *policy* is the product, it must be a strategy with a metric.
- **Design problem (method first):** Requirements: N elevators, F floors; hall calls with direction; car calls; each elevator serves stops in its direction before reversing (LOOK); doors open for a tick at each stop; capacity limit; emergency stop; dispatcher chooses which car takes a hall call; step-based simulation; pluggable dispatch strategy; displays update. Key fork — *where does "which stops next" live?* **(A) the controller decides every move of every car each tick** — omniscient God controller; cars are dumb; hard to test one car alone. **(B) each `Elevator` is a state machine owning its stop set and direction logic; the `Controller` only assigns hall calls via a `Dispatcher` strategy** — cars testable alone; the strategy is isolated; the controller is thin. **(C) each elevator runs its own thread (Active Object)** — realistic, but the simulation becomes nondeterministic and untestable; keep for a later exercise.
- **Thought process → decision:** Divide by *rate of change*: elevator physics (move one floor per tick, open doors, reverse when no stops ahead) is stable → inside `Elevator`; assignment policy is the product → `Dispatcher` strategy; coordination is thin → `Controller`. Deterministic `step()` so tests can assert positions after k ticks → **B**. States as an enum with rich per-state logic in `step()`; the State pattern (Day 34) is a valid alternative — with four states and one main method, the enum is smaller.
- **Code (core):**

```python
"""Day 43 — Elevator: each car is a LOOK state machine; the dispatcher is a strategy; the controller is thin."""

from __future__ import annotations

from dataclasses import dataclass, field
from enum import Enum, auto
from typing import Protocol


class Direction(Enum):
    UP = 1
    DOWN = -1


class DoorState(Enum):
    CLOSED = auto()
    OPEN = auto()


@dataclass(frozen=True)
class HallCall:
    floor: int
    direction: Direction


class Elevator:
    """Serves stops in the current direction (LOOK), reverses when none remain ahead."""

    def __init__(self, elevator_id: int, floors: int, capacity: int = 8) -> None:
        self.id = elevator_id
        self._floors = floors
        self.capacity = capacity
        self.current_floor = 0
        self.direction: Direction | None = None
        self.doors = DoorState.CLOSED
        self._stops: set[int] = set()
        self.passengers = 0
        self.emergency = False
        self.log: list[str] = []

    @property
    def is_idle(self) -> bool:
        return self.direction is None and not self._stops

    @property
    def stops(self) -> frozenset[int]:
        return frozenset(self._stops)

    def add_stop(self, floor: int) -> None:
        if not 0 <= floor < self._floors:
            raise ValueError(f"floor {floor} out of range")
        if floor == self.current_floor and self.doors is DoorState.CLOSED:
            self.doors = DoorState.OPEN                  # already here: just open
            return
        self._stops.add(floor)
        if self.direction is None:
            self.direction = Direction.UP if floor > self.current_floor else Direction.DOWN

    def stop_emergency(self) -> None:
        self.emergency = True
        self._stops.clear()
        self.direction = None
        self.log.append(f"E{self.id}: EMERGENCY STOP at {self.current_floor}")

    def step(self) -> None:
        """One tick: close doors, or move one floor, or open at a stop, or reverse, or idle."""
        if self.emergency:
            return
        if self.doors is DoorState.OPEN:                  # doors stay open exactly one tick
            self.doors = DoorState.CLOSED
            return
        if self.current_floor in self._stops:
            self._stops.discard(self.current_floor)
            self.doors = DoorState.OPEN
            self.log.append(f"E{self.id}: open at {self.current_floor}")
            if not self._stops:
                self.direction = None
            return
        if self.direction is None:
            return
        ahead = [f for f in self._stops if (f - self.current_floor) * self.direction.value > 0]
        if not ahead:                                     # LOOK: nothing ahead → reverse toward remaining stops
            self.direction = Direction.DOWN if self.direction is Direction.UP else Direction.UP
            return
        self.current_floor += self.direction.value

    def cost_to_serve(self, call: HallCall) -> int:
        """Estimated ticks to reach the call; used by dispatchers. Infinity if moving away."""
        distance = abs(call.floor - self.current_floor)
        if self.is_idle:
            return distance
        heading_toward = (call.floor - self.current_floor) * (self.direction.value if self.direction else 0) >= 0
        if heading_toward and self.direction is call.direction:
            return distance + len(self._stops)            # stops along the way cost a tick each
        return distance + 2 * self._floors                # must finish current sweep first


class Dispatcher(Protocol):
    def assign(self, elevators: list[Elevator], call: HallCall) -> Elevator: ...


class LeastCostDispatcher:
    def assign(self, elevators: list[Elevator], call: HallCall) -> Elevator:
        candidates = [e for e in elevators if not e.emergency]
        if not candidates:
            raise RuntimeError("no elevator in service")
        return min(candidates, key=lambda e: (e.cost_to_serve(call), e.passengers, e.id))


class Controller:
    def __init__(self, elevators: list[Elevator], dispatcher: Dispatcher) -> None:
        self._elevators = elevators
        self._dispatcher = dispatcher
        self.tick = 0

    def hall_call(self, floor: int, direction: Direction) -> Elevator:
        elevator = self._dispatcher.assign(self._elevators, HallCall(floor, direction))
        elevator.add_stop(floor)
        return elevator

    def car_call(self, elevator_id: int, floor: int) -> None:
        self._elevators[elevator_id].add_stop(floor)

    def step(self, ticks: int = 1) -> None:
        for _ in range(ticks):
            for elevator in self._elevators:
                elevator.step()
            self.tick += 1


if __name__ == "__main__":
    controller = Controller([Elevator(0, floors=10), Elevator(1, floors=10)], LeastCostDispatcher())
    controller._elevators[1].current_floor = 8
    car = controller.hall_call(7, Direction.DOWN)      # elevator 1 (at 8, idle) is cheaper
    controller.car_call(car.id, 2)
    controller.hall_call(3, Direction.UP)              # elevator 0 (idle at 0) takes it
    controller.step(12)
    for e in controller._elevators:
        print(e.id, e.current_floor, e.doors.name, sorted(e.stops), *e.log, sep=" | ")
```

- **Python internals:** A tick-based `step()` is a *cooperative* design: everything advances under one call, so tests are deterministic and there are no locks — the same reason game engines and `asyncio` use a loop. Multiplying `(f - current) * direction.value` to test "ahead of me" is the tiny trick that keeps the LOOK logic branch-free; `Enum` values chosen as `±1` make it possible — representation choices that shrink code (Day 25's `IntEnum` again). `min(key=tuple)` implements a *lexicographic tie-break* (cost, then load, then id) — deterministic dispatch is testable dispatch.
- **Build & drill:** Implement `SameDirectionFirstDispatcher` and a simple *destination-dispatch* variant; write a simulator that feeds 200 random calls and reports average wait per strategy — *measure*, then pick. Add capacity: a car at capacity skips hall calls (where? `cost_to_serve` returns infinity). Add an `Observer` for floor displays. Convert `Elevator` to State classes (Day 34) and compare line counts and clarity honestly.
- **Recall:** Why does the elevator own its stops but not its assignment? What is LOOK and how did you encode "ahead"? Why deterministic ticks instead of threads?

### Day 44 — Board games: Tic-Tac-Toe → Chess (polymorphic rules, move validation, undo, extensibility)

- **Concept & why it matters:** Games are LLD's purest exercise in **polymorphic rules** — each piece answers "where can I move?" differently — plus **immutable positions**, **a board that validates**, **Command-based moves with undo** (needed for both "take back" and for *checking legality by simulation*), and a **game state machine** (in progress / check / checkmate / stalemate / draw). New force: designing so that *rules extend* (new piece, new board size, variants like Chess960) without touching the game loop. Tic-Tac-Toe is the warm-up (n×n board, O(n) win check by maintaining row/column/diagonal counters — the classic interview follow-up).
- **Real-world case study:** **Stockfish and python-chess** separate *board representation*, *move generation*, *legality (check detection)*, and *game state* into distinct layers; python-chess in particular is a well-designed Python codebase worth reading: `Board.push(move)`/`pop()` is Command/Memento (a move stack), `Board.legal_moves` is a lazy generator, and piece movement is table-driven. Game studios use the same layering so rule variants ship as data.
- **Design problem (method first):** Chess requirements: 8×8 board; six piece kinds with movement rules; turns; capture; move must not leave your own king in check; detect check, checkmate, stalemate; undo; (castling, en passant, promotion as extensions). Key fork — *where does legality live?* **(A) one `Game.is_legal(move)` with `if piece.kind == …`** — a 200-line switch; adding a piece edits it. **(B) each `Piece.candidate_moves(board, from)` yields pseudo-legal targets (movement pattern only); `Game` filters by "does this leave my king in check?" by *applying, testing, undoing*** — rules per piece are polymorphic and small; check logic written once; simulation reuses the same `apply/undo` as the user-facing undo. **(C) precomputed attack tables/bitboards** — what engines do for speed; overkill for LLD, mention it. Second fork — *board mutability*: mutable board with an undo stack (efficient) vs immutable board copies per move (simple, memory-heavy; fine for Tic-Tac-Toe).
- **Thought process → decision:** Piece kinds vary; the check rule is universal → **B**. Sliding pieces (rook, bishop, queen) share a "walk in directions until blocked" helper — composition of a `directions` tuple, not a class hierarchy of `SlidingPiece` mixins (simpler). `apply` returns an *undo record* (captured piece, previous squares) — Memento in miniature — so legality checks are `apply → in_check? → undo`. For Tic-Tac-Toe, use immutable boards; for Chess, the mutable board + undo stack.
- **Code (core — pawns simplified to forward moves and diagonal captures; castling/en passant/promotion are the drill):**

```python
"""Day 44 — Chess core: polymorphic pseudo-legal moves; legality by apply/test/undo."""

from __future__ import annotations

from abc import ABC, abstractmethod
from collections.abc import Iterator
from dataclasses import dataclass
from enum import Enum


class Color(Enum):
    WHITE = "w"
    BLACK = "b"

    @property
    def other(self) -> Color:
        return Color.BLACK if self is Color.WHITE else Color.WHITE


@dataclass(frozen=True, slots=True)
class Square:
    file: int   # 0..7 (a..h)
    rank: int   # 0..7 (1..8)

    def offset(self, df: int, dr: int) -> Square | None:
        f, r = self.file + df, self.rank + dr
        return Square(f, r) if 0 <= f < 8 and 0 <= r < 8 else None

    def __str__(self) -> str:
        return f"{'abcdefgh'[self.file]}{self.rank + 1}"


class Piece(ABC):
    symbol = "?"

    def __init__(self, color: Color) -> None:
        self.color = color

    @abstractmethod
    def candidate_moves(self, board: Board, at: Square) -> Iterator[Square]:
        """Pseudo-legal targets: movement pattern + blocking + capture rules, ignoring check."""

    def _slide(self, board: Board, at: Square, directions: tuple[tuple[int, int], ...]) -> Iterator[Square]:
        for df, dr in directions:
            square = at.offset(df, dr)
            while square is not None:
                occupant = board.piece_at(square)
                if occupant is None:
                    yield square
                else:
                    if occupant.color is not self.color:
                        yield square                     # capture, then stop
                    break
                square = square.offset(df, dr)

    def _jump(self, board: Board, at: Square, offsets: tuple[tuple[int, int], ...]) -> Iterator[Square]:
        for df, dr in offsets:
            square = at.offset(df, dr)
            if square is not None:
                occupant = board.piece_at(square)
                if occupant is None or occupant.color is not self.color:
                    yield square

    def __repr__(self) -> str:
        return self.symbol.upper() if self.color is Color.WHITE else self.symbol


ORTHOGONAL = ((1, 0), (-1, 0), (0, 1), (0, -1))
DIAGONAL = ((1, 1), (1, -1), (-1, 1), (-1, -1))


class Rook(Piece):
    symbol = "r"
    def candidate_moves(self, board: Board, at: Square) -> Iterator[Square]:
        return self._slide(board, at, ORTHOGONAL)

class Bishop(Piece):
    symbol = "b"
    def candidate_moves(self, board: Board, at: Square) -> Iterator[Square]:
        return self._slide(board, at, DIAGONAL)

class Queen(Piece):
    symbol = "q"
    def candidate_moves(self, board: Board, at: Square) -> Iterator[Square]:
        return self._slide(board, at, ORTHOGONAL + DIAGONAL)

class King(Piece):
    symbol = "k"
    def candidate_moves(self, board: Board, at: Square) -> Iterator[Square]:
        return self._jump(board, at, ORTHOGONAL + DIAGONAL)

class Knight(Piece):
    symbol = "n"
    def candidate_moves(self, board: Board, at: Square) -> Iterator[Square]:
        return self._jump(board, at, ((1, 2), (2, 1), (-1, 2), (-2, 1), (1, -2), (2, -1), (-1, -2), (-2, -1)))

class Pawn(Piece):
    symbol = "p"
    def candidate_moves(self, board: Board, at: Square) -> Iterator[Square]:
        forward = 1 if self.color is Color.WHITE else -1
        one = at.offset(0, forward)
        if one is not None and board.piece_at(one) is None:
            yield one
            start_rank = 1 if self.color is Color.WHITE else 6
            two = at.offset(0, 2 * forward)
            if at.rank == start_rank and two is not None and board.piece_at(two) is None:
                yield two
        for df in (-1, 1):
            diag = at.offset(df, forward)
            if diag is not None and (target := board.piece_at(diag)) is not None and target.color is not self.color:
                yield diag


@dataclass(frozen=True)
class Move:
    src: Square
    dst: Square


@dataclass(frozen=True)
class UndoRecord:                                        # Memento for one move
    move: Move
    captured: Piece | None


class Board:
    def __init__(self) -> None:
        self._grid: dict[Square, Piece] = {}

    def place(self, piece: Piece, at: Square) -> None:
        self._grid[at] = piece

    def piece_at(self, square: Square) -> Piece | None:
        return self._grid.get(square)

    def pieces(self, color: Color) -> Iterator[tuple[Square, Piece]]:
        return ((sq, p) for sq, p in list(self._grid.items()) if p.color is color)

    def apply(self, move: Move) -> UndoRecord:
        piece = self._grid.pop(move.src)
        captured = self._grid.get(move.dst)
        self._grid[move.dst] = piece
        return UndoRecord(move, captured)

    def undo(self, record: UndoRecord) -> None:
        piece = self._grid.pop(record.move.dst)
        self._grid[record.move.src] = piece
        if record.captured is not None:
            self._grid[record.move.dst] = record.captured

    def king_square(self, color: Color) -> Square:
        return next(sq for sq, p in self._grid.items() if isinstance(p, King) and p.color is color)

    def is_attacked(self, square: Square, by: Color) -> bool:
        return any(square in p.candidate_moves(self, sq) for sq, p in self.pieces(by))


class IllegalMoveError(Exception): ...


class Game:
    def __init__(self, board: Board) -> None:
        self.board = board
        self.turn = Color.WHITE
        self._history: list[UndoRecord] = []

    def in_check(self, color: Color) -> bool:
        return self.board.is_attacked(self.board.king_square(color), by=color.other)

    def legal_moves(self, color: Color) -> Iterator[Move]:
        for src, piece in self.board.pieces(color):
            for dst in list(piece.candidate_moves(self.board, src)):
                move = Move(src, dst)
                record = self.board.apply(move)              # simulate ...
                leaves_king_safe = not self.in_check(color)  # ... test ...
                self.board.undo(record)                      # ... undo BEFORE yielding: the caller never
                if leaves_king_safe:                         #     observes the board in a simulated state
                    yield move

    def make_move(self, move: Move) -> None:
        piece = self.board.piece_at(move.src)
        if piece is None or piece.color is not self.turn:
            raise IllegalMoveError(f"no {self.turn.name.lower()} piece on {move.src}")
        if move not in set(self.legal_moves(self.turn)):
            raise IllegalMoveError(f"{move.src}->{move.dst} is not legal")
        self._history.append(self.board.apply(move))
        self.turn = self.turn.other

    def undo(self) -> None:
        if not self._history:
            raise IndexError("no moves to undo")
        self.board.undo(self._history.pop())
        self.turn = self.turn.other

    def status(self) -> str:
        has_moves = any(True for _ in self.legal_moves(self.turn))
        if has_moves:
            return "check" if self.in_check(self.turn) else "in progress"
        return "checkmate" if self.in_check(self.turn) else "stalemate"
```

- **Python internals:** `legal_moves` is a generator that *mutates and restores* the board around each candidate — and it undoes *before* `yield`, deliberately. A `try/finally` around the `yield` would leave the board in the simulated state while the consumer holds the value, restoring it only when the generator is closed or garbage-collected (`GeneratorExit` raised at the `yield`); that happens promptly in CPython thanks to reference counting, but it is a trap on other runtimes and a confusing invariant to hand a caller. Iterating `list(self._grid.items())` snapshots before mutation — modifying a dict while iterating it raises `RuntimeError`. Frozen `slots` `Square` objects as dict keys give O(1) board lookup with tiny memory; `dict` beats an 8×8 list for sparse boards and makes "iterate pieces" cheap. `any(square in p.candidate_moves(...))` short-circuits on the first attacker — the generator is never fully materialized.
- **Build & drill:** Implement Tic-Tac-Toe with an immutable `Board` and O(1) win detection via counters; generalize to n×n and k-in-a-row. Add castling, en passant, and promotion to chess — each is a test of whether `Move` needs to grow (it does: a `kind` field or subclasses — decide and defend). Add Snake & Ladder with a `Dice` strategy and a `Board` of jumps as data. Write the `legal_moves` performance note: what would you cache?
- **Recall:** Why pseudo-legal per piece and legality in `Game`? Why is `apply/undo` the same mechanism for simulation and for user undo? Mutable board + undo stack vs immutable boards — when each?

### Day 45 — LRU / LFU Cache and a generic cache with eviction strategies, TTL, and thread safety

- **Concept & why it matters:** The most-asked coding+design hybrid. Forces: **O(1) get and put**, an **eviction policy** that varies (LRU, LFU, FIFO, TTL), **thread safety** without serializing every read into a bottleneck, **statistics** (hit ratio) for tuning, and a clear **interface** (`get` returns a sentinel or raises? `put` overwrites?). Also the data-structure insight: LRU = hash map + doubly-linked list (or `OrderedDict`, which *is* that); LFU = frequency buckets each an ordered set, plus a min-frequency pointer, for O(1).
- **Real-world case study:** **Redis** exposes eviction as configuration (`allkeys-lru`, `allkeys-lfu`, `volatile-ttl`) and famously uses *approximate* LRU (sampling) because exact LRU's bookkeeping cost wasn't worth it at scale — a policy-vs-cost tradeoff you should be able to articulate. **CPython's `functools.lru_cache`** is a C implementation of exactly the dict + circular doubly-linked list design. **Caffeine** (Java) uses a windowed TinyLFU because pure LRU is fooled by scans — knowing *why a policy fails* is senior-level.
- **Design problem (method first):** Requirements: capacity-bounded key→value cache; `get`/`put` O(1); pluggable eviction (LRU, LFU); optional per-entry TTL; thread-safe; hit/miss stats; `put` on existing key updates value and recency. Key fork — *how is the policy separated from storage?* **(A) `LRUCache` and `LFUCache` as two independent classes** — each complete, duplicated TTL/stats/locking code, adding FIFO = third copy. **(B) one `Cache` class holding entries + a `EvictionPolicy` object notified on `access`/`insert`/`remove` and asked `victim()`** — storage, TTL, stats, locking written once; policies are small; each policy maintains only ordering metadata. **(C) inherit: `BaseCache` with abstract `_on_access/_on_insert/_victim` hooks (Template Method)** — works, but composition (B) lets you swap policy at construction and test policies alone. Tradeoffs: A is the interview-coding answer, B the design answer, C acceptable.
- **Thought process → decision:** Policies vary, everything else is shared → **B**. One `RLock` around each public operation — cache ops are microseconds, so a single lock is not the bottleneck people fear (say so; measure if challenged). TTL checked lazily on `get` (expired entry → treat as miss and remove) plus optional sweeps — avoids a timer thread. `get` returns a `default` (like `dict.get`) rather than raising — caches miss constantly; exceptions for misses would be exceptions as control flow (Day 26).
- **Code (core):**

```python
"""Day 45 — generic cache with pluggable O(1) eviction policies, lazy TTL, and one lock."""

from __future__ import annotations

import threading
import time
from collections import OrderedDict, defaultdict
from collections.abc import Callable, Hashable
from dataclasses import dataclass
from typing import Generic, Protocol, TypeVar

K = TypeVar("K", bound=Hashable)
V = TypeVar("V")


class EvictionPolicy(Protocol[K]):
    def on_insert(self, key: K) -> None: ...
    def on_access(self, key: K) -> None: ...
    def on_remove(self, key: K) -> None: ...
    def victim(self) -> K: ...


class LRUPolicy(Generic[K]):
    """OrderedDict is a hash map + doubly linked list: move_to_end and popitem(last=False) are O(1)."""
    def __init__(self) -> None:
        self._order: OrderedDict[K, None] = OrderedDict()
    def on_insert(self, key: K) -> None:
        self._order[key] = None
    def on_access(self, key: K) -> None:
        self._order.move_to_end(key)
    def on_remove(self, key: K) -> None:
        self._order.pop(key, None)
    def victim(self) -> K:
        return next(iter(self._order))


class LFUPolicy(Generic[K]):
    """Frequency buckets, each an insertion-ordered set; ties broken by LRU within the bucket."""
    def __init__(self) -> None:
        self._freq: dict[K, int] = {}
        self._buckets: defaultdict[int, OrderedDict[K, None]] = defaultdict(OrderedDict)
        self._min_freq = 0
    def on_insert(self, key: K) -> None:
        self._freq[key] = 1
        self._buckets[1][key] = None
        self._min_freq = 1
    def on_access(self, key: K) -> None:
        freq = self._freq[key]
        del self._buckets[freq][key]
        if not self._buckets[freq]:
            del self._buckets[freq]
            if self._min_freq == freq:
                self._min_freq += 1
        self._freq[key] = freq + 1
        self._buckets[freq + 1][key] = None
    def on_remove(self, key: K) -> None:
        freq = self._freq.pop(key, None)
        if freq is not None:
            self._buckets[freq].pop(key, None)
            if not self._buckets[freq]:
                del self._buckets[freq]
    def victim(self) -> K:
        if not self._freq:
            raise LookupError("nothing to evict")
        while self._min_freq not in self._buckets:          # defensive: never index a bucket that was emptied
            self._min_freq += 1
        return next(iter(self._buckets[self._min_freq]))


@dataclass
class _Entry(Generic[V]):
    value: V
    expires_at: float | None


@dataclass(frozen=True)
class CacheStats:
    hits: int
    misses: int
    evictions: int

    @property
    def hit_ratio(self) -> float:
        total = self.hits + self.misses
        return self.hits / total if total else 0.0


class Cache(Generic[K, V]):
    def __init__(self, capacity: int, policy: EvictionPolicy[K], clock: Callable[[], float] = time.monotonic) -> None:
        if capacity <= 0:
            raise ValueError("capacity must be positive")
        self._capacity, self._policy, self._clock = capacity, policy, clock
        self._entries: dict[K, _Entry[V]] = {}
        self._lock = threading.RLock()
        self._hits = self._misses = self._evictions = 0

    def get(self, key: K, default: V | None = None) -> V | None:
        with self._lock:
            entry = self._entries.get(key)
            if entry is None or (entry.expires_at is not None and entry.expires_at <= self._clock()):
                if entry is not None:
                    self._remove(key)                       # lazily expire
                self._misses += 1
                return default
            self._policy.on_access(key)
            self._hits += 1
            return entry.value

    def put(self, key: K, value: V, ttl_seconds: float | None = None) -> None:
        if ttl_seconds is not None and ttl_seconds <= 0:
            raise ValueError("ttl must be positive")
        expires_at = self._clock() + ttl_seconds if ttl_seconds is not None else None
        with self._lock:
            if key in self._entries:
                self._entries[key] = _Entry(value, expires_at)
                self._policy.on_access(key)
                return
            if len(self._entries) >= self._capacity:
                self._remove(self._policy.victim())
                self._evictions += 1
            self._entries[key] = _Entry(value, expires_at)
            self._policy.on_insert(key)

    def _remove(self, key: K) -> None:
        self._entries.pop(key, None)
        self._policy.on_remove(key)

    def __len__(self) -> int:
        with self._lock:
            return len(self._entries)

    @property
    def stats(self) -> CacheStats:
        with self._lock:
            return CacheStats(self._hits, self._misses, self._evictions)


if __name__ == "__main__":
    lru: Cache[str, int] = Cache(2, LRUPolicy())
    lru.put("a", 1); lru.put("b", 2); lru.get("a"); lru.put("c", 3)
    print(sorted(k for k in ("a", "b", "c") if lru.get(k) is not None), lru.stats)   # a, c survive
    lfu: Cache[str, int] = Cache(2, LFUPolicy())
    lfu.put("a", 1); lfu.put("b", 2); lfu.get("a"); lfu.get("a"); lfu.put("c", 3)
    print(sorted(k for k in ("a", "b", "c") if lfu.get(k) is not None))                # a (freq 3), c
```

- **Python internals:** `OrderedDict` is implemented in C as a dict plus a doubly-linked list of keys; `move_to_end` and `popitem(last=False)` are O(1), so it *is* the classic LRU structure — implement the raw dict + `Node` linked list once by hand (drill) so you can do it on a whiteboard, then use `OrderedDict` in real code. Plain `dict` preserves insertion order but has no `move_to_end` (you'd delete and reinsert — also O(1), and a legitimate FIFO/LRU trick). `RLock` (not `Lock`) because `put` calls `_remove` which may later call locked helpers — re-entrancy avoids self-deadlock. `time.monotonic` for TTLs; the injected clock makes expiry testable without sleeping. The generic `Protocol[K]` gives type-checked policies.
- **Build & drill:** Implement the hand-rolled doubly-linked-list LRU (interview form) and `FIFOPolicy` (5 lines). Add a `ReadWriteLock` (Day 39) and benchmark vs `RLock` at 95% reads — report whether it helped. Add a `sweep()` for expired entries and `get_or_compute(key, fn)` with **single-flight** (only one thread computes a missing key — the cache-stampede fix). Write the LRU-fooled-by-scan test and explain why LFU/TinyLFU resist it.
- **Recall:** Why is `OrderedDict` an LRU? How does LFU stay O(1)? Why lazy TTL? Why does `get` return a default instead of raising?

### Day 46 — Rate Limiter (token bucket, leaky bucket, sliding windows; per-client, thread-safe)

- **Concept & why it matters:** Rate limiting protects a system from overload and abuse and is a favourite because the *algorithms* have distinct, measurable tradeoffs: **fixed window** (simple; double-burst at boundaries), **sliding window log** (exact; O(requests) memory), **sliding window counter** (approximate; O(1)), **token bucket** (allows bursts up to capacity, smooth average — the industry default), **leaky bucket** (smooths output to a constant rate — a queue). New forces: **time as an input** (inject the clock), **per-key state** with cleanup (millions of clients), the **`allow()` decision as a pure function of state + time**, and the honest limitation that an in-process limiter is *per instance* — distributed limiting needs a shared store (say it, don't design it here).
- **Real-world case study:** **Stripe's rate limiters** blog post describes four limiters layered (request rate, concurrent requests, fleet-usage load shedder, worker-utilization shedder), all token-bucket based on Redis, and explains *why* bursts must be allowed (retry storms) but bounded. **GitHub's API** uses fixed hourly windows exposed in headers (simple for clients to reason about). **Nginx's `limit_req`** is a leaky bucket. The choice of algorithm is a *product decision about burst behaviour*, and you should be able to say which one a client would prefer and why.
- **Design problem (method first):** Requirements: limit requests per client id (e.g. 100/min), configurable algorithm, `allow(client_id) -> bool` plus `retry_after` hint, thread-safe, bounded memory (idle clients expire), injectable clock, testable without sleeping. Key fork — *algorithm as strategy or as separate classes?* **(A) `TokenBucketLimiter`, `SlidingWindowLimiter`, … each with its own dict-of-clients and lock** — duplication of per-client bookkeeping and cleanup. **(B) a `RateLimiter` that owns the per-client map, locking, and idle cleanup, delegating to a `LimitAlgorithm` that creates and updates per-client state** — algorithms are pure and tiny; the shell is written once. **(C) one limiter class with an `algorithm` enum and branches** — the smell.
- **Thought process → decision:** **B**. Per-client state objects are small dataclasses owned by the algorithm; the limiter shell stores them in a dict guarded by one lock (or sharded locks by hash if measured necessary), and evicts clients idle longer than the window. `allow()` returns a small result object (`allowed`, `retry_after`) — richer than `bool`, still cheap, and what an HTTP layer needs for `429` + `Retry-After`.
- **Code (core):**

```python
"""Day 46 — rate limiter shell + pluggable algorithms (token bucket, sliding window log/counter)."""

from __future__ import annotations

import threading
import time
from collections import deque
from collections.abc import Callable
from dataclasses import dataclass, field
from typing import Generic, Protocol, TypeVar

S = TypeVar("S")


@dataclass(frozen=True)
class Verdict:
    allowed: bool
    retry_after: float = 0.0          # seconds until the next request could pass (0 when allowed)


class LimitAlgorithm(Protocol[S]):
    def new_state(self, now: float) -> S: ...
    def check(self, state: S, now: float) -> Verdict: ...
    def idle_ttl(self) -> float: ...


@dataclass
class BucketState:
    tokens: float
    updated_at: float


class TokenBucket:
    """Allows bursts up to `capacity`; refills at `rate` tokens/second; O(1) memory per client."""

    def __init__(self, rate: float, capacity: int) -> None:
        if rate <= 0 or capacity <= 0:
            raise ValueError("rate and capacity must be positive")
        self._rate, self._capacity = rate, capacity

    def new_state(self, now: float) -> BucketState:
        return BucketState(tokens=float(self._capacity), updated_at=now)

    def check(self, state: BucketState, now: float) -> Verdict:
        elapsed = max(0.0, now - state.updated_at)
        state.tokens = min(self._capacity, state.tokens + elapsed * self._rate)     # lazy refill
        state.updated_at = now
        if state.tokens >= 1:
            state.tokens -= 1
            return Verdict(True)
        return Verdict(False, retry_after=(1 - state.tokens) / self._rate)

    def idle_ttl(self) -> float:
        return self._capacity / self._rate


@dataclass
class WindowLogState:
    timestamps: deque[float] = field(default_factory=deque)


class SlidingWindowLog:
    """Exact: at most `limit` requests in any trailing `window` seconds. O(limit) memory per client."""

    def __init__(self, limit: int, window_seconds: float) -> None:
        self._limit, self._window = limit, window_seconds

    def new_state(self, now: float) -> WindowLogState:
        return WindowLogState()

    def check(self, state: WindowLogState, now: float) -> Verdict:
        cutoff = now - self._window
        while state.timestamps and state.timestamps[0] <= cutoff:
            state.timestamps.popleft()
        if len(state.timestamps) < self._limit:
            state.timestamps.append(now)
            return Verdict(True)
        return Verdict(False, retry_after=state.timestamps[0] + self._window - now)

    def idle_ttl(self) -> float:
        return self._window


@dataclass
class WindowCounterState:
    window_start: float
    current: int = 0
    previous: int = 0


class SlidingWindowCounter:
    """Approximate: weights the previous window by overlap. O(1) memory, no boundary double-burst."""

    def __init__(self, limit: int, window_seconds: float) -> None:
        self._limit, self._window = limit, window_seconds

    def new_state(self, now: float) -> WindowCounterState:
        return WindowCounterState(window_start=now - (now % self._window))

    def check(self, state: WindowCounterState, now: float) -> Verdict:
        start = now - (now % self._window)
        if start != state.window_start:
            windows_passed = round((start - state.window_start) / self._window)
            state.previous = state.current if windows_passed == 1 else 0
            state.current = 0
            state.window_start = start
        overlap = 1 - (now - start) / self._window
        estimate = state.previous * overlap + state.current
        if estimate + 1 <= self._limit:
            state.current += 1
            return Verdict(True)
        return Verdict(False, retry_after=self._window - (now - start))

    def idle_ttl(self) -> float:
        return 2 * self._window


class RateLimiter(Generic[S]):
    """Owns per-client state, locking, and idle cleanup. Algorithm-agnostic."""

    def __init__(self, algorithm: LimitAlgorithm[S], clock: Callable[[], float] = time.monotonic) -> None:
        self._algorithm, self._clock = algorithm, clock
        self._states: dict[str, tuple[S, float]] = {}          # client -> (state, last_seen)
        self._lock = threading.Lock()
        self._last_sweep = clock()

    def allow(self, client_id: str) -> Verdict:
        now = self._clock()
        with self._lock:
            entry = self._states.get(client_id)
            state = entry[0] if entry else self._algorithm.new_state(now)
            verdict = self._algorithm.check(state, now)
            self._states[client_id] = (state, now)
            if now - self._last_sweep > self._algorithm.idle_ttl():
                self._sweep(now)
            return verdict

    def _sweep(self, now: float) -> None:
        ttl = self._algorithm.idle_ttl()
        for client, (_, last_seen) in list(self._states.items()):
            if now - last_seen > ttl:
                del self._states[client]
        self._last_sweep = now


if __name__ == "__main__":
    fake_now = [1000.0]
    limiter = RateLimiter(TokenBucket(rate=1.0, capacity=3), clock=lambda: fake_now[0])
    print([limiter.allow("asha").allowed for _ in range(4)])       # [True, True, True, False]
    print(round(limiter.allow("asha").retry_after, 2))
    fake_now[0] += 2.0
    print([limiter.allow("asha").allowed for _ in range(3)])       # [True, True, False]
```

- **Python internals:** The token bucket's *lazy refill* (compute tokens from elapsed time on each call) means no timer thread and O(1) work — the standard trick; it relies on a monotonic clock (`time.monotonic`) that never goes backwards. `deque.popleft()` is O(1) (a doubly-linked block list) where `list.pop(0)` is O(n) — the right structure for the sliding log. Floating-point token counts drift negligibly; if exactness matters, count in integer "micro-tokens." Sweeping inside `allow` amortizes cleanup without a background thread — a design choice to say out loud (alternative: a daemon sweeper). A `Protocol[S]` generic algorithm keeps the shell type-safe over different state shapes.
- **Build & drill:** Add `FixedWindow` and a test that shows its boundary double-burst, then the same test passing under `SlidingWindowCounter`. Add a **leaky bucket** as a *queue-based smoother* (different interface: `enqueue`/`drain`), and explain why it's a different tool. Shard the lock by `hash(client_id) % 16` and measure. Write the "how would you make this distributed?" answer in eight lines (Redis `INCR` + `EXPIRE`, Lua for atomicity, clock skew, hot keys).
- **Recall:** Token bucket vs sliding window: which allows bursts and why does that matter? How does lazy refill avoid a timer? Why inject the clock? What does an in-process limiter *not* guarantee?

### Day 47 — Logging framework and a Pub/Sub message queue (Kafka-lite): pipelines, offsets, and delivery guarantees

- **Concept & why it matters:** Two "infrastructure" LLD problems that test whether you can design *a library others build on*. **Logger**: a hierarchy of named loggers with level thresholds, handlers (console/file/memory/remote), formatters, filters, propagation to parents, and thread safety — Chain of Responsibility + Strategy + Composite in one. **Message queue**: topics, an **append-only log** per topic (or partition), producers appending, **consumer groups** tracking **offsets**, **at-least-once** delivery via acknowledgement, retention. New forces: **the log as a data structure** (immutable, indexed by offset — the seed of event sourcing on Day 48), **consumer position as consumer-owned state**, and **delivery semantics as an explicit contract** (at-most-once vs at-least-once vs "exactly-once effect" via idempotent consumers).
- **Real-world case study:** **Python's `logging` module** is the reference implementation of today's logger — read `logging/__init__.py` after designing yours: `Logger.callHandlers` walks the parent chain, `Handler.handle` applies filters then `emit`, and a module-level `RLock` protects the handler list. **Apache Kafka** is the reference for the queue: the insight that made it scale was *not* tracking per-message state on the broker — consumers keep their own offset into an immutable log, so the broker does sequential appends and reads. **RabbitMQ** took the opposite route (broker-side acks per message), which is why the two behave so differently under load.
- **Design problem (method first — the queue; the logger is the drill):** Requirements: create topics; publish messages (ordered within a topic); multiple consumer groups each receive every message once *per group*; a consumer polls a batch and acks; un-acked messages are redelivered after a visibility timeout; retention by count; thread-safe. Key fork — *where does "delivered/acked" state live?* **(A) per-message state on the broker (`pending`/`acked` flags per group)** — simple mental model; O(messages × groups) state; redelivery needs scanning. **(B) per-group *offset* into an append-only log, plus a small in-flight map for redelivery** — O(1) state per group; the log is shared and immutable; redelivery is "reset offset to the oldest un-acked." **(C) push delivery to registered callbacks (Observer)** — simplest API, but the broker then owns retry/backpressure for every consumer's speed; polling lets consumers control pace.
- **Thought process → decision:** **B** with polling: the log is the truth, offsets are consumer-owned, and *acks advance the committed offset only when contiguous* (like Kafka's commit semantics, simplified). At-least-once is the guarantee: a crash between processing and ack causes redelivery — document it and make consumers idempotent (Day 51 shows idempotency keys). Retention drops the head of the log; a group whose offset is behind the retained head has *lost messages* — surface that as an error, never silently skip.
- **Code (core):**

```python
"""Day 47 — a Kafka-lite broker: append-only topic logs, consumer-group offsets, at-least-once acks."""

from __future__ import annotations

import threading
import time
from collections import deque
from collections.abc import Callable
from dataclasses import dataclass, field
from typing import Any


class BrokerError(Exception): ...
class UnknownTopicError(BrokerError): ...
class MessagesLostError(BrokerError): ...


@dataclass(frozen=True)
class Message:
    offset: int
    key: str | None
    payload: Any
    produced_at: float


class TopicLog:
    """Append-only, offset-indexed, bounded by retention. Immutable messages."""

    def __init__(self, retention: int) -> None:
        self._messages: deque[Message] = deque()
        self._next_offset = 0
        self._retention = retention

    def append(self, key: str | None, payload: Any, now: float) -> Message:
        message = Message(self._next_offset, key, payload, now)
        self._messages.append(message)
        self._next_offset += 1
        while len(self._messages) > self._retention:
            self._messages.popleft()
        return message

    @property
    def head_offset(self) -> int:                      # oldest retained
        return self._messages[0].offset if self._messages else self._next_offset

    @property
    def end_offset(self) -> int:                       # next to be written
        return self._next_offset

    def read(self, from_offset: int, limit: int) -> list[Message]:
        if from_offset < self.head_offset:
            raise MessagesLostError(f"offset {from_offset} < retained head {self.head_offset}")
        start = from_offset - self.head_offset
        return [self._messages[i] for i in range(start, min(start + limit, len(self._messages)))]


@dataclass
class GroupState:
    committed: int                                     # everything below this offset is acked
    next_to_deliver: int                               # high-water mark: first offset never delivered
    in_flight: dict[int, float] = field(default_factory=dict)   # offset -> deadline for ack
    acked: set[int] = field(default_factory=set)       # acked above `committed` (gaps still open)


class Broker:
    def __init__(self, *, retention: int = 10_000, visibility_timeout: float = 30.0,
                 clock: Callable[[], float] = time.monotonic) -> None:
        self._topics: dict[str, TopicLog] = {}
        self._groups: dict[tuple[str, str], GroupState] = {}
        self._retention, self._visibility, self._clock = retention, visibility_timeout, clock
        self._lock = threading.RLock()

    def create_topic(self, name: str) -> None:
        with self._lock:
            self._topics.setdefault(name, TopicLog(self._retention))

    def publish(self, topic: str, payload: Any, key: str | None = None) -> int:
        with self._lock:
            return self._topic(topic).append(key, payload, self._clock()).offset

    def poll(self, topic: str, group: str, max_messages: int = 10) -> list[Message]:
        """Redeliver expired in-flight messages first, then never-delivered ones. At-least-once."""
        with self._lock:
            log = self._topic(topic)
            state = self._groups.setdefault(
                (topic, group), GroupState(committed=log.head_offset, next_to_deliver=log.head_offset)
            )
            now = self._clock()
            batch: list[Message] = []
            for offset, deadline in sorted(state.in_flight.items()):
                if deadline <= now and len(batch) < max_messages:
                    batch.extend(log.read(offset, 1))
                    state.in_flight[offset] = now + self._visibility
            for message in log.read(state.next_to_deliver, max_messages - len(batch)):
                state.in_flight[message.offset] = now + self._visibility
                state.next_to_deliver = message.offset + 1
                batch.append(message)
            return batch

    def ack(self, topic: str, group: str, offset: int) -> None:
        with self._lock:
            state = self._groups[(topic, group)]
            if state.in_flight.pop(offset, None) is None:
                raise BrokerError(f"offset {offset} is not in flight for group {group!r}")
            state.acked.add(offset)
            while state.committed in state.acked:            # advance only while contiguous
                state.acked.remove(state.committed)
                state.committed += 1

    def lag(self, topic: str, group: str) -> int:
        with self._lock:
            return self._topic(topic).end_offset - self._groups[(topic, group)].committed

    def _topic(self, name: str) -> TopicLog:
        try:
            return self._topics[name]
        except KeyError:
            raise UnknownTopicError(name) from None


if __name__ == "__main__":
    now = [0.0]
    broker = Broker(visibility_timeout=5.0, clock=lambda: now[0])
    broker.create_topic("orders")
    for i in range(3):
        broker.publish("orders", {"order": i}, key=str(i))
    batch = broker.poll("orders", "billing", max_messages=2)
    print([m.offset for m in batch])                     # [0, 1]
    broker.ack("orders", "billing", 1)                   # ack out of order: committed stays at 0
    now[0] += 6.0                                        # offset 0 times out → redelivered
    print([m.offset for m in broker.poll("orders", "billing")])      # [0, 2]
    broker.ack("orders", "billing", 0); broker.ack("orders", "billing", 2)
    print("lag:", broker.lag("orders", "billing"), "| analytics sees all:", [m.offset for m in broker.poll("orders", "analytics")])
```

- **Python internals:** `deque` with `popleft` gives O(1) retention trimming; indexing a deque is O(n) in the worst case (it's a linked list of blocks) — for a real broker use a list with a base offset or segment files. Frozen `Message` objects can be handed to many consumer threads without copying or locking — immutability as a concurrency tool (Day 27). `RLock` because `poll` and `ack` share helpers. The "committed advances only while contiguous" loop is the essence of offset-based acks: out-of-order acks are remembered but not committed until the gap closes — draw it on paper; it's a favourite follow-up.
- **Build & drill:** Design and code the **logging framework**: `Logger` hierarchy by dotted name (`app.payments` → `app` → root), `Level` enum, `Handler` ABC with `ConsoleHandler`/`FileHandler`/`MemoryHandler`, `Formatter` strategy, `Filter` chain, propagation, and a lock on the handler list; then compare with `logging`'s source. For the broker: add *partitions by key* (same key → same partition → ordered), consumer-group rebalancing across N consumers, and a `dead_letter` topic after K redeliveries. Write the at-least-once test: crash (no ack) → redelivery → idempotent consumer processes once.
- **Recall:** Why do offsets beat per-message flags? What does at-least-once mean for consumers? Why must committed advance only contiguously? Why poll instead of push here?

### Day 48 — Splitwise and a Money Ledger (split strategies, debt simplification, double-entry, event sourcing)

- **Concept & why it matters:** Money systems test *exactness* (Day 12), **split strategies** with validation (equal / exact / percentage — the parts must sum to the whole, to the paisa), **balance derivation** (compute from the history, don't store and drift), **debt simplification** (an algorithm: minimize transactions with a greedy two-heap method — and knowing its limits), and the **double-entry ledger** where every transaction is a set of postings whose debits equal credits — the invariant that has kept accountants honest since 1494. New force: **the ledger as an append-only source of truth from which balances are a fold** — event sourcing, which Day 47's log prepared you for.
- **Real-world case study:** **Splitwise's "simplify debts"** feature (which they document as an approximation — the exact minimum-transactions problem is NP-hard) and the classic **Square/Stripe/Modern Treasury ledgers**: every balance is derived from immutable journal entries, corrections are *new reversing entries*, never edits, and the system refuses any entry whose postings don't sum to zero. Uber's and Airbnb's payment teams have written about migrating *to* double-entry after "balance column" designs drifted by cents across millions of rows.
- **Design problem (method first):** Requirements: users, groups; add an expense paid by one user, split among participants by a strategy; per-user balances within a group; settle up; "who owes whom" simplified; every amount exact; complete audit history. Key fork — *stored balances vs derived from the ledger?* **(A) a `balances[user]` dict updated on each expense** — fast reads; any bug or crash mid-update drifts it forever; no audit trail. **(B) an append-only list of `Expense`/`Settlement` records; balances computed by folding** — always consistent with history; O(n) reads (cache if needed); corrections are new records. **(C) a true double-entry `Ledger` of `JournalEntry`s with postings that must balance; expenses and settlements are *translated* into entries; balances are account sums** — the strongest invariant; slightly more modelling; the shape used by real finance systems. Tradeoffs: A is what beginners write and what drifts; B is enough for a group app; C when money leaves the app.
- **Thought process → decision:** Go with **C** for the core (it's the transferable skill) and expose Splitwise semantics on top: each expense becomes an entry crediting the payer's receivable and debiting each participant's payable; a settlement is the reverse. Split strategies as a Protocol; each returns a list of `(user, Money)` shares and *must* sum exactly to the total — use `Money.allocate` (Day 12) for equal splits so remainders are distributed, never lost. Debt simplification as a pure function over net balances (greedy heaps), clearly labelled as a heuristic.
- **Code (core; `Money` from Day 12 assumed importable):**

```python
"""Day 48 — split strategies with exact allocation, a double-entry ledger, and debt simplification."""

from __future__ import annotations

import heapq
from collections import defaultdict
from dataclasses import dataclass, field
from datetime import datetime, timezone
from decimal import Decimal
from typing import Protocol
from uuid import UUID, uuid4

from money import Money          # Day 12's value object: exact minor units, allocate(), + and -


class LedgerError(Exception): ...
class UnbalancedEntryError(LedgerError): ...
class InvalidSplitError(LedgerError): ...


# --- split strategies: each must return shares that sum exactly to the total -------------------
class SplitStrategy(Protocol):
    def split(self, total: Money, participants: list[str]) -> dict[str, Money]: ...


class EqualSplit:
    def split(self, total: Money, participants: list[str]) -> dict[str, Money]:
        if not participants:
            raise InvalidSplitError("no participants")
        return dict(zip(participants, total.allocate(len(participants))))      # remainder paise distributed


class ExactSplit:
    def __init__(self, amounts: dict[str, Money]) -> None:
        self._amounts = dict(amounts)

    def split(self, total: Money, participants: list[str]) -> dict[str, Money]:
        if set(self._amounts) != set(participants):
            raise InvalidSplitError("exact amounts must cover exactly the participants")
        summed = sum(self._amounts.values(), Money("0", total.currency))
        if summed != total:
            raise InvalidSplitError(f"shares {summed} do not equal total {total}")
        return dict(self._amounts)


class PercentSplit:
    def __init__(self, percents: dict[str, Decimal]) -> None:
        if sum(percents.values()) != Decimal("100"):
            raise InvalidSplitError("percentages must sum to 100")
        self._percents = dict(percents)

    def split(self, total: Money, participants: list[str]) -> dict[str, Money]:
        if set(self._percents) != set(participants):
            raise InvalidSplitError("percentages must cover exactly the participants")
        shares = {u: total * (p / 100) for u, p in self._percents.items()}
        drift = total - sum(shares.values(), Money("0", total.currency))      # rounding residue
        first = participants[0]
        shares[first] = shares[first] + drift                                  # assign residue explicitly
        return shares


# --- double-entry ledger ---------------------------------------------------------------------
@dataclass(frozen=True)
class Posting:
    account: str            # e.g. "user:asha"
    amount: Money           # positive = debit (they owe more), negative = credit (they are owed)


@dataclass(frozen=True)
class JournalEntry:
    description: str
    postings: tuple[Posting, ...]
    id: UUID = field(default_factory=uuid4)
    recorded_at: datetime = field(default_factory=lambda: datetime.now(timezone.utc))

    def __post_init__(self) -> None:
        if len(self.postings) < 2:
            raise UnbalancedEntryError("an entry needs at least two postings")
        currency = self.postings[0].amount.currency
        total = sum((p.amount for p in self.postings), Money("0", currency))
        if total != Money("0", currency):
            raise UnbalancedEntryError(f"postings sum to {total}, not zero")


class Ledger:
    """Append-only. Balances are a fold over entries — never stored, never drift."""

    def __init__(self) -> None:
        self._entries: list[JournalEntry] = []

    def record(self, entry: JournalEntry) -> None:
        self._entries.append(entry)

    def balances(self, currency: str) -> dict[str, Money]:
        totals: defaultdict[str, Money] = defaultdict(lambda: Money("0", currency))
        for entry in self._entries:
            for posting in entry.postings:
                totals[posting.account] = totals[posting.account] + posting.amount
        return dict(totals)

    @property
    def entries(self) -> tuple[JournalEntry, ...]:
        return tuple(self._entries)


class Group:
    def __init__(self, name: str, members: list[str], currency: str = "INR") -> None:
        self.name, self.members, self.currency = name, list(members), currency
        self.ledger = Ledger()

    def add_expense(self, description: str, paid_by: str, total: Money, participants: list[str], strategy: SplitStrategy) -> None:
        unknown = (set(participants) | {paid_by}) - set(self.members)
        if unknown:
            raise LedgerError(f"not group members: {sorted(unknown)}")
        shares = strategy.split(total, participants)
        postings = [Posting(f"user:{u}", share) for u, share in shares.items()]       # each owes their share
        postings.append(Posting(f"user:{paid_by}", total * -1))                        # payer is owed the total
        self.ledger.record(JournalEntry(description, tuple(postings)))

    def settle(self, from_user: str, to_user: str, amount: Money) -> None:
        self.ledger.record(JournalEntry(f"{from_user} pays {to_user}",
                                        (Posting(f"user:{from_user}", amount * -1), Posting(f"user:{to_user}", amount))))

    def net_balances(self) -> dict[str, Money]:
        """Positive = owes the group; negative = is owed."""
        return {acct.removeprefix("user:"): bal for acct, bal in self.ledger.balances(self.currency).items()}


def simplify_debts(net: dict[str, Money], currency: str) -> list[tuple[str, str, Money]]:
    """Greedy: largest debtor pays largest creditor. Few transactions in practice; not guaranteed minimal."""
    zero = Money("0", currency)
    debtors = [(-b._minor, u) for u, b in net.items() if b > zero]        # heapq is a min-heap → negate
    creditors = [(b._minor, u) for u, b in net.items() if b < zero]
    heapq.heapify(debtors); heapq.heapify(creditors)
    transfers: list[tuple[str, str, Money]] = []
    while debtors and creditors:
        owed_minor, debtor = heapq.heappop(debtors)
        credit_minor, creditor = heapq.heappop(creditors)
        paid = min(-owed_minor, -credit_minor)
        transfers.append((debtor, creditor, Money._from_minor(paid, currency)))
        if -owed_minor - paid:
            heapq.heappush(debtors, (owed_minor + paid, debtor))
        if -credit_minor - paid:
            heapq.heappush(creditors, (credit_minor + paid, creditor))
    return transfers
```

- **Python internals:** `sum(iterable, start)` with a `Money` start value works because `Money.__add__` exists (Day 13) — and would fail without an explicit start because `0 + Money` has no meaning; that's why `__radd__` matters in the drill. `heapq` is a min-heap over tuples, so negating amounts gives max-heap behaviour and the user name breaks ties deterministically. `str.removeprefix` (3.9+) avoids the classic `s[len(prefix):]` off-by-one. Frozen `JournalEntry.__post_init__` enforces the balance invariant at construction — an unbalanced entry *cannot exist* (Day 26's principle in its most important application). Note `Money._minor` access from the same package — "friend" access; expose `minor_units` as a property in a real codebase.
- **Build & drill:** Add `__lt__`/`__gt__` to `Money` if missing; write tests that `PercentSplit(33.33/33.33/33.34)` and `EqualSplit` over 3 people on ₹100 both sum *exactly*. Add a *reversal* (correct a wrong expense by recording the inverse entry) and a per-group *statement* that replays history. Prove `simplify_debts` is not always minimal with a counter-example. Then rebuild Day 11's bank account on top of the `Ledger` (accounts as ledger accounts) — event-sourced balances.
- **Recall:** Why are balances derived, not stored? What invariant does `JournalEntry` enforce, and where? Why does equal split use `allocate`? What is the limitation of greedy simplification?

### Day 49 — Movie Ticket Booking (BookMyShow): seat holds with expiry, double-booking prevention, payment timeouts

- **Concept & why it matters:** The first *truly concurrent* business system: many users see the same seat map and race for the same seats. Forces: **a temporary hold** (seat locked for N minutes while paying) as a first-class state with **expiry**, **atomic multi-seat reservation** (all requested seats or none), **per-show locking granularity**, a **booking state machine** (HELD → CONFIRMED / EXPIRED / CANCELLED), **payment as an external step that can fail or time out**, and **idempotent confirmation** (the payment callback may arrive twice). The correctness bar: *no seat is ever sold twice, and no seat is stuck held forever.*
- **Real-world case study:** **Ticketmaster's Taylor Swift Eras Tour pre-sale (Nov 2022)** collapsed under 3.5 billion requests; the post-incident analysis centred on hold/queue mechanics and bot traffic — hold expiry and queueing are the system, not features. **BookMyShow** and airline seat maps all use timed holds (typically 5–10 minutes) exactly so an abandoned checkout can't block inventory. Database teams describe the same invariant with `SELECT … FOR UPDATE` or optimistic version columns; in-process, it's a lock per show.
- **Design problem (method first):** Requirements: theatres, screens, shows; seat map per show with seat types and prices; a user holds one or more seats for T minutes; hold expires automatically; confirming a hold creates a booking after payment; cancellation refunds by policy; the same seats can never be double-held or double-booked; concurrent users. Key fork — *lock granularity and hold ownership*: **(A) one global lock** — correct, serializes every show in the system; unacceptable for 10k shows. **(B) one lock per `Show`; the show owns seat states and holds; holds have a deadline checked lazily and by a sweeper** — contention scoped to one show (the natural unit — users of different shows never conflict); atomic multi-seat is a loop under the show lock. **(C) per-seat locks** — finer, but multi-seat holds need multiple locks in order, and "all or nothing" gets complicated for no real gain. Second fork — hold expiry: *timer per hold* (many timers) vs *lazy check on every access + periodic sweep* (simple, robust).
- **Thought process → decision:** **B** with lazy expiry + sweep. The `Show` is the aggregate: it owns the seat states, the holds, and the lock; `BookingService` orchestrates payment and translates outcomes into show operations. Confirmation is idempotent by hold id (a second confirm returns the same booking). Payment failure or timeout releases the hold. Explicit states for a seat: `AVAILABLE`, `HELD(hold_id, until)`, `BOOKED(booking_id)` — modelled as a small tagged union so illegal combinations can't exist.
- **Code (core):**

```python
"""Day 49 — Show as the concurrency aggregate: seat states, timed holds, atomic multi-seat, idempotent confirm."""

from __future__ import annotations

import threading
from collections.abc import Callable
from dataclasses import dataclass, field
from datetime import datetime, timedelta, timezone
from decimal import Decimal
from enum import Enum, auto
from uuid import UUID, uuid4


class BookingError(Exception): ...
class SeatUnavailableError(BookingError): ...
class HoldExpiredError(BookingError): ...
class UnknownHoldError(BookingError): ...


@dataclass(frozen=True)
class Seat:
    row: str
    number: int
    price: Decimal

    @property
    def label(self) -> str:
        return f"{self.row}{self.number}"


class SeatStatus(Enum):
    AVAILABLE = auto()
    HELD = auto()
    BOOKED = auto()


@dataclass
class Hold:
    id: UUID
    user_id: str
    seats: tuple[Seat, ...]
    expires_at: datetime
    booking_id: UUID | None = None                # set once confirmed → idempotent re-confirm


@dataclass(frozen=True)
class Booking:
    id: UUID
    user_id: str
    show_id: UUID
    seats: tuple[Seat, ...]
    total: Decimal


class Show:
    """Aggregate root: everything about one show's seats happens under one lock."""

    HOLD_DURATION = timedelta(minutes=10)

    def __init__(self, seats: list[Seat], clock: Callable[[], datetime] = lambda: datetime.now(timezone.utc)) -> None:
        self.id = uuid4()
        self._clock = clock
        self._status: dict[str, SeatStatus] = {s.label: SeatStatus.AVAILABLE for s in seats}
        self._seats = {s.label: s for s in seats}
        self._holds: dict[UUID, Hold] = {}
        self._held_by_seat: dict[str, UUID] = {}
        self._bookings: dict[UUID, Booking] = {}
        self._lock = threading.Lock()

    def available_seats(self) -> list[Seat]:
        with self._lock:
            self._expire_holds()
            return [self._seats[l] for l, s in self._status.items() if s is SeatStatus.AVAILABLE]

    def hold(self, user_id: str, labels: list[str]) -> Hold:
        if not labels:
            raise BookingError("select at least one seat")
        with self._lock:
            self._expire_holds()
            unknown = [l for l in labels if l not in self._seats]
            if unknown:
                raise BookingError(f"unknown seats {unknown}")
            taken = [l for l in labels if self._status[l] is not SeatStatus.AVAILABLE]
            if taken:
                raise SeatUnavailableError(f"seats not available: {taken}")     # all-or-nothing: nothing changed yet
            hold = Hold(uuid4(), user_id, tuple(self._seats[l] for l in labels), self._clock() + self.HOLD_DURATION)
            for label in labels:
                self._status[label] = SeatStatus.HELD
                self._held_by_seat[label] = hold.id
            self._holds[hold.id] = hold
            return hold

    def confirm(self, hold_id: UUID) -> Booking:
        """Idempotent: confirming an already-confirmed hold returns the same booking."""
        with self._lock:
            hold = self._holds.get(hold_id)
            if hold is None:
                raise UnknownHoldError(str(hold_id))
            if hold.booking_id is not None:
                return self._bookings[hold.booking_id]
            if hold.expires_at <= self._clock():
                self._release(hold)
                raise HoldExpiredError(f"hold {hold_id} expired")
            booking = Booking(uuid4(), hold.user_id, self.id, hold.seats, sum((s.price for s in hold.seats), Decimal("0")))
            for seat in hold.seats:
                self._status[seat.label] = SeatStatus.BOOKED
                del self._held_by_seat[seat.label]
            hold.booking_id = booking.id
            self._bookings[booking.id] = booking
            return booking

    def release(self, hold_id: UUID) -> None:
        with self._lock:
            hold = self._holds.get(hold_id)
            if hold is None or hold.booking_id is not None:
                return                                     # nothing to release: idempotent
            self._release(hold)

    def _release(self, hold: Hold) -> None:               # caller holds the lock
        for seat in hold.seats:
            if self._status[seat.label] is SeatStatus.HELD:
                self._status[seat.label] = SeatStatus.AVAILABLE
                self._held_by_seat.pop(seat.label, None)
        del self._holds[hold.id]

    def _expire_holds(self) -> None:                      # lazy sweep; also run periodically
        now = self._clock()
        for hold in [h for h in self._holds.values() if h.booking_id is None and h.expires_at <= now]:
            self._release(hold)


class PaymentGateway:                                    # port (Day 24); a fake for the demo
    def charge(self, user_id: str, amount: Decimal) -> bool:
        return amount < Decimal("100000")


class BookingService:
    def __init__(self, gateway: PaymentGateway) -> None:
        self._gateway = gateway

    def book(self, show: Show, user_id: str, labels: list[str]) -> Booking:
        hold = show.hold(user_id, labels)                 # step 1: reserve atomically
        try:
            total = sum((s.price for s in hold.seats), Decimal("0"))
            if not self._gateway.charge(user_id, total):  # step 2: pay (slow, external, may fail)
                raise BookingError("payment declined")
            return show.confirm(hold.id)                  # step 3: confirm (idempotent)
        except Exception:
            show.release(hold.id)                         # any failure: never strand a hold
            raise
```

- **Python internals:** The status dict plus separate `_held_by_seat`/`_holds` maps are *indexes over one truth* — all mutated only under the show lock, which is what makes them safe to keep in sync (contrast Day 6's warning: multiple structures are fine when one lock and one class own them). `threading.Lock` (not `RLock`) is deliberate: internal helpers marked "caller holds the lock" never re-acquire, and a plain lock makes an accidental re-entrancy a visible deadlock in tests rather than a silent one. The service's `try/except Exception … raise` is compensation (Day 24): the hold is released on *any* failure path, including the gateway raising unexpectedly. Real systems replace the in-process lock with a DB transaction or a Redis lock — the *shape* (aggregate + atomic hold + idempotent confirm) stays identical.
- **Build & drill:** Write the race test: 100 threads each trying to hold seat A1 → exactly one succeeds. Add a background sweeper thread calling `available_seats()` every second and a test where a hold expires mid-payment. Add `cancel(booking_id)` with a refund policy strategy (full > 24h, 50% > 2h, else none). Model `Theatre → Screen → Show` and a `SeatType` price map. Then write the "how does this work across two servers?" answer (DB row locks or optimistic versioning; Redis `SET NX PX` for holds).
- **Recall:** Why is the `Show` the lock's unit? How is all-or-nothing achieved without rollback code? Why must `confirm` be idempotent? Lazy expiry vs timers — tradeoff?

### Day 50 — Ride Sharing (Uber) and Food Delivery: matching strategies, a spatial index, and the trip lifecycle

- **Concept & why it matters:** Marketplace systems introduce **matching** (which driver for this rider?) as a strategy with a **spatial index** behind it (you cannot scan a million drivers per request), a **trip state machine** with two actors (rider and driver both drive transitions), **pricing** as a strategy (base + distance + time + surge), and **driver availability** as state that must flip atomically when matched (two riders, one driver — the booking race again). New force: **geo queries** — the simplest LLD-scale index is a uniform grid (geohash-lite): bucket drivers by cell, search the neighbouring cells, expand rings until enough candidates.
- **Real-world case study:** **Uber's H3** hexagonal grid and **Lyft's** S2-based indexing exist because "nearest driver" at scale is an index problem, not a loop; their dispatch systems match in *batches* every few seconds (a global optimization) rather than greedily per request — a strategy swap you can model. **Swiggy/Zomato** face the same shape with a third party (the restaurant's prep time) and batch orders per rider — the *matching strategy* is the competitive core of these companies.
- **Design problem (method first):** Requirements: riders request a trip (pickup, dropoff); drivers go online/offline and report location; match a rider to a nearby available driver by strategy; the driver accepts or rejects (timeout → next candidate); trip lifecycle REQUESTED → MATCHED → STARTED → COMPLETED (or CANCELLED with rules); fare computed by a pricing strategy; notifications to both. Key fork — *how do you find nearby drivers?* **(A) scan all drivers and sort by distance** — O(n log n) per request; fine to 1k drivers, dead at 100k. **(B) a grid index: `dict[(cell_x, cell_y), set[driver_id]]`; query the 3×3 (then 5×5…) neighbourhood** — O(candidates) per request; cell size is a tuning knob; simple to implement and explain. **(C) a KD-tree / R-tree / geohash library** — better for skewed density; more dependency and complexity — name it as the production step. Second fork — *matching strategy*: nearest-first vs highest-rating-within-radius vs batch assignment.
- **Thought process → decision:** **B** for the index (owned by a `DriverRegistry` that also owns availability state and its lock so "find and reserve a driver" is atomic), **Strategy** for matching (receives candidates and returns an ordered list to try), **State** enum + transition table for the trip (thin per-state behaviour), pricing as a callable. The registry, not the matcher, flips availability — the matcher only *ranks*; atomic reservation stays with the owner of the state.
- **Code (core):**

```python
"""Day 50 — ride matching: grid index + atomic driver reservation; trip state machine; pricing strategy."""

from __future__ import annotations

import math
import threading
from collections.abc import Callable, Iterator
from dataclasses import dataclass, field
from decimal import Decimal
from enum import Enum, auto
from typing import Protocol
from uuid import UUID, uuid4


@dataclass(frozen=True, slots=True)
class Location:
    lat: float
    lon: float

    def distance_km(self, other: Location) -> float:      # equirectangular approximation: fine at city scale
        dlat = math.radians(other.lat - self.lat)
        dlon = math.radians(other.lon - self.lon) * math.cos(math.radians((self.lat + other.lat) / 2))
        return 6371 * math.hypot(dlat, dlon)


class DriverStatus(Enum):
    OFFLINE = auto()
    AVAILABLE = auto()
    ON_TRIP = auto()


@dataclass
class Driver:
    id: str
    location: Location
    rating: float = 5.0
    status: DriverStatus = DriverStatus.OFFLINE


class DriverRegistry:
    """Owns driver state and the spatial index; 'find nearby' and 'reserve' are atomic here."""

    def __init__(self, cell_km: float = 1.0) -> None:
        self._cell_deg = cell_km / 111.0                   # ~111 km per degree of latitude
        self._drivers: dict[str, Driver] = {}
        self._cells: dict[tuple[int, int], set[str]] = {}
        self._lock = threading.Lock()

    def _cell(self, loc: Location) -> tuple[int, int]:
        return (int(loc.lat // self._cell_deg), int(loc.lon // self._cell_deg))

    def upsert(self, driver: Driver) -> None:
        with self._lock:
            old = self._drivers.get(driver.id)
            if old is not None:
                self._cells.get(self._cell(old.location), set()).discard(driver.id)
            self._drivers[driver.id] = driver
            self._cells.setdefault(self._cell(driver.location), set()).add(driver.id)

    def set_status(self, driver_id: str, status: DriverStatus) -> None:
        with self._lock:
            self._drivers[driver_id].status = status

    def nearby_available(self, origin: Location, max_rings: int = 3, want: int = 5) -> list[Driver]:
        with self._lock:
            cx, cy = self._cell(origin)
            found: list[Driver] = []
            for ring in range(max_rings + 1):              # expand outward until enough candidates
                for dx in range(-ring, ring + 1):
                    for dy in range(-ring, ring + 1):
                        if max(abs(dx), abs(dy)) != ring:
                            continue                       # only the ring's perimeter
                        for driver_id in self._cells.get((cx + dx, cy + dy), ()):
                            driver = self._drivers[driver_id]
                            if driver.status is DriverStatus.AVAILABLE:
                                found.append(driver)
                if len(found) >= want:
                    break
            return found

    def try_reserve(self, driver_id: str) -> bool:
        with self._lock:                                   # check-then-act under the lock: no double dispatch
            driver = self._drivers[driver_id]
            if driver.status is not DriverStatus.AVAILABLE:
                return False
            driver.status = DriverStatus.ON_TRIP
            return True


class MatchingStrategy(Protocol):
    def rank(self, pickup: Location, candidates: list[Driver]) -> list[Driver]: ...


class NearestFirst:
    def rank(self, pickup: Location, candidates: list[Driver]) -> list[Driver]:
        return sorted(candidates, key=lambda d: d.location.distance_km(pickup))


class BestRatedWithinRadius:
    def __init__(self, radius_km: float) -> None:
        self._radius = radius_km

    def rank(self, pickup: Location, candidates: list[Driver]) -> list[Driver]:
        close = [d for d in candidates if d.location.distance_km(pickup) <= self._radius]
        return sorted(close, key=lambda d: (-d.rating, d.location.distance_km(pickup)))


class TripStatus(Enum):
    REQUESTED = auto()
    MATCHED = auto()
    STARTED = auto()
    COMPLETED = auto()
    CANCELLED = auto()


_TRANSITIONS = {
    TripStatus.REQUESTED: {TripStatus.MATCHED, TripStatus.CANCELLED},
    TripStatus.MATCHED: {TripStatus.STARTED, TripStatus.CANCELLED},
    TripStatus.STARTED: {TripStatus.COMPLETED},
    TripStatus.COMPLETED: set(),
    TripStatus.CANCELLED: set(),
}


@dataclass
class Trip:
    rider_id: str
    pickup: Location
    dropoff: Location
    id: UUID = field(default_factory=uuid4)
    status: TripStatus = TripStatus.REQUESTED
    driver_id: str | None = None
    fare: Decimal | None = None

    def transition(self, target: TripStatus) -> None:
        if target not in _TRANSITIONS[self.status]:
            raise ValueError(f"{self.status.name} -> {target.name} not allowed")
        self.status = target


PricingStrategy = Callable[[Trip, Decimal], Decimal]     # (trip, surge multiplier) -> fare


def standard_pricing(trip: Trip, surge: Decimal) -> Decimal:
    base, per_km = Decimal("40"), Decimal("12")
    distance = Decimal(str(round(trip.pickup.distance_km(trip.dropoff), 2)))
    return ((base + per_km * distance) * surge).quantize(Decimal("1"))


class DispatchService:
    def __init__(self, registry: DriverRegistry, matcher: MatchingStrategy, pricing: PricingStrategy,
                 driver_accepts: Callable[[Driver, Trip], bool]) -> None:
        self._registry, self._matcher, self._pricing, self._driver_accepts = registry, matcher, pricing, driver_accepts
        self._trips: dict[UUID, Trip] = {}

    def request(self, rider_id: str, pickup: Location, dropoff: Location) -> Trip:
        trip = Trip(rider_id, pickup, dropoff)
        self._trips[trip.id] = trip
        for driver in self._matcher.rank(pickup, self._registry.nearby_available(pickup)):
            if not self._registry.try_reserve(driver.id):    # someone else got them first: next
                continue
            if self._driver_accepts(driver, trip):             # offer with timeout in production
                trip.driver_id = driver.id
                trip.transition(TripStatus.MATCHED)
                return trip
            self._registry.set_status(driver.id, DriverStatus.AVAILABLE)   # rejected: give them back
        trip.transition(TripStatus.CANCELLED)
        return trip

    def start(self, trip_id: UUID) -> None:
        self._trips[trip_id].transition(TripStatus.STARTED)

    def complete(self, trip_id: UUID, surge: Decimal = Decimal("1.0")) -> Decimal:
        trip = self._trips[trip_id]
        trip.transition(TripStatus.COMPLETED)
        trip.fare = self._pricing(trip, surge)
        assert trip.driver_id is not None
        self._registry.set_status(trip.driver_id, DriverStatus.AVAILABLE)
        return trip.fare
```

- **Python internals:** Floor division on floats (`loc.lat // cell`) buckets coordinates into integer cells in one operation; tuples of ints are hashable keys, so the grid is a plain dict — O(1) cell lookup with no library. The ring-expansion loop visits only the *perimeter* (`max(abs(dx), abs(dy)) == ring`), avoiding re-scanning inner cells. `try_reserve` is the atomic check-then-act (Day 27) that makes concurrent dispatch safe; because the registry owns both the index and the status, the lock covers both. `math.hypot` is numerically stabler than `sqrt(a*a + b*b)`.
- **Build & drill:** Add offer timeouts (a driver who doesn't answer in 15 s is skipped) using Day 39's scheduler or a `Condition`. Implement batch matching (collect requests for 2 s, assign greedily by total distance) as another `MatchingStrategy` — note what changes in the service. Model **food delivery**: `Order → Restaurant (prep time) → DeliveryPartner`, and decide whether the partner is matched at order time or at "food ready" time (argue with data). Add surge as a function of demand/supply per cell. Publish trip events on the Day 32 bus.
- **Recall:** Why does the registry (not the matcher) flip availability? How does the grid make "nearby" cheap? What's the tradeoff of cell size? Which transitions may the rider vs the driver trigger?

### Day 51 — E-commerce: cart, inventory reservation, order lifecycle, payment, and idempotency keys

- **Concept & why it matters:** The system that ties together most of what you know: a **cart** (mutable, per user, priced on read), **inventory with reservations** (available = on-hand − reserved; reservations expire — Day 49's hold, generalized), an **order state machine** (CREATED → PAID → SHIPPED → DELIVERED, CANCELLED/REFUNDED branches), **payment as an external, failure-prone step**, **compensation** when a later step fails (release reservation, refund), and the new force: **idempotency keys** — the client retries "place order" after a timeout, and you must not create two orders or charge twice. Also **events** (Day 32) for notifications and analytics, published *after* the state change commits.
- **Real-world case study:** **Stripe's `Idempotency-Key` header** is the industry reference: the server stores the key with the first response and replays it on retries for 24 hours — Stripe's engineering blog explains the failure modes it prevents (network timeouts after the charge succeeded). **Amazon's** order pipeline reserves inventory at checkout and releases on payment failure; overselling on Prime Day is the failure they design against. Every payments postmortem you'll read involves a retry that wasn't idempotent.
- **Design problem (method first):** Requirements: products with stock; cart add/remove/quantity; checkout creates an order, reserves stock, charges payment, confirms; if payment fails, release stock; order lifecycle with legal transitions; cancellation before shipping refunds; retries of "place order" with the same idempotency key return the original result; concurrent checkouts on the same product never oversell. Key fork — *how is "never oversell" enforced?* **(A) check `stock >= qty` then decrement in the service** — check-then-act across objects; two checkouts pass the check together. **(B) `Inventory.reserve(sku, qty)` as an atomic operation under the inventory's lock, tracking reservations separately from on-hand so a failed payment can release** — one owner; available derived; oversell impossible. **(C) optimistic versioning (compare-and-set on a version number, retry on conflict)** — the DB-friendly version of B; mention it. Second fork — *idempotency storage*: in the service (dict key → result), with TTL.
- **Thought process → decision:** **B** for inventory (Information Expert + one lock), a **transition table** for the order (thin behaviour), an `OrderService.place_order(idempotency_key, …)` that first checks the idempotency store (under a lock, so two concurrent identical requests don't both proceed — store an *in-progress* marker), then reserves, charges, confirms, and stores the result; on any failure, compensates and stores the failure too (a retry of a failed request should fail the same way, not re-attempt blindly — or should it? decide and document: here, failures are *not* cached so a transient decline can be retried with a new key).
- **Code (core):**

```python
"""Day 51 — atomic inventory reservations, an order state machine, and idempotent order placement."""

from __future__ import annotations

import threading
from dataclasses import dataclass, field
from decimal import Decimal
from enum import Enum, auto
from typing import Protocol
from uuid import UUID, uuid4


class ShopError(Exception): ...
class OutOfStockError(ShopError): ...
class PaymentDeclinedError(ShopError): ...
class RequestInProgressError(ShopError): ...


@dataclass(frozen=True)
class CartLine:
    sku: str
    quantity: int
    unit_price: Decimal


class Cart:
    def __init__(self, user_id: str) -> None:
        self.user_id = user_id
        self._lines: dict[str, CartLine] = {}

    def set_quantity(self, sku: str, quantity: int, unit_price: Decimal) -> None:
        if quantity < 0:
            raise ValueError("quantity cannot be negative")
        if quantity == 0:
            self._lines.pop(sku, None)
        else:
            self._lines[sku] = CartLine(sku, quantity, unit_price)

    @property
    def lines(self) -> tuple[CartLine, ...]:
        return tuple(self._lines.values())

    @property
    def total(self) -> Decimal:
        return sum((l.unit_price * l.quantity for l in self._lines.values()), Decimal("0"))


class Inventory:
    """available = on_hand - reserved. Reserve/release/commit are atomic; oversell is impossible."""

    def __init__(self) -> None:
        self._on_hand: dict[str, int] = {}
        self._reserved: dict[str, int] = {}
        self._lock = threading.Lock()

    def add_stock(self, sku: str, quantity: int) -> None:
        with self._lock:
            self._on_hand[sku] = self._on_hand.get(sku, 0) + quantity

    def available(self, sku: str) -> int:
        with self._lock:
            return self._on_hand.get(sku, 0) - self._reserved.get(sku, 0)

    def reserve(self, lines: tuple[CartLine, ...]) -> None:
        with self._lock:                                          # all lines or none
            short = [l.sku for l in lines if self._on_hand.get(l.sku, 0) - self._reserved.get(l.sku, 0) < l.quantity]
            if short:
                raise OutOfStockError(f"insufficient stock for {short}")
            for line in lines:
                self._reserved[line.sku] = self._reserved.get(line.sku, 0) + line.quantity

    def release(self, lines: tuple[CartLine, ...]) -> None:
        with self._lock:
            for line in lines:
                self._reserved[line.sku] -= line.quantity

    def commit(self, lines: tuple[CartLine, ...]) -> None:        # reservation becomes a real decrement
        with self._lock:
            for line in lines:
                self._reserved[line.sku] -= line.quantity
                self._on_hand[line.sku] -= line.quantity

    def restock(self, lines: tuple[CartLine, ...]) -> None:       # a refund returns goods to on-hand
        with self._lock:
            for line in lines:
                self._on_hand[line.sku] = self._on_hand.get(line.sku, 0) + line.quantity


class OrderStatus(Enum):
    CREATED = auto()
    PAID = auto()
    SHIPPED = auto()
    DELIVERED = auto()
    CANCELLED = auto()
    REFUNDED = auto()


_TRANSITIONS = {
    OrderStatus.CREATED: {OrderStatus.PAID, OrderStatus.CANCELLED},
    OrderStatus.PAID: {OrderStatus.SHIPPED, OrderStatus.REFUNDED},
    OrderStatus.SHIPPED: {OrderStatus.DELIVERED},
    OrderStatus.DELIVERED: set(), OrderStatus.CANCELLED: set(), OrderStatus.REFUNDED: set(),
}


@dataclass
class Order:
    user_id: str
    lines: tuple[CartLine, ...]
    total: Decimal
    id: UUID = field(default_factory=uuid4)
    status: OrderStatus = OrderStatus.CREATED
    payment_ref: str | None = None

    def transition(self, target: OrderStatus) -> None:
        if target not in _TRANSITIONS[self.status]:
            raise ShopError(f"{self.status.name} -> {target.name} not allowed")
        self.status = target


class PaymentGateway(Protocol):
    def charge(self, user_id: str, amount: Decimal, idempotency_key: str) -> str: ...
    def refund(self, payment_ref: str) -> None: ...


_IN_PROGRESS = object()                                           # sentinel stored while a request runs


class OrderService:
    def __init__(self, inventory: Inventory, gateway: PaymentGateway) -> None:
        self._inventory, self._gateway = inventory, gateway
        self._orders: dict[UUID, Order] = {}
        self._idempotency: dict[str, object] = {}                # key -> Order | _IN_PROGRESS
        self._lock = threading.Lock()

    def place_order(self, cart: Cart, idempotency_key: str) -> Order:
        with self._lock:                                          # claim the key atomically
            existing = self._idempotency.get(idempotency_key)
            if existing is _IN_PROGRESS:
                raise RequestInProgressError("identical request is being processed")
            if isinstance(existing, Order):
                return existing                                   # replay: same result, no second charge
            if not cart.lines:
                raise ShopError("cart is empty")
            self._idempotency[idempotency_key] = _IN_PROGRESS
        try:
            order = Order(cart.user_id, cart.lines, cart.total)
            self._inventory.reserve(order.lines)
            try:
                order.payment_ref = self._gateway.charge(cart.user_id, order.total, idempotency_key)
            except Exception:
                self._inventory.release(order.lines)              # compensation
                raise
            self._inventory.commit(order.lines)
            order.transition(OrderStatus.PAID)
            with self._lock:
                self._orders[order.id] = order
                self._idempotency[idempotency_key] = order         # store the result for replays
            return order
        except Exception:
            with self._lock:
                self._idempotency.pop(idempotency_key, None)       # failed: allow a genuine retry
            raise

    def cancel(self, order_id: UUID) -> None:
        order = self._orders[order_id]
        if order.status is OrderStatus.PAID:
            assert order.payment_ref is not None
            self._gateway.refund(order.payment_ref)
            self._inventory.restock(order.lines)                  # goods come back; money goes back
            order.transition(OrderStatus.REFUNDED)
        else:
            order.transition(OrderStatus.CANCELLED)
```

- **Python internals:** A module-level `object()` sentinel is the idiom for a unique marker that can't collide with real values (`None` might be a legitimate value); comparing with `is` is exact. The idempotency dict does double duty as a *mutex per key*: claiming the key under the lock and releasing it on failure is a tiny lock manager — the same idea as `SET NX` in Redis. Note the *two* lock scopes: the service lock is held only for the dictionary operations, never during the slow `charge` call — holding a lock across I/O is how you serialize your whole system (Day 27's rule). `isinstance(existing, Order)` narrows the `object`-typed value for the checker.
- **Build & drill:** Write the refund-path tests for `cancel` (PAID → REFUNDED restores stock and refunds exactly once; SHIPPED cannot be cancelled). Add reservation *expiry* (a cart abandoned mid-checkout) using Day 49's lazy approach. Write the concurrency test: 20 threads placing orders for the last 5 units → exactly 5 succeed. Test idempotent replay: same key twice → one charge (count gateway calls with a fake). Publish `OrderPaid`/`OrderRefunded` events after the state change. Draw the sequence diagram for a payment timeout followed by a client retry.
- **Recall:** Why separate `reserved` from `on_hand`? Why claim the idempotency key *before* doing work? Why is the lock released during `charge`? Should failed requests be cached under the key — what did you decide and why?

### Day 52 — Hotel Booking and Meeting-Room Scheduler / Calendar: intervals, recurrence, conflicts, and availability search

- **Concept & why it matters:** Time-interval systems: **half-open intervals** (`[start, end)` so adjacent bookings don't collide), **conflict detection** in O(log n) via a sorted list + `bisect` (or an interval tree when queries dominate), **recurring events** (weekly stand-up for 12 weeks) expanded lazily within a query window, **availability search** ("find a room free 2–3 pm for 8 people with a projector") as filtering + interval checks, and **time zones** as a boundary concern (store UTC, present local). New force: **choosing a data structure by the query mix** — inserts vs overlap queries vs range scans — and being able to say the Big-O for each option.
- **Real-world case study:** **Google Calendar's** "find a time" and **Microsoft's Bookings** are interval-conflict engines with recurrence rules (RFC 5545 RRULE — the standard's complexity is why libraries like `dateutil.rrule` exist; never hand-roll full RRULE). **Airbnb/Booking.com** availability calendars are per-listing sorted interval sets; their double-booking incidents came from timezone and inclusive/exclusive boundary bugs — exactly the half-open rule.
- **Design problem (method first):** Requirements: rooms with capacity and amenities; book a room for `[start, end)`; reject conflicts; recurring bookings (weekly, N occurrences) with conflict checks against every occurrence; cancel one occurrence or the series; "find available rooms" for a window and constraints; list a room's schedule for a day. Key fork — *how are a room's bookings stored?* **(A) an unsorted list; check overlap by scanning** — O(n) per check; fine for a room with 50 bookings, poor at 50k. **(B) a sorted list of `(start, end)` with `bisect` — O(log n) to locate, O(1) neighbours to check for overlap (only the predecessor and successor can conflict when the set is non-overlapping)** — simple, fast, standard. **(C) an interval tree / augmented BST** — O(log n + k) for *overlapping-range* queries; needed when you query "everything overlapping [a, b)" often. Second fork — recurrence: *materialize all occurrences* on creation (simple; N rows; the check is N inserts) vs *store the rule and expand on query* (compact; every query expands).
- **Thought process → decision:** **B** for storage (the sorted list's invariant "non-overlapping" makes the neighbour check sufficient — write that proof in a comment). For recurrence: *materialize* occurrences into the same sorted structure with a shared `series_id` — the conflict check for a series is then just N inserts with rollback on the first conflict (all-or-nothing, like Day 49), and "cancel one occurrence" is a delete. Expand-on-query is the better trade when series are infinite; ours are bounded. Half-open intervals via the `DateRange`-style value object from Day 26.
- **Code (core):**

```python
"""Day 52 — meeting rooms: sorted non-overlapping intervals with bisect, recurring series, availability search."""

from __future__ import annotations

import bisect
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from uuid import UUID, uuid4


class SchedulingError(Exception): ...
class ConflictError(SchedulingError): ...


@dataclass(frozen=True, order=True)
class Interval:
    """Half-open [start, end). Ordered by start so bisect works on lists of Intervals."""
    start: datetime
    end: datetime

    def __post_init__(self) -> None:
        if self.start >= self.end:
            raise SchedulingError("start must be before end")

    def overlaps(self, other: Interval) -> bool:
        return self.start < other.end and other.start < self.end


@dataclass(frozen=True)
class Booking:
    interval: Interval
    organizer: str
    title: str
    id: UUID = field(default_factory=uuid4)
    series_id: UUID | None = None


class Room:
    """Invariant: bookings are sorted by start and pairwise non-overlapping.
    Given that invariant, a new interval can only conflict with its predecessor or successor."""

    def __init__(self, name: str, capacity: int, amenities: frozenset[str] = frozenset()) -> None:
        if capacity <= 0:
            raise ValueError("capacity must be positive")
        self.name, self.capacity, self.amenities = name, capacity, amenities
        self._starts: list[datetime] = []          # parallel key list for bisect (kept in sync privately)
        self._bookings: list[Booking] = []

    def is_free(self, interval: Interval) -> bool:
        index = bisect.bisect_left(self._starts, interval.start)
        neighbours = self._bookings[max(0, index - 1): index + 1]
        return not any(b.interval.overlaps(interval) for b in neighbours)

    def book(self, interval: Interval, organizer: str, title: str, series_id: UUID | None = None) -> Booking:
        if not self.is_free(interval):
            raise ConflictError(f"{self.name} is busy during {interval.start:%H:%M}-{interval.end:%H:%M}")
        booking = Booking(interval, organizer, title, series_id=series_id)
        index = bisect.bisect_left(self._starts, interval.start)
        self._starts.insert(index, interval.start)
        self._bookings.insert(index, booking)
        return booking

    def cancel(self, booking_id: UUID) -> None:
        for index, booking in enumerate(self._bookings):
            if booking.id == booking_id:
                del self._starts[index]; del self._bookings[index]
                return
        raise SchedulingError(f"no booking {booking_id} in {self.name}")

    def schedule(self, day: datetime) -> list[Booking]:
        window = Interval(day.replace(hour=0, minute=0, second=0, microsecond=0), day.replace(hour=0, minute=0, second=0, microsecond=0) + timedelta(days=1))
        lo = bisect.bisect_left(self._starts, window.start)
        return [b for b in self._bookings[lo:] if b.interval.start < window.end]


class Scheduler:
    def __init__(self, rooms: list[Room]) -> None:
        self._rooms = {r.name: r for r in rooms}

    def find_available(self, interval: Interval, attendees: int, amenities: frozenset[str] = frozenset()) -> list[Room]:
        return sorted(
            (r for r in self._rooms.values()
             if r.capacity >= attendees and amenities <= r.amenities and r.is_free(interval)),
            key=lambda r: (r.capacity, r.name),            # smallest adequate room first
        )

    def book_series(self, room_name: str, first: Interval, organizer: str, title: str, *, weekly_occurrences: int) -> list[Booking]:
        """All occurrences or none: roll back on the first conflict."""
        if weekly_occurrences <= 0:
            raise ValueError("need at least one occurrence")
        room = self._rooms[room_name]
        series_id = uuid4()
        created: list[Booking] = []
        try:
            for week in range(weekly_occurrences):
                shift = timedelta(weeks=week)
                created.append(room.book(Interval(first.start + shift, first.end + shift), organizer, title, series_id))
        except ConflictError:
            for booking in created:
                room.cancel(booking.id)
            raise
        return created

    def cancel_series(self, room_name: str, series_id: UUID) -> int:
        room = self._rooms[room_name]
        doomed = [b.id for b in room._bookings if b.series_id == series_id]
        for booking_id in doomed:
            room.cancel(booking_id)
        return len(doomed)


if __name__ == "__main__":
    t = datetime(2026, 9, 7, 10, 0)
    rooms = [Room("Mars", 4, frozenset({"tv"})), Room("Jupiter", 12, frozenset({"tv", "projector"}))]
    scheduler = Scheduler(rooms)
    scheduler.book_series("Mars", Interval(t, t + timedelta(hours=1)), "asha", "standup", weekly_occurrences=4)
    print([r.name for r in scheduler.find_available(Interval(t, t + timedelta(minutes=30)), 3)])        # Jupiter
    print([r.name for r in scheduler.find_available(Interval(t + timedelta(hours=1), t + timedelta(hours=2)), 3)])   # Mars, Jupiter
    try:
        rooms[0].book(Interval(t + timedelta(weeks=2, minutes=30), t + timedelta(weeks=2, hours=2)), "ravi", "review")
    except ConflictError as err:
        print("conflict:", err)
```

- **Python internals:** `bisect` requires a list that stays sorted; keeping a *parallel key list* (`_starts`) lets you bisect on `datetime` without a `key=` (3.10+ supports `bisect(key=...)`, which removes the parallel list — use it). `list.insert` is O(n) memmove, but for thousands of bookings per room that's microseconds; a `sortedcontainers.SortedList` gives O(log n) inserts when it matters — name it. `order=True` on the frozen dataclass makes `Interval` comparable by `(start, end)`. Never use naive `datetime`s across zones: store timezone-aware UTC and convert with `zoneinfo` at the presentation boundary — the "adjacent bookings collide" bug is usually a DST bug in disguise.
- **Build & drill:** Implement an **interval tree** (or use the augmented sorted list) and add `overlapping(interval) -> list[Booking]`; benchmark against the scan for 100k bookings. Add "cancel this occurrence only" and "move the series." Build the **hotel** version: rooms with types and nightly rates, availability by date range (nights are half-open days), and an overbooking policy knob (hotels *do* overbook — a strategy with a percentage). Handle time zones properly with `zoneinfo` and a DST-transition test.
- **Recall:** Why half-open intervals? Why is checking only the two neighbours sufficient? Materialize vs expand for recurrence — the tradeoff? What structure would you pick if overlap-range queries dominated?

### Day 53 — Social Feed (Twitter) and Q&A (Stack Overflow): graphs, fan-out strategies, votes, and reputation rules

- **Concept & why it matters:** Two "content" systems with different cores. **Feed**: a **follow graph** (adjacency sets), posts, and the central design choice — **fan-out on write** (push each new post into every follower's precomputed inbox: fast reads, expensive writes for celebrities) vs **fan-out on read** (merge followees' recent posts at read time with a k-way heap merge: cheap writes, slower reads) vs **hybrid** (push for normal users, pull for celebrities). **Q&A**: questions, answers, comments, **votes** with rules (one vote per user per post, changeable), **reputation** as a derived score with event-driven rules (+10 upvote on answer, −2 for downvoting), **accepted answer** as an invariant (at most one per question, only by the asker), and moderation states. New force: **choosing a strategy by read/write ratio and skew**, and **derived values that must stay consistent under concurrent votes** (recompute vs increment).
- **Real-world case study:** **Twitter's timeline architecture** (public talks by Raffi Krikorian) used fan-out on write into Redis lists for most users and pulled celebrity tweets at read time — the hybrid — because Lady Gaga's 30M followers made a single tweet cost 30M writes. **Stack Overflow** runs on a handful of servers partly because reputation and vote counts are recomputed by scheduled jobs from the vote log (derived, correctable) rather than trusted counters; their "recalc" tooling exists because incremented counters *do* drift.
- **Design problem (method first — feed; Q&A is the drill):** Requirements: users follow/unfollow; post; home feed = latest N posts from followees, newest first; pluggable fan-out strategy; celebrities handled; unfollow removes their posts from the feed; concurrent posting. Key fork is the fan-out strategy itself — implement both behind one interface so the tradeoff is measurable, then the hybrid.
- **Thought process → decision:** Model `SocialGraph` (follow sets both directions, since fan-out needs followers and read needs followees), `PostStore` (per-user sorted-by-time posts), and a `FeedStrategy` Protocol with `on_post(author, post)` and `feed(user, limit)`. **Push** keeps a bounded `deque` inbox per user; **Pull** merges followees' post lists with `heapq.merge`; **Hybrid** pushes unless the author's follower count exceeds a threshold, and the read path pulls from celebrity followees and merges with the inbox. Post ids are time-ordered (a monotonic counter) so merging by id equals merging by time.
- **Code (core):**

```python
"""Day 53 — social feed with swappable fan-out strategies: push (write), pull (read), hybrid."""

from __future__ import annotations

import heapq
import threading
from collections import defaultdict, deque
from dataclasses import dataclass, field
from itertools import count
from typing import Protocol

_post_ids = count(1)                                      # monotonic → id order == time order


@dataclass(frozen=True, order=True)
class Post:
    id: int
    author: str = field(compare=False)
    text: str = field(compare=False)


class SocialGraph:
    def __init__(self) -> None:
        self._following: defaultdict[str, set[str]] = defaultdict(set)
        self._followers: defaultdict[str, set[str]] = defaultdict(set)
        self._lock = threading.Lock()

    def follow(self, follower: str, followee: str) -> None:
        if follower == followee:
            raise ValueError("cannot follow yourself")
        with self._lock:
            self._following[follower].add(followee)
            self._followers[followee].add(follower)

    def unfollow(self, follower: str, followee: str) -> None:
        with self._lock:
            self._following[follower].discard(followee)
            self._followers[followee].discard(follower)

    def following(self, user: str) -> frozenset[str]:
        with self._lock:
            return frozenset(self._following[user])

    def followers(self, user: str) -> frozenset[str]:
        with self._lock:
            return frozenset(self._followers[user])


class PostStore:
    def __init__(self) -> None:
        self._by_author: defaultdict[str, list[Post]] = defaultdict(list)   # append-only, ascending ids
        self._lock = threading.Lock()

    def add(self, author: str, text: str) -> Post:
        with self._lock:
            post = Post(next(_post_ids), author, text)
            self._by_author[author].append(post)
            return post

    def recent(self, author: str, limit: int) -> list[Post]:
        with self._lock:
            return self._by_author[author][-limit:][::-1]          # newest first


class FeedStrategy(Protocol):
    def on_post(self, post: Post) -> None: ...
    def feed(self, user: str, limit: int) -> list[Post]: ...


class PullFeed:
    """Fan-out on read: cheap writes; reads merge k followee lists (O(k log k) with a heap)."""
    def __init__(self, graph: SocialGraph, posts: PostStore) -> None:
        self._graph, self._posts = graph, posts

    def on_post(self, post: Post) -> None:
        pass

    def feed(self, user: str, limit: int) -> list[Post]:
        streams = [self._posts.recent(f, limit) for f in self._graph.following(user)]
        return list(heapq.merge(*streams, reverse=True))[:limit]    # each stream is newest-first


class PushFeed:
    """Fan-out on write: each post is copied into every follower's bounded inbox; reads are O(limit)."""
    def __init__(self, graph: SocialGraph, inbox_size: int = 500) -> None:
        self._graph = graph
        self._inbox: defaultdict[str, deque[Post]] = defaultdict(lambda: deque(maxlen=inbox_size))
        self._lock = threading.Lock()

    def on_post(self, post: Post) -> None:
        followers = self._graph.followers(post.author)
        with self._lock:
            for follower in followers:                              # O(followers) — the celebrity problem
                self._inbox[follower].append(post)

    def feed(self, user: str, limit: int) -> list[Post]:
        with self._lock:
            following = self._graph.following(user)
            return [p for p in reversed(self._inbox[user]) if p.author in following][:limit]   # unfollow filter


class HybridFeed:
    """Push for normal authors; pull for celebrities; merge at read time."""
    def __init__(self, graph: SocialGraph, posts: PostStore, celebrity_threshold: int = 10_000) -> None:
        self._graph, self._posts, self._threshold = graph, posts, celebrity_threshold
        self._push = PushFeed(graph)

    def _is_celebrity(self, user: str) -> bool:
        return len(self._graph.followers(user)) >= self._threshold

    def on_post(self, post: Post) -> None:
        if not self._is_celebrity(post.author):
            self._push.on_post(post)

    def feed(self, user: str, limit: int) -> list[Post]:
        celebrities = [f for f in self._graph.following(user) if self._is_celebrity(f)]
        streams = [self._push.feed(user, limit), *(self._posts.recent(c, limit) for c in celebrities)]
        return list(heapq.merge(*streams, reverse=True))[:limit]


class Timeline:
    def __init__(self, graph: SocialGraph, posts: PostStore, strategy: FeedStrategy) -> None:
        self._graph, self._posts, self._strategy = graph, posts, strategy

    def post(self, author: str, text: str) -> Post:
        post = self._posts.add(author, text)
        self._strategy.on_post(post)
        return post

    def home(self, user: str, limit: int = 20) -> list[Post]:
        return self._strategy.feed(user, limit)
```

- **Python internals:** `heapq.merge(*iterables, reverse=True)` does a lazy k-way merge of already-sorted inputs in O(n log k) — the exact primitive for fan-out on read; it requires each input sorted the same way (newest-first here), and `Post` ordering by `id` alone (`compare=False` on other fields) makes it correct. `deque(maxlen=n)` is a bounded ring buffer: appending past capacity evicts the oldest in O(1) — the inbox is size-capped by construction. Note the *unfollow* handling: push inboxes still hold the unfollowed author's posts, so the read path filters — the honest cost of precomputation is stale derived data; the alternative is scrubbing inboxes on unfollow (O(inbox)).
- **Build & drill:** Benchmark push vs pull with 1 celebrity (100k followers) and 10k normal users — write the numbers down and set the hybrid threshold from data. Build the **Stack Overflow** core: `Question`/`Answer`/`Comment` (Composite-ish), `Vote` with one-per-user-per-post invariant and change semantics, `Reputation` as a *fold over vote events* (Day 48's ledger idea) with a `recalculate(user)` that must equal the incremental value in tests, `accept_answer` enforcing "only the asker, at most one," and a `Badge` Observer. Add feed *ranking* as a strategy (recency vs engagement) and note it composes with fan-out.
- **Recall:** Push vs pull vs hybrid — the deciding variables? Why does `heapq.merge` fit? What's the cost of precomputed inboxes on unfollow? Increment vs recompute for reputation — why keep both?

### Day 54 — Key-Value Store with Transactions and TTL (nested transactions, undo logs, and the overlay tradeoff)

- **Concept & why it matters:** A store with `get/set/delete`, **transactions** with `begin/commit/rollback` that **nest**, and **TTL** expiry. New forces: **implementing atomicity in memory** three ways — **undo log** (record the inverse of each write; rollback replays inverses), **snapshot** (copy the whole map on `begin`; rollback restores), **overlay/layered maps** (each transaction is a delta layer; reads walk layers top-down; commit merges into the parent) — and knowing their costs: undo log is O(writes) memory and O(1) reads; snapshot is O(size) per begin; overlay is O(depth) reads. Plus **tombstones** (a delete inside a transaction must shadow the parent's value), **expiry with a min-heap** of deadlines, and the interface question: what does `get` of a missing key return?
- **Real-world case study:** **Redis** `MULTI`/`EXEC` (queued commands, no nesting), **SQLite savepoints** (nested transactions via `SAVEPOINT`/`RELEASE`/`ROLLBACK TO` — implemented with a journal, i.e. an undo log), and **Git's index** as an overlay over the working tree. **Docker's layered filesystem** is the overlay approach at OS scale — and its known cost is read amplification across many layers, exactly the overlay tradeoff.
- **Design problem (method first):** Requirements: `set(k, v, ttl=None)`, `get(k)`, `delete(k)`, `begin()`, `commit()`, `rollback()`; transactions nest (inner commit folds into outer; outer rollback discards everything); reads inside a transaction see its own writes; TTL expiry; thread-safe for single-store use; `count(value)` as a stretch. Key fork — undo log vs snapshot vs overlay: **(A) snapshot** — `begin` copies the dict: trivial, O(n) per begin, unusable for large stores. **(B) undo log** — each write records `(key, previous_value_or_absent)`; rollback replays in reverse; nested = a stack of logs; commit merges the inner log into the outer (so an outer rollback can still undo it); reads hit the base map directly (fast). **(C) overlay** — a stack of delta dicts with tombstones; reads walk top-down; commit merges the top into the next. Tradeoffs: B is optimal for read-heavy transactions with few writes (the common case); C is elegant and easy to reason about for `count()` but pays on reads; A is for tiny stores or demos.
- **Thought process → decision:** **B** (undo log) as the primary — say why: reads are the hot path and stay O(1); memory scales with writes, not store size. Implement commit as *merging the inner undo log into the outer* (prepend, preserving order) so nested semantics are exact. Model "absent" with a sentinel so "key didn't exist" and "key was `None`" are distinct. TTL: lazy on `get` plus a heap for `sweep()`; a `set` inside a transaction that later rolls back must also roll back the expiry — store expiry alongside the value in one record so the undo log covers both.
- **Code (core):**

```python
"""Day 54 — KV store with nested transactions via undo logs, and TTL via lazy check + heap sweep."""

from __future__ import annotations

import heapq
import threading
import time
from collections.abc import Callable
from dataclasses import dataclass
from typing import Any

_ABSENT = object()                                       # distinct from None


class TransactionError(Exception): ...


@dataclass(frozen=True)
class Record:
    value: Any
    expires_at: float | None


class KeyValueStore:
    def __init__(self, clock: Callable[[], float] = time.monotonic) -> None:
        self._data: dict[str, Record] = {}
        self._undo_stack: list[list[tuple[str, Record | object]]] = []    # one undo log per open transaction
        self._expiry_heap: list[tuple[float, str]] = []
        self._clock = clock
        self._lock = threading.RLock()

    # --- transactions ---------------------------------------------------------------------------
    def begin(self) -> None:
        with self._lock:
            self._undo_stack.append([])

    def commit(self) -> None:
        with self._lock:
            if not self._undo_stack:
                raise TransactionError("no open transaction")
            inner = self._undo_stack.pop()
            if self._undo_stack:
                self._undo_stack[-1].extend(inner)       # outer must still be able to undo the inner's writes
            # at top level, the writes are already in _data: nothing to do

    def rollback(self) -> None:
        with self._lock:
            if not self._undo_stack:
                raise TransactionError("no open transaction")
            for key, previous in reversed(self._undo_stack.pop()):
                if previous is _ABSENT:
                    self._data.pop(key, None)
                else:
                    self._data[key] = previous          # type: ignore[assignment]

    @property
    def depth(self) -> int:
        return len(self._undo_stack)

    # --- data ------------------------------------------------------------------------------------
    def _record_undo(self, key: str) -> None:
        if self._undo_stack:
            self._undo_stack[-1].append((key, self._data.get(key, _ABSENT)))

    def set(self, key: str, value: Any, ttl_seconds: float | None = None) -> None:
        if ttl_seconds is not None and ttl_seconds <= 0:
            raise ValueError("ttl must be positive")
        with self._lock:
            self._record_undo(key)
            expires_at = self._clock() + ttl_seconds if ttl_seconds is not None else None
            self._data[key] = Record(value, expires_at)
            if expires_at is not None:
                heapq.heappush(self._expiry_heap, (expires_at, key))

    def get(self, key: str, default: Any = None) -> Any:
        with self._lock:
            record = self._data.get(key)
            if record is None:
                return default
            if record.expires_at is not None and record.expires_at <= self._clock():
                self._record_undo(key)                  # expiry is a write too (so rollback can restore)
                del self._data[key]
                return default
            return record.value

    def delete(self, key: str) -> bool:
        with self._lock:
            if key not in self._data:
                return False
            self._record_undo(key)
            del self._data[key]
            return True

    def sweep(self) -> int:
        """Remove expired keys eagerly. Heap entries may be stale (key re-set) — verify before deleting."""
        removed = 0
        with self._lock:
            now = self._clock()
            while self._expiry_heap and self._expiry_heap[0][0] <= now:
                expires_at, key = heapq.heappop(self._expiry_heap)
                record = self._data.get(key)
                if record is not None and record.expires_at == expires_at:
                    self._record_undo(key)
                    del self._data[key]
                    removed += 1
        return removed

    def count(self, value: Any) -> int:                  # O(n): the overlay design makes this cheaper — drill
        with self._lock:
            now = self._clock()
            return sum(1 for r in self._data.values() if r.value == value and (r.expires_at is None or r.expires_at > now))


if __name__ == "__main__":
    now = [0.0]
    store = KeyValueStore(clock=lambda: now[0])
    store.set("a", 1)
    store.begin()
    store.set("a", 2); store.set("b", 3, ttl_seconds=5)
    store.begin()
    store.delete("a"); store.set("c", 9)
    print(store.get("a"), store.get("c"))              # None 9  (inner sees its own writes)
    store.rollback()                                    # inner only
    print(store.get("a"), store.get("c"))              # 2 None
    store.commit()                                      # outer commits a=2, b=3
    now[0] += 6
    print(store.get("a"), store.get("b"), store.depth)  # 2 None 0  (b expired lazily)
    store.begin(); store.set("a", 100); store.rollback()
    print(store.get("a"))                               # 2
```

- **Python internals:** The undo stack is a `list` of `list`s — `append`/`pop` at the end are O(1), and `extend` on commit is O(inner writes) — the whole transaction machinery is two list operations. Recording `self._data.get(key, _ABSENT)` *before* each write is the undo log; replaying in `reversed` order is essential when the same key is written twice in one transaction (the earliest previous value must win). The expiry heap holds *possibly stale* entries (a key re-set with a new TTL leaves the old deadline in the heap) — the sweep verifies `record.expires_at == expires_at` before deleting, the standard "lazy deletion" heap technique that avoids O(n) heap removal. `RLock` because `get` may perform an expiry write while `sweep` also holds the lock in tests calling both.
- **Build & drill:** Implement the **overlay** version (`list[dict]` with a `TOMBSTONE`), make `count()` O(1) by maintaining value→count in each layer, and write the comparison paragraph. Add `keys(prefix)` with a sorted structure. Add persistence via an **append-only command log** (Day 33/47) with replay on startup and periodic snapshots (Memento). Then design the **in-memory file system** (Day 36) *on top of* this store (paths as keys) and note what became easier and harder.
- **Recall:** Undo log vs snapshot vs overlay — cost of begin, read, write, rollback for each? Why replay the undo log in reverse? Why does commit merge into the outer log? How do stale heap entries get handled?

### Day 55 — Spreadsheet Engine: formula parsing, dependency graphs, cycle detection, and incremental recalculation

- **Concept & why it matters:** The most *algorithmic* LLD problem, and a great one because every part is a design choice: **cells** with values or **formulas** (`=A1+B2*2`, `=SUM(A1:A3)`), a **parser** (tokenizer + recursive descent — the only parsing you need to know by heart), an **AST** evaluated by a Visitor (Day 36), a **dependency graph** (which cells does this formula read?) with **reverse edges** (who depends on me?) so a change triggers **incremental recalculation** of exactly the affected cells in **topological order**, **cycle detection** (`A1 = B1`, `B1 = A1` must be rejected, not loop), and **error values** (`#DIV/0!`, `#REF!`, `#CYCLE!`) that propagate as values rather than exceptions. New force: **propagating change through a graph efficiently and safely** — the same shape as build systems, reactive UIs, and Excel itself.
- **Real-world case study:** **Excel's recalculation engine** (documented by Microsoft) maintains a dependency chain and dirty flags, recalculates only dirty cells in dependency order, and detects circular references with an explicit error state; its multithreaded recalc partitions the graph. **Build systems (Make, Bazel)** and **React/Solid signals** are the same DAG-propagation problem. Google Sheets' collaborative version adds conflict resolution on top — the graph core is unchanged.
- **Design problem (method first):** Requirements: set a cell to a number, text, or formula; formulas support `+ - * /`, parentheses, cell refs, `SUM(range)`; reading a cell gives its computed value; changing a cell updates dependents automatically; cycles are rejected at set time; errors are values; undo (Day 33). Key fork — *when to compute?* **(A) lazy: compute on read, recursively, with memoization and a "visiting" set for cycles** — simple; every read after a change may recompute a large subtree; cycles found only when read. **(B) eager: on set, rebuild the cell's dependencies, check for a cycle (DFS from the cell through *dependents* back to itself), then recompute the affected subgraph in topological order** — reads O(1); changes cost O(affected); cycle detected at set time (the requirement). **(C) hybrid with dirty flags** — mark dependents dirty on set, compute on read; Excel's approach; best when many changes happen between reads. Tradeoffs: A for a REPL, B for the stated requirements, C when writes vastly outnumber reads.
- **Thought process → decision:** **B**: the requirement "cycles rejected at set time" and "dependents update automatically" both point at eager. Structure: `Tokenizer` → `Parser` (recursive descent, precedence climbing) → AST nodes (frozen dataclasses) → `evaluate` visitor with a `Sheet` lookup; `Sheet` owns cells, `deps[cell]` (what it reads) and `dependents[cell]` (who reads it), performs cycle check via DFS over dependents before committing the change (and rolls back the dependency edges if a cycle is found), then a topological sort (Kahn's algorithm) over the affected set to recompute. Errors are a small `CellError` value type so `=A1+1` where `A1` is `#DIV/0!` yields `#DIV/0!` — errors propagate without exceptions.
- **Code (core):**

```python
"""Day 55 — spreadsheet: tokenizer → recursive-descent parser → AST → dependency DAG → topological recalc."""

from __future__ import annotations

import re
from collections import defaultdict, deque
from dataclasses import dataclass
from decimal import Decimal, InvalidOperation
from functools import singledispatch

TOKEN = re.compile(r"\s*(?:(?P<num>\d+(?:\.\d+)?)|(?P<ref>[A-Z]+\d+)|(?P<name>[A-Z]+)|(?P<op>[-+*/(),:]))")


class SheetError(Exception): ...
class ParseError(SheetError): ...
class CycleError(SheetError): ...


@dataclass(frozen=True)
class CellError:
    code: str                    # "#DIV/0!", "#REF!", "#VALUE!"


# --- AST ----------------------------------------------------------------------------------------
@dataclass(frozen=True)
class Num:
    value: Decimal

@dataclass(frozen=True)
class Ref:
    cell: str

@dataclass(frozen=True)
class Range:
    start: str
    end: str

@dataclass(frozen=True)
class BinOp:
    op: str
    left: object
    right: object

@dataclass(frozen=True)
class Call:
    name: str
    args: tuple[object, ...]


class Parser:
    """expr := term (('+'|'-') term)* ; term := factor (('*'|'/') factor)* ; factor := num | ref | ref ':' ref | name '(' args ')' | '(' expr ')' | '-' factor"""

    def __init__(self, text: str) -> None:
        self._tokens = [(m.lastgroup, m.group(m.lastgroup)) for m in TOKEN.finditer(text) if m.lastgroup]
        if "".join(t for _, t in self._tokens) != text.replace(" ", ""):
            raise ParseError(f"unexpected characters in {text!r}")
        self._pos = 0

    def parse(self) -> object:
        node = self._expr()
        if self._pos != len(self._tokens):
            raise ParseError("trailing tokens")
        return node

    def _peek(self) -> tuple[str, str] | None:
        return self._tokens[self._pos] if self._pos < len(self._tokens) else None

    def _take(self, kind: str | None = None, text: str | None = None) -> str:
        token = self._peek()
        if token is None or (kind and token[0] != kind) or (text and token[1] != text):
            raise ParseError(f"expected {text or kind}, got {token}")
        self._pos += 1
        return token[1]

    def _expr(self) -> object:
        node = self._term()
        while (t := self._peek()) and t[1] in "+-":
            node = BinOp(self._take(), node, self._term())
        return node

    def _term(self) -> object:
        node = self._factor()
        while (t := self._peek()) and t[1] in "*/":
            node = BinOp(self._take(), node, self._factor())
        return node

    def _factor(self) -> object:
        token = self._peek()
        if token is None:
            raise ParseError("unexpected end of formula")
        kind, text = token
        if kind == "num":
            return Num(Decimal(self._take()))
        if kind == "ref":
            start = self._take()
            if (t := self._peek()) and t[1] == ":":
                self._take(); return Range(start, self._take("ref"))
            return Ref(start)
        if kind == "name":
            name = self._take(); self._take(text="(")
            args: list[object] = [self._expr()]
            while (t := self._peek()) and t[1] == ",":
                self._take(); args.append(self._expr())
            self._take(text=")")
            return Call(name, tuple(args))
        if text == "(":
            self._take(); node = self._expr(); self._take(text=")"); return node
        if text == "-":
            self._take(); return BinOp("-", Num(Decimal(0)), self._factor())
        raise ParseError(f"unexpected token {text!r}")


# --- dependency extraction & evaluation (Visitors via singledispatch) ----------------------------
def expand_range(start: str, end: str) -> list[str]:
    (c1, r1), (c2, r2) = _split(start), _split(end)
    return [f"{chr(c)}{r}" for c in range(min(c1, c2), max(c1, c2) + 1) for r in range(min(r1, r2), max(r1, r2) + 1)]

def _split(ref: str) -> tuple[int, int]:
    match = re.fullmatch(r"([A-Z])(\d+)", ref)
    if match is None:
        raise ParseError(f"only single-letter columns supported: {ref}")
    return ord(match.group(1)), int(match.group(2))


@singledispatch
def references(node: object) -> set[str]:
    return set()

@references.register
def _(node: Ref) -> set[str]:
    return {node.cell}

@references.register
def _(node: Range) -> set[str]:
    return set(expand_range(node.start, node.end))

@references.register
def _(node: BinOp) -> set[str]:
    return references(node.left) | references(node.right)

@references.register
def _(node: Call) -> set[str]:
    return set().union(*(references(a) for a in node.args))


Value = Decimal | str | CellError | None


class Sheet:
    def __init__(self) -> None:
        self._raw: dict[str, str] = {}
        self._ast: dict[str, object] = {}
        self._values: dict[str, Value] = {}
        self._deps: dict[str, set[str]] = defaultdict(set)        # cell -> cells it reads
        self._dependents: dict[str, set[str]] = defaultdict(set)  # cell -> cells that read it

    def set(self, cell: str, raw: str) -> None:
        old_raw, old_deps = self._raw.get(cell), set(self._deps[cell])
        try:
            self._install(cell, raw)
            if self._reaches_itself(cell):
                raise CycleError(f"setting {cell} creates a circular reference")
        except SheetError:
            if old_raw is None:
                self._remove(cell)
            else:
                self._install(cell, old_raw)                 # restore edges and AST
            raise
        self._recalculate_from(cell)

    def get(self, cell: str) -> Value:
        return self._values.get(cell)

    def _install(self, cell: str, raw: str) -> None:
        for dep in self._deps[cell]:
            self._dependents[dep].discard(cell)
        self._raw[cell] = raw
        if raw.startswith("="):
            ast = Parser(raw[1:].upper()).parse()
            self._ast[cell] = ast
            self._deps[cell] = references(ast)
        else:
            self._ast.pop(cell, None)
            self._deps[cell] = set()
        for dep in self._deps[cell]:
            self._dependents[dep].add(cell)

    def _remove(self, cell: str) -> None:
        for dep in self._deps.pop(cell, set()):
            self._dependents[dep].discard(cell)
        self._raw.pop(cell, None); self._ast.pop(cell, None); self._values.pop(cell, None)

    def _reaches_itself(self, start: str) -> bool:
        stack, seen = [start], set()
        while stack:
            for nxt in self._dependents[stack.pop()]:
                if nxt == start:
                    return True
                if nxt not in seen:
                    seen.add(nxt); stack.append(nxt)
        return False

    def _affected(self, start: str) -> set[str]:
        affected, stack = {start}, [start]
        while stack:
            for nxt in self._dependents[stack.pop()]:
                if nxt not in affected:
                    affected.add(nxt); stack.append(nxt)
        return affected

    def _recalculate_from(self, start: str) -> None:
        affected = self._affected(start)
        indegree = {c: len(self._deps[c] & affected) for c in affected}     # Kahn's algorithm on the subgraph
        ready = deque(c for c, d in indegree.items() if d == 0)
        while ready:
            cell = ready.popleft()
            self._values[cell] = self._compute(cell)
            for dependent in self._dependents[cell]:
                if dependent in affected:
                    indegree[dependent] -= 1
                    if indegree[dependent] == 0:
                        ready.append(dependent)

    def _compute(self, cell: str) -> Value:
        raw = self._raw.get(cell)
        if raw is None:
            return None
        if cell not in self._ast:
            try:
                return Decimal(raw)
            except InvalidOperation:
                return raw
        return evaluate(self._ast[cell], self)


@singledispatch
def evaluate(node: object, sheet: Sheet) -> Value:
    raise TypeError(type(node))

@evaluate.register
def _(node: Num, sheet: Sheet) -> Value:
    return node.value

@evaluate.register
def _(node: Ref, sheet: Sheet) -> Value:
    value = sheet.get(node.cell)
    return Decimal(0) if value is None else value

@evaluate.register
def _(node: BinOp, sheet: Sheet) -> Value:
    left, right = evaluate(node.left, sheet), evaluate(node.right, sheet)
    for side in (left, right):
        if isinstance(side, CellError):
            return side                                     # errors propagate as values
        if not isinstance(side, Decimal):
            return CellError("#VALUE!")
    if node.op == "/":
        return left / right if right != 0 else CellError("#DIV/0!")
    return {"+": left + right, "-": left - right, "*": left * right}[node.op]

@evaluate.register
def _(node: Call, sheet: Sheet) -> Value:
    if node.name != "SUM":
        return CellError("#NAME?")
    total = Decimal(0)
    for arg in node.args:
        cells = expand_range(arg.start, arg.end) if isinstance(arg, Range) else None
        values = [sheet.get(c) for c in cells] if cells else [evaluate(arg, sheet)]
        for v in values:
            if isinstance(v, CellError):
                return v
            if isinstance(v, Decimal):
                total += v
    return total


if __name__ == "__main__":
    s = Sheet()
    s.set("A1", "10"); s.set("A2", "20"); s.set("A3", "=A1+A2"); s.set("B1", "=SUM(A1:A3)*2")
    print(s.get("A3"), s.get("B1"))                     # 30 120
    s.set("A1", "5"); print(s.get("A3"), s.get("B1"))   # 25 100  (incremental)
    s.set("C1", "=A1/0"); print(s.get("C1"))            # CellError('#DIV/0!')
    try:
        s.set("A1", "=B1")
    except CycleError as err:
        print("refused:", err); print(s.get("A1"))     # 5 — the old value survived the rollback
```

- **Python internals:** A regex with *named alternatives* and `m.lastgroup` is a compact tokenizer; the `"".join(...) != text` check makes sure nothing was silently skipped by `\s*`. Recursive descent maps grammar rules to methods one-to-one — the precedence of `*` over `+` is encoded by `_term` being *inside* `_expr`. Kahn's algorithm with a `deque` gives O(V + E) topological order; restricting `indegree` to `deps & affected` is what makes recalculation *incremental*. `singledispatch` visitors on frozen AST dataclasses (Day 36) keep evaluation and dependency-extraction separate from the tree. Note the rollback in `set`: installing edges, testing for a cycle, and *reverting on failure* is a small transaction (Day 54) over the graph.
- **Build & drill:** Add `AVERAGE`, `MIN`, `MAX`, `IF`; multi-letter columns (`AA1`); and `#REF!` on deleting a referenced cell. Wrap `set` in Day 33's Command for undo/redo. Implement the *dirty-flag hybrid* (C) and compare performance for 10k random sets then one read. Add an Observer so a UI can subscribe to "cell X changed." Then write down the connection to build systems and reactive frameworks in five lines.
- **Recall:** Eager vs lazy vs dirty — which requirement decided it? How do you detect a cycle *before* committing a change? Why do errors propagate as values? Why is Kahn's algorithm restricted to the affected subgraph?

### Day 56 — Chat Application (WhatsApp): conversations, ordering, delivery states, presence, and groups

- **Concept & why it matters:** Messaging combines the Mediator (Day 37 — participants talk to the service, never to each other), Observer (push to online devices), and *per-conversation ordering* (a monotonically increasing sequence number issued by the conversation, not by the client's clock). New forces: **delivery state per recipient** (`SENT → DELIVERED → READ` for *each* member of a group — a matrix, not a scalar), **offline queuing** (undelivered messages stored until the recipient connects), **idempotent send** (the client retries with a `client_message_id`), **presence** (online/offline/last-seen, and who may see it), **group membership changes** that affect who receives what from *when*, and **read receipts** as their own events. Also honesty about end-to-end encryption: the server holds ciphertext; the design must not depend on reading content.
- **Real-world case study:** **WhatsApp** (originally Erlang, ~50 engineers for 900M users) is the canonical proof that the *message routing core* is small when designed right: per-user queues, acks at each hop (server-received, delivered, read — the one/two/blue ticks are exactly today's state machine), and store-until-delivered. **Slack's** "channel-scoped sequence numbers" and **Discord's** snowflake ids solve ordering the same way this design does. Their well-known bug class: messages appearing out of order across devices when ordering relied on client timestamps.
- **Design problem (method first):** Requirements: 1:1 and group conversations; send a message (idempotent by client id); server assigns per-conversation sequence numbers; online recipients receive a push; offline recipients get messages on reconnect in order; per-recipient delivered/read receipts; presence and last-seen; group add/remove members; history fetch since a sequence number. Key fork — *who owns ordering and delivery state?* **(A) the client timestamps messages; the server broadcasts** — out-of-order on clock skew; no server-side truth. **(B) each `Conversation` assigns the next sequence number under its lock and stores the message; a `DeliveryTracker` per message holds per-recipient states; a `ChatService` mediates, pushing to `OnlineRegistry` connections and leaving the rest for pull-on-reconnect via `history_since`** — one owner for ordering (the conversation), one for per-recipient state (the tracker), and the service coordinates. **(C) per-user inbox queues (push model, like Day 53's feed)** — good for offline delivery; ordering across group members is then per-inbox, still driven by the conversation's sequence.
- **Thought process → decision:** **B** with the reconnect path implemented as *pull since last seen sequence* (simplest correct offline story; a per-user inbox is the optimization). Idempotency via `(sender, client_message_id)` stored on the conversation. Presence is a separate small aggregate (`PresenceService`) with privacy as a strategy (everyone / contacts / nobody). Membership changes are *messages too* (system messages with a sequence number) so "who was in the group at sequence N" is derivable — event sourcing's gift again (Day 48).
- **Code (core):**

```python
"""Day 56 — chat core: conversation-owned sequencing, per-recipient delivery states, push + pull-on-reconnect."""

from __future__ import annotations

import threading
from collections.abc import Callable
from dataclasses import dataclass, field
from datetime import datetime, timezone
from enum import Enum, auto
from uuid import UUID, uuid4


class ChatError(Exception): ...
class NotAMemberError(ChatError): ...


class DeliveryState(Enum):
    SENT = auto()          # accepted by the server
    DELIVERED = auto()     # reached the recipient's device
    READ = auto()

    def can_advance_to(self, target: DeliveryState) -> bool:
        return target.value == self.value + 1


@dataclass(frozen=True)
class Message:
    conversation_id: UUID
    sequence: int
    sender: str
    body: bytes                            # opaque: the server never needs to read it (E2E-friendly)
    client_message_id: str
    sent_at: datetime
    system: bool = False
    id: UUID = field(default_factory=uuid4)


class Conversation:
    """Owns ordering (sequence numbers), membership, history, idempotency, and per-recipient delivery state."""

    def __init__(self, members: set[str], *, is_group: bool) -> None:
        if len(members) < 2:
            raise ChatError("a conversation needs at least two members")
        if not is_group and len(members) != 2:
            raise ChatError("a direct conversation has exactly two members")
        self.id = uuid4()
        self.is_group = is_group
        self._members = set(members)
        self._messages: list[Message] = []                    # index == sequence - 1
        self._by_client_id: dict[tuple[str, str], Message] = {}
        self._delivery: dict[UUID, dict[str, DeliveryState]] = {}
        self._lock = threading.Lock()

    @property
    def members(self) -> frozenset[str]:
        with self._lock:
            return frozenset(self._members)

    def append(self, sender: str, body: bytes, client_message_id: str, *, system: bool = False) -> tuple[Message, bool]:
        """Returns (message, is_new). Idempotent on (sender, client_message_id)."""
        with self._lock:
            if not system and sender not in self._members:
                raise NotAMemberError(f"{sender} is not in this conversation")
            existing = self._by_client_id.get((sender, client_message_id))
            if existing is not None:
                return existing, False
            message = Message(self.id, len(self._messages) + 1, sender, body, client_message_id, datetime.now(timezone.utc), system)
            self._messages.append(message)
            self._by_client_id[(sender, client_message_id)] = message
            self._delivery[message.id] = {m: DeliveryState.SENT for m in self._members if m != sender}
            return message, True

    def history_since(self, sequence: int, limit: int = 100) -> list[Message]:
        with self._lock:
            return self._messages[sequence: sequence + limit]

    def mark(self, message_id: UUID, recipient: str, state: DeliveryState) -> bool:
        """Monotonic per recipient: SENT → DELIVERED → READ; anything else is ignored (idempotent receipts)."""
        with self._lock:
            states = self._delivery.get(message_id)
            if states is None or recipient not in states:
                return False
            if states[recipient].can_advance_to(state):
                states[recipient] = state
                return True
            return False

    def delivery_summary(self, message_id: UUID) -> DeliveryState:
        """Group semantics: the message's state is the *minimum* across recipients (blue ticks only when all read)."""
        with self._lock:
            states = self._delivery[message_id].values()
            return min(states, key=lambda s: s.value)

    def add_member(self, actor: str, new_member: str) -> Message:
        with self._lock:
            if actor not in self._members:
                raise NotAMemberError(actor)
            if not self.is_group:
                raise ChatError("cannot add members to a direct conversation")
            self._members.add(new_member)
        message, _ = self.append(actor, f"{actor} added {new_member}".encode(), f"sys-add-{new_member}-{uuid4()}", system=True)
        return message


class OnlineRegistry:
    """Who is connected, and how to push to them. Presence lives here."""

    def __init__(self) -> None:
        self._push: dict[str, Callable[[Message], None]] = {}
        self._last_seen: dict[str, datetime] = {}
        self._lock = threading.Lock()

    def connect(self, user: str, push: Callable[[Message], None]) -> None:
        with self._lock:
            self._push[user] = push

    def disconnect(self, user: str) -> None:
        with self._lock:
            self._push.pop(user, None)
            self._last_seen[user] = datetime.now(timezone.utc)

    def push(self, user: str, message: Message) -> bool:
        with self._lock:
            handler = self._push.get(user)
        if handler is None:
            return False
        try:
            handler(message)
            return True
        except Exception:                                    # noqa: BLE001 — a dead connection must not break the send
            self.disconnect(user)
            return False

    def presence(self, user: str) -> str:
        with self._lock:
            if user in self._push:
                return "online"
            seen = self._last_seen.get(user)
            return f"last seen {seen:%H:%M}" if seen else "offline"


class ChatService:
    """Mediator: clients talk to the service; the service routes."""

    def __init__(self, online: OnlineRegistry) -> None:
        self._online = online
        self._conversations: dict[UUID, Conversation] = {}

    def create(self, members: set[str], *, is_group: bool = False) -> Conversation:
        conversation = Conversation(members, is_group=is_group)
        self._conversations[conversation.id] = conversation
        return conversation

    def send(self, conversation_id: UUID, sender: str, body: bytes, client_message_id: str) -> Message:
        conversation = self._conversations[conversation_id]
        message, is_new = conversation.append(sender, body, client_message_id)
        if is_new:
            for member in conversation.members - {sender}:
                if self._online.push(member, message):        # offline members will pull via history_since
                    conversation.mark(message.id, member, DeliveryState.DELIVERED)
        return message

    def acknowledge_read(self, conversation_id: UUID, message_id: UUID, reader: str) -> None:
        conversation = self._conversations[conversation_id]
        conversation.mark(message_id, reader, DeliveryState.DELIVERED)   # a read implies delivered; no-op if already
        conversation.mark(message_id, reader, DeliveryState.READ)

    def sync(self, conversation_id: UUID, user: str, last_sequence: int) -> list[Message]:
        conversation = self._conversations[conversation_id]
        messages = conversation.history_since(last_sequence)
        for message in messages:
            conversation.mark(message.id, user, DeliveryState.DELIVERED)
        return messages
```

- **Python internals:** The `bytes` body makes the "server never reads content" property explicit in the type — encryption becomes a client concern and the design doesn't depend on decoding. Sequence = `len(self._messages) + 1` under the conversation lock is the simplest monotonic counter with no gaps; `history_since(seq)` is a slice because index == sequence − 1 — a representation choice that makes sync O(k). The delivery matrix is `dict[message_id, dict[recipient, state]]`; the *summary* as `min` over an ordered `Enum` gives group semantics in one expression. Pushing happens *outside* the conversation lock (the service iterates a frozen `members` snapshot) — never call unknown code (a client's push handler) while holding a lock (Day 27). `can_advance_to` encodes the monotonic state machine in one line and makes duplicate receipts harmless.
- **Build & drill:** Add per-user *inbox queues* for offline delivery and compare with pull-on-reconnect. Add typing indicators (ephemeral — not stored) and presence privacy as a strategy. Add `remove_member` and derive "members at sequence N" from system messages. Model message *edits/deletes* as new events referencing the original (never mutate history). Run a concurrency test: 50 threads sending to one group → sequences 1..50 with no gaps or duplicates. Draw the sequence diagram for send → deliver → read with one offline member.
- **Recall:** Why does the conversation assign sequence numbers? Why is delivery state a matrix, and how is the group summary computed? How is idempotent send achieved? Why push outside the lock?

### Day 57 — Consolidation IV: three rapid designs and the problems checkpoint

- **Do:** teach-backs for Days 42–56 — for each system, in 90 seconds: the aggregate that owns consistency, the strategy that varies, the state machine, the concurrency unit, and the one decision you'd defend hardest. Rebuild the "force → pattern" matrix from memory, then extend it with a "system → forces" column for all fifteen systems.
- **Three rapid designs (45 minutes each, paper + skeleton code with signatures and the core method bodies; no full implementation):**
  1. **Airline reservation** — flights, fare classes with inventory per class (nested availability: a Y seat can be sold as Y or as part of a higher class's allotment), seat selection (Day 49's holds), overbooking policy (Day 52's strategy), PNR as the aggregate, cancellation/refund rules by fare class (Chain or table?), and a waitlist (Observer: notify on seat release).
  2. **Cricket scoreboard (Cricbuzz)** — match, innings, over, ball as a composite; every ball an *event* (event sourcing: the scorecard is a fold); strike rotation and bowler change as a state machine; commentary as an Observer; run-rate/required-rate as derived values; the "what happens if the scorer corrects a ball?" question (a correction event, never an edit — Day 48).
  3. **Warehouse inventory management** — SKUs, locations (Composite: warehouse → zone → shelf → bin), stock movements as a *ledger* (every move is a double-entry between locations, Day 48), reservations (Day 51), cycle counts producing *adjustment* entries, reorder rules as a Chain or Strategy, and a pick-list optimizer as a Strategy (nearest-neighbour vs zone-based).
- **Grade each with the rubric:** one owner per invariant? every varying concern a strategy/registry? lifecycle drawn as a state machine? concurrency unit named and justified? extension test passed for two plausible changes? errors designed as a hierarchy? If any design fails two of these, redo it tomorrow morning before Day 58.
- **Checkpoint self-test (blank page, 45 minutes, unseen problem):** design and skeleton-code an **Online Auction (eBay-style)**: listings with start/end times, bids that must exceed the current high bid by an increment, proxy (automatic maximum) bidding, anti-sniping extension, concurrent bids, winner determination at close, payment hold, notifications. Grade yourself as above. You should now finish with time to spare and with *at least two rejected alternatives written down* for the core decisions (where does the bid-validation invariant live? how is the close triggered — scheduler or lazy?). If you can, you are ready to work without this plan's walkthroughs.

---

# PHASE 5 — Independence & Capstone (Days 58–60)

The plan has been handing you walkthroughs. These three days remove them. The goal is not another system — it's proof that you can produce the walkthrough *yourself*, under time, and defend it.

### Day 58 — The LLD interview method under pressure: pacing, communication, follow-ups, and three mocks

- **Concept & why it matters:** Interviews (and design reviews at work) test the method under constraints: 45–60 minutes, a vague prompt, an evaluator who probes. What they grade: **requirements discipline** (you asked before you drew), **entity/ownership clarity** (one owner per invariant), **API precision** (signatures with types, not hand-waving), **justified patterns** (a force named for each), **extension test** (you volunteered "if X changes, only Y changes"), **concurrency awareness** (you named the unit of consistency and the lock/transaction), **error design**, and **communication** (you narrated tradeoffs, and you *wrote down rejected alternatives*). What fails candidates: coding before requirements, `isinstance` chains, God classes, Singleton reflexes, "the GIL makes it thread-safe," and silence while thinking.
- **The 45-minute clock (rehearse it until it's automatic):**
  - **0–5 min — requirements:** restate the problem in one sentence; list 5–8 functional requirements as bullets; ask 3 clarifying questions (scale? concurrency? persistence? which operations must be fast? what's out of scope?); write "out of scope" explicitly.
  - **5–10 — entities:** nouns → entities / value objects / enums; mark the aggregate that owns consistency.
  - **10–15 — relationships + APIs:** draw the class diagram; write the 4–6 public method signatures with types.
  - **15–30 — core logic:** code (or pseudo-code, if asked for design only) the aggregate's key methods and one strategy; keep talking: "this belongs here because…"
  - **30–38 — walk-through + stress:** happy path, one failure, one concurrent scenario, the extension test. Fix what breaks.
  - **38–45 — follow-ups:** they will ask 2–3 of the standard ones below. Leave time for them.
- **Standard follow-ups and what a strong answer contains:**
  - *"How would you make it thread-safe?"* — name the unit of consistency (the aggregate); one lock (or transaction) around check-then-act; why finer locks aren't worth it yet; immutable messages; never hold a lock across I/O (Days 27, 49).
  - *"How would you persist this?"* — Repository ports per aggregate, a Unit of Work as the transaction boundary, in-memory fakes for tests (Day 38); where the schema's constraints back your invariants.
  - *"How would you add X?"* — the extension test: name the one class/registration that changes; if more changes, say so and propose the refactor (Strategy/registry/State).
  - *"Why not a Singleton / why not inheritance here?"* — force-based answer: "one instance is a wiring decision" (Day 30); "the variation is combinational, so composition" (Day 17).
  - *"What breaks at scale?"* — the O(n) scan you'd index (Day 6), the lock that'd contend (shard by aggregate), the derived value that'd drift (recompute from a log, Day 48), the in-process limiter/cache that's per-instance (Day 46).
  - *"What did you consider and reject?"* — you should have two written down already.
- **Three mock problems (45 minutes each, blank page, timer on; then grade with the rubric below):**
  1. **Digital wallet / UPI-style payments** — wallet balances (ledger, Day 48), P2P transfer as a two-account atomic operation (lock order, Day 27), idempotent transfer requests (Day 51), transaction states (INITIATED → PENDING_BANK → SUCCESS/FAILED/REVERSED — Day 34), daily limits (rate limiting by amount, Day 46), and a reconciliation report. *Key fork:* balances as counters vs derived from the ledger; the strong answer chooses the ledger and explains reversal as a new entry.
  2. **Learning management system** — courses, modules, lessons (Composite, Day 36), enrolments with prerequisites (a DAG; cycle detection, Day 55), progress tracking (state per lesson per learner — a matrix like Day 56's delivery states), quizzes with grading strategies, certificates on completion (Observer). *Key fork:* progress as stored percentage vs derived from lesson-completion events.
  3. **Snake game (real-time)** — board, snake as a deque of cells (O(1) head insert / tail pop), food spawner strategy, collision rules, a tick loop (Day 43), input queue decoupled from the tick (producer–consumer, Day 39), speed levels (State or table), replay from a recorded input log (Command, Day 33). *Key fork:* immutable game-state snapshots per tick vs in-place mutation with an undo/replay log.
- **The rubric (score each 0–2; 12+ of 16 means interview-ready):** requirements & scope stated · one owner per invariant · APIs typed before bodies · lifecycle drawn as a state machine where one exists · each pattern tied to a named force · extension test passed for two changes · concurrency unit named and defended · rejected alternatives written down.
- **Build & drill:** record yourself (audio) doing one mock; listen back for silence longer than 20 seconds and for any decision you made without saying *why*. Redo that mock the next morning. Write your personal "clarifying questions" card (8 questions) and your "follow-ups" card, and keep both for Day 59.
- **Recall:** Recite the 45-minute clock. Give the strong answer to "how would you make it thread-safe?" in four sentences. Name the eight rubric criteria.

### Day 59 — Capstone I: design document and core build

- **Choose one** (all three exercise every phase; pick the domain you can explain to a friend):
  1. **Food-delivery platform** — restaurants and menus (Composite), carts and orders with a lifecycle (Day 51), partner matching with a spatial index (Day 50), delivery tracking (State + Observer), payments with idempotency (Day 51), ratings, promotions as pricing strategies (Day 31), restaurant-side order acceptance timeouts (Day 39).
  2. **Hospital management** — patients, doctors with schedules (Day 52 intervals + recurrence), appointments with holds (Day 49), admissions and bed allocation (Day 42 allocation strategy), prescriptions with interaction checks (Chain), billing on a ledger (Day 48), notifications, role-based permissions (Flag enums + a policy chain).
  3. **Multi-tenant appointment-booking SaaS (Calendly-like)** — tenants and users, availability rules and recurrences (Day 52), booking with conflict detection and holds (Days 49/52), buffer times and time zones, notifications and reminders (Day 39 scheduler), webhooks to customers (Observer + retry decorators), per-tenant rate limits (Day 46), and a plugin registry for calendar integrations (Adapter + registry).
- **Deliverable 1 — the design document (write it *before* coding; 2–4 pages):** one-paragraph problem statement; numbered functional requirements; non-functional requirements (concurrency, persistence, expected scale, latency-sensitive operations); *out of scope*; the **class diagram** (aggregates marked, ownership arrows); **state diagrams** for every entity with a lifecycle; **sequence diagrams** for the three most important flows; the **concurrency model** (units of consistency, what locks/transactions exist, what's immutable); the **error hierarchy**; the **extension tests** you'll run ("add a payment method," "add a new tenant plan," …) and which classes they touch; and a **decision log** — at least six decisions, each with the alternatives considered and the reason for the choice.
- **Deliverable 2 — the core build (today):** the domain model (entities, value objects, enums, invariants enforced in one place each), the aggregates with their key methods, the strategies/registries the design calls for, the service layer with Unit of Work and in-memory repositories (Day 38), the event bus for notifications (Day 32). Tests for every invariant and every state transition. `mypy --strict` clean. Commit at every green step.
- **Method reminder:** run the six steps on paper for the *whole* system, then again for each aggregate. Where you get stuck, apply the rubric question that fits: *who owns this state? what varies? what must never be inconsistent?* If a class exceeds ~200 lines or has more than one reason to change, split it (Day 21). Keep the pure domain free of I/O.

### Day 60 — Capstone II: hardening, audit, retrospective, and what comes next

- **Harden:** concurrency tests for every aggregate (threads racing on the same booking/order/seat — exactly one wins); idempotency tests for every externally retried operation; a hold/reservation expiry test with a fake clock; a "kill it mid-flow" test (an exception between two steps) proving compensation restores consistency; a SQLite Unit of Work behind the same service tests (Day 38) to prove the domain didn't leak.
- **Pattern audit (Day 40):** for every class, one line naming its force. Delete anything without one. Confirm no Singleton in application code, no `isinstance` chains for behaviour, no mutable shared state without an owner and a lock, no float money, no string statuses, no `except: pass`.
- **Document (the portfolio):** a `README` with the design doc, the UML, the decision log, the *pattern → force → class* table, how to run the tests, and a short "what I would do differently" section. This repository plus your 60 days of commits is your proof of skill.
- **The final synthesis (from memory, aloud, 5 minutes):** explain the course's thesis — *LLD is deciding which object owns which state and behaviour so that the thing most likely to change is the cheapest thing to change* — using your capstone as the example: name three invariants and their single owners, three things that vary and the strategy/registry that isolates each, two lifecycles and their state machines, the unit of consistency and how you protect it, and the two decisions you'd defend hardest with the alternatives you rejected. If you can do this without notes, you have reached the goal this plan set out: you can design and code a system in Python and *reason* about it, not copy it.
- **Retrospective (write it down):** which days were hardest and why; which pattern you over-used and which you under-used; which internals block changed how you code; what your next 30 days should target.
- **What comes next (beyond LLD):**
  - **High-level / distributed design** — the *backend + agentic* study plan in this repository picks up exactly here: networking, databases, caching, queues, consistency, and production engineering are the layer above the objects you now design.
  - **Domain-Driven Design** — Evans' *Domain-Driven Design* and Vernon's *Implementing DDD*: aggregates, bounded contexts, and domain events formalize what Days 26, 38, and 48 started.
  - **Asynchronous Python** — `asyncio` in depth, once your systems talk to networks (Day 39 gave you the model).
  - **Databases** — indexing, transactions, isolation levels: the persistence side of every Unit of Work you wrote.
  - **Read real code** — python-chess (Day 44's design at production quality), the `logging` module (Day 47), cosmicpython's example repository (Day 38), SQLAlchemy's query builder (Day 29), `functools.lru_cache`'s C source (Day 45). Reading well-designed code after designing the same thing yourself is the fastest way to calibrate.

---

# The master mapping table (the real point of Phase 4)

There are far fewer *distinct decisions* than there are systems. Know this table cold; for an unseen problem, fill in its row first.

| System (day) | Aggregate that owns consistency | What varies → strategy/registry | Lifecycle (state machine) | Unit of concurrency | Signature patterns |
|---|---|---|---|---|---|
| Parking Lot (42) | `ParkingLot` (spots + tickets) | allocation, fee | ticket (in → out) | the lot (coarse lock) | Strategy, Factory/registry, value objects |
| Elevator (43) | `Elevator` (its stops) | dispatch algorithm | car (idle/moving/doors) | tick loop (no locks) | State/enum, Strategy, Observer |
| Chess (44) | `Board` via `Game` | piece movement (polymorphism) | game (progress/check/mate) | single-threaded | Polymorphism, Command/Memento (apply/undo) |
| Cache (45) | `Cache` | eviction policy | entry (fresh/expired) | one `RLock` | Strategy, lazy TTL |
| Rate limiter (46) | `RateLimiter` per-client state | algorithm | — | one lock (shardable) | Strategy, injected clock |
| Message queue (47) | `TopicLog` + `GroupState` | — | message (in-flight/acked) | broker lock | Append-only log, offsets, at-least-once |
| Splitwise/Ledger (48) | `Ledger` (entries) | split strategy | — | single-writer ledger | Value objects, double-entry invariant, event sourcing |
| Ticket booking (49) | `Show` (seats + holds) | refund policy | hold → confirmed/expired | the show (lock per show) | Timed holds, idempotent confirm, compensation |
| Ride sharing (50) | `DriverRegistry` (availability + index) | matching, pricing | trip | registry lock (`try_reserve`) | Spatial grid index, Strategy, State table |
| E-commerce (51) | `Inventory` (reservations) | payment gateway | order | inventory lock; idempotency key claim | Reservation, idempotency, compensation, events |
| Scheduler/Calendar (52) | `Room` (sorted intervals) | overbooking/recurrence policy | booking | per room | Half-open intervals, bisect, all-or-nothing series |
| Social feed (53) | `SocialGraph` + `PostStore` | fan-out strategy | — | per structure locks | Push/pull/hybrid, k-way merge |
| KV store (54) | `KeyValueStore` | — | transaction (open/committed) | one `RLock` | Undo log, tombstones, lazy-deletion heap |
| Spreadsheet (55) | `Sheet` (cells + DAG) | functions | cell (value/error) | single-threaded | Parser, Visitor, topological recalculation |
| Chat (56) | `Conversation` (sequence + delivery) | presence privacy | delivery (sent/delivered/read) | per conversation | Mediator, Observer, idempotent send |

**The seven questions that fill any row:** What is the one operation this system exists to do? What must never be observed inconsistent, and which object owns it? What is most likely to change, and is it a strategy, a registry, or data? Which entities have a lifecycle, and what are the legal transitions? What is the unit of concurrency, and how is check-then-act protected? Which values must be exact, immutable, or derived rather than stored? What does "add X" touch?

---

# Problem bank (solve after Day 57; each with its core fork)

Run the six-step method and the rubric on each. Aim for one every two or three days after the plan ends.

1. **ATM** — card/PIN/session state machine; cash dispensing as change-making (Day 25); transaction log. *Fork:* where does the daily-limit rule live?
2. **Coffee/vending with recipes** — recipes as data; ingredient inventory; concurrent dispensers. *Fork:* reserve ingredients before or during brewing?
3. **Snake & Ladder / Ludo** — board as data; dice strategy; turn order. *Fork:* rules engine per game vs a generic board-game framework.
4. **Tic-Tac-Toe n×n, k-in-a-row** — O(1) win detection with counters. *Fork:* immutable boards vs mutation + undo.
5. **Sudoku validator/solver** — constraint sets; backtracking as a strategy. *Fork:* validation as Visitor over regions.
6. **Online judge (LeetCode)** — submissions, sandboxed execution (a port), verdict state machine, leaderboards (Day 5 top-k). *Fork:* synchronous vs queued execution.
7. **Pub/Sub library (in-process)** — Day 32 generalized with topics, wildcards, and delivery guarantees. *Fork:* sync vs async dispatch per subscriber.
8. **Notification service** — templates, channels, preferences, quiet hours, dedup, retries (Day 41 checkpoint). *Fork:* rules as chain vs data.
9. **Logging framework** — Day 47 drill in full. *Fork:* propagation vs explicit handler lists.
10. **URL shortener (LLD scope)** — id generation strategies (counter/base62 vs hash), collision policy, expiry. *Fork:* who owns uniqueness?
11. **Task manager (Jira-lite)** — issues, workflows as configurable state machines per project, assignments, comments. *Fork:* hard-coded lifecycle vs per-project transition tables.
12. **Airline reservation** — Day 57's rapid design, fully coded.
13. **Cricket scoreboard** — Day 57's rapid design, fully coded.
14. **Warehouse inventory** — Day 57's rapid design, fully coded.
15. **Online auction** — Day 57's checkpoint, fully coded.
16. **Restaurant table reservation + waitlist** — capacity by table size, time slots (Day 52), holds (Day 49), waitlist Observer. *Fork:* table assignment at booking or arrival?
17. **Car rental** — vehicles, branches, reservations by date range, pricing with insurance add-ons (Decorator). *Fork:* Decorator vs a list of add-on strategies for pricing.
18. **Library v2 with reservations & fines** — Day 20 + queues + ledger (Day 48). *Fork:* fines as derived from loan events.
19. **Music/video streaming playlist & playback** — playlists (Composite), playback state machine, shuffle/repeat strategies, resume position. *Fork:* Observer for "now playing" vs polling.
20. **Learning management system** — Day 58 mock, fully coded.
21. **Digital wallet / UPI** — Day 58 mock, fully coded.
22. **Stock exchange order book** — limit/market orders, price-time priority matching (heaps per side), partial fills, cancel. *Fork:* matching as a strategy; the book as the aggregate under one lock.
23. **Distributed-lock-free-ish local file sync (Dropbox-lite, single machine)** — file tree (Composite), change detection, conflict policy. *Fork:* event log vs snapshot diff.
24. **Text editor with collaborative cursors (single process)** — Day 33 + per-user cursors + Observer. *Fork:* operation log vs snapshots for history.
25. **Traffic signal controller for an intersection** — phases as State, timing plans as strategies, pedestrian requests as events, emergency preemption. *Fork:* fixed-time table vs adaptive strategy.

---

# Self-assessment rubric (use it on every design, forever)

| Criterion | 0 — missing | 1 — partial | 2 — solid |
|---|---|---|---|
| Requirements & scope | coded immediately | listed some; no out-of-scope | numbered FRs, NFRs, explicit out-of-scope, questions asked |
| Ownership of invariants | state scattered; check-then-act in callers | mostly one owner; some leaks | every invariant enforced in exactly one method of its owner |
| APIs before bodies | none written | some signatures | typed signatures for every public operation before implementation |
| Lifecycles | statuses as strings/ifs | enum, transitions implicit | state diagram drawn; illegal transitions impossible |
| Patterns tied to forces | pattern fever or none | patterns used, forces unstated | each pattern named with the force; alternatives rejected in writing |
| Extension test | not attempted | one change considered | two plausible changes traced to one class each |
| Concurrency | "the GIL handles it" | a lock somewhere | unit of consistency named; check-then-act protected; no lock across I/O |
| Errors & values | bare exceptions, float money, `except: pass` | some hierarchy | domain exception hierarchy; exact/immutable value objects; errors as contract |

---

# Final word

This is a **60-day sequence, not a 60-day deadline.** If a day needs two, take two — the ordering is the value, not the calendar. But never skip the four things that turn reading into skill: **the paper design before the code**, **the tests**, **the internals block**, and **the teach-back**. Notes you can recognize are worthless; systems you can design from a blank page and defend line by line are the entire point.

You asked to be "perfect at LLD." Perfection isn't a state; it's the reflex of asking the right questions — *who owns this? what varies? what must never be inconsistent? what does change cost?* — and having enough reps that the answers arrive as design rather than as recall. By Day 60 you will have run those questions across sixty days, twenty systems, twenty-three patterns, and a capstone you built yourself. That is what "capable of LLD" means in practice, and it is exactly what this plan was built to make true.
