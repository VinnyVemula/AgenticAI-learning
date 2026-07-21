# Day P1 — Python Fundamentals I: Values, Control Flow, Functions

> **What this covers:** the smallest set of Python you need to write and run real programs — variables, the core types (`int`, `float`, `str`, `bool`, `None`), operators and truthiness, control flow (`if`/`elif`/`else`, `for`, `while`, `break`/`continue`), functions (`def`, arguments, defaults, keyword arguments, `return`, scope), f-strings, and how to read a traceback.
> **Who it's for:** someone who has never written Python. Zero assumed knowledge.
> **The ONE idea that unites the backend and agentic layers:** *an "AI agent" is not magic — it is a `while` loop, an `if`/`elif` chain, a few function calls, and a list that grows. Every construct you learn today IS the control plane of every agent you will ever build.*
>
> **How to read this:** every concept runs the ladder **intuition → analogy (with where it breaks) → concrete worked example → visual → under-the-hood**. Anything you can run has a `### Runnable example` block: paste it, run it, read the line-by-line explanation. When something feels abstract, jump to its example.
>
> **Depth tiers:** **[CORE]** open every box · **[WORKING]** use it, know the tradeoffs · **[AWARE]** know it exists.
>
> *(P1–P4 are prerequisite days: per the course rules they omit the `### System design`, `### Case studies`, and `### In production` blocks — those begin at Day 1. Everything else runs at full depth.)*

---

# PART 1 — BACKEND: The Python You Will Use Every Single Day

## Overview & motivation — why a "programming language" at all

**The problem before programming languages.** A computer's processor understands only *machine code* — raw numbers that mean "add these two registers," "jump to this address." Humans wrote programs that way in the 1940s–50s, and it was so slow and error-prone that a single program took months. The fix, invented in stages (assembly → FORTRAN → C → scripting languages), was to let humans write instructions in something closer to English and have a program translate it for the machine.

**Python's specific deal.** Python (released 1991, Guido van Rossum) is an *interpreted* language: a program called the **interpreter** (`python`) reads your text file top-to-bottom and executes it line by line. You don't compile a binary first; you just run the file. That's why Python won for scripting, data work, backend services, and — the reason this course exists — nearly all AI/LLM tooling. The Anthropic SDK, FastAPI, Pydantic: all Python.

**What "running a program" means, concretely:**

```
you write text into a file        prog.py
        │
        ▼
you invoke the interpreter        $ python prog.py
        │
        ▼
interpreter parses the text       (syntax errors caught HERE, before anything runs)
        │
        ▼
interpreter executes statements   top to bottom, one at a time
        │
        ▼
program prints / returns / crashes with a traceback
```

Everything in this note happens inside step "executes statements." Let's open that box.

---

## 1. Variables, expressions, statements **[CORE]**

**Depth: [CORE]** — this is the atom of all code; everything else is built from it.

### Intuition — why variables exist

A program computes values: numbers, pieces of text, yes/no answers. The moment you compute something you almost always need to *use it again later* — and re-computing it every time would be wasteful and unreadable. So every language gives you a way to **give a value a name**. Before named variables existed (raw machine code), programmers referred to values by *memory address* — literally "the number stored at slot 0x4F3A" — which is why a single typo could corrupt an unrelated part of the program. Variables solve that: humans handle names, the interpreter handles addresses.

Three words you must be able to define precisely, because every error message uses them:

- A **value** is a piece of data that exists in memory while the program runs: `42`, `3.14`, `"hello"`, `True`, `None`.
- An **expression** is any fragment of code that *evaluates to a value*: `2 + 3`, `price * 1.18`, `name.upper()`, even just `42`. Expressions can nest: `(2 + 3) * (10 - 4)`.
- A **statement** is a complete instruction — a "sentence" of the program: an assignment (`x = 5`), an `if`, a `for`, a `return`. Statements *do* something; expressions *are* something. `x = 2 + 3` is one statement containing one expression.

**Assignment** (`x = 5`) is the statement that binds a name to a value. Read `=` as "**make this name point at** this value" — never as mathematical equality (equality testing is `==`, section 3).

### Analogy — name tags, not boxes

A Python variable is a **name tag on a string, tied to an object** — not a box that contains the object. `x = 5` ties the tag "x" to the object `5`. `y = x` ties a *second* tag to the *same* object — nothing is copied. Reassigning `x = 99` moves *x's tag* to a new object; `y` still points at `5`.

*Where the analogy breaks:* real name tags can only be read by humans; in Python the tag is how the *program itself* reaches the object — an object with zero tags pointing at it becomes unreachable and Python eventually reclaims its memory (this cleanup is called **garbage collection**; treat it as a black box until much later). Also, unlike a physical tag, a Python name lives in a specific *namespace* (section 7 — scope), so two functions can each have their own `x` without collision.

### Worked example — trace the tags

Follow the interpreter through four statements, tracking which name points where:

```
statement          names → objects                what happened
─────────────────  ─────────────────────────────  ─────────────────────────────
x = 10             x → 10                         object 10 created, tag x tied
y = x              x → 10,  y → 10                SECOND tag on the SAME object
x = x + 5          x → 15,  y → 10                15 is a NEW object; x re-tied
                                                  (10 was never modified!)
z = "10"           x → 15, y → 10, z → "10"       "10" is text, a different
                                                  object entirely from 10
```

The key surprise for beginners is line 3: `x = x + 5` does **not** change the object `10`. The right side `x + 5` is an *expression* — it evaluates to a brand-new object `15` — and then the assignment moves x's tag onto it. Numbers and strings in Python are **immutable**: they never change in place; operations produce new objects.

### Visual — namespace as a lookup table

Internally, "the current set of name tags" is a table mapping names to objects:

```
      namespace (a lookup table)              objects in memory
   ┌───────────┬──────────────┐            ┌──────┐  ┌──────┐  ┌───────┐
   │ name      │ points to    │            │  15  │  │  10  │  │ "10"  │
   ├───────────┼──────────────┤            └──▲───┘  └──▲───┘  └───▲───┘
   │ x         │ ────────────────────────────┘         │           │
   │ y         │ ──────────────────────────────────────┘           │
   │ z         │ ──────────────────────────────────────────────────┘
   └───────────┴──────────────┘
```

When the interpreter sees `y` in an expression, it looks the name up in this table and substitutes the object. If the name is *not* in the table you get `NameError: name 'y' is not defined` — the single most common beginner error, and the one you'll deliberately trigger in section 8.

### Runnable example — bindings, rebinding, and proof that assignment copies nothing

```python
# names.py — no installs needed; Python 3 standard library only
x = 10
y = x                      # y now points at the SAME object as x
print(x, y)                # both names resolve to 10
print(id(x) == id(y))      # id() returns the object's unique identity number
                           # equal ids == literally the same object in memory

x = x + 5                  # right side evaluates to a NEW object 15; x re-binds
print(x, y)                # x moved; y still points at the old 10
print(id(x) == id(y))      # different objects now

greeting = "hello"
shout = greeting.upper()   # .upper() is an expression: it PRODUCES a new string
print(greeting, shout)     # the original was not modified (strings are immutable)
```

```
$ python names.py
# -> 10 10
# -> True
# -> 15 10
# -> False
# -> hello HELLO
```

**Why this works, line by line.** `x = 10` creates an integer object and binds the name `x` to it in the current namespace. `y = x` looks up `x`, gets the object, and binds a second name to *that same object* — `id()` (a built-in function returning each object's unique identity for its lifetime) proves it by printing `True` for `id(x) == id(y)`. `x = x + 5` first evaluates the expression `x + 5` — producing a *new* object `15`, because integers are immutable — then re-binds `x`; `y` is untouched, so the second `id` comparison prints `False`. Finally `greeting.upper()` shows the same immutability rule for strings: methods on immutable objects always *return new objects* rather than editing the original, which is why we must capture the result in `shout`. If you ever write `greeting.upper()` on its own line and wonder why nothing changed — this is why.

---

## 2. The core types — `int`, `float`, `str`, `bool`, `None` (and "everything is an object") **[CORE]**

**Depth: [CORE]** — you will touch these five types in every program you ever write, including every agent.

### Intuition — why types exist

`10` and `"10"` look similar but must behave differently: `10 + 5` should be `15`, while `"10" + "5"` should be the text `"105"` (gluing strings together). A **type** is the interpreter's answer to "what operations make sense on this value, and what do they mean?" Every value in Python carries its type with it at runtime — ask with the built-in `type(...)`.

What came before: in early languages (and still in C), the *variable* had the type and the raw bytes were trusted to match — get it wrong and you'd silently reinterpret garbage. Python attaches the type to the **value**, checks at runtime, and refuses nonsense with a clear error (`TypeError`) instead of corrupting data. That's the trade: a bit slower, massively safer and friendlier.

### The five you need today

| Type | What it holds | Literal examples | The one thing to remember |
|---|---|---|---|
| `int` | whole numbers | `42`, `-7`, `0` | unlimited size — no overflow, ever |
| `float` | decimal numbers | `3.14`, `-0.5`, `2.0` | stored in binary → tiny rounding errors are *normal* |
| `str` | text (Unicode) | `"hello"`, `'hi'`, `""` | immutable; `+` concatenates, `*` repeats |
| `bool` | truth values | `True`, `False` | literally a subtype of `int` (True == 1) |
| `NoneType` | "no value here" | `None` | the *deliberate absence* of a value; test with `is None` |

**Why `None` exists (term necessity).** Programs constantly need to say "there is no answer": a search found nothing, a user didn't fill a field, a function had nothing to return. Before a dedicated null object, languages used *sentinel values* — return `-1` for "not found," `0` for "empty" — and it caused decades of bugs because `-1` and `0` are also legitimate answers. `None` is a single, unmistakable object meaning *absence*, with its own type, so it can never be confused with a real result. There is exactly one `None` in the whole program (a **singleton**), which is why the idiomatic test is identity, `x is None`, not equality.

**Why `bool` is a subtype of `int` (history).** Python didn't have `True`/`False` until version 2.3 (PEP 285, 2002) — before that, people literally used `1` and `0`. For backwards compatibility, `bool` was made a subclass of `int`, so `True + True` is `2` to this day. Harmless trivia most days; occasionally the explanation for a weird bug.

### Analogy — types are like units on a measurement

`10` meters and `10` kilograms are both "10," but adding them is meaningless — the *unit* tells you what operations make sense. Python types work the same: the type travels with the value and gates the operations. *Where it breaks:* physical units convert smoothly (meters→feet); Python types convert only when *you explicitly ask* (`int("10")`, `str(10)`, `float(3)`) — with one big exception, `int` + `float` auto-promotes to `float`, because that conversion can't lose meaning.

### Worked example — mixing types, the errors and the conversions

```
expression          result        why
──────────────────  ────────────  ──────────────────────────────────────────
10 + 5              15            int + int → int
10 + 2.5            12.5          int auto-promotes to float (safe)
"10" + "5"          "105"         str + str = concatenation
"10" + 5            TypeError!    Python refuses to guess: text + number
"10" * 3            "101010"      str * int = repetition (this one is defined)
int("10") + 5       15            YOU asked for the conversion → fine
str(10) + "5"       "105"         and the other direction
int("ten")          ValueError!   conversion requested but impossible
```

Two different errors, and the difference matters: **`TypeError`** = "this operation doesn't exist between these types"; **`ValueError`** = "the operation exists, but this particular value can't do it." Tracebacks (section 8) will name which one you hit.

### Under the hood — everything is an object

This is the sentence that organizes all of Python: **every value — every int, string, function, even `None` and types themselves — is an object.** An object is a chunk of memory holding three things:

```
        a Python object (e.g. the int 42)
   ┌─────────────────────────────────────┐
   │ identity   — its address/id()       │   unique while it lives
   │ type       — pointer to `int`       │   decides available operations
   │ value/data — the payload (42)       │   plus a reference count
   └─────────────────────────────────────┘
```

Consequences you can observe today:

- Values have **methods** — functions attached to the object, called with a dot: `"hi".upper()`, `(255).bit_length()`. Even a bare integer has ~60 methods.
- `type(x)` works on *anything*, because everything carries its type: `type(42)` is `<class 'int'>`, `type(None)` is `<class 'NoneType'>`, and `type(int)` is `<class 'type'>` (yes — types are objects too).
- There is no split between "primitives" and "objects" as in Java (`int` vs `Integer`) — one uniform rule. That uniformity is *the problem this design solves*: every value can be passed to functions, stored in collections, and inspected the same way.

Two more under-the-hood facts worth knowing now: Python `int` is **arbitrary precision** — `2 ** 1000` just works, no overflow, because ints grow as many internal digits as needed (unlike C's fixed 64 bits). And `float` is an IEEE-754 double — 64 bits of *binary* fraction — which is why `0.1 + 0.2` prints `0.30000000000000004`: 0.1 has no exact binary representation, exactly as 1/3 has no exact decimal one. That's not a Python bug; every mainstream language does it. Rule of thumb: never compare floats with `==`; never use floats for money.

### Runnable example — a tour of the five types and the "everything is an object" claim

```python
# types_tour.py
print(type(42), type(3.14), type("hi"), type(True), type(None))

# int: arbitrary precision — this would overflow in C/Java
print(2 ** 100)

# float: binary representation → visible rounding
print(0.1 + 0.2)
print(0.1 + 0.2 == 0.3)          # the classic gotcha

# str: immutable, but rich in methods that return NEW strings
s = "Agentic AI"
print(s.lower(), s.replace("AI", "Engineering"), len(s))

# bool is an int in disguise
print(True + True, True == 1)

# None: singleton — always test with `is`
result = None
print(result is None, type(result))

# everything is an object: even a number literal has methods
print((255).bit_length())        # how many binary digits does 255 need?
print("42".isdigit())            # can this string convert cleanly to int?
```

```
$ python types_tour.py
# -> <class 'int'> <class 'float'> <class 'str'> <class 'bool'> <class 'NoneType'>
# -> 1267650600228229401496703205376
# -> 0.30000000000000004
# -> False
# -> agentic ai Agentic Engineering 10
# -> 2 True
# -> True <class 'NoneType'>
# -> 8
# -> True
```

**Why this works, line by line.** `type(...)` reads the type pointer inside each object and prints its class. `2 ** 100` demonstrates arbitrary-precision ints — the interpreter allocates as many internal digits as the result needs, so the 31-digit answer is exact. `0.1 + 0.2` shows IEEE-754 rounding: both operands are stored as the *nearest representable binary fraction*, and the tiny errors accumulate into the visible `...0004`; consequently `== 0.3` is `False`, which is why real code compares floats with a tolerance instead. The string methods `.lower()` and `.replace()` return brand-new strings (immutability again — compare with section 1), and `len(s)` counts characters, spaces included. `True + True` evaluates to `2` because `bool` subclasses `int`. `result is None` uses **identity** (`is`) to test for the singleton — the officially recommended style (PEP 8), because `is None` can never be fooled the way `==` can be by exotic objects. The last two lines are the punchline: `(255)` — parentheses so the dot isn't read as a decimal point — and `"42"` are plain literals, yet both answer method calls, because *every* value is a full object.

### The string toolkit — the six methods today's builds need **[WORKING]**

**Depth: [WORKING]** — strings have ~50 methods; you need six today, and the full tour waits for P2. Know the interface, use them fluently, leave the Unicode internals boxed.

**Why methods instead of standalone functions (term necessity):** operations that belong to a type live *on* the type — `text.split()` instead of a global `split(text)` — so that (a) each type can implement the operation its own way, and (b) your editor can list everything a value can do the moment you type the dot. That dot-discovery is how professionals explore unfamiliar objects.

| Method / tool | What it does | Worked micro-example |
|---|---|---|
| `s.split()` | chop into a list of words on whitespace | `"a  quick fox".split()` → `['a', 'quick', 'fox']` |
| `s.lower()` / `s.upper()` | new string, case-folded | `"The THE the".lower()` → `"the the the"` |
| `s.strip()` | new string, edge whitespace removed | `"  hi \n".strip()` → `"hi"` |
| `s.count(sub)` | how many times `sub` occurs | `"banana".count("an")` → `2` |
| `sub in s` | membership (an operator, §3) | `"fox" in "the fox ran"` → `True` |
| `len(s)` | character count (a built-in, not a method) | `len("hello ")` → `6` |

Remember the immutability rule from §1: every method above **returns a new string** — the original is never modified, so you must capture the result (`clean = raw.strip()`).

### Runnable example — the exact pipeline `wordcount.py` will use

```python
# string_toolkit.py
raw_line = "  The quick brown fox — THE fox! \n"

clean = raw_line.strip()             # trim the edges (newlines from files, spaces from users)
folded = clean.lower()               # case-fold so "The" and "THE" count as the same word
words = folded.split()               # whitespace-chop into a list of word-strings

print(f"cleaned: {clean!r}")         # !r = show it repr-style, quotes and all
print(f"words ({len(words)}): {words}")

target = "the"
hits = 0
for w in words:                      # accumulator pattern, previewing §5
    if w.strip("—!.,?") == target:   # strip punctuation from EACH word before comparing
        hits = hits + 1
print(f"{target!r} appears {hits} times")
print(f"quick check with .count(): {folded.count(target)} (counts substrings — see below)")
```

```
$ python string_toolkit.py
# -> cleaned: 'The quick brown fox — THE fox!'
# -> words (7): ['the', 'quick', 'brown', 'fox', '—', 'the', 'fox!']
# -> 'the' appears 2 times
# -> quick check with .count(): 2 (counts substrings — see below)
```

**Why this works, line by line.** `strip()` removes the leading spaces and trailing `" \n"` — file lines almost always end in `\n`, so stripping is the standard first move. `lower()` folds case so the comparison later treats `The`/`THE`/`the` as one word. `split()` with no argument splits on *any run* of whitespace — note it kept the dash as a "word" and left `fox!` glued to its punctuation, which is why the loop strips punctuation from each word (`w.strip("—!.,?")` — `strip` with an argument removes those specific characters from the edges) before the `==` test. The final line shows the tempting shortcut and its trap: `folded.count("the")` counts *substrings*, so in text containing `"them"` or `"other"` it over-counts — here it happens to agree (2), but the loop-with-`==` is the correct word-level tool. That distinction — substring vs word — is exactly the judgment call your `wordcount.py` build has to make; make it consciously.

---

## 3. Operators and truthiness **[CORE]**

**Depth: [CORE]** — operators are how expressions compute; truthiness is how Python decides every `if` and `while` you will ever write, including the guard on every agent loop.

### Intuition — operators

An **operator** is a symbol that combines values into an expression: `+`, `-`, `*`, `/`, `==`, `and`. They exist because `add(multiply(a, b), c)` is unreadable — infix symbols mirror the math notation humans already know. Each operator is, under the hood, a method call on the left object (`a + b` really runs `a.__add__(b)`), which is *why* `+` can mean addition for ints and concatenation for strings — each type defines its own `__add__`. Name that mechanism (**operator overloading**) and leave the box closed for now.

The table you must know cold:

| Category | Operators | Gotchas to burn in now |
|---|---|---|
| Arithmetic | `+  -  *  /  //  %  **` | `/` **always** returns float (`10/2` → `5.0`); `//` floors (`7//2` → `3`); `%` is remainder (`7%2` → `1`) — the heart of FizzBuzz; `**` is power |
| Comparison | `==  !=  <  <=  >  >=` | return `bool`; chain like math: `0 <= x < 10` works |
| Logical | `and  or  not` | short-circuit, and return an *operand*, not always a bool (below) |
| Identity | `is  is not` | same *object*? — only for `None` at your level |
| Membership | `in  not in` | `"z" in "pizza"` → `True`; works on strings today, lists/dicts from P2 |

**Why `%` matters today:** `x % y` is the remainder after dividing x by y. `15 % 3 == 0` means "15 is divisible by 3." That single fact *is* the FizzBuzz build at the end of this note.

### Intuition — truthiness

Every value in Python can stand in for `True` or `False` when a condition needs one. The rule is beautifully uniform: **empty or zero things are falsy; everything else is truthy.**

Falsy values (memorize — this is the complete practical list): `False`, `None`, `0`, `0.0`, `""` (empty string), and the empty collections you'll meet in P2 (`[]`, `{}`, `()`).

Why this exists (term necessity): the pattern "do something only if this string/list/result is non-empty" is so common that writing `if len(s) != 0` or `if result != None` everywhere was recognized as noise. Truthiness lets you write `if s:` and `if result:` — shorter, and by convention *clearer*, because it reads as "if there's anything there." Before Python standardized this, C code used the same trick with `0` — Python generalized it to every type via the `__bool__`/`__len__` protocol (an object is falsy if its `__bool__` returns False or its `__len__` returns 0 — that's the whole mechanism).

### Analogy — the bouncer's quick glance

A bouncer at a club door doesn't interview each guest ("state your exact age and membership number") — one glance: *is there anything here or not?* `if response:` is that glance. *Where it breaks:* the glance can't distinguish *kinds* of emptiness. If `0` is a **legitimate value** in your domain (a temperature of 0°, a count of 0 items that you still want to record), the truthiness glance wrongly treats it like "nothing." In those cases you must ask the precise question: `if x is not None:` rather than `if x:`. This exact bug — using `if x:` when `x=0` was a real answer — is a classic, so tag it now.

### Worked example — short-circuit evaluation, traced

`and` and `or` don't just return True/False — they return **one of their operands**, and they stop evaluating as soon as the answer is known (**short-circuit**):

```
expression                evaluates to    trace
────────────────────────  ──────────────  ───────────────────────────────────────
"" or "default"           "default"       "" is falsy → or moves on, returns
                                          the second operand as-is
"user input" or "default" "user input"    first is truthy → returned IMMEDIATELY,
                                          second never even evaluated
0 and 1/0                  0              0 is falsy → and stops → the 1/0
                                          crash NEVER HAPPENS (short-circuit!)
5 and "yes"                "yes"          5 truthy → and must check the second
                                          → returns it as-is
not ""                     True           not always returns a real bool
```

Two idioms fall straight out of this table, and you will see both in every real codebase:

- `name = user_input or "anonymous"` — "use the input, or a default if it's empty."
- `if items and items[0] == target:` — the truthy check *guards* the access; if `items` is empty, short-circuit means the crashing part is never evaluated.

### Runnable example — operators, truthiness, and the short-circuit guard

```python
# truthiness.py
# --- arithmetic gotchas ---
print(10 / 2, 7 // 2, 7 % 2, 2 ** 10)     # true div, floor div, remainder, power
print(15 % 3 == 0, 15 % 4 == 0)           # divisibility test — the FizzBuzz core

# --- comparison chaining ---
x = 7
print(0 <= x < 10)                        # reads like math, works like math

# --- what bool() says about each value ---
for value in [0, 1, -1, 0.0, "", "hi", None, True]:
    print(repr(value), "->", bool(value))

# --- short-circuit in action ---
name = "" or "anonymous"                  # empty → falls through to default
print(name)
crash_avoided = 0 and (1 / 0)             # 1/0 would crash — but never runs
print(crash_avoided)
```

```
$ python truthiness.py
# -> 5.0 3 1 1024
# -> True False
# -> True
# -> 0 -> False
# -> 1 -> True
# -> -1 -> True
# -> 0.0 -> False
# -> '' -> False
# -> 'hi' -> True
# -> None -> False
# -> True -> True
# -> anonymous
# -> 0
```

**Why this works, line by line.** `10 / 2` prints `5.0` — true division *always* yields float, even when it divides evenly; use `//` when you want an int. `15 % 3 == 0` is the divisibility idiom: remainder zero means "divides exactly." The chained comparison `0 <= x < 10` is evaluated as `(0 <= x) and (x < 10)` with `x` evaluated once — Python is one of the few languages where chaining works like written math. The `for` loop (formally introduced next section — here it just visits each value in the list) feeds each value to `bool(...)`, which invokes the truthiness protocol: `0`, `0.0`, `""`, and `None` come back `False`; *any* non-zero number (including `-1` — a common surprise) and any non-empty string come back `True`. `repr(value)` prints the value in its literal form (quotes around strings) so you can tell `''` from nothing. `"" or "anonymous"` returns the second operand because the first is falsy — the default-value idiom. And `0 and (1 / 0)` prints `0` without crashing: `and` saw a falsy left side, so the division by zero on the right was **never evaluated**. That's not luck — it's a guarantee you'll use to write guards.

---

## 4. Control flow I — `if` / `elif` / `else` **[CORE]**

**Depth: [CORE]** — branching is half of what "programming" means (the other half is looping).

### Intuition — why branching exists

A program that runs the same statements top to bottom every time can only ever do one thing. Real programs *decide*: is the password correct? is the number divisible by 3? did the model ask to use a tool? `if` is the statement that makes execution **conditional** — before it (in the very first machines), "deciding" meant physically rewiring the computer. The conditional jump instruction is arguably the most important invention in computing; `if` is its human-readable form.

The shape, and the three rules that govern it:

```python
if condition_1:          # evaluated first — truthiness rules from section 3
    ...                  #   runs only if condition_1 is truthy
elif condition_2:        # checked ONLY if condition_1 was falsy
    ...
else:                    # runs only if everything above was falsy
    ...
```

1. **Indentation is the syntax.** The indented block *is* the body — there are no braces. Four spaces is the universal convention. Mixing indentation levels is a `IndentationError`.
2. **First truthy branch wins; the rest are skipped.** Even if three conditions are all true, only the first one's body runs. Order matters — this is the FizzBuzz trap (below).
3. `elif` and `else` are optional. A bare `if` is fine.

### Analogy — the airport security lane

Passengers (execution) reach a fork: crew badge? → crew lane; pre-screened? → fast lane; otherwise → standard lane. Each passenger takes exactly **one** lane, checked top to bottom, first match wins. *Where it breaks:* an airport can open several lanes at once; `if/elif/else` never can — it is strictly one-of. If you need "do ALL checks independently," you write separate `if` statements (no `elif`), a distinction that matters constantly.

### Worked example — the ordering trap (this is the heart of FizzBuzz)

Task: for a number `n`, print `"FizzBuzz"` if divisible by both 3 and 5, `"Fizz"` if by 3, `"Buzz"` if by 5, else the number.

```
WRONG ORDER (n = 15):                    RIGHT ORDER (n = 15):
if n % 3 == 0:        ← 15%3==0 True     if n % 15 == 0:      ← True → "FizzBuzz"
    print("Fizz")     ← prints "Fizz"    elif n % 3 == 0:       (skipped)
elif n % 5 == 0:        (skipped —       elif n % 5 == 0:       (skipped)
    print("Buzz")        first match     else:                  (skipped)
elif n % 15 == 0:        already won!)       print(n)
    print("FizzBuzz") ← NEVER REACHED
```

With the specific-case-last ordering, `n % 15 == 0` can never be reached, because any multiple of 15 is also a multiple of 3 and gets captured by the first branch. **Rule: most specific condition first.** Trace it for n = 15, 9, 10, 7 and confirm: FizzBuzz, Fizz, Buzz, 7.

### Under the hood

The interpreter compiles `if cond:` to bytecode that (1) evaluates the expression `cond`, (2) calls the truthiness protocol on the result (`__bool__`/`__len__`, section 3), (3) executes a *conditional jump*: fall into the block if truthy, jump past it if falsy. `elif` chains are just nested jumps. That's the whole mechanism — an `if` costs one test and one jump, essentially free.

### Runnable example — a grading function exercising every branch

```python
# grade.py
def grade(score):                    # def = function definition; full story in section 7
    if score >= 90:
        return "A"
    elif score >= 75:
        return "B"
    elif score >= 60:
        return "C"
    else:
        return "F"

for s in [95, 90, 74, 60, 12]:
    print(s, "->", grade(s))
```

```
$ python grade.py
# -> 95 -> A
# -> 90 -> A
# -> 74 -> C
# -> 60 -> C
# -> 12 -> F
```

**Why this works, line by line.** `grade` packages the decision so it can be reused (functions in full in section 7 — for now: `def` names a block, `return` sends a value back). For `95`: the first condition `95 >= 90` is truthy, the body returns `"A"`, and — key point — `return` exits the function immediately, so no later branch is even checked. For `90`: `>=` includes the boundary, so it's still an A — off-by-one boundaries (`>` vs `>=`) are one of the most common logic bugs in all of software; test *at* the boundary, as this loop does. For `74`: the first two tests are falsy, `74 >= 60` is truthy → `"C"`. For `12`: everything is falsy, so the `else` catches it. Note the chain runs top-down and *first match wins* — the conditions don't need upper bounds (`score >= 75` doesn't need `and score < 90`) precisely because anything ≥ 90 was already captured a line earlier. That's the same ordering discipline as the FizzBuzz trace above.

---

## 5. Control flow II — `for`, `while`, `break`, `continue` **[CORE]**

**Depth: [CORE]** — loops are the other half of programming, and the `while` loop specifically is the skeleton of every LLM agent (Part 2 will make that literal).

### Intuition — why loops exist

Computers earn their keep by repeating things millions of times without complaint. Before loops (in the straight-line programs of the 1940s), "repeat 100 times" meant physically writing the instructions 100 times. Loops compress repetition into a description of it:

- **`for` — "for each item in this collection, do the block."** Use when you're processing *things that exist*: characters of a string, lines of a file, numbers in a range, messages in a chat history. You know the itinerary in advance.
- **`while` — "keep doing the block as long as this condition stays truthy."** Use when you *don't* know how many rounds you need: keep asking until the user quits, keep retrying until the network responds, **keep calling the model until it stops requesting tools**. The condition is re-checked before every lap.

And the two control valves that work inside either loop:

- **`break`** — leave the loop entirely, right now.
- **`continue`** — abandon this lap only; jump straight to the next iteration.

Why they exist: without them, expressing "stop early when found" or "skip invalid items" requires contorted flag variables and nested ifs. `break`/`continue` state the intent directly.

### Analogy — the assembly line and the thermostat

A **`for` loop is an assembly line**: a conveyor delivers each item, you perform the same operation on each, and the line stops when the last item passes. A **`while` loop is a thermostat**: it doesn't know how many heating cycles today needs — it just re-checks "too cold?" forever and acts while the answer is yes. `break` is the emergency stop button on either machine; `continue` is tossing a defective item off the conveyor and letting the next one arrive.

*Where the analogies break:* the assembly line implies items are independent, but loop iterations can carry state between laps (a running total, a growing string) — each lap can see what previous laps did. And a real thermostat runs concurrently with the room; a `while` loop *is* the only thing running — if its condition never becomes falsy and no `break` fires, the program is stuck forever (the **infinite loop**), consuming 100% CPU and printing nothing. Every `while` you write must have a visible way out; for agent loops we make that a counted guard (`MAX_ITERS`) — you'll see it in Part 2.

### Worked example — a search with `break` and `continue`, traced lap by lap

Find the first even number greater than 10 in a sequence, skipping negatives:

```python
numbers = [3, -4, 7, 12, 18]
found = None
for n in numbers:
    if n < 0:
        continue          # defective item — skip this lap
    if n > 10 and n % 2 == 0:
        found = n
        break             # got it — stop the whole line
```

```
lap  n    n < 0?  skip?     n>10 and even?   action
───  ───  ──────  ────────  ───────────────  ─────────────────────────
1    3    no      —         no (3 ≤ 10)      fall through, next lap
2    -4   YES     continue  (never checked)  jump to next lap
3    7    no      —         no               next lap
4    12   no      —         YES              found = 12, break
5    18   (never reached — break already exited)
```

After the loop, `found` is `12`. Note the interplay: `continue` skipped only lap 2; `break` ended the loop so lap 5 never ran. Also note `found = None` before the loop — the sentinel pattern: initialize to "no answer," so after the loop `if found is None:` tells you whether the search succeeded. `None` (section 2) earning its keep.

### Visual — the two loop shapes

```
   for item in collection:              while condition:
   ┌────────────────────────┐           ┌────────────────────────┐
   │ any items left? ──no──►│ exit      │ condition truthy? ─no─►│ exit
   │       │yes             │           │       │yes             │
   │       ▼                │           │       ▼                │
   │  bind next item        │           │  run the body          │
   │  run the body          │           │  (body must eventually │
   │       │                │           │   change the condition │
   │       └──── back up ───│           │   or break — or you    │
   └────────────────────────┘           │   loop forever!)       │
                                        │       └─── back up ────│
                                        └────────────────────────┘
   break: jump straight to "exit" from anywhere in the body
   continue: jump straight to "back up" from anywhere in the body
```

### Under the hood — what `for` actually does

`for item in thing:` doesn't index with a counter like older languages. It calls `iter(thing)` to get an **iterator** (an object that hands out items one at a time), then repeatedly calls `next()` on it, binding each result to `item`, until the iterator signals exhaustion. That protocol is why `for` works uniformly over strings, lists, files, ranges, and (later) network streams — anything that can produce items one at a time. Name the mechanism (**the iterator protocol**), treat it as a black box until Phase 1. `range(n)` is a small object that produces `0, 1, ..., n-1` on demand *without* building the whole list in memory — which is why `range(10**9)` is instant.

### Runnable example — `for` over `range` and strings; `while` with a counted guard

```python
# loops.py
# --- for over a range: the classic counted loop ---
total = 0
for i in range(1, 6):            # produces 1, 2, 3, 4, 5 (stop value excluded!)
    total = total + i
print("sum 1..5 =", total)

# --- for over a string: strings are iterable, char by char ---
vowels = 0
for ch in "agentic":
    if ch in "aeiou":
        vowels = vowels + 1
print("vowels:", vowels)

# --- while: unknown trip count, with a hard safety guard ---
n = 1
steps = 0
MAX_STEPS = 100                  # the guard — NEVER write a while without an exit
while n < 1000:
    n = n * 3                    # 3, 9, 27, 81, 243, 729, 2187
    steps = steps + 1
    if steps >= MAX_STEPS:
        break                    # belt-and-braces: cap the laps no matter what
print("first power of 3 over 1000:", n, "after", steps, "laps")

# --- break/continue together: first even number > 10 ---
found = None
for candidate in [3, -4, 7, 12, 18]:
    if candidate < 0:
        continue
    if candidate > 10 and candidate % 2 == 0:
        found = candidate
        break
print("found:", found)
```

```
$ python loops.py
# -> sum 1..5 = 15
# -> vowels: 3
# -> first power of 3 over 1000: 2187 after 7 laps
# -> found: 12
```

**Why this works, line by line (loops.py).** `range(1, 6)` yields 1 through 5 — the stop value is *excluded*, Python's universal half-open convention (it makes lengths and splits compose cleanly; internalize it now and a hundred off-by-one bugs never happen). `total = total + i` is the **accumulator pattern**: a variable initialized before the loop, updated every lap — the single most common loop shape in existence (sums, counts, string building, and in Part 2, growing a chat history). The string loop shows that `for` iterates *characters* of a string, and `ch in "aeiou"` is the membership operator from section 3 doing a five-way comparison in one expression. The `while` loop runs an unpredictable number of laps — we genuinely don't know in advance that the answer takes 7 triplings — and carries `MAX_STEPS` as a guard: here it's redundant (the condition provably terminates), but writing the guard *unconditionally* is the habit that saves you when a condition depends on something external (user input, a network, a language model) that might never cooperate. The final block replays the worked example above; run the trace table against it mentally and confirm `12`.

### `input()` — the `while` loop's favorite companion **[WORKING]**

**Depth: [WORKING]** — one built-in function, but it's how interactive programs (including Part 2's chat loop) get data from a human, so meet it properly.

**Intuition & term necessity.** Programs need data from outside themselves. Before interactive input, you edited the source code to change the data — absurd. `input(prompt)` pauses the program, shows `prompt`, waits for the user to type a line and press Enter, and **returns what they typed as a `str` — always a `str`**, even if they typed `42`. That last clause is the whole trap: `input()` never returns an int, so arithmetic on it needs an explicit `int(...)` conversion (§2), which raises `ValueError` on garbage — a crash you'll learn to catch on Day P2.

**Worked example — the read-until-quit shape** (trace it: user types `5`, then `12`, then presses bare Enter):

```python
# read_loop.py
total = 0
while True:                          # deliberately infinite — the break IS the exit
    line = input("number (blank to stop)> ")
    if not line:                     # truthiness: bare Enter gives "" → falsy → done
        break
    total = total + int(line)        # str → int; crashes on "twelve" (P2 fixes that)
print(f"total: {total}")
```

```
$ python read_loop.py
# -> number (blank to stop)> 5
# -> number (blank to stop)> 12
# -> number (blank to stop)>
# -> total: 17
```

**Why this works.** `while True:` is a legitimate, idiomatic pattern — *when the exit is a visible `break` right at the top of the body*. The condition can't be written in the `while` header because you must read the line *before* you can test it; `while True` + immediate guard is the standard solution (called "loop-and-a-half"). `if not line:` is the §3 truthiness guard: `input()` returns `""` on a bare Enter. Everything else is the accumulator pattern. Part 2's `chat.py` is this exact skeleton with a model call where `int(line)` sits — recognize it when you get there.

### The loop `else` clause **[AWARE]**

**Depth: [AWARE]** — know it exists so it doesn't confuse you in other people's code; you don't need to write it. A `for` or `while` may carry an `else:` block that runs **only if the loop finished without hitting `break`** — designed for search loops ("if we never broke, we never found it"). It reads misleadingly ("else" suggests "if the loop didn't run," which is wrong), so most style guides — and this course — prefer the `found = None` sentinel pattern from the worked example above. One concrete look so you can parse it on sight, then treat it as a closed box:

```python
for n in [3, 7, 9]:
    if n % 2 == 0:
        print("first even:", n)
        break
else:                                # belongs to the FOR, not the if
    print("no even numbers")         # runs because no break fired
# -> no even numbers
```

---

## 6. Functions — `def`, arguments, defaults, keyword args, `return`, scope **[CORE]**

**Depth: [CORE]** — functions are how programs stop being scripts and start being systems. Also: the traceback you'll read in section 8 is literally a printout of function calls, so this section is a prerequisite for debugging anything.

### Intuition — why functions exist

Without functions, reusing logic means copy-pasting it, and fixing a bug means finding *every* copy. Functions solve three problems at once:

1. **Reuse** — write the logic once, call it anywhere.
2. **Abstraction** — a good name (`grade(score)`) lets readers use the logic without re-reading its body. This is the mechanism that lets humans build systems too large to hold in one head.
3. **Isolation** — variables inside a function are invisible outside it (scope, below), so functions can't trample each other's names.

Vocabulary, precisely:

```python
def greet(name, punctuation="!"):      # DEF-inition: creates the function object
    message = f"Hello, {name}{punctuation}"
    return message                     # send a value back to the caller

text = greet("Ada")                    # CALL: run the body with name="Ada"
```

- **`def`** is a *statement* that runs like any other: when the interpreter reaches it, it creates a **function object** (everything is an object — functions too) and binds the name `greet` to it. The body does **not** run at definition time — only at call time.
- **Parameters** are the names in the definition (`name`, `punctuation`); **arguments** are the values you pass in the call (`"Ada"`). The call binds arguments to parameters, then executes the body.
- **`return`** does two things at once: *ends the function immediately* and *hands a value back*. A function with no `return` (or a bare `return`) returns `None` — silently. This is the source of a top-3 beginner bug: printing inside a function instead of returning, then wondering why `result` is `None`.
- **Default values** (`punctuation="!"`) make a parameter optional — the caller can omit it.
- **Keyword arguments** let the caller name what they're passing: `greet(name="Ada", punctuation="?")`. Order stops mattering, and calls become self-documenting. This matters enormously in Part 2: *every* Anthropic SDK call is made almost entirely with keyword arguments (`model=...`, `max_tokens=...`, `messages=...`).

### Analogy — the coffee machine

A function is a coffee machine: inputs (beans, water — arguments) go in defined slots (hoppers — parameters), a fixed internal process runs, and one output comes out the spout (`return`). The machine has internal parts (its local variables) you never see and can't reach from outside. Defaults are the machine's presets — "medium strength unless you turn the dial." *Where it breaks:* a coffee machine physically consumes its inputs; a Python function receives *references* to the caller's objects (name tags, section 1) — for the immutable types of today that distinction is invisible, but from P2 onward (mutable lists/dicts) a function *can* modify an object the caller also holds. File that away; it will matter.

### Worked example — positional, keyword, default: every call shape traced

```python
def price(amount, tax_rate=0.18, currency="INR"):
    total = amount * (1 + tax_rate)
    return f"{currency} {total:.2f}"
```

```
call                                          binding that happens              result
────────────────────────────────────────────  ────────────────────────────────  ─────────────
price(100)                                    amount=100, tax=0.18, cur="INR"   INR 118.00
price(100, 0.05)                              positional: 2nd slot → tax=0.05   INR 105.00
price(100, currency="USD")                    tax keeps default, cur overridden USD 118.00
price(amount=100, tax_rate=0.0)               all by name — order-free          INR 100.00
price(currency="USD", amount=50)              keywords in ANY order             USD 59.00
price()                                       ✗ TypeError: missing required
                                                argument 'amount'
price(100, amount=100)                        ✗ TypeError: got multiple values
                                                for argument 'amount'
```

Rules distilled: positional arguments fill parameter slots left-to-right; keyword arguments fill by name; required parameters (no default) *must* be supplied; a parameter can't be filled twice. Style rule you'll see everywhere: once a call has more than ~2 arguments, pass them by keyword — `price(100, tax_rate=0.05)` beats `price(100, 0.05)` for the reader.

### Docstrings — the function's built-in manual **[WORKING]**

A string literal placed as the *first statement* of a function body is its **docstring** — documentation attached to the function object itself (everything is an object; objects can carry their manual). Why it exists: comments (`# ...`) are for readers of the source; docstrings are readable by *tools* — `help(price)` in the interactive interpreter prints it, editors show it on hover, and (later in the course) frameworks like FastAPI and LLM tool registries read docstrings to describe your functions to the outside world. Convention: triple quotes, first line a one-sentence summary in the imperative ("Return the taxed price."), even when one line is all you need.

```python
def price(amount, tax_rate=0.18):
    """Return the amount with tax applied, formatted as a currency string."""
    return f"INR {amount * (1 + tax_rate):.2f}"

print(price.__doc__)                 # the docstring is data ON the object
# -> Return the amount with tax applied, formatted as a currency string.
```

From today onward, every function in your builds gets a one-line docstring. It costs five seconds and it is the habit that later makes your tools self-describing to the model (Part 2 §4's `"description"` field is a docstring's job done by hand — SDK helpers you'll meet on Day 23 generate it *from* the docstring).

### Scope — where a name is visible

**The problem scope solves:** if all variables were global (one shared namespace, as in early BASIC), any function could accidentally overwrite any other function's data, and no code could be understood in isolation. Scope gives each function call its own private namespace.

The rules for today:

1. A name **assigned inside a function** is **local** — it exists only during that call, in that call's namespace, and vanishes when the call returns.
2. A name **only read** inside a function is looked up outward: local first, then the **enclosing** function (if nested — preview, rare at your level), then the **global** (module file) namespace, then Python's **built-ins** (`print`, `len`, ...). The lookup order is called **LEGB** — Local, Enclosing, Global, Built-in.
3. Assignment inside a function **never** touches a global of the same name — it silently creates a *new local*. (There's a `global` keyword to override this; treat needing it as a design smell and don't use it.)

```
              LEGB lookup for the name `x` used inside g()
   ┌ Built-in ───────────────────────────────────────────────┐
   │  print, len, range, ...                       (4th)     │
   │  ┌ Global (the module file) ────────────────────────┐   │
   │  │  x = "global"                            (3rd)   │   │
   │  │  ┌ Enclosing (outer function, if nested) ────┐   │   │
   │  │  │                                    (2nd)  │   │   │
   │  │  │  ┌ Local (this call's frame) ─────────┐   │   │   │
   │  │  │  │  x = "local"?  ← checked FIRST     │   │   │   │
   │  │  │  └────────────────────────────────────┘   │   │   │
   │  │  └────────────────────────────────────────────┘   │   │
   │  └────────────────────────────────────────────────────┘  │
   └──────────────────────────────────────────────────────────┘
```

### Under the hood — call frames (the bridge to tracebacks)

Every function *call* creates a **frame**: a workspace holding that call's local namespace plus a bookmark for where to resume in the caller. Frames stack: if `main()` calls `parse()` which calls `to_int()`, the stack is three frames deep. `return` pops the top frame. **When an error occurs, Python prints that stack of frames — and that printout is exactly what a traceback is** (section 8). Understand frames and tracebacks become obvious.

```
      the call stack while to_int() is running
      ┌──────────────────────────┐  ← top: youngest call
      │ frame: to_int()  locals  │     (an error HERE is
      ├──────────────────────────┤      reported bottom-most
      │ frame: parse()   locals  │      in the traceback)
      ├──────────────────────────┤
      │ frame: main()    locals  │
      ├──────────────────────────┤
      │ frame: module (global)   │  ← bottom: where it all started
      └──────────────────────────┘
```

### Runnable example — definition vs call, `return` vs `print`, defaults, keywords, scope

```python
# functions.py
def price(amount, tax_rate=0.18, currency="INR"):
    total = amount * (1 + tax_rate)          # `total` is LOCAL — born and dies here
    return f"{currency} {total:.2f}"

print(price(100))                            # defaults kick in
print(price(100, 0.05))                      # positional override
print(price(currency="USD", amount=50))      # keywords: any order, self-documenting

# --- return vs print: the classic trap ---
def bad_double(n):
    print(n * 2)                             # SHOWS the value... returns None

def good_double(n):
    return n * 2                             # HANDS BACK the value

x = bad_double(21)                           # prints 42 as a side effect...
print("bad_double gave me:", x)              # ...but x is None!
y = good_double(21)
print("good_double gave me:", y)

# --- scope: assignment inside a function creates a LOCAL ---
count = 0                                    # global

def increment():
    count = 1                                # new LOCAL count — global untouched
    return count

increment()
print("global count is still:", count)

# --- reading a global from inside is fine (LEGB lookup) ---
GREETING = "hello"                           # module-level constant

def shout():
    return GREETING.upper()                  # not assigned here → looked up in Global

print(shout())
print(type(price))                           # functions are objects too
```

```
$ python functions.py
# -> INR 118.00
# -> INR 105.00
# -> USD 59.00
# -> 42
# -> bad_double gave me: None
# -> good_double gave me: 42
# -> global count is still: 0
# -> HELLO
# -> <class 'function'>
```

**Why this works, line by line.** The three `price` calls replay the worked-example table: defaults fill unsupplied slots, positional arguments fill left-to-right, keywords fill by name in any order — and `{total:.2f}` inside the f-string (next section) formats to two decimals. The `bad_double`/`good_double` pair demonstrates the trap in the flesh: `bad_double(21)` *displays* `42` on the screen — a side effect — but because the function has no `return`, the call *expression* evaluates to `None`, which is what lands in `x`. If you take one habit from this section: **functions should `return` their result; `print` belongs at the edges of a program**, because a returned value can be reused, tested, and composed, while printed text is gone. The scope block shows rule 3 in action: `count = 1` inside `increment` creates a *local* `count` in that call's frame; the frame is destroyed on return, and the global `count` proves untouched at `0`. `shout` shows the reading direction is fine — `GREETING` isn't assigned in the function, so LEGB lookup walks outward and finds the global. Finally `type(price)` prints `<class 'function'>`: the function you defined is an ordinary object bound to a name — which means functions can be passed to other functions, stored, and inspected. That fact powers half of what you'll meet later (decorators, callbacks, FastAPI route handlers, tool registries in agents).

---

## 7. f-strings — building text from values **[CORE]**

**Depth: [CORE]** — trivially small feature, used in literally every program; and in Part 2, f-strings are *how prompts get built*, which makes them agent infrastructure.

### Intuition — the problem of gluing text and values

Programs constantly need text with values embedded: `"Order 42 shipped to Ada"`. Concatenation is miserable — `"Order " + str(order_id) + " shipped to " + name` — you must convert every non-string by hand and the result is unreadable. Python went through two earlier solutions — `%` formatting (inherited from C, cryptic) and `.format()` (verbose: `"Order {} shipped to {}".format(order_id, name)` — values live far from their slots). **f-strings** (PEP 498, Python 3.6, 2016) solved it definitively: prefix the string with `f`, and write any *expression* inside `{}` — evaluated on the spot, converted to text, spliced in.

```python
name, order_id, total = "Ada", 42, 1234.5
line = f"Order {order_id} for {name}: {total:.2f} ({'big' if total > 1000 else 'small'})"
# -> "Order 42 for Ada: 1234.50 (big)"
```

Anything after a `:` inside the braces is a **format spec** — a mini-language for presentation. The four you need today: `:.2f` (2 decimal places), `:,` (thousands separators → `1,234.5`), `:>10` (right-align in 10 characters), `:05d` (zero-pad to width 5 → `00042`).

### Analogy — the fill-in-the-blanks form

An f-string is a printed form with labeled blanks: the template is fixed, the blanks are filled from live values at the moment you "print the form." *Where it breaks:* a paper form's blanks accept whatever you scribble; an f-string's braces hold *live code that executes* — `{1/0}` inside an f-string genuinely crashes with `ZeroDivisionError`. Power and responsibility: keep brace expressions simple (a variable, an attribute, a light computation); anything clever belongs on its own line above.

### Worked example — from painful to clean

```
goal: "Ada scored 92.35% (rank 003)"  from  name="Ada", score=0.92349, rank=3

concatenation:  name + " scored " + str(round(score*100, 2)) + "% (rank " + \
                str(rank).zfill(3) + ")"                       ← 5 conversions, unreadable
.format():      "{} scored {:.2f}% (rank {:03d})".format(name, score*100, rank)
                                                               ← slots far from values
f-string:       f"{name} scored {score * 100:.2f}% (rank {rank:03d})"
                                                               ← reads like the output
```

### Runnable example — interpolation, format specs, and the `=` debug trick

```python
# fstrings.py
name = "Ada"
score = 0.92349
rank = 3
big_number = 1234567.891

print(f"{name} scored {score * 100:.2f}% (rank {rank:03d})")
print(f"revenue: {big_number:,.2f}")             # thousands separators + 2dp
print(f"[{name:>10}] [{name:<10}] [{name:^10}]") # align right / left / center

# the = specifier: prints the expression AND its value — instant debugging
print(f"{score=}")
print(f"{score * 100 = :.1f}")

# braces hold real expressions — including calls and conditionals
print(f"{name.upper()} is {'passing' if score > 0.5 else 'failing'}")

# literal braces: double them
print(f"a JSON-ish template: {{\"name\": \"{name}\"}}")
```

```
$ python fstrings.py
# -> Ada scored 92.35% (rank 003)
# -> revenue: 1,234,567.89
# -> [       Ada] [Ada       ] [   Ada    ]
# -> score=0.92349
# -> score * 100 = 92.3
# -> ADA is passing
# -> a JSON-ish template: {"name": "Ada"}
```

**Why this works, line by line.** The first print splices three expressions into one template: `{score * 100:.2f}` evaluates the multiplication *then* applies the format spec (two decimals), and `{rank:03d}` zero-pads the int to width 3. The alignment line shows `>`/`<`/`^` padding a value inside a fixed width — how you make columns line up in terminal output. `f"{score=}"` is the debug specifier (Python 3.8+): it prints the expression text, an equals sign, and the value — the fastest possible "what is this variable right now?" while debugging, and it composes with format specs as the next line shows. The conditional-expression line demonstrates that braces accept any expression, including the **conditional expression** (`A if cond else B`, often called the *ternary*): the whole thing is an *expression* — it evaluates to `A` when `cond` is truthy, else to `B` — which is exactly why it's legal inside braces where §4's `if` *statement* would not be. It exists for one-value choices (`label = "big" if total > 1000 else "small"`); the moment a choice involves more than picking between two values, use the real `if` statement — nested ternaries are famously unreadable. The last line shows the escape rule: to output a literal `{` or `}`, double it — needed whenever your text itself contains braces (JSON templates, which you will meet constantly). Under the hood, the interpreter compiles an f-string at *parse time* into direct build-the-string instructions — it's the fastest of the three formatting styles as well as the most readable, so there is no reason to use the others in new code.

---

## 8. Reading tracebacks — errors are messages, not punishments **[CORE]**

**Depth: [CORE]** — the difference between a programmer who progresses and one who stalls is almost entirely *whether they read error output*. This skill pays compound interest every day of the next 100 days.

### Intuition — what a traceback is and why it exists

When a running program hits an operation it cannot perform — an undefined name, `"10" + 5`, dividing by zero — Python raises an **exception**: a signal that normal execution can't continue. If nothing handles it (handling arrives on Day P2 with `try/except`), the program stops and prints a **traceback**: a report of *what went wrong* and *the exact chain of function calls that led there*.

Before tracebacks (in C, still today), a crashing program printed at best `Segmentation fault` — no location, no cause; you attached a debugger and spelunked. Python's traceback hands you the diagnosis for free. It only *looks* scary because it's verbose; its structure is rigid and completely learnable in five minutes:

```
Traceback (most recent call last):        ← banner: "read the calls top-down,
  File "boom.py", line 12, in <module>       the LAST one listed is where it died"
    print(describe(7))                    ← oldest frame: module level called describe
  File "boom.py", line 8, in describe
    label = build_label(n)                ← describe called build_label
  File "boom.py", line 4, in build_label
    return f"value is {valu}"             ← youngest frame: THE crime scene
NameError: name 'valu' is not defined     ← THE VERDICT: exception type + message
```

**The reading algorithm — bottom-up:**

1. **Last line first.** It names the exception type (`NameError`) and the message (`name 'valu' is not defined`). This is *what* happened. Nine times out of ten the last line alone tells you the fix.
2. **Second-to-last frame next.** File, line number, and the source line — this is *where* it happened.
3. **Walk upward only if you need the story** — each frame above answers "and who called that?", tracing back to your entry point. You'll need the upper frames when the crime scene is in *library* code (a frame in `site-packages/...`): then the bug is almost always in the **lowest frame that is YOUR file** — you passed the library something bad.

This is exactly the call-frame stack from section 6, printed at the moment of death: the bottom of the traceback is the top of the stack (the youngest call). "Most recent call **last**" — the banner literally tells you the ordering.

### Analogy — the air-crash investigation report

A traceback is a flight recorder transcript: the final line is the moment of impact (what failed); the lines above reconstruct the flight path (who called whom). Investigators read the impact first, then trace backward. *Where it breaks:* an air-crash report takes months of inference; a traceback is exact and instant — every line number is literally true. There is no interpretation step; beginners fail only by *not reading it at all* (eyes glaze, copy-paste to a search engine without reading). Read the last line aloud, in words. Every time.

### Worked example — the five exceptions you'll actually meet this week

| Last line says | It means | Typical cause at your level |
|---|---|---|
| `NameError: name 'x' is not defined` | name not found in any scope (LEGB, §6) | typo; used before assignment; wrong case (`Name` vs `name`) |
| `TypeError: can only concatenate str (not "int") to str` | operation undefined between these types | `"total: " + 5` — convert first or use an f-string |
| `ValueError: invalid literal for int() with base 10: 'ten'` | right operation, impossible value | `int(user_input)` on non-numeric text |
| `IndentationError: expected an indented block` | block structure broken | missing/inconsistent indentation after a `:` — caught at *parse* time, before anything runs |
| `SyntaxError: invalid syntax` | Python couldn't even parse the line | missing `:` , unclosed quote/paren — also parse-time; the caret `^` points at (or just after) the problem |

Note the two families: `SyntaxError`/`IndentationError` fire **before the program runs** (the parse step from the overview diagram) and have no call stack; everything else fires **at runtime** on the exact line executed, with the full frame story.

### Runnable example — deliberately crash a three-deep call and autopsy the traceback

```python
# boom.py — this program is SUPPOSED to crash. That's the exercise.
def build_label(n):
    return f"value is {valu}"      # BUG: typo — the parameter is `n`, not `valu`

def describe(n):
    label = build_label(n)
    return label.upper()

print("before the crash")          # proves the program ran fine until the bad line
print(describe(7))
print("after the crash")           # never prints — the exception ends the program
```

```
$ python boom.py
# -> before the crash
# -> Traceback (most recent call last):
# ->   File "boom.py", line 9, in <module>
# ->     print(describe(7))
# ->   File "boom.py", line 6, in describe
# ->     label = build_label(n)
# ->   File "boom.py", line 3, in build_label
# ->     return f"value is {valu}"
# -> NameError: name 'valu' is not defined
```

**Why this works (and fails), line by line.** The program parses cleanly — the typo is a legal *name*, so nothing is wrong until that name is *looked up* at runtime. Execution proceeds: the two `def` statements create function objects (bodies not run — §6), `"before the crash"` prints, then `describe(7)` pushes a frame, which calls `build_label(7)`, pushing another. Inside `build_label`, the f-string evaluates its brace expression `{valu}`: LEGB lookup — not local (the local is `n`), no enclosing, not global, not built-in — exhausted, so Python raises `NameError`. No handler exists anywhere up the stack, so the exception unwinds every frame, the interpreter prints the traceback, and the process exits — which is why `"after the crash"` never appears. Now apply the reading algorithm to the output: **last line** — `NameError: name 'valu' is not defined`: something referenced `valu`, which doesn't exist. **Frame above it** — `boom.py`, line 3, in `build_label`, source shown: there's the typo, fix `valu` → `n`, done. The two frames above that (in `describe`, in `<module>`) are the flight path — unneeded here, indispensable when the bottom frame is inside a library you didn't write. Total time from crash to diagnosis when you read bottom-up: about ten seconds. That's the skill.

---

## Part 1 wrap — how today's pieces interlock

```
values & types (§1–2)  ──feed──►  expressions with operators (§3)
        │                                   │ truthiness
        ▼                                   ▼
   f-strings (§7)                 conditions for if / while (§4–5)
   render values                            │
   into text                                ▼
        │                    functions package the logic (§6)
        │                          │ calls create frames
        └───────► real programs ◄──┘        │ frames crash
                                            ▼
                                   tracebacks report the frame stack (§8)
```

**Interview-style checks for Part 1** *(answer from memory, then verify against the sections)*: 1) What's the difference between an expression and a statement? Give one of each. 2) Why does `x = x + 5` not modify the object `10`? 3) List every falsy value you know. 4) Why must the `n % 15` check come first in FizzBuzz? 5) When do you reach for `while` instead of `for`? 6) A function without `return` returns what — and what bug does that cause? 7) In `f"{total:.2f}"`, what does each part do? 8) In which direction do you read a traceback, and what does the last line contain?

---

# PART 2 — AGENTIC AI: The Same Constructs, Running an LLM

> **Ground rule for this Part:** nothing new about *Python* is introduced here. Every line of code below is built from Part 1's constructs — variables, the five types, `if`, `for`, `while`, `break`, functions with keyword arguments, f-strings, truthiness, tracebacks. What *is* new is the domain: calling a **large language model (LLM)** — a program that takes text in and produces text out — through Anthropic's Python library. Where a line uses something from a later day (dictionaries, imports, exceptions), it is flagged **`(preview — take on faith until Day P2/P3/22)`**. Your job today is not to master the API; it's to *recognize your own Python* inside real agentic code.

## Overview & motivation — what an "LLM call" is, mechanically

An LLM like Claude runs on Anthropic's servers, not your laptop. "Using the model" means your Python program sends text over the network and receives text back. The **SDK** (software development kit — Anthropic's official `anthropic` library) wraps all networking behind one ordinary function call, so from where you sit, *talking to a frontier AI model is calling a Python function with keyword arguments*. Section 6 of Part 1 is the entire skill.

```
your Python program                         Anthropic's servers
┌──────────────────────────┐   request      ┌──────────────────┐
│ client.messages.create(  │ ─────────────► │   Claude (LLM)   │
│   model=...,             │                │  reads the text, │
│   max_tokens=...,        │   response     │  writes a reply  │
│   messages=[...],        │ ◄───────────── │                  │
│ )                        │                └──────────────────┘
└──────────────────────────┘
   a function call with          the reply arrives as an OBJECT
   KEYWORD ARGUMENTS (§6!)       you inspect with for + if (§4–5!)
```

**Setup (once):** `pip install anthropic`, and an API key exported as the environment variable `ANTHROPIC_API_KEY` (an environment variable is a named value your operating system hands to programs — take the mechanics on faith until Day P4). Every example below assumes both.

---

## 1. Your first model call — keyword arguments in the wild **[CORE]**

### Intuition

Look at this call through Part 1 eyes before running it: one `import` *(preview — imports are Day P2; today: "load a library so its names become available")*, one variable binding a client object, one function call with three **keyword arguments**, one `for` loop, one `if`. You already know every construct.

### Runnable example — the minimal complete request

```python
# first_call.py
# pip install anthropic          (and export ANTHROPIC_API_KEY first)
import anthropic                          # (preview — imports: Day P2)

client = anthropic.Anthropic()            # builds the client; reads the key from the env

response = client.messages.create(        # ← a FUNCTION CALL, nothing more
    model="claude-opus-4-8",              #    keyword argument: which model (a str)
    max_tokens=1000,                      #    keyword argument: reply length cap (an int)
    messages=[                            #    keyword argument: the conversation so far
        {"role": "user", "content": "In one sentence: what is a Python traceback?"}
    ],                                    # (preview — {…} is a dict, […] a list: Day P2.
)                                         #  today: "a labeled record inside a sequence")

for block in response.content:            # response.content is a sequence of blocks
    if block.type == "text":              # keep only the text ones
        print(block.text)
```

```
$ python first_call.py
# -> A Python traceback is the error report Python prints when a program
# -> crashes, showing the chain of function calls that led to the failure —
# -> read from the bottom up.
#    (wording varies run to run — the model writes fresh text each time;
#     the SHAPE of the response is stable, and that's what your code relies on)
```

**Why this works, line by line.** `anthropic.Anthropic()` is a call that returns a *client object* — bind it to a name exactly like any value in §1; it finds your API key in the environment so no secret appears in the code (never hardcode keys). `client.messages.create(...)` is the whole API: a single function called with **keyword arguments** — the `price(currency="USD", amount=50)` pattern from Part 1 §6, and now you see why keyword args matter: with this many parameters, naming them is the only readable option. `model` is a plain `str`; `max_tokens` a plain `int` capping how much the model may write (you pay per token — a token is roughly ¾ of a word); `messages` is the conversation: a **list of labeled records**, each with a `role` (`"user"` = you, `"assistant"` = the model) and `content` (the text). The response lands in an ordinary object; `response.content` is a sequence of *content blocks* — the model can return several kinds (text, tool requests, thinking) — so the idiomatic extraction is precisely Part 1: a `for` loop over the sequence with an `if` filtering on `block.type == "text"`. Note the honesty caveat in the output: LLM replies are **non-deterministic** — assert on shapes (`block.type`), never on exact wording.

*(Two conventions to absorb now, verified against the current API: the model id is exactly `"claude-opus-4-8"` — no date suffix; and current Claude models do not accept a `temperature` parameter — randomness/steering is done via prompting. You'll meet the full parameter surface on Day 22.)*

---

## 2. A chat history is a list you grow with a loop **[CORE]**

### Intuition

The API is **stateless**: the model remembers *nothing* between calls. Every request must carry the entire conversation so far. So "a chatbot with memory" is nothing but: a list variable, `append` in a loop, and the whole list passed on each call. Which is Part 1 §5's **accumulator pattern** — a variable initialized before the loop, grown every lap — with a network call in the middle.

### Analogy — the pen-pal with amnesia

Each letter you send to a brilliant pen-pal with total amnesia must include photocopies of every previous letter, or they can't follow the thread. The growing envelope = your `history` list. *Where it breaks:* postage for a fat envelope grows linearly; LLM cost grows with *every token re-sent every turn*, so long conversations get expensive fast — the reason context management exists later in the course (Day 24+). For today: history is a list; you grow it; you resend it.

### Runnable example — a working multi-turn chat in ~20 lines of P1 Python

```python
# chat.py
# pip install anthropic
import anthropic                                     # (preview — Day P2)

client = anthropic.Anthropic()
history = []                                         # the accumulator (§5) — empty list

MAX_TURNS = 20                                       # guard, exactly like MAX_STEPS in §5
turns = 0
while turns < MAX_TURNS:                             # the chat loop is a WHILE loop
    user_text = input("you> ")                       # read a line from the keyboard
    if not user_text:                                # TRUTHINESS (§3): empty string is falsy
        break                                        # blank line = quit → BREAK (§5)

    history.append({"role": "user", "content": user_text})   # grow the record

    response = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=1000,
        messages=history,                            # send EVERYTHING so far — statelessness
    )

    reply = ""                                       # accumulator for the text blocks
    for block in response.content:                   # FOR + IF extraction, same as ex. 1
        if block.type == "text":
            reply = reply + block.text

    print(f"claude> {reply}")                        # f-string (§7) renders the turn
    history.append({"role": "assistant", "content": reply})  # record the model's turn too
    turns = turns + 1
```

```
$ python chat.py
# -> you> My name is Vinay. Remember it.
# -> claude> Nice to meet you, Vinay — I'll keep that in mind for this conversation.
# -> you> What's my name?
# -> claude> Your name is Vinay.
# -> you>
#    (blank line → truthiness guard fires → break → program ends)
#    (wording varies; the memory effect is real and comes ONLY from resending history)
```

**Why this works, line by line.** `history = []` is the accumulator; `MAX_TURNS`/`turns` is the counted guard from §5 — a `while` over user input is exactly the "unknown trip count" case, and the guard caps it unconditionally. `if not user_text:` is truthiness earning its keep: `input()` returns `""` for a blank line, `""` is falsy, `not ""` is `True`, `break` exits — three sections of Part 1 in one line. Each lap appends the user's record, sends the **whole** list (`messages=history` — statelessness means the second call literally contains turn one, which is the *only* reason the model can answer "What's my name?"), extracts text with the `for`/`if` idiom into a `reply` accumulator, prints it through an f-string, and appends the assistant's record so the *next* lap resends it too. The alternating `user`/`assistant` roles in the list are required by the API — a malformed sequence produces an error response, whose traceback you now know how to read. Strip away the network call and this program is: a while loop, two appends, a for loop, an if, and two f-strings. **That is the entire architecture of a chatbot.**

---

## 3. f-strings build prompts — and `count_tokens` prices them **[CORE]**

### Intuition

A **prompt** is the text you send the model. Real applications almost never send raw user input — they wrap it in instructions, context, and data: "You are a support agent for X. The customer's order status is Y. Answer this question: Z." That wrapping is *string building from live values* — which Part 1 §7 already gave you. **f-strings are prompt engineering's assembly line.** And because you pay per token, the companion skill is measuring a prompt's size before sending — the API has a purpose-built function for that (never estimate with third-party tokenizers; they're for other models and miscount).

### Runnable example — a templated prompt + its real token price

```python
# prompt_builder.py
# pip install anthropic
import anthropic                                      # (preview — Day P2)

client = anthropic.Anthropic()

def build_prompt(product, question, tone="friendly"):     # defaults + keywords: §6
    return (
        f"You are a customer-support assistant for {product}. "
        f"Answer in a {tone} tone, in at most two sentences.\n\n"
        f"Customer question: {question}"
    )

prompt = build_prompt("AcmeDB", "How do I reset my admin password?")
print("--- the prompt we built ---")
print(prompt)

# price it BEFORE sending — count_tokens is the official meter
count = client.messages.count_tokens(
    model="claude-opus-4-8",
    messages=[{"role": "user", "content": prompt}],
)
print(f"--- this prompt costs {count.input_tokens} input tokens ---")

response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=300,
    messages=[{"role": "user", "content": prompt}],
)
for block in response.content:
    if block.type == "text":
        print(block.text)
```

```
$ python prompt_builder.py
# -> --- the prompt we built ---
# -> You are a customer-support assistant for AcmeDB. Answer in a friendly tone,
# -> in at most two sentences.
# ->
# -> Customer question: How do I reset my admin password?
# -> --- this prompt costs 39 input tokens ---
# -> Happy to help! Head to Settings → Admin → Security and click "Reset
# -> password" — you'll get a confirmation email within a minute or two.
#    (token count is exact and stable for identical text; the reply wording varies)
```

**Why this works, line by line.** `build_prompt` is a Part 1 function through and through: two required parameters, one default (`tone="friendly"`), returning a value built by adjacent f-strings (Python glues adjacent string literals inside parentheses into one string — a common multi-line-text idiom; `\n` is the newline character). The braces splice live values into fixed instruction text — swap `product` and every customer gets the same quality wrapper around their own question, which is precisely why templating beats hand-writing prompts. `count_tokens` is the same call shape as `create` but returns only a count object — read `count.input_tokens` (an ordinary `int`) — letting you check cost *before* committing; production systems budget with it constantly. Two cautions to install today: first, **f-string prompt-building means user text becomes part of the instructions** — a malicious `question` like `"Ignore your instructions and..."` is called *prompt injection*, the defining security problem of this field (a whole day later in the course; today, just notice the mechanism: it's *your* f-string that splices attacker text into the prompt). Second, note what we *didn't* pass: no `temperature`, no sampling knobs — current Claude models reject them; you steer with words, in the prompt you just learned to build.

---

## 4. The agent loop is a `while` loop with `if`/`elif` on `stop_reason` **[CORE]**

### Intuition — what turns a chatbot into an agent

A chatbot only *talks*. An **agent** can *act*: you describe **tools** (functions of yours it may request — "look up the weather," "query the database"), and the model, instead of answering directly, can reply "please run tool X with arguments Y." **Your code** runs the tool, appends the result to the conversation, and calls the model again — around and around until the model says it's done. The model *requests*; your Python *executes and decides*. And the decider is nothing but Part 1:

```
                    THE AGENT LOOP (this diagram IS the code below)
        ┌────────────────────────────────────────────────────────┐
        │  while iterations < MAX_ITERS:            (§5: while)  │
        │      response = call the model            (§6: call)   │
        │      if response.stop_reason == "tool_use":  (§4: if)  │
        │          run the requested function       (§6: call)   │
        │          append result to history         (§5: accum.) │
        │          continue looping                              │
        │      else:                                             │
        │          break — the model gave a final answer (§5)    │
        └────────────────────────────────────────────────────────┘
```

`stop_reason` is a string the API sets on every response saying *why the model stopped writing*: `"end_turn"` (finished a normal answer), `"tool_use"` (wants a tool run), `"max_tokens"` (hit your cap — the reply is truncated). Branching on it is a textbook `if`/`elif` chain on a `str` — §2 and §4, verbatim.

### Analogy — the executive and the assistant

The model is a brilliant executive who cannot touch a computer; your program is the assistant who can. The executive says "fetch me the Q3 numbers" (a tool request); the assistant fetches and reports back; the executive either asks for something else or dictates the final memo (`end_turn`). The assistant also enforces office hours — after N round-trips, meeting over (`MAX_ITERS`). *Where it breaks:* a human assistant would refuse an obviously bad instruction; your loop executes whatever tool call arrives, so *the loop is also the safety boundary* — validation and permission checks live in your `if` statements, not in the model. That single sentence is most of "agent safety," previewed.

### Runnable example — a complete tool-using agent, one tool, MAX_ITERS-guarded

```python
# agent_loop.py
# pip install anthropic
import anthropic                                     # (preview — Day P2)

client = anthropic.Anthropic()

def get_word_count(text):                            # THE TOOL: an ordinary §6 function
    words = text.split()                             # .split() chops a string on spaces
    return f"{len(words)} words"

TOOLS = [                                            # describe the tool TO the model
    {                                                # (preview — dicts: Day P2; the three
        "name": "get_word_count",                    #  labels are: what it's called,
        "description": "Count the words in a piece of text. Use for any question about text length.",
        "input_schema": {                            #  when to use it, and what arguments
            "type": "object",                        #  it takes — a machine-readable
            "properties": {                          #  version of a §6 def signature)
                "text": {"type": "string", "description": "The text to count"}
            },
            "required": ["text"],
        },
    }
]

history = [{"role": "user",
            "content": "How many words are in this sentence: 'The quick brown fox jumps over the lazy dog'?"}]

MAX_ITERS = 5                                        # the guard — an agent loop without
iterations = 0                                       # one can ping-pong forever, on YOUR bill

while iterations < MAX_ITERS:                        # ← THE agent loop (§5)
    iterations = iterations + 1
    response = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=1000,
        tools=TOOLS,                                 # offer the tool on every call
        messages=history,
    )

    if response.stop_reason == "tool_use":           # ← the pivot: a str == test (§3–4)
        history.append({"role": "assistant", "content": response.content})
        for block in response.content:               # find the request block(s)
            if block.type == "tool_use":
                print(f"[model requested {block.name} with input {block.input}]")
                result = get_word_count(block.input["text"])   # WE run OUR function
                history.append({                     # send the result back, tagged with
                    "role": "user",                  # the request's id so the model can
                    "content": [{                    # match answer to question
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result,
                    }],
                })
        continue                                     # §5: next lap — model sees the result

    # any other stop_reason: the model is done talking → extract and leave
    for block in response.content:
        if block.type == "text":
            print(block.text)
    break                                            # §5: mission complete, exit the loop
```

```
$ python agent_loop.py
# -> [model requested get_word_count with input {'text': 'The quick brown fox jumps over the lazy dog'}]
# -> The sentence contains 9 words.
#    (the tool-request lap is deterministic in SHAPE — stop_reason=="tool_use",
#     then a final text turn; the closing wording varies)
```

**Why this works, line by line.** `get_word_count` is a plain function — the model never executes it; it can only *ask*. `TOOLS` is its advertisement: name, plain-English description ("when to use me" — the model reads this to decide), and the argument specification, which you should recognize as a machine-readable `def` signature: one required string parameter named `text`. The `while iterations < MAX_ITERS` frame is §5's counted guard made mandatory: a model that (through a bug or a bad prompt) requested tools forever would otherwise loop unbounded, and every lap costs real money — **never ship an agent loop without an iteration cap**. Lap 1: the model reads the question, sees a word-counting tool on offer, and instead of answering returns `stop_reason == "tool_use"` — the `if` catches that plain string comparison, we append the model's turn to history (the protocol requires its request to be on the record), the inner `for`/`if` finds the `tool_use` block, and `block.input["text"]` hands us the argument *the model chose* to pass — which we forward to our own function. The result goes back into history as a `tool_result` tagged with `block.id`, so the model can match answers to requests (it may make several at once — the reason this is a loop-and-tag protocol rather than a single variable). `continue` starts lap 2: the model now sees its own request *and* the result, stops needing tools, and returns ordinary text with `stop_reason == "end_turn"` — the `if` falls through to the extraction `for`/`if`, we print, `break`. Count the constructs: one `while`, one guard, one `if` on a string, two `for`+`if` extractions, one `continue`, one `break`, one function definition, one function call. **An "AI agent" is Part 1 of this note arranged in a circle.**

---

## 5. Truthiness guards the loop — empty results and missing text **[WORKING]**

**Depth: [WORKING]** — a small pattern, but it's the difference between an agent that degrades gracefully and one that crashes or, worse, silently lies.

### Intuition

Tools fail *quietly* all the time: the search finds nothing, the file is empty, the API returns an empty list. If you feed emptiness onward unexamined, two bad things happen: crashes (indexing into an empty sequence) or — the agent-specific hazard — the model receives `""` as a tool result and *makes something up* rather than saying "no data" (models abhor a vacuum; this failure mode is called **hallucination**). The fix is Part 1 §3, verbatim: a truthiness check at every boundary where emptiness is possible, converting silent emptiness into an *explicit, honest message*.

### Worked example + runnable — the guard pattern

```python
# guards.py — pure Part-1 Python; run it, no API needed
def search_orders(query):
    """Pretend database search. Returns matching text, or '' when nothing matches."""
    if query == "order 1042":
        return "Order 1042: shipped 2026-07-19, arriving Thursday."
    return ""                                        # the quiet failure: empty, not error

def tool_result_for(query):
    result = search_orders(query)
    if not result:                                   # TRUTHINESS: "" is falsy (§3)
        return "No matching orders found."           # explicit truth beats silent emptiness
    return result

print(tool_result_for("order 1042"))
print(tool_result_for("order 9999"))

# the same guard, protecting text extraction from a response-like sequence
blocks = []                                          # imagine: a response with no text blocks
reply = ""
for block in blocks:
    reply = reply + str(block)
if reply:                                            # §3 again: only act if there IS text
    print(f"model said: {reply}")
else:
    print("model returned no text this turn (tool-only turn) — not an error")
```

```
$ python guards.py
# -> Order 1042: shipped 2026-07-19, arriving Thursday.
# -> No matching orders found.
# -> model returned no text this turn (tool-only turn) — not an error
```

**Why this works, line by line.** `search_orders` models the real world: failure is an empty string, not an exception — nothing crashes, something is just *absent*. `if not result:` is the §3 bouncer's glance — `""` is falsy — and the function substitutes an explicit sentence. That sentence matters more here than in ordinary code: it becomes the `tool_result` content the *model* reads next lap, and `"No matching orders found."` steers it to answer honestly, where `""` invites invention. The second block shows the same guard on the output side: in the agent loop of example 4, a `tool_use` turn often contains *no* text blocks at all, so `reply` stays `""` — truthiness distinguishes "nothing to print, and that's fine" from real content. One inherited caution from §3: if a tool can legitimately return `0` or `"0"`, the bare truthiness glance misfires — ask the precise question (`is None`, `== ""`) in those domains.

---

## 6. When the agent breaks, a traceback tells you which layer broke **[CORE]**

### Intuition

Agent code has more layers than plain scripts — your code, the SDK, the network, the remote API — and a failed turn produces a traceback whose *frames walk those layers*. The §8 algorithm doesn't change: **last line first** for the what, then the **lowest frame in YOUR file** for the where. The new skill is layer attribution: SDK frames (`.../site-packages/anthropic/...`) between your frame and the error usually mean *you passed something bad* or *the service refused you* — the exception type says which.

### Runnable example — a classic agent bug, crashed and autopsied

```python
# broken_turn.py — SUPPOSED to crash; the autopsy is the lesson
# pip install anthropic
import anthropic                                     # (preview — Day P2)

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=500,
    messages=[{"role": "user", "content": "Say hi."}],
)

print(response.content.text)      # BUG: content is a SEQUENCE of blocks, not one block
```

```
$ python broken_turn.py
# -> Traceback (most recent call last):
# ->   File "broken_turn.py", line 13, in <module>
# ->     print(response.content.text)
# -> AttributeError: 'list' object has no attribute 'text'
```

**Why this fails, and the ten-second autopsy.** Apply §8: *last line* — `AttributeError: 'list' object has no attribute 'text'`. Translation into Part 1 vocabulary: `response.content` is a `list` (a sequence — because the model may return several blocks: text, tool requests, thinking), and a list has no `.text`; the individual *blocks inside it* do. *Frame above* — line 13, your file, source shown: there's the expression. Fix: the `for block in response.content: if block.type == "text":` idiom from every working example above — which is *why* that idiom exists. This is the single most common first-week agent bug, and notice what diagnosing it required: nothing but §8's bottom-up reading plus §2's type awareness. Two more you'll meet soon, so you recognize the last lines on sight: `anthropic.AuthenticationError: ... invalid x-api-key` — your key isn't set/valid (the environment, not your logic); and `anthropic.BadRequestError: ... 'temperature' is not supported` — you passed a parameter current models reject (delete it; steer via the prompt, as noted in example 3). Both arrive as clean typed exceptions with readable messages — the traceback discipline transfers unchanged.

---

## Part 2 wrap — the recognition table

| Agentic concept | It is literally... | Part 1 section |
|---|---|---|
| an API call to Claude | a function call with keyword arguments | §6 |
| chat memory | a list accumulator grown in a loop, resent each turn | §5 |
| a prompt template | an f-string with live values spliced in | §7 |
| the agent loop | `while` + guard, `if` on `stop_reason`, `break`/`continue` | §4–5 |
| a tool | an ordinary `def`-ed function the model may *request* | §6 |
| `MAX_ITERS` | the counted `while` guard | §5 |
| empty-result handling | a truthiness check at the boundary | §3 |
| a failed turn | a traceback — read bottom-up, find your lowest frame | §8 |

**Interview-style checks for Part 2:** 1) Why must the whole history be resent every call, and which Part 1 pattern implements that? 2) What does `stop_reason == "tool_use"` mean, and who actually executes the tool? 3) Why is `MAX_ITERS` non-negotiable in an agent loop? 4) Why return `"No results found."` instead of `""` from a tool? 5) `AttributeError: 'list' object has no attribute 'text'` — what happened, and what's the fix?

---

# PART 3 — THE BRIDGE: How a `while` Loop Becomes an Agent's Control Plane

> Everything here references Parts 1 and 2 only — no new concepts. This Part exists to fuse the two halves into one mental model you can carry forward.

## 3.1 The dependency map — who provides what

The agentic layer doesn't *replace* the Python layer; it **runs on it**. Every arrow below is a dependency you can now name precisely:

```
   AGENTIC LAYER (Part 2)                     PYTHON LAYER (Part 1)
   ─────────────────────                      ────────────────────
   "the agent decides what to do next"  ──►  if/elif on stop_reason (§4)
   "the agent keeps working until done" ──►  while + break, MAX_ITERS guard (§5)
   "the agent remembers the conversation"──► list accumulator, resent each call (§5)
   "the agent uses a tool"              ──►  your def-ed function, called by
                                             YOUR code after an if (§6)
   "the prompt adapts to the user"      ──►  f-string interpolation (§7)
   "the agent handles empty data"       ──►  truthiness guard (§3)
   "the agent run failed"               ──►  a traceback across your frames
                                             and the SDK's (§8, §6 frames)
```

Read the left column as a product manager would, the right as the interpreter would. **The intelligence lives on Anthropic's servers; the control plane — every decision about looping, stopping, executing, recording, and guarding — is your Part 1 Python.** When people say "the agent did X," what ran was your `while` loop; the model only ever *suggested* X in a response object your `if` inspected.

## 3.2 One annotated turn — the full round trip through both layers

Trace a single tool-using turn of Part 2 example 4, tagging each step with its layer:

```
step  layer      what happens                                    construct
────  ─────────  ──────────────────────────────────────────────  ─────────────
 1    Python     while guard checks iterations < MAX_ITERS       §5 while
 2    Python     f-string built the user prompt (earlier)        §7
 3    Python     client.messages.create(model=..., tools=...,    §6 kwargs
                 messages=history) — function call, network I/O
 4    Model      Claude reads history, decides it needs a tool,
                 returns a response object, stop_reason set
 5    Python     if response.stop_reason == "tool_use":          §3 ==, §4 if
 6    Python     for/if finds the tool_use block                 §5 for, §4 if
 7    Python     result = get_word_count(block.input["text"])    §6 call
 8    Python     truthiness guard: if not result: substitute     §3
 9    Python     history.append(tool_result record)              §5 accumulator
10    Python     continue → next lap → model sees the result     §5 continue
11    Model      Claude answers in text, stop_reason "end_turn"
12    Python     else-path: for/if extracts text, print, break   §4–5
```

Twelve steps; the model appears in exactly two of them. If any Python step misbehaves — the guard is missing (step 1), the comparison string is typo'd (step 5), the result isn't appended (step 9) — the *agent* misbehaves, and no amount of model quality can compensate. Conversely, when a turn dies, the traceback's frames tell you which step number you're staring at: a frame in your file at step 7 is your tool's bug; an SDK exception at step 3 is a request/auth problem. The layers fail *together*, but the traceback separates them — that's why §8 was [CORE].

## 3.3 The failure symmetry — same discipline, both directions

- **Python breaks under the agent:** a `NameError` in your tool function kills the turn mid-loop. The traceback's bottom frame is your `def` — fix it like any §8 exercise. The agent layer added nothing but more frames above it.
- **The agent layer breaks over your Python:** the model returns something your code didn't expect — no text blocks (a tool-only turn), an empty result, a truncation (`stop_reason == "max_tokens"`). Nothing raises; your *logic* is simply wrong for the shape it received. The defenses are the quiet Part 1 habits: extract with `for`/`if` instead of `content[0]`, guard with truthiness, branch on `stop_reason` explicitly instead of assuming `"end_turn"`.

One sentence to keep: **crashes are debugged with tracebacks (loud failures, §8); model-shape surprises are prevented with guards (quiet failures, §3–5) — an agent engineer needs both reflexes, and both are Day P1 skills.**

## 3.4 Why this bridge holds for the whole course

Every escalation coming later is an elaboration of today's skeleton, not a replacement: Day P2's dicts formalize the `{"role": ..., "content": ...}` records you took on faith; P3's type hints annotate the `def` signatures; Week 2's ReAct loop is example 4's `while` with better prompts; Week 3's tool contracts deepen the `TOOLS` description; the production weeks wrap the same loop in FastAPI endpoints and retries. When any of it feels overwhelming, run this note's decompression: *find the while, find the ifs, find the appends, read the traceback bottom-up.* The skeleton never changes.

---

# Wrap-up — Memory, Practice, and Proof

## Cheat Sheet — Day P1 on one screen

```python
# ── values, names, types ─────────────────────────────────────────
x = 10                    # bind a name (a tag, not a box); rebinding moves the tag
type(x)                   # every value carries its type: int, float, str, bool, NoneType
int("10"); str(10)        # conversions are explicit; int("ten") → ValueError
x is None                 # the absence test — identity, not ==
0.1 + 0.2 != 0.3          # floats round in binary; never == floats, never floats for money

# ── operators & truthiness ───────────────────────────────────────
10 / 3                    # 3.333... — / ALWAYS float
10 // 3; 10 % 3           # 3 and 1 — floor div, remainder;  n % k == 0  → divisible
0 <= x < 10               # chaining works like math
falsy = [False, None, 0, 0.0, "", [], {}]   # everything else is truthy
name = raw or "default"   # or returns first truthy operand; and short-circuits

# ── control flow ─────────────────────────────────────────────────
if cond:  ...             # first truthy branch wins; MOST SPECIFIC FIRST
elif other:  ...
else:  ...
for item in seq:  ...     # known itinerary; range(1, 6) → 1..5 (stop excluded)
while cond:  ...          # unknown trip count; ALWAYS have an exit + a counted guard
break                     # leave the loop now
continue                  # skip to the next lap

# ── functions ────────────────────────────────────────────────────
def f(a, b=2, *, c="x"):  # required, default, (keyword-only: preview)
    return a * b          # return ends the call AND hands back a value
f(1, c="y")               # keyword args: order-free, self-documenting
# no return → returns None (print ≠ return!)
# assignment inside a function creates a LOCAL (LEGB lookup: Local→Enclosing→Global→Built-in)

# ── f-strings ────────────────────────────────────────────────────
f"{name} owes {total:.2f}"    # any expression in braces; :.2f, :,, :>10, :03d
f"{score=}"                   # debug: prints  score=0.92349
f"{{literal braces}}"         # double them

# ── tracebacks ───────────────────────────────────────────────────
# READ BOTTOM-UP: last line = type + message (the WHAT);
# frame above = file/line (the WHERE); walk up only for the story.
# Library frames in the middle? → your bug is the lowest frame in YOUR file.

# ── the agent skeleton (Part 2) ──────────────────────────────────
# while iters < MAX_ITERS:
#     response = client.messages.create(model=..., tools=..., messages=history)
#     if response.stop_reason == "tool_use": run tool, append result, continue
#     else: extract text with for/if, break
```

## Build This — the Day P1 deliverables

Three small programs. Definition of done for each is a command you run and the exact behavior you must observe.

**1. `fizzbuzz.py` — CLI FizzBuzz.** For every number from 1 to N: print `FizzBuzz` if divisible by 15, `Fizz` if by 3, `Buzz` if by 5, otherwise the number. N comes from the command line (`sys.argv[1]` — take the two lines of `sys` boilerplate on faith until P4: `import sys` then `n = int(sys.argv[1])`); default to 100 if no argument is given (hint: `len(sys.argv)` + an `if`, or the `or` idiom). Structure it as a function `fizzbuzz(n) -> str` returning the label for ONE number, plus a loop that calls it — returning, not printing, inside the function (§6).

*Definition of done:*
```
$ python fizzbuzz.py 15
# -> 1, 2, Fizz, 4, Buzz, Fizz, 7, 8, Fizz, Buzz, 11, Fizz, 13, 14, FizzBuzz
     (one per line is fine)
$ python fizzbuzz.py
# -> ...runs 1..100, line 100 is "Buzz"
```

**2. `wordcount.py` — text-file statistics.** Read a text file named on the command line and print: total lines, total words, total characters, and how many times a target word (second CLI argument) appears — case-insensitively. Reading a file is two lines you take on faith until P3: `with open(filename) as f:` then `text = f.read()`. Everything else is P1: `.split()` for words, a `for` loop with an `if` and an accumulator for the target count, `.lower()` for case-folding, f-strings for the report. First create a `sample.txt` with a few sentences. *(A per-word frequency table needs dicts — that upgrade is part of Day P2's build.)*

*Definition of done:*
```
$ python wordcount.py sample.txt the
# -> lines: 4
# -> words: 57
# -> characters: 312
# -> 'the' appears 5 times
     (your numbers will match YOUR sample.txt — verify one count by hand)
$ python wordcount.py missing.txt the
# -> a traceback ending in: FileNotFoundError: [Errno 2] No such file or
# -> directory: 'missing.txt'     ← expected today! graceful handling arrives with
                                    try/except on Day P2. READ this traceback bottom-up.
```

**3. `boom.py` — cause and autopsy a traceback.** Write a program with two functions where the inner one references a variable that doesn't exist (reuse §8's or invent your own three-frame version). Run it. Then, in comments at the bottom of the file, write in your own words: (a) what the last line says, (b) which file/line the bug is on, (c) what the frames above tell you. *Definition of done:* the traceback prints with **three** frames (`<module>` → outer → inner), and your three comment answers are correct against §8's algorithm.

## Practice — graded exercises (do these after the builds)

**Easy** *(pure single-section drills)*
1. Without running it: what does `print(7 // 2, 7 % 2, 7 / 2)` output? Then run it.
2. Write one line that prints `pi ≈ 3.14` from `pi = 3.14159` using an f-string format spec.
3. Predict `bool(x)` for `x` in: `" "` (a single space), `"0"`, `0.5`, `None`. Verify.
4. Write `is_leap(year)`: divisible by 4, except centuries unless divisible by 400 — pure `if`/`elif` ordering practice. Check 2000 (True), 1900 (False), 2024 (True).

**Medium** *(two or more sections combined)*
5. Write `clamp(n, low=0, high=100)` returning `n` forced into the range — then call it three ways: all positional, mixed, all keyword.
6. Using only a `for` loop, an accumulator, and truthiness: given `text = "the cat sat on the mat"`, build a single string of every word that is NOT `"the"`, space-separated, with no trailing space. (Hint: build a count too, or strip at the end.)
7. Write a `while` loop that keeps calling `input()` and echoes each line in UPPERCASE, quits on blank input, and refuses (with a message, then `continue`) any line longer than 20 characters. Which construct handles the refusal, and why not `break`?
8. Take `boom.py` from the Build and add a *fourth* frame (one more function in the chain). Before running, write down what the traceback will look like; run and compare frame-for-frame.

**Hard** *(design pressure)*
9. FizzBuzz, but the rules arrive as data: given `divisor_a, word_a, divisor_b, word_b` as parameters, write `flexbuzz(n, divisor_a=3, word_a="Fizz", divisor_b=5, word_b="Buzz")` returning the right label — the combined case must still work for ANY pair of divisors. What replaces the hardcoded `% 15`? (Answer to check yourself: `n % divisor_a == 0 and n % divisor_b == 0` — the specific-first *ordering* survives even when the numbers are parameters.)
10. Write `safe_ratio(a, b)` that returns `a / b`, but returns `None` when `b` is zero — then write calling code that distinguishes "ratio is 0.0" from "ratio undefined" *correctly*. Explain in a comment why `if result:` is the wrong guard here and `if result is not None:` is right. (This is §3's bouncer-analogy break, in code.)

**Scenario / thought experiments**
11. An agent loop was shipped without `MAX_ITERS`. The tool it calls has a bug: it always returns `""`. Using Part 2 §4–5, narrate what happens turn by turn, what it costs, and which TWO one-line fixes (one guard, one substitution) prevent it.
12. A teammate's traceback shows six frames: yours at top and bottom, four `site-packages/anthropic/...` frames in between, last line `TypeError: expected str, got int`. Where do you look first, what probably happened, and why do you *not* open the SDK source?

## Active Recall & Self-Test

Answer from memory — no scrolling. Then check.

1. Define *value*, *expression*, *statement* — one sentence each, one example each.
2. After `a = [we haven't done lists — use ints:] a = 7; b = a; a = a + 1` — what is `b`, and *why* (name-tag model)?
3. Which of these are falsy: `-1`, `0.0`, `"False"`, `""`, `None`, `0`?
4. Why does `"" or "default"` evaluate to `"default"`? What does `5 and "yes"` evaluate to?
5. Write FizzBuzz's `if` chain from memory. Why must `% 15` come first?
6. When is `while` the right loop, and what two things must every `while` have?
7. What do `break` and `continue` each do, in one sentence each?
8. `def f(x): print(x * 2)` — what does `y = f(3)` leave in `y`, and why is that a bug factory?
9. What is LEGB, and what happens when you *assign* to a name inside a function?
10. Write, from memory, an f-string printing `total` with two decimals and thousands separators.
11. Recite the traceback reading algorithm. Where is the bug when the bottom frames are inside `site-packages`?
12. (Part 2) What Python construct is the agent loop, what does it branch on, and what guard must it carry?

**Teach-back prompt (60 seconds, out loud, no notes):** *"Explain to a new teammate why an LLM agent is 'just' a while loop — walk through one tool-using turn, naming the Python construct at each step, and finish with what MAX_ITERS protects against."* If you stumble, re-read Part 3 §3.2 and try again tomorrow.

## Spaced-Repetition Flashcards

| Q | A |
|---|---|
| Expression vs statement? | Expression *evaluates to a value* (`2+3`); statement *does something* (`x = 5`, `if`, `return`). |
| What does `=` do in Python? | Binds a name (tag) to an object — no copying, not math equality (that's `==`). |
| The five core types? | `int` (unlimited whole), `float` (IEEE-754, rounds), `str` (immutable text), `bool` (subtype of int), `NoneType` (the singleton `None` = absence). |
| Complete falsy list (P1 level)? | `False, None, 0, 0.0, ""` (+ empty collections `[] {} ()` from P2). |
| `10 / 2` returns...? | `5.0` — `/` always float. Want an int? `//`. |
| Divisibility test? | `n % k == 0`. |
| What does short-circuit mean? | `and`/`or` stop evaluating once the answer is known, and return an *operand* — enables guards (`x and x[0]`) and defaults (`x or "d"`). |
| `if/elif/else` selection rule? | Top-down, first truthy branch wins, exactly one branch runs → most specific condition first. |
| `for` vs `while`? | `for`: iterate things that exist (known itinerary). `while`: repeat until a condition changes (unknown trip count) — needs an exit + counted guard. |
| `break` vs `continue`? | `break` exits the loop entirely; `continue` abandons this lap and starts the next. |
| Function with no `return` returns...? | `None` — and `print` inside is a side effect, not a result. |
| Keyword arguments — why? | Order-free, self-documenting calls; required once a call has many params (every SDK call: `model=`, `max_tokens=`, `messages=`). |
| Assignment inside a function affects globals? | No — it creates a *local* in that call's frame (LEGB); globals are readable but not writable by plain assignment. |
| f-string for 2 decimals? | `f"{x:.2f}"`; debug form `f"{x=}"`; literal braces `{{ }}`. |
| Traceback reading order? | Bottom-up: last line = exception type + message; frame above = your file/line; library frames → your bug is the lowest frame in YOUR code. |
| The agent loop in one line? | `while` under `MAX_ITERS`: call model → `if stop_reason == "tool_use"` run tool + append + `continue`, else extract text + `break`. |
| Why resend full history each call? | The API is stateless — the model remembers nothing; the list accumulator *is* the memory. |
| Why never return `""` from a tool? | The model may hallucinate to fill the void — return an explicit `"No results found."` (truthiness guard at the boundary). |

## Primary Sources

Verify against the originals; these are stable, official, and worth bookmarking:

- **Python Tutorial §3 — An Informal Introduction to Python** (numbers, strings, first steps): https://docs.python.org/3/tutorial/introduction.html
- **Python Tutorial §4 — More Control Flow Tools** (`if`, `for`, `range`, `break`/`continue`, defining functions, default/keyword arguments): https://docs.python.org/3/tutorial/controlflow.html
- **Python Tutorial §8 — Errors and Exceptions** (tracebacks, exception types): https://docs.python.org/3/tutorial/errors.html
- **Library reference — Built-in Types** (truth-value testing lives here — the official falsy list): https://docs.python.org/3/library/stdtypes.html#truth-value-testing
- **PEP 498 — Literal String Interpolation** (the f-string design document): https://peps.python.org/pep-0498/
- **PEP 285 — Adding a bool type** (why `True + True == 2`): https://peps.python.org/pep-0285/
- **PEP 8 — Style Guide** (the `is None` recommendation, naming, indentation): https://peps.python.org/pep-0008/
- **Language reference — the Data Model** ("everything is an object," `__bool__`, `__add__` — skim now, return later): https://docs.python.org/3/reference/datamodel.html
- **Anthropic SDK & API docs** (Part 2's surface; flag: model names/parameters drift — verify before relying): https://platform.claude.com/docs/ and https://github.com/anthropics/anthropic-sdk-python

## Key Takeaways & Summary

**10-second version.** Python programs are values with types, bound to names, combined by operators, steered by `if` and loops, packaged into functions — and when they crash, the traceback tells you exactly where, bottom-up. An LLM agent is those same pieces arranged in a `while` loop around an API call.

**1-minute version.** Everything in Python is an object carrying its type; names are tags, and assignment moves tags, never copies objects. Expressions evaluate to values; statements act. Truthiness lets any value answer yes/no — empty and zero are falsy — powering every `if` and `while` condition and the guard idioms (`x or default`, `if not result`). `for` iterates what exists; `while` repeats until a condition changes and must always carry an exit plus a counted guard. Functions bind arguments to parameters (positionally or by keyword), run in their own frame with their own local names (LEGB), and hand back a value with `return` — no return means `None`. f-strings splice live expressions into text with format specs. When anything fails, read the traceback from the last line up: type and message first, then your lowest frame. And the punchline: an AI agent is a `while` loop with an `if` on `stop_reason`, a list accumulator for memory, f-strings for prompts, your `def`-ed functions as tools, and `MAX_ITERS` as the leash.

**5-minute version.** Reconstruct the chapter aloud, section by section: (1) names bind to objects in a namespace; immutables never change in place, so `x = x + 5` makes a new object — prove with `id()`. (2) The five types and *why each exists*: unlimited `int` (no overflow class of bugs), binary `float` (rounding is physics, not a bug — no `==`, no money), immutable `str`, `bool` as an `int` subtype (PEP 285 compatibility), and `None` as the honest absence that killed sentinel-value bugs — tested with `is` because it's a singleton. (3) Operators are per-type method calls under the hood (`__add__`), which is why `+` concatenates strings; `%` gives remainders and thus divisibility; comparisons chain; `and`/`or` short-circuit and return operands, enabling guards and defaults; falsy = empty-or-zero, via `__bool__`/`__len__`. (4) `if/elif/else` runs exactly one branch, first truthy wins → most specific first (the FizzBuzz ordering proof). (5) `for` runs the iterator protocol over anything that yields items; `range` is lazy and half-open; `while` re-checks a condition each lap and demands an exit; `break` leaves, `continue` skips; the accumulator pattern builds results across laps. (6) `def` creates a function object at runtime; calls bind arguments to parameters — defaults fill gaps, keywords free you from order — and push a frame whose locals die at `return`; reading walks LEGB outward, assignment stays local. (7) f-strings evaluate brace expressions in place with format specs (`:.2f`, `:,`, `:03d`, `=` for debugging) — the definitive fix for `%`-formatting and `.format()`. (8) A traceback is the frame stack printed at the moment of death — read bottom-up; parse-time errors (Syntax/Indentation) have no stack; in layered code the bug is your lowest frame. Then Part 2: the SDK makes model calls a keyword-argument function call; statelessness makes chat history a resent list accumulator; prompts are f-string templates priced by `count_tokens`; the agent loop branches on `stop_reason` under `MAX_ITERS`; truthiness guards keep empty tool results honest; and failed turns are ordinary tracebacks spanning your frames and the SDK's. Part 3 fuses them: the model suggests, your Python decides — the control plane is entirely Day P1 material.

**Expert summary.** P1 establishes the execution model that everything later leans on: objects with identity/type/value, name binding into namespaces, the truthiness protocol (`__bool__`/`__len__`), operator dispatch (`__add__` et al.), the iterator protocol behind `for`, call frames and LEGB resolution behind functions, and the traceback as a stack unwind report. The agentic reading: an LLM integration is I/O at the edges of a plain event loop — a stateless request/response API means conversation state is client-owned (list accumulator), control flow is client-owned (`while`/`if` on `stop_reason` with a bounded iteration budget), and failure handling is client-owned (typed exceptions surfacing through ordinary tracebacks; quiet shape-mismatches prevented by guards). The competencies that compound from here: reading errors bottom-up without flinching, returning values instead of printing them, guarding every boundary where emptiness is possible, and never writing an unbounded loop around something you don't control — a model very much included.

## Mental Models

- **Name tags, not boxes** (§1): assignment ties a tag to an object; rebinding moves the tag; `y = x` is a second tag, zero copying.
- **The bouncer's glance** (§3): truthiness asks only "is there anything here?" — fast and usually right; ask the precise question (`is None`) when `0` is a real answer.
- **Airport security lane** (§4): `if/elif/else` sends each execution down exactly one lane, checked top-down — order the lanes most-specific first.
- **Assembly line vs thermostat** (§5): `for` processes items that exist; `while` re-checks a condition — and a thermostat with a broken sensor runs forever, hence the guard.
- **The coffee machine** (§6): arguments in slots, private internals, one spout (`return`). No spout usage = the drink (`None`) you didn't mean to serve.
- **The flight recorder** (§8): the traceback is the crash transcript — impact (last line) first, flight path (frames) above.
- **The executive and the assistant** (Part 2): the model dictates, your loop executes and enforces office hours (`MAX_ITERS`). The assistant's judgment — your `if`s — is the safety boundary.

## Mnemonics

- **"Zero, empty, None — the falsy clan."** (0, 0.0, "", empty collections, None, False. Everything else: truthy.)
- **"Slash gives a float, double-slash gives the floor, percent gives what's left."** (`/`, `//`, `%`.)
- **"Most specific first"** — the `if`-chain ordering rule (FizzBuzz: 15 before 3 before 5).
- **"break leaves the loop; continue skips a lap."**
- **LEGB** — Local, Enclosing, Global, Built-in: the name-lookup path, inside-out.
- **"Return the result; print at the edges."** — the §6 habit in six words.
- **"Last line first."** — the entire traceback method in three words.
- **"No leash, no loop."** — never a `while` around an external system (users, networks, models) without a counted guard.

## Common Confusions

| Confusion | Resolution |
|---|---|
| `=` vs `==` | `=` binds a name (statement); `==` tests equality (expression). Writing `if x = 5:` is a `SyntaxError` — Python protects you. |
| `print` vs `return` | `print` shows a value on screen (side effect, gone forever); `return` hands the value to the caller for further use. Functions should return; a call to a print-only function evaluates to `None`. |
| Variables are boxes | They're *tags*. `y = x` gives one object two tags; nothing is copied; rebinding `x` doesn't touch `y`. |
| `10 / 2` is `5` | It's `5.0` — `/` always yields float. `//` yields the floored int. |
| `0.1 + 0.2 == 0.3` should be True | Binary floats can't represent 0.1 exactly; every language rounds the same way. Compare with a tolerance; use ints (paise/cents) for money. |
| `is` vs `==` | `==` asks "equal value?"; `is` asks "the *same object*?" — use `is` only for `None` (a singleton) at this level. |
| `elif` chain vs separate `if`s | A chain picks exactly ONE branch; separate `if`s each get evaluated independently. Choose deliberately. |
| `range(1, 6)` includes 6 | It doesn't — half-open: 1,2,3,4,5. Python's universal stop-excluded convention. |
| Assigning inside a function updates the global | It creates a new *local* instead — the global is untouched (and reading is fine; that's LEGB). |
| Empty reply from a tool is fine | To the model, `""` is a vacuum it may fill by inventing facts — always substitute an explicit "no results" message (truthiness guard). |
| The model runs my tools | Never. It *requests*; your loop's `if` decides and your code executes. The control plane is yours. |
| Tracebacks are read top-down | Bottom-up: the last line is the verdict; the frame above it is the crime scene; upper frames are the route. |
