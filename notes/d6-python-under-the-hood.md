# Day 6 — Python Under the Hood: Bytecode, PyObjects, and Why "Slow" Is the Wrong Question

> **Framing.** Today you open the box marked `python`. You learn what actually happens between the moment you press Enter on `x = a + b` and the moment a result appears: your text becomes tokens, tokens become a tree, the tree becomes **bytecode**, and a single C loop (`ceval`) walks that bytecode one instruction at a time, pushing and popping a stack of **PyObjects** — where even the number `1` is a 28-byte heap object with a type pointer and a reference count. You learn *why* that makes Python 10–100× slower than C per operation, and — the part that matters — *why that mostly doesn't bill you a cent on a real backend*, because your server spends 99.9% of its life waiting on a network, not running bytecode (Day 1's latency pyramid, cashed in).
>
> **Who it's for.** Someone who has written Python but has never seen its bytecode, never asked why an `int` costs 28 bytes, and half-believes "Python is slow" is a reason to avoid it. We build the whole machine from zero.
>
> **The ONE idea that unites the backend and agentic layers:** *Interpreter overhead only matters where there is no bigger wait to hide behind.* A CRUD API and an LLM agent are both dominated by I/O — the network call dwarfs the interpreter by three-to-six orders of magnitude — so Python's per-operation cost is noise, and choosing it buys you developer velocity for free. The *instant* you have a tight CPU-bound loop with no I/O to hide behind — a log parser, a cosine-similarity sweep over an embedding store, a token-by-token transform — the interpreter's cost stops being noise and becomes the bill, and you reach for native code (numpy, Cython, Rust-via-pyo3) that runs compiled and *releases the GIL*. Knowing which regime you are in is the entire skill, and it is identical whether you are building a backend service or an agent.

**A note on versions.** CPython's *internals* drift faster than almost anything else in this course. Bytecode opcodes, object byte-sizes, and the interpreter loop changed materially in 3.11 (the specializing adaptive interpreter, PEP 659), again in 3.12 (immortal objects, PEP 683; comprehension inlining, PEP 709), and 3.13 ships an *experimental* JIT (PEP 744) and *experimental* free-threaded/no-GIL build (PEP 703). **Every concrete transcript below targets CPython 3.11 on a 64-bit machine and I flag it as such.** The *concepts* (a stack machine over PyObjects with a refcount header) are stable across all of them; the exact byte offsets and opcode names are not. When a number matters, run it on *your* interpreter — `python --version` first, always.

**A note on this note's numbers.** I generated this note in an environment where I could not execute Python, so the `dis` and `getsizeof` transcripts below are hand-derived from CPython 3.11's documented behaviour (these I know precisely), and the `cProfile` / `timeit` transcripts are **representative** — the *shape and ratios* are the lesson, the absolute nanoseconds are machine-specific. Each runnable block is complete and pasteable; **run it yourself** — that is the Build task, and confirming these numbers on your own box is exactly the skill.

**Reading order.** Part 1 builds the CPython machine. Part 2 puts an LLM agent on top of it and asks where the interpreter's cost actually shows up. Part 3 is where they become one system and share a failure mode.

---

# PART 1 — BACKEND

## 1.1 Interpreted vs compiled — the distinction that isn't binary

**Depth: [CORE]**

### Intuition

A computer's CPU only understands **machine code**: a stream of bytes encoding instructions like "add register 1 to register 2" (Day 1). Your Python source is text. Something has to bridge that gap, and there are two classic strategies:

- **Compiled (ahead-of-time):** a separate program (the compiler) reads your *entire* source once, translates it to machine code, and writes a binary file. You ship the binary; the CPU runs it directly. C, Rust, and Go work this way. The translation cost is paid **once, before the program ever runs**.
- **Interpreted:** a program (the interpreter) reads your source and *executes it as it goes*, doing the translation work **every time the program runs**. No separate binary; you ship the source and the interpreter runs it.

The trade is stark and it explains the whole day: compiling lets a smart compiler spend seconds analysing your code and emit machine code that runs in 1 nanosecond per operation; interpreting means re-deciding what to do at every single operation, which costs tens to hundreds of nanoseconds *per operation* — but you get instant startup, no build step, and the ability to run the same source on any machine that has the interpreter.

**The distinction isn't binary, and this is the part most people get wrong.** Python is not "interpreted" the way a 1980s BASIC that re-parsed each line was. CPython **compiles** your source to an intermediate language called **bytecode**, and *then* interprets the bytecode. So Python has a compile step (fast, dumb, ~microseconds) and an interpret step (the `ceval` loop). Java and C# do the same but go further, JIT-compiling their bytecode to machine code at runtime. "Compiled vs interpreted" is really a spectrum of *when* and *how much* translation happens.

### Analogy — the recipe

A **compiled** language is a recipe a professional kitchen has already turned into a precise, wordless assembly line: every station knows exactly what to do, no reading happens during service, plates fly out. Setting up the line took an afternoon.

An **interpreted** language is a home cook reading the recipe aloud one line at a time — "now dice the onion… now, what's next… heat the pan…" — re-reading and re-deciding at every step. No setup; you can start cooking the instant you have the card. But every dish is slower because you're *reading while cooking*.

**Where the analogy breaks (non-negotiable):** the home cook re-reads the *English recipe* every time. CPython does **not** re-read your `.py` text on every operation — it reads it *once*, turns it into a terse shorthand (bytecode), and then re-reads *the shorthand*. So it's a home cook who first rewrites the recipe into their own two-word-per-step shorthand and then cooks from *that*, every service. The re-reading cost is real but far smaller than parsing English each time — which is precisely why "Python re-parses your code line by line" is a myth, and why the `.pyc` cache (below) exists.

### Worked example — the same program, three points on the spectrum

Consider `total = a + b` where `a` and `b` are integers.

| Language | What happens to `a + b` | Cost of the add at runtime |
|---|---|---|
| **C** (compiled) | Compiler emits a single x86 `add` instruction into a binary | ~1 CPU cycle (~0.3 ns) |
| **Python** (CPython, bytecode-interpreted) | Compiled *once* to a `BINARY_OP` bytecode; at runtime the `ceval` loop decodes it, calls `PyNumber_Add`, which looks up the `+` slot on `int`, unboxes both objects, adds, and boxes a new `int` object | ~30–100 ns (§1.4 measures why) |
| **Java** (bytecode + JIT) | Compiled to JVM bytecode, then the JIT compiles the hot method to machine code after ~10k runs | ~1 ns once JIT-warmed |

That ~100× gap on a single add is real. Whether it matters is the question §1.6 answers — and the answer is "almost never on a backend, because you're not doing billions of adds, you're waiting on a database."

### Under the hood

CPython the reference implementation ("CPython" = the C program you download from python.org; there are others — PyPy, GraalPy, MicroPython) does this the moment you import or run a module:

1. **Compile** source → bytecode (fast, one pass, no optimization to speak of).
2. **Cache** the bytecode to a `.pyc` file under `__pycache__/` so the compile step is skipped next time the source is unchanged. The `.pyc` starts with a *magic number* (bytecode version tag) + a source hash/mtime, so a stale cache is detected. This is why the *first* import of a big library is slower than the second.
3. **Interpret** the bytecode in the `ceval` loop (§1.2).

You can watch the compile step directly:

```python
# compile_step.py — any OS. stdlib only.  Run: python compile_step.py
import dis
src = "total = a + b\n"
code = compile(src, filename="<demo>", mode="exec")  # source TEXT -> a code object
print(type(code))            # -> <class 'code'>
print(code.co_code[:8])      # -> the raw bytecode BYTES, e.g. b'\x97\x00e\x00...'
dis.dis(code)                # human-readable disassembly (see §1.5)
```

`compile()` is the Python compiler exposed as a function: text in, a **code object** out (a bundle of bytecode plus metadata — constants, variable names, line table). `run` and `import` call it for you. Primary source: the `compile` builtin and the `dis`/`code` docs at docs.python.org.

**Deliberate stop.** I am not covering PyPy's tracing JIT or GraalPy here — they're alternative *implementations* and Depth: [AWARE]. When people say "Python" in a backend context they mean CPython 99% of the time; that's what we open.

---

## 1.2 CPython's pipeline — source → tokens → AST → bytecode → the `ceval` loop

**Depth: [CORE]**

### Intuition

"Turn text into running behaviour" is too big a leap to do in one step, so CPython does it in a **pipeline** of small, well-defined stages, each consuming the previous stage's output. This is the same divide-and-conquer that every compiler uses, and knowing the stages by name is what lets you reason about *where* a given behaviour comes from — a `SyntaxError` is the parser, a `NameError` is the interpreter, a slow function is the `ceval` loop.

The five stages:

```
  your .py text
       │  ①  TOKENIZER (lexer)      Parser/tokenizer.c
       ▼
  token stream:  NAME'total'  OP'='  NAME'a'  OP'+'  NAME'b'  NEWLINE
       │  ②  PARSER (PEG)          Parser/pegen  — PEP 617
       ▼
  AST (abstract syntax tree):   Assign(target=Name'total',
                                       value=BinOp(Name'a', Add, Name'b'))
       │  ③  COMPILER              Python/compile.c  (+ symbol table pass)
       ▼
  CODE OBJECT: bytecode  LOAD_NAME a; LOAD_NAME b; BINARY_OP +; STORE_NAME total
       │        (cached to __pycache__/*.pyc via marshal, so ①–③ are skipped next time)
       │  ④  ceval INTERPRETER LOOP  Python/ceval.c  — _PyEval_EvalFrameDefault
       ▼
  side effects: PyObjects created, the name `total` bound, results on the value stack
```

Stages ①–③ are the **compiler** (fast, run once, cached). Stage ④ is the **interpreter** (the `ceval` loop), and it is where all the runtime cost lives.

### Analogy — translating a book through committee

Imagine translating a novel from English to a code only a specific typewriter understands.

1. **Tokenizer** = the person who splits the raw text into *words and punctuation*, throwing away irrelevant whitespace but noting where sentences end. They don't understand grammar; they just chunk.
2. **Parser** = the grammarian who takes that word-stream and builds a *sentence-diagram tree* — subject, verb, object, nested clauses. This is where "grammatically invalid" is caught.
3. **Compiler** = the person who walks the sentence-diagram and writes down a flat list of *typewriter keystrokes* that reproduce the meaning.
4. **Interpreter** = the typist who actually presses the keys, one after another, producing output.

**Where the analogy breaks:** a human translator understands *meaning* and can fix an ambiguous sentence using world knowledge. Every CPython stage is **purely mechanical and local** — the compiler has no idea what your program *does*, it only performs a fixed structural rewrite. That's why Python won't warn you that `if x = 5` is probably a typo the way a human editor would (well — the parser rejects it as a syntax error, but the compiler will happily compile provably-infinite loops or obvious logic bugs). There is no "understanding" anywhere; there is only tokens → tree → bytecode → dispatch.

### Worked example — watch each stage on `total = a + b`

Each stage is exposed as a stdlib module, so you can print the intermediate forms:

```python
# pipeline.py — any OS. stdlib only.  Run: python pipeline.py
import tokenize, ast, dis, io

src = "total = a + b\n"

# ① TOKENIZER — text -> tokens
print("--- ① tokens ---")
for tok in tokenize.generate_tokens(io.StringIO(src).readline):
    if tok.string.strip():                       # skip blank/indent noise
        print(f"{tokenize.tok_name[tok.type]:10} {tok.string!r}")

# ② PARSER — tokens -> AST
print("\n--- ② AST ---")
tree = ast.parse(src)
print(ast.dump(tree, indent=2))

# ③ + ④ COMPILER -> bytecode (which ceval will run)
print("\n--- ③ bytecode ---")
dis.dis(compile(src, "<demo>", "exec"))
```

Representative output (CPython 3.11):

```
--- ① tokens ---
NAME       'total'
OP         '='
NAME       'a'
OP         '+'
NAME       'b'
NEWLINE    '\n'

--- ② AST ---
Module(
  body=[
    Assign(
      targets=[Name(id='total', ctx=Store())],
      value=BinOp(
        left=Name(id='a', ctx=Load()),
        op=Add(),
        right=Name(id='b', ctx=Load())))],
  type_ignores=[])

--- ③ bytecode ---
  0           0 RESUME                   0
  1           2 LOAD_NAME                0 (a)
              4 LOAD_NAME                1 (b)
              6 BINARY_OP                0 (+)
             10 STORE_NAME               2 (total)
             12 LOAD_CONST               0 (None)
             14 RETURN_VALUE
```

**Why this works, line by line.**

- `tokenize.generate_tokens` runs stage ① and yields `TokenInfo` records. Notice it produced `NAME 'a'` — just the *characters* — with **no idea** that `a` is a variable that must exist. Tokenizing is pure lexical chunking; it cannot fail on `a` being undefined, only on illegal characters. This is why an unclosed string is a tokenizer/parse error, not a runtime error.
- `ast.parse` runs stages ①+②. The **AST** is the structural meaning: `Assign` whose value is a `BinOp` of two `Name`s with an `Add`. Crucially, `Name(id='a', ctx=Load())` vs `Name(id='total', ctx=Store())` — the tree already knows `a` is being *read* and `total` is being *written*. That `Load`/`Store` context is what the compiler turns into `LOAD_*`/`STORE_*` opcodes. Tools like linters, formatters (Black), and type checkers (mypy) operate on the AST, never on your text — the AST is the real "source of truth" for tooling.
- `compile(...)` runs stage ③ and hands `dis.dis` a **code object**. The bytecode is a flat instruction list for a **stack machine**: `LOAD_NAME a` pushes `a`'s value, `LOAD_NAME b` pushes `b`'s, `BINARY_OP +` pops both and pushes their sum, `STORE_NAME total` pops the sum into the name. The trailing `LOAD_CONST None` + `RETURN_VALUE` is because a module (like every function) implicitly returns `None`.
- `RESUME 0` at the top is a 3.11-era bookkeeping opcode (it exists for the specializing interpreter and for generator/coroutine resumption). It's a no-op for straight-line code — mentioned so you're not surprised by it.

Note it emitted `LOAD_NAME`, not `LOAD_FAST`, because this is *module-level* code where names live in a dict. Inside a function, locals become `LOAD_FAST` (an array index), and *that difference is the whole benchmark in §1.5*.

### Under the hood — the `ceval` loop is a stack machine

Stage ④, `_PyEval_EvalFrameDefault` in `Python/ceval.c`, is the beating heart of CPython. Stripped to its essence it is:

```c
/* wildly simplified — the real thing is ~4000 lines */
for (;;) {
    opcode  = NEXTOP();          /* read the next 1-byte instruction */
    oparg   = NEXTARG();         /* and its argument */
    switch (opcode) {
        case LOAD_FAST:  PUSH(fastlocals[oparg]);         break;
        case BINARY_OP:  { PyObject *r = POP(), *l = POP();
                           PUSH(PyNumber_Add(l, r)); }     break;
        case STORE_FAST: fastlocals[oparg] = POP();        break;
        case RETURN_VALUE: return POP();
        /* ... ~200 more cases ... */
    }
}
```

Three facts do all the work:

1. **It's a stack machine.** There is a per-frame **value stack**; almost every opcode pushes and pops it. `a + b` is "push, push, pop-pop-add-push." This is why the disassembly reads like Reverse Polish Notation.
2. **Dispatch is the tax.** Every single operation pays for `NEXTOP` + the `switch`/jump + the case body. On a real CPU this is a hard-to-predict indirect branch. Real CPython uses **computed gotos** (a GCC extension — a jump table indexed by opcode) instead of a `switch` when available, precisely to help the CPU's branch predictor; that alone is worth ~15–20%. This dispatch overhead, paid per operation, *is* "why Python is slow" (§1.6).
3. **The operands are PyObject pointers.** `POP()` yields a `PyObject *`, and `PyNumber_Add` has to *discover at runtime* that they're ints and how to add them (§1.4). The loop itself doesn't know types. That is the price and the power of dynamic typing.

**Frames.** Each function call creates a **frame** holding the value stack, the local variables, and a pointer back to the caller. In 3.11 frames were made much cheaper ("zero-cost" frames, an internal `_PyInterpreterFrame`), which is a big part of 3.11's speedup. When you see a traceback, you are reading the chain of frames.

**Deliberate stop.** I am not opening the PEG parser's grammar actions, the symbol-table pass that decides which names are local, or the peephole/AST optimizer (which does constant-fold `2 + 3` → `5` at compile time). You now know the five stages precisely enough to place any behaviour; the parser internals are compiler-course material. Primary sources: `dis`, `ast`, `tokenize` module docs; `Python/ceval.c`, `Python/compile.c` in the CPython source tree; PEP 617 (new PEG parser).

---

## 1.3 Everything is a PyObject — why a Python `int` costs 28 bytes, not 8

**Depth: [CORE]**

### Intuition

In C, the integer `1` is 8 bytes (on a 64-bit machine) sitting directly in a register or on the stack — just the raw bits `00…001`. In Python, the integer `1` is a **heap object**: a struct in dynamically-allocated memory with a header describing *what it is* and *how many references point to it*, and only then the actual value. Everything in Python — every int, every string, every list, every function, every class, even every type — is one of these objects, reached through a **pointer**. This uniformity is *why* the language is so flexible (you can put anything in a list, ask anything its type, add attributes) and *why* it uses so much more memory and runs slower than C: you can never just "have a 1", you always have "a pointer to a boxed 1."

The header that every object carries has two mandatory fields, and understanding them explains half of Python's behaviour:

- **`ob_refcnt`** (8 bytes): the **reference count** — how many things currently point at this object. When it hits 0, the object is freed immediately (Day 4). This is Python's primary memory-management mechanism.
- **`ob_type`** (8 bytes): a pointer to the object's **type object** (`int`, `str`, `list`…). *This* is what makes `type(x)`, `isinstance`, and every method call work — and it's what every operation must dereference to know what to do (§1.4).

That's a 16-byte header (`PyObject`) on *every* object before a single byte of actual value. Variable-sized objects (ints, strings, lists, tuples) carry an additional 8-byte `ob_size` field (making a 24-byte `PyVarObject` header) recording how many elements/digits they hold.

### Analogy — the museum artifact vs the sticky note

A C `int` is a **sticky note**: the number is written directly on it, it weighs nothing, you can hold a thousand in your hand. A Python `int` is a **museum artifact in a display case**: the object itself (the number) sits inside a case that carries a *placard* (its type: "Roman coin, 1st century" = `int`) and a *loan-tracking log* (the refcount: "on loan to 3 exhibits"). The case and its metadata weigh far more than the coin. And you never handle the coin directly — you're always given the *case's catalogue number* (a pointer).

**Where the analogy breaks:** a museum case's metadata is *descriptive* — remove it and the coin is unchanged. Python's header is **load-bearing and functional**: delete the `ob_type` pointer and the object literally cannot be added, printed, or have a method called on it, because *every* operation dispatches through it; zero the `ob_refcnt` and the object is destroyed. The metadata isn't a label *about* the object, it's the machinery that makes the object *usable at all*. That's the difference between data and a live object.

### Worked example — the 28-byte int, dissected

```python
# pyobject_size.py — any OS. stdlib only.  Run: python pyobject_size.py
import sys
for v in [0, 1, 255, 256, 2**30, 2**60, 2**100]:
    print(f"{v!s:>32}  ->  {sys.getsizeof(v):>3} bytes")
print("empty string     ->", sys.getsizeof(""),  "bytes")
print("one-char string  ->", sys.getsizeof("a"), "bytes")
print("float 1.0        ->", sys.getsizeof(1.0), "bytes")
print("empty list       ->", sys.getsizeof([]),  "bytes")
```

Representative output (CPython 3.11, 64-bit):

```
                               0  ->   24 bytes
                               1  ->   28 bytes
                             255  ->   28 bytes
                             256  ->   28 bytes
                      1073741824  ->   32 bytes
          1152921504606846976     ->   36 bytes
 1267650600228229401496703205376  ->   40 bytes
empty string     -> 49 bytes
one-char string  -> 50 bytes
float 1.0        -> 24 bytes
empty list       -> 56 bytes
```

**The 28-byte breakdown for a small int** (value `1`):

```
┌─────────────────────────────┬────────┐
│ ob_refcnt   (reference count)│ 8 bytes│  ← how many references point here
│ ob_type     (pointer to int) │ 8 bytes│  ← "I am an int" — every op reads this
│ ob_size     (number of digits)│ 8 bytes│  ← PyVarObject: ints are variable-length
│ ob_digit[0] (the value: one   │ 4 bytes│  ← the ACTUAL number (a 30-bit "digit")
│              30-bit digit)    │        │
└─────────────────────────────┴────────┘
                        total = 28 bytes    to store what C does in 8 (or 4)
```

**Why this works, and why the numbers are what they are.**

- **`sys.getsizeof(1) == 28`.** 24-byte `PyVarObject` header + one 4-byte "digit". CPython stores big integers in base 2³⁰: each `ob_digit` slot holds 30 bits of the value. The number `1` needs one digit → 28 bytes. This is the famous fact: **a Python int is ~3.5× the size of a C int, before you even count the pointer you need to reach it.**
- **`getsizeof(0) == 24`.** Zero needs *zero* digits (`ob_size == 0`), so it's just the 24-byte header. A beautiful confirmation that the size is header + digits.
- **`255`, `256` still 28; `2**30` jumps to 32.** Anything from 1 up to 2³⁰−1 fits in one 30-bit digit → 28 bytes. `2**30` needs a second digit → 32 bytes. `2**60` → 3 digits → 36. `2**100` → 4 digits → 40. Python integers are **arbitrary precision** — they grow a digit at a time and never overflow — and this layout is the mechanism. (C's `int` silently overflows at 2³¹; Python's cannot, and this is the memory you pay for that guarantee.)
- **The empty string is 49 bytes.** A `str` header is even heavier (it caches length, a hash, and an encoding-kind flag); one-char ASCII adds exactly 1 byte. The point isn't the exact number, it's that **there is no cheap scalar in Python** — the smallest useful object is dozens of bytes. This is why a list of 1,000,000 Python ints costs ~28 MB of int objects **plus** 8 MB of pointers, where a C array or numpy array costs ~8 MB total. It's the entire reason numpy exists (§2.2).

### Under the hood — the small-int and interned-string caches

If every `int` were a fresh 28-byte allocation, `for i in range(1000000)` would allocate and free a million objects. CPython avoids this for common values with two **caches**:

- **Small-int cache.** CPython pre-creates and permanently keeps the integers **−5 through 256** (`NSMALLNEGINTS=5`, `NSMALLPOSINTS=257`). Every time your code produces `1`, it gets a pointer to the *same* shared object. This is why `a = 256; b = 256; a is b` → **True**, but `a = 257; b = 257; a is b` → often **False** (257 is outside the cache, so each is a fresh object). *This is the #1 "gotcha" interview question, and now you know it's a memory optimization, not magic.* (Use `==` for value equality, `is` only for identity — `None`, sentinels.)
- **String interning.** Strings that look like identifiers (and all string *literals* in your source) are **interned**: stored once in a global table so identical strings share one object. This makes the dict lookups that dominate Python (§1.4) faster, because comparing interned strings can be a pointer comparison instead of a character-by-character scan.

```python
# caches.py — Run: python caches.py
a = 256; b = 256
print("256 is 256 ?", a is b)      # -> True  (both point at the cached object)
c = 257; d = 257
print("257 is 257 ?", c is d)      # -> False (each freshly allocated; outside cache)
x = "hello"; y = "hello"
print("'hello' is 'hello' ?", x is y)   # -> True  (interned string literal)
z = "".join(["h","e","l","l","o"])      # built at runtime -> not auto-interned
print("built 'hello' is ?", x is z)     # -> False (usually)  use == for value!
```

In CPython 3.12+ this got a further twist: **immortal objects** (PEP 683) give `None`, `True`, `False`, small ints, and interned strings a sentinel refcount so their reference count is never touched — which removes cache-line contention in the future free-threaded build. So exact `getsizeof` and refcount behaviour shifts across 3.11 → 3.12 → 3.13; **verify on your interpreter**. The *concept* — shared cached objects for hot values — is stable.

**Deliberate stop.** Reference counting itself and the cycle-collecting garbage collector (generations, `gc.collect()`) were covered at Day 4 [WORKING] — I'm cross-referencing, not repeating. The one-line recap: the refcount header you just met is decremented when a reference goes away and the object is freed at 0; a separate generational GC exists only to break *reference cycles* that refcounting alone can't. Instagram famously *disabled* that GC (Day 4, and §1.10 here). Primary sources: `Objects/longobject.c` (see `_PyLong_SmallInts`, `NSMALLPOSINTS`), `Include/object.h` (the `PyObject`/`PyVarObject` structs), `sys.getsizeof` docs, PEP 683 (immortal objects).

---

## 1.4 The cost of dynamic typing — every operation is a lookup

**Depth: [CORE]**

### Intuition

Here is the sentence that explains Python's speed better than any other: **the interpreter does not know the types of your variables, so it must ask — at runtime, on every operation.** In C, the compiler *knows* `a` and `b` are `int`s, so `a + b` compiles to one machine `add`. In Python, `a + b` reaches the `ceval` loop with two anonymous `PyObject *` pointers, and the interpreter has to *discover* what they are and how "+" applies to them, every single time the line runs, even inside a loop that runs a billion times with the same types.

"Discover" means **pointer chasing and dict lookups**, and each one is tens of nanoseconds. That's the per-operation tax. C does the type reasoning *once, at compile time, for free*; Python does it *every time, at runtime, forever*. Dynamic typing is a real feature — it's why you can write `def add(a, b): return a + b` and have it work for ints, floats, strings, lists, and your own classes — but it is not free, and the bill comes due on the hot inner loop.

### Analogy — the letter with no address format

A compiled language is like a mail system where every envelope's format is fixed and known in advance: the sorting machine knows byte 0–5 is the ZIP code, always, and routes at full speed without looking. Python is a mail system where every envelope is a mystery package: the clerk must *open* each one, read a label inside that says "this is a letter, here's how to handle letters," look up the letter-handling procedure in a binder, and only then act — for **every** package, even a thousand identical ones in a row.

**Where the analogy breaks:** the clerk could *learn* — "these are all letters, I'll stop checking." Classic CPython (pre-3.11) genuinely did **not** learn; it re-opened every envelope forever. This is exactly the inefficiency PEP 659's specializing interpreter (§1.7) attacks: after seeing `int + int` a few times, 3.11+ *rewrites that bytecode in place* to a specialized `BINARY_OP_ADD_INT` that skips the general lookup — the clerk finally learns for this specific mailbox. So the analogy-break is itself the frontier of modern CPython.

### Worked example — what `a + b` actually costs, step by step

When `BINARY_OP (+)` runs on two objects in classic (non-specialized) CPython, `PyNumber_Add(a, b)` does roughly:

```
1. read a->ob_type                          (pointer deref)
2. read a->ob_type->tp_as_number->nb_add    (two more derefs — the "+" slot)
3. if b is a different type, also fetch b->ob_type->...->nb_add for __radd__ rules
4. call nb_add(a, b):
     a. check both are ints (long_add in longobject.c)
     b. UNBOX: read a->ob_digit and b->ob_digit into C longs
     c. add the C longs                      (THE single instruction C would have used)
     d. BOX: allocate a NEW 28-byte PyLongObject for the result, set its header,
        write the digit, hand back a pointer
5. push the result pointer onto the value stack
```

Step 4c — the actual arithmetic — is **one** CPU instruction. Everything around it (the type lookups in 1–3, the unbox in 4b, and especially the *heap allocation* of a new boxed result in 4d) is the overhead. That's why the measured cost is ~30–100 ns instead of ~0.3 ns: **the add is 1% of the work; boxing, unboxing, and type dispatch are the other 99%.**

Now scale it: attribute access `obj.x` and global access `len` are *worse*, because they're dict lookups, not slot lookups:

| Operation | What the interpreter must do | Rough cost |
|---|---|---|
| `x` (local variable) | index into the frame's C array of locals (`LOAD_FAST`) | ~1–5 ns |
| `GLOBAL_NAME` (module global) | hash the name, look it up in the module's dict, maybe fall back to builtins dict (`LOAD_GLOBAL`) | ~15–40 ns |
| `obj.attr` (instance attribute) | hash `"attr"`, search the instance `__dict__`, then walk the type's MRO if absent (`LOAD_ATTR`) | ~30–80 ns |
| `obj.method()` | `LOAD_ATTR` to find the function, bind `self`, then `CALL` | ~80–200 ns |
| `a + b` (int) | type-slot dispatch + unbox + add + box (above) | ~30–100 ns |

**The engineering payoff of this table:** in a tight loop, *hoist* globals and attributes into locals. `LOAD_FAST` is an array index; `LOAD_GLOBAL`/`LOAD_ATTR` are hash lookups. This is a real, standard Python optimization, and §1.5 measures it with `dis` and `timeit` so you can *see* the opcode difference cause the timing difference. (Numbers are order-of-magnitude, CPython 3.11 pre-specialization; measure your own.)

### Under the hood — types are just dictionaries of functions

The reason "everything is a lookup" is that a Python **type** is, mechanically, a struct (`PyTypeObject`) full of C function pointers ("slots") plus a `__dict__` of Python-level methods. `int` has a `tp_as_number->nb_add`; `str` has a `sq_concat`; your class has whatever's in its `__dict__` and its bases'. When the interpreter meets `a + b` it walks `a`'s type to find the right slot. When it meets `a.foo` it hash-searches dicts along the **MRO** (method resolution order — the linearized inheritance chain). There is no shortcut in the general case: the interpreter cannot assume, because the very next iteration `a` might be a different type. This is the mechanism behind duck typing, operator overloading, and monkey-patching — and behind the cost.

**Deliberate stop.** I'm not opening the descriptor protocol (`__get__`/`__set__`, how `@property` and methods actually bind) or the full MRO C3-linearization algorithm — both are Depth: [WORKING] for a later OOP day. You now know precisely *why* a Python operation costs 10–100× a C one: runtime type discovery + boxing, on every op. Primary sources: `Objects/abstract.c` (`PyNumber_Add`), `Objects/typeobject.c`, the Python data model docs (`docs.python.org/3/reference/datamodel.html`).

---

## 1.5 The `dis` module — reading the machine, and proving LOAD_FAST beats LOAD_GLOBAL

**Depth: [CORE]**

### Intuition

`dis` (disassemble) is the window into stage ③: it shows you the exact bytecode the compiler produced for any function, so you can *read what the interpreter will do*. This is not academic — it's how you settle "which of these two ways is faster?" arguments with evidence instead of folklore, and it's how you understand why the standard optimizations (hoist to locals, avoid attribute lookups in loops, use built-in vectorized operations) actually work. If §1.2–1.4 were the theory, `dis` is the microscope.

### Runnable example — disassemble a function and read every opcode aloud

```python
# dis_read.py — any OS. stdlib only.  Run: python dis_read.py
import dis

def add(a, b):
    total = a + b
    return total

dis.dis(add)
```

Output (CPython 3.11):

```
  4           0 RESUME                   0

  5           2 LOAD_FAST                0 (a)
              4 LOAD_FAST                1 (b)
              6 BINARY_OP                0 (+)
             10 STORE_FAST               2 (total)

  6          12 LOAD_FAST                2 (total)
             14 RETURN_VALUE
```

**Read it aloud, one opcode at a time** (the left number is the source line; the middle number is the byte offset into the bytecode):

- `RESUME 0` — "housekeeping: (re)enter this frame." A near-no-op present since 3.11; it's where the interpreter checks for things like signals/tracing on entry. Ignore it for logic.
- `LOAD_FAST 0 (a)` — "push local variable slot 0, named `a`, onto the value stack." **`LOAD_FAST` = read from the frame's C array of locals by index.** No hashing, no dict — a raw array index. This is the fastest way to read a value in Python.
- `LOAD_FAST 1 (b)` — "push local slot 1, `b`." Stack now holds `[a, b]`.
- `BINARY_OP 0 (+)` — "pop two, add them (arg 0 selects `+`), push the result." This is the §1.4 dance: type dispatch, unbox, add, box. Stack now holds `[a+b]`.
- `STORE_FAST 2 (total)` — "pop the top and store it into local slot 2, `total`." Stack empty.
- `LOAD_FAST 2 (total)` — "push `total` back." (Yes, we just stored it and immediately reload it — the compiler is not clever enough to keep it on the stack across the statement boundary. A JIT would; the bytecode compiler doesn't.)
- `RETURN_VALUE` — "pop the top and return it to the caller."

**The offset gap 6 → 10 is a clue, not a typo.** `BINARY_OP` is at offset 6, the next instruction at 10 — a 4-byte gap for a 2-byte opcode. The missing 2 bytes are a hidden **inline CACHE** entry the specializing interpreter (§1.7) uses to remember "last time this was int+int." Reveal them with `dis.dis(add, show_caches=True)`. Those caches are how 3.11+ *learns* the types and skips the general dispatch — the analogy-break from §1.4, made concrete in the bytecode.

### Runnable example — LOAD_FAST vs LOAD_GLOBAL, in bytecode and in time

This is the Build task's benchmark. First *see* the opcode difference, then *measure* the time difference — and understand the one causes the other.

```python
# fast_vs_global.py — any OS. stdlib only.  Run: python fast_vs_global.py
import dis, timeit

LIMIT = 1000                      # a module GLOBAL

def uses_global(n):
    total = 0
    for i in range(n):
        if i < LIMIT:             # LIMIT is looked up as a GLOBAL every iteration
            total += i
    return total

def uses_local(n):
    total = 0
    limit = LIMIT                 # hoist the global into a LOCAL, once
    for i in range(n):
        if i < limit:             # now a LOAD_FAST every iteration
            total += i
    return total

print("=== uses_global: the compare line ===")
dis.dis(uses_global)
print("\n=== uses_local: the compare line ===")
dis.dis(uses_local)

# Measure: 20,000 calls each. Absolute ns are machine-specific; the RATIO is the point.
g = timeit.timeit(lambda: uses_global(1000), number=20_000)
l = timeit.timeit(lambda: uses_local(1000),  number=20_000)
print(f"\nuses_global: {g:.3f}s")
print(f"uses_local : {l:.3f}s")
print(f"global is ~{g/l:.2f}x the time of local")
```

The bytecode difference for the `if i < LIMIT/limit:` line (CPython 3.11):

```
=== uses_global (the comparison) ===
             ... LOAD_FAST                2 (i)
             ... LOAD_GLOBAL             ? (LIMIT)        <-- dict lookup: globals, then builtins
             ... COMPARE_OP              0 (<)
             ...

=== uses_local (the comparison) ===
             ... LOAD_FAST                2 (i)
             ... LOAD_FAST                3 (limit)       <-- array index: no dict, no hashing
             ... COMPARE_OP              0 (<)
```

Representative timing (CPython 3.11, one laptop — **run yours**):

```
uses_global: 0.612s
uses_local : 0.451s
global is ~1.36x the time of local
```

**Why this works, and why the ratio is what it is.**

- The *only* difference between the two functions is `LOAD_GLOBAL (LIMIT)` vs `LOAD_FAST (limit)` on the comparison line. Everything else is identical bytecode.
- **`LOAD_FAST` is an array index into the frame's locals** — the compiler assigns every local a numbered slot at compile time (it can, because it can see all assignments in the function via the symbol table). O(1), no hashing, ~a couple of nanoseconds.
- **`LOAD_GLOBAL` is a dict lookup** — hash the string `"LIMIT"`, probe the module's globals dict; if absent, probe the builtins dict (this two-dict fallback is why `len`, `range`, `print` — all builtins — are *slightly* slower to reference than your own module globals). In a loop of 1,000 iterations × 20,000 calls = 20 million lookups, the ~15–30 ns each adds up to the measured gap.
- The ~1.3–1.4× ratio is modest here because `LOAD_GLOBAL` is *itself* specialized/cached in 3.11+ (the inline cache remembers the dict version and the found value's location, turning most lookups into a version-check). On CPython 3.10 and earlier the ratio was larger (~1.5–2×). **This is a live example of PEP 659 narrowing a classic gap — verify on your version.** The lesson stands: in a genuinely hot loop, `limit = LIMIT` (or `_local_len = len`) hoists a dict lookup out to a single array index and is free to write. Standard-library code does this constantly; grep CPython's own `.py` files for lines like `_len = len`.

### Runnable example — `cProfile`: the hotspot is never where you guess

Profiling answers "where does the time *actually* go?" — and the answer is reliably surprising, which is the entire reason you must measure instead of guess.

```python
# profile_me.py — any OS. stdlib only.  Run: python -m cProfile -s cumtime profile_me.py
def slow_string_build(n):
    # the "obvious" suspect: building a big string
    s = ""
    for i in range(n):
        s += str(i)
    return s

def innocent_looking(data):
    # looks trivial, but calls a costly helper in a loop
    return [expensive_transform(x) for x in data]

def expensive_transform(x):
    total = 0
    for _ in range(200):          # the REAL hotspot hides in here
        total += (x * x) % 97
    return total

def main():
    s = slow_string_build(50_000)
    result = innocent_looking(range(5_000))
    return len(s), sum(result)

if __name__ == "__main__":
    print(main())
```

Representative `cProfile` output (`python -m cProfile -s cumtime profile_me.py`):

```
(250000, 12118480)
         55011 function calls in 0.281 seconds

   Ordered by: cumulative time

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
        1    0.000    0.000    0.281    0.281 profile_me.py:19(main)
        1    0.004    0.004    0.216    0.216 profile_me.py:9(innocent_looking)
     5000    0.205    0.000    0.212    0.000 profile_me.py:13(expensive_transform)
        1    0.061    0.061    0.061    0.061 profile_me.py:2(slow_string_build)
    50000    0.007    0.000    0.007    0.000 {built-in method builtins.str}
        1    0.000    0.000    0.000    0.000 {built-in method builtins.len}
```

**How to read this, and the lesson.**

- **`tottime`** = time in that function *excluding* subcalls. **`cumtime`** = time *including* subcalls. `ncalls` = call count. Sort by `tottime` to find where the CPU literally sits; sort by `cumtime` to find the expensive *call chain*.
- The naive suspect, `slow_string_build`, cost **0.061 s**. The innocent-looking list comprehension cost **0.216 s cumulative** — and `tottime` pins **0.205 s inside `expensive_transform`**, called 5,000 times. **The hotspot was the helper nobody looked at, not the string-building everyone blames.** This is the universal experience of profiling: your intuition points at the wrong line ~70% of the time, because you weight code by how *ugly* it looks, not how *often* it runs. `ncalls × per-call cost` is what matters, and only the profiler knows both.
- The fix follows the data: optimize `expensive_transform` (or call it less), and *don't bother* rewriting the string builder — it's 20% of the time and premature. (For the record, `"".join(list_of_strs)` is the right way to build strings, but the profiler just told you it's not today's problem.)
- **`cProfile` is a deterministic/tracing profiler**: it instruments *every* call, which is why it can miscount very cheap functions (the measurement overhead rivals their runtime) and why you should not run it on latency-sensitive production traffic. For a *live* service you use a **sampling** profiler like `py-spy` instead (Part 3) — it periodically peeks at the stack with near-zero overhead and needs no code change. Same question ("where's the time?"), different tool for the setting.

**Under the hood.** `cProfile` is a C extension (`_lsprof`) that hooks CPython's per-call trace mechanism; `timeit` disables the garbage collector and runs your snippet in a tight loop to reduce noise; `dis` walks the code object's `co_code` bytes and pretty-prints them against the opcode table (`dis.opmap`). Primary sources: `dis`, `cProfile`, `timeit`, and `profile` module docs; `Lib/dis.py`, `Modules/_lsprof.c` in CPython.

---

## 1.6 Why Python is "slow" — and why that mostly doesn't matter for a backend

**Depth: [CORE]**

### Intuition

You now know exactly *why* Python is slow per operation: bytecode dispatch overhead (§1.2) + runtime type discovery and boxing (§1.4) + everything-is-a-heap-object (§1.3). Summed up, a pure-Python numeric loop runs ~30–100× slower than the equivalent C. That number is real and it terrifies beginners into avoiding Python for backends. **They have the wrong mental model of where a backend spends its time.**

Here is the reframe, and it is the most important idea in this Part. A backend request does almost no computation. It spends its life **waiting** — on a database query, on a cache, on a downstream API, on the network to the client. Recall Day 1's latency pyramid:

```
  operation                  time         "how far is that?" (Day 1's scale)
  ─────────────────────────  ───────────  ──────────────────────────────────
  one Python bytecode op     ~30 ns       you are here
  L1 cache reference          ~1 ns
  RAM reference               ~100 ns
  SSD random read             ~100 µs      ~3,000× a bytecode op
  same-datacenter round trip  ~500 µs      ~15,000× a bytecode op
  DB query (indexed)          ~1–10 ms     ~30,000–300,000×
  cross-continent round trip  ~100–150 ms  ~5,000,000×
  LLM API call (Part 2)       ~1–10 s      ~30,000,000–300,000,000×
```

If your endpoint does 10,000 bytecode ops (0.3 ms) and one 5 ms database query, the interpreter is **6% of the wall time**. Rewrite the whole thing in Rust and, at best, you shave that 0.3 ms → ~0.003 ms and save 6% — while spending 10× the development time and losing the entire Python ecosystem. **The interpreter overhead is hiding in the shadow of the I/O wait, and you cannot optimize what you're not spending time on.** This is why Instagram, Dropbox, Reddit, YouTube's early stack, and countless others ran enormous businesses on Python: for I/O-bound work, "slow" is a rounding error.

### Analogy — the slow chef in a restaurant where the ingredients arrive by mail

You run a restaurant. Your chef chops 30% slower than a world-class chef. But every dish needs an ingredient that arrives *by mail-order*, taking 2 days. Does hiring the faster chef make dinner come sooner? No — the meal is gated by the 2-day mail wait, not the 10 vs 13 minutes of chopping. The chef's speed is invisible next to the shipping delay. Your money is better spent on faster shipping (a better database, a cache, a CDN) or ordering ingredients in parallel (async/concurrency — Days 19–22) than on a faster chef (a faster language).

**Where the analogy breaks:** if you open a second restaurant that does **no** mail-order — say, a juice bar that just blends fruit on hand, all local work, no waiting — then the chef's speed is suddenly *everything*, because there's no shipping delay to hide behind. That "juice bar" is the CPU-bound service: a log processor, a numeric kernel, an embedding-similarity sweep. There, Python's slow chop is the whole bill, and you *do* hire the faster chef (native code). Knowing which restaurant you're running — I/O-bound or CPU-bound — is the entire language-choice decision (§ System design ①).

### Worked example — the arithmetic that settles it

An API endpoint that fetches a user and returns JSON:

| Step | Work | Time |
|---|---|---|
| Parse request, route, validate (Pydantic) | ~5,000 bytecode ops + some C | ~0.4 ms |
| **`SELECT * FROM users WHERE id=$1`** (indexed, same-DC) | wait on the network + DB | **~3 ms** |
| Serialize the row to JSON | ~2,000 bytecode ops + C | ~0.15 ms |
| **Total wall time** | | **~3.55 ms** |
| Python's share | 0.4 + 0.15 = 0.55 ms | **~15%** |
| The DB wait's share | 3 ms | **~85%** |

Now the punchline for concurrency (previews Days 19–21): during that 3 ms DB wait, a well-written async Python server **does other requests' work** — the wait is *free capacity*, not idle time. So the interpreter's 15% isn't even 15% of a busy server's throughput; the server is bound by how many concurrent waits it can juggle, which is an async/epoll question (Day 5's `epoll`, Day 21's event loop), *not* an interpreter-speed question. **This is why "Python is slow" and "Python backends are slow" are different claims, and the second one is usually false.**

### When it *does* matter (the honest boundary)

Python's per-op cost bites hard, and you must not hand-wave it, exactly when there is **no I/O to hide behind**:

- **CPU-bound loops over lots of data**: parsing millions of log lines, image/audio transforms, numeric simulation, cryptographic hashing in pure Python, big-matrix math.
- **Tight per-element work at scale**: computing cosine similarity across 100k embeddings in a Python `for` loop (§2.2), BM25 scoring a large corpus, per-token processing.
- **Latency-critical hot paths** where even microseconds matter (a trading engine's order book — § System design ①).

The response is never "rewrite everything in Rust." It's "push the *hot loop* — usually a small, well-identified fraction of the code (the profiler tells you which, §1.5) — into native code that runs compiled *and releases the GIL*." That's § System design ②, and numpy is the everyday example: `numpy.dot` is a compiled BLAS call that does a million multiply-adds in one C loop with no per-element PyObject overhead, ~100× faster than the Python equivalent, and it drops the GIL so other threads run meanwhile.

### Under the hood — this is why the GIL is (mostly) tolerable

There's a deep connection here to concurrency (Day 20). The **GIL** (Global Interpreter Lock) means only one thread runs Python *bytecode* at a time — which would be crippling for CPU-bound parallelism. But the GIL is **released during I/O waits** (and inside well-written C extensions like numpy). So for I/O-bound backends — the common case — threads still give you real concurrency: while thread A waits on a socket (GIL released), thread B runs bytecode. The same physics that makes "slow" not matter (you're mostly waiting) is what makes the GIL not matter for typical backends. It bites precisely, and only, on CPU-bound pure-Python parallelism — the same boundary as this whole section. Day 20 opens the GIL fully; today, note that it and interpreter-speed are two faces of the same "are you I/O-bound or CPU-bound?" question.

**Deliberate stop.** The GIL, threads vs processes vs coroutines, and the event loop are Days 19–21 [CORE] — cross-referenced, not opened here. Primary sources: Day 1 latency numbers (Jeff Dean / Peter Norvig's "Latency Numbers Every Programmer Should Know"); the CPython GIL design notes.

---

## 1.7 The moving frontier — faster CPython, free-threading, and a JIT

**Depth: [AWARE]** — know these exist and roughly what they change; treat as black boxes unless a project forces you deeper. **Everything here drifts across versions — verify current before relying on it.**

CPython is, as of the mid-2020s, being made faster and more parallel after years of stability. Three efforts you should be able to name:

- **The specializing adaptive interpreter (PEP 659, shipped in 3.11).** This is the "Faster CPython" project (led by Mark Shannon, funded by Microsoft, with Guido van Rossum on the team). The idea, foreshadowed in §1.4's analogy-break: opcodes start *generic* (`BINARY_OP`) and, after the interpreter observes the actual types a few times, **rewrite themselves in place** into *specialized* forms (`BINARY_OP_ADD_INT`) that skip the general type dispatch, storing what they learned in the **inline CACHE** entries you saw as offset-gaps in §1.5. No JIT, no machine-code generation — just a smarter bytecode interpreter that adapts. Net effect: 3.11 was ~10–60% faster than 3.10 on typical code, and 3.12/3.13 continued the trend. This is why classic "LOAD_GLOBAL is much slower than LOAD_FAST" advice has *softened* on modern versions.
- **Free-threaded / no-GIL CPython (PEP 703, experimental in 3.13).** A build option (`--disable-gil`, the "3.13t" builds) that removes the Global Interpreter Lock, allowing true multi-core parallelism for pure-Python threads (Day 20). It changes reference counting to be thread-safe (biased/atomic refcounts) and relies heavily on immortal objects (PEP 683, §1.3). As of 3.13 it is **experimental, slower single-threaded, and not the default** — most C extensions must be updated to support it. Track it; do not build on it in production yet.
- **An experimental JIT (PEP 744, experimental in 3.13).** A "copy-and-patch" JIT that compiles hot bytecode regions to machine code at runtime. Early, off by default, modest gains so far. The long game is to close more of the gap to compiled languages without a rewrite.

**The one-sentence takeaway for a backend/agent engineer:** CPython is getting faster and (eventually) parallel, which makes "just use Python" the right default *even more often* — but the gains are incremental, they don't change the I/O-bound-vs-CPU-bound calculus of §1.6, and the exact numbers and stability move every release. Primary sources: PEP 659, PEP 703, PEP 744, PEP 683; the "Faster CPython" project on GitHub (`faster-cpython/ideas`); the 3.11/3.12/3.13 "What's New" docs. **Re-verify status against the version you actually run.**

---

## 1.8 System design ① — Language choice: Python vs Go vs Rust for four services

**Depth: [CORE]** (this is the payoff of §1.6 — a decision framework, not a religion.)

**The problem.** You're the tech lead for a platform with four services. For each, pick a language and *defend it with the I/O-bound-vs-CPU-bound reasoning from this Part*, not tribal preference. The three candidates, in one line each:

- **Python (CPython):** slowest per-op, richest ecosystem (web, data, ML, glue), fastest to write, dynamically typed. GIL limits pure-Python CPU parallelism.
- **Go:** compiled, garbage-collected, ~10–40× faster than Python per-op, *excellent* cheap concurrency (goroutines), single static binary, small memory footprint, fast compiles. Simpler than Rust, less raw-fast.
- **Rust:** compiled, no GC, ~C-speed, memory-safe via the borrow checker, no runtime pauses, steepest learning curve and slowest to write. The choice when microseconds or predictability are non-negotiable.

**The decision framework (one question):** *Where does this service spend its wall-clock time — waiting on I/O, or burning CPU?* Plus two modifiers: *how tight is the tail-latency requirement*, and *how much does developer velocity / ecosystem matter*.

| Service | Bound by | Pick | Why (defend it) |
|---|---|---|---|
| **CRUD API** (REST over a database) | **I/O** (DB, network) — §1.6's exact case | **Python** (FastAPI) | ~85% of wall time is the DB wait; interpreter overhead is a rounding error. Python's velocity, Pydantic validation, and ecosystem win outright. Go is a fine *alternative* if you want a single binary + lower memory per instance and don't need Python's libraries; Rust is over-engineering here — you'd pay 10× dev time to optimize the 15% that doesn't dominate. |
| **ML inference gateway** (routes requests to models, does light pre/post-processing) | **I/O** (network to the model server/GPU); heavy math lives in **native** libs already | **Python** (glue) | The gateway is orchestration: receive request → call the model (network/GPU wait) → shape the response. The *actual math* is in torch/onnx/numpy — compiled C/CUDA that releases the GIL. Python is the industry-standard glue precisely because the CPU work isn't in Python. (A pure high-QPS *router* with no ML libs could be Go; but the moment you touch tokenizers/tensors, Python's ecosystem is decisive.) |
| **Log processor** (parse millions of lines/sec, extract fields, aggregate) | **CPU** — §1.6's "juice bar"; little I/O to hide behind | **Go** (or Rust) | This is the case where Python's per-line interpreter overhead *is* the bill and there's no wait to shadow it. Go gives you 10–40× throughput, trivial concurrency for parallel shards, one deployable binary, and stays readable. Rust if you need the absolute max throughput / lowest memory and the team can carry it. Python is viable *only* if you push the hot parse loop into a native lib (pyarrow, a Rust extension via pyo3) — which is admitting the CPU work shouldn't be in Python (§1.9). |
| **Trading engine** (order matching, microsecond latency, deterministic) | **CPU + latency tail** — the strictest case | **Rust** (or C++/carefully-tuned Java) | GC pauses and interpreter jitter are disqualifying: a 10 ms GC stall during a market event is a catastrophe. Rust gives C-level speed with *no* GC pauses and memory safety. Python is out on three counts — interpreter latency, GIL, and unpredictable GC/refcount timing (cross-ref Day 1's LMAX Disruptor: this domain builds architecture around CPU-cache mechanical sympathy; a boxed-PyObject language can't play). |

**The meta-lesson.** Three of four services could reasonably be Python, and the *one* that can't is the one that's CPU-bound *and* latency-critical. "What language should we use?" is almost always really "is this path I/O-bound or CPU-bound, and how tight is the tail?" — and that question you can now answer from first principles. **Don't pick a language per company; pick it per service, from the workload's physics.** (And note: most real platforms are *polyglot* — Python services + a couple of Go/Rust hot-path services + native libraries inside the Python ones. Dropbox and Instagram, §1.10, are exactly this.)

**Failure mode of this decision.** The classic mistake is choosing Rust/Go for a CRUD API "for performance" and shipping 3× slower *as a business* (missed features, slower iteration) to save latency that was never on the critical path — premature optimization at the language level. The opposite mistake is writing a CPU-bound log-cruncher in pure Python and being confused when one core melts at 5% of the target throughput. Both come from skipping the one question.

---

## 1.9 System design ② — The escape hatch for CPU-bound hot paths in a Python service

**Depth: [CORE]**

**The problem.** You have a Python service (say the FastAPI CRUD/ML gateway from §1.8) that is 95% I/O-bound and correctly in Python — but one endpoint does something genuinely CPU-heavy: resizing images, computing similarities over a big array, parsing a 50 MB document, or a numeric transform. Profiling (§1.5) confirms that *one function* is 80% of that endpoint's CPU. You do **not** want to rewrite the service. How do you make the hot path fast without leaving Python?

**Step 0 — profile first, always.** §1.5's whole point: the hot path is rarely where you guess. Optimize the function the profiler indicts, nothing else. And check the *cheap* wins first: a better algorithm or the right stdlib call often beats any language change (an O(n²) loop rewritten O(n) beats Rust-ifying the O(n²)).

**The escape-hatch ladder, cheapest to heaviest:**

```
                              adds     true        dev      deploy
  option                      latency  parallelism? cost    surface   when to reach for it
  ─────────────────────────  ───────  ───────────  ─────  ────────  ───────────────────────
  1. algorithm / better stdlib  none    n/a          low    none      always try first
  2. vectorize (numpy/pyarrow)  none    yes (drops   low    none      hot path is array math
                                        the GIL)
  3. Cython / native ext       ~none    yes (can     med    build     small, hot, numeric
     (C / Rust-via-pyo3)                 drop GIL)           step      kernel; reused a lot
  4. subprocess / process pool  ms      yes (separate med    none      embarrassingly parallel
     (multiprocessing)                   interpreters)                CPU chunks; big inputs
  5. sidecar service (Go/Rust)  network yes (own      high   a new     large component; own
     over localhost/gRPC        hop      process)            service   lifecycle; multi-consumer
```

**Option by option, with the trade-off named:**

1. **Algorithm / stdlib.** Free. `set` membership instead of `list` scan; `str.join` instead of `+=`; `functools.lru_cache` on a pure hot function; `bisect`, `heapq`, `collections.Counter`. The most under-used escape hatch, because it's not glamorous. Do this before anything below.
2. **Vectorize with numpy / pandas / pyarrow.** If the hot loop is arithmetic over many elements, replace the Python `for` with one vectorized call. This moves the loop into compiled C **and releases the GIL** (so it also parallelizes), typically ~50–200× (§2.2 measures it for embeddings). No build step, no new deployable — you just `pip install numpy`. This solves the *majority* of real CPU hot-paths in data/ML/agent code.
3. **Native extension — Cython, or Rust via `pyo3`/`maturin`, or C via CFFI.** When the hot path is a small, well-defined kernel that vectorization can't express (custom parsing, a stateful algorithm), compile *that piece* to native code and call it from Python. **Rust-via-pyo3** is the modern favorite (memory-safe, great tooling with `maturin`, can release the GIL with `py.allow_threads`); Cython lets you gradually type-annotate Python into C. Cost: a compile/build step and a second language in the repo, but only for one file. This is how Pydantic v2 (Rust core), `orjson`, `tokenizers`, and `polars` get their speed — a thin native core under a Python API.
4. **Subprocess / process pool (`multiprocessing`, `ProcessPoolExecutor`).** Sidestep the GIL entirely by running the CPU work in *separate interpreter processes* — true multi-core parallelism with zero native code. Cost: **serialization** (arguments and results are `pickle`d and copied between processes) and **memory** (each process is a full interpreter). Great for chunky, independent CPU jobs (resize 1,000 images, process 100 files) where the per-task work dwarfs the pickle cost. Bad for tiny tasks (pickle overhead dominates) or huge shared data (copying it is the bottleneck).
5. **Sidecar service in Go/Rust.** Promote the hot path to its *own* service the Python app calls over localhost (HTTP/gRPC/a queue). Reach for this when the component is *large*, has its *own* lifecycle/scaling needs, or is *reused* by several services — e.g., a shared image-transcoding service. Cost is the highest: a network hop's latency, serialization, a separate deploy/monitor/on-call surface, and — critically — **you have now created a distributed system** with all of Phase 5's failure modes (retries, timeouts, partial failure). Don't reach here to speed up one function; reach here when the function has grown into a *product boundary*.

**The decision rule.** Climb the ladder only as far as the profiler forces you. Most services never leave rung 2. Rungs 4–5 are justified by *organizational* facts (parallelism across cores, independent scaling, reuse) as much as raw speed. **The anti-pattern is jumping to rung 5 ("microservice it in Go") to fix what rung 2 (numpy) solves in an afternoon** — you trade a one-line `pip install` for a distributed system.

**Failure modes of the escape hatch.**
- **Native extension that *holds* the GIL** (a C loop that never calls `Py_BEGIN_ALLOW_THREADS`) gives you speed but *not* parallelism, and if it runs long it blocks the whole event loop (Part 3). Release the GIL around the compute.
- **Process pool with huge arguments**: pickling a 2 GB array to a worker costs more than the compute saved. Use shared memory (`multiprocessing.shared_memory`) or memory-mapped files, or don't fork.
- **Sidecar with a synchronous call inside an async handler**: now you've added a network wait *and* risk blocking the loop if the client isn't async (Part 3, Day 21).

**Cross-reference.** §2.2 applies exactly this ladder to an agent's embedding-similarity hot path (rung 2, numpy). Part 3 covers the shared failure mode where a CPU-bound hot path — native *or* not — blocks the async event loop and stalls every concurrent request.

---

## 1.10 Case studies

### ① Dropbox — a Python business that reached for native code *surgically*, not by rewriting

**What happened.** Dropbox built essentially its entire product — the desktop client and much of the server — in Python, and grew to hundreds of millions of users on it. Python was the *reason* they could iterate fast in the early years. As they scaled, they did **not** rewrite in a faster language; they made three surgical moves, each mapping onto this Part:

1. **Invested in types instead of rewriting (mypy).** Rather than flee dynamic typing, Dropbox bankrolled **mypy** — the static type checker — to make a multi-million-line Python codebase maintainable and catch bugs the interpreter can't (Guido van Rossum worked on it there). Type hints became the "readable contracts + tooling" you met on Day P3. The lesson: dynamic typing's *maintenance* cost is real and separate from its *speed* cost, and gradual typing addresses the former without changing the language.
2. **Reached for Rust on the one hot, correctness-critical core.** Dropbox rewrote the *sync engine* — the piece that reconciles file state across devices, where both CPU efficiency and absolute correctness matter — from Python to **Rust** (the "Nucleus" rewrite). This is §1.9 rung 5 done right: not "rewrite the company in Rust," but "the sync engine grew into a hot, safety-critical product boundary, so it earned its own native implementation." Primary source: Dropbox engineering blog, *"Rewriting the heart of our sync engine"* (2020).
3. **Used Go for high-throughput backend infrastructure**, and (earlier) **sponsored Pyston**, a JIT-compiled Python fork, as a bet on making Python itself faster (later spun out). §1.8's "polyglot platform" in the flesh.

**Engineering lesson tied to the concept.** A company can be Python-first *and* fast, if it (a) treats the slow-per-op interpreter as irrelevant on I/O-bound paths, (b) manages dynamic typing's maintenance cost with tooling, and (c) drops to native code *only* on the identified hot, critical cores — exactly the §1.9 ladder, at company scale. **Verify current:** Dropbox's stack has evolved; treat specific service breakdowns as of their published posts, not gospel today.

### ② Instagram / Meta's Cinder — making Python faster at scale *instead of* leaving it

**What happened.** Instagram runs one of the largest Django/Python deployments on earth (the web tier is Python). Faced with Python's cost at that scale, Meta's response was, again, **not** a rewrite — it was to make *their* Python faster:

- **Cinder** (`github.com/facebookincubator/cinder`) is Meta's internal, performance-oriented fork of CPython, open-sourced "for reference" (explicitly *not* a supported product). It bundles several techniques that are a masterclass in this Part's material:
  - a **method-at-a-time JIT** compiling hot Python functions to machine code;
  - **"Static Python"** — a typed bytecode compiler that uses type annotations to emit specialized opcodes that skip the §1.4 runtime type checks (annotations become *speed*, not just documentation);
  - **immortal objects** — objects (like small ints/interned strings) whose refcount is never touched, reducing copy-on-write page churn across forked workers; this idea flowed **upstream into CPython as PEP 683** (§1.3);
  - inline caching / "Shadowcode" and eager async evaluation — several of which parallel or fed the upstream **Faster CPython** work (§1.7).
- Earlier and simpler: Instagram famously **disabled CPython's cyclic garbage collector** to stop it from touching (and thus dirtying, via refcount writes) shared memory pages across their forked worker processes, reclaiming ~10% memory headroom via better copy-on-write sharing. That's the Day 4 case study, and it's the *same* root cause immortal objects later addressed more surgically. Primary source: Instagram engineering, *"Dismissing Python Garbage Collection at Instagram"* (2017).

**Engineering lesson tied to the concept.** At sufficient scale, "Python is slow" becomes worth *engineering away at the interpreter level* rather than migrating — because the cost of rewriting millions of lines and abandoning the ecosystem exceeds the cost of building a faster Python. Cinder's tricks (JIT, type-annotation-driven specialization, immortal objects) are precisely attacks on the three costs you learned: dispatch overhead (§1.2), runtime type discovery (§1.4), and refcount churn on shared objects (§1.3). Many of them are now *in* mainstream CPython. **Verify current:** Cinder tracks specific CPython versions and is a research/reference artifact — do not treat it as a deployable runtime without reading its own caveats.

*(Per Principle 7: neither of these is a "failure/postmortem" in the outage sense — Python-internals topics don't have a canonical public postmortem the way fsyncgate does for Day 5. The honest failure here is the *anti-pattern* both companies avoided: the premature full rewrite. I'm not inventing an outage to fill a quota.)*

---

## 1.11 In production (Part 1)

**Best practices, beginner → senior.**

| Level | Habit |
|---|---|
| Beginner | Don't fear "Python is slow." Know your service is I/O-bound (§1.6) before optimizing anything. Use the stdlib; don't hand-roll what `collections`/`itertools` already do fast in C. |
| Intermediate | Profile before optimizing (`cProfile` offline, `py-spy` live) — never guess the hotspot (§1.5). Hoist globals/attrs to locals in genuinely hot loops. Reach for numpy/pandas when the hot path is array math. Use `sys.getsizeof`/`tracemalloc` (Day 4) when memory matters. |
| Senior | Climb the §1.9 escape-hatch ladder only as far as the profiler forces. Release the GIL in native extensions. Never do CPU-bound work on the async event loop (Part 3). Choose language *per service* from the workload's physics (§1.8), and keep the platform polyglot without turning every function into a microservice. |

**Monitoring & observability — what's worth watching.**
1. **CPU time vs wall time per endpoint.** If CPU ≪ wall, you're I/O-bound and interpreter speed is irrelevant — stop optimizing Python and look at the I/O (DB, downstream). If CPU ≈ wall, you found a hot path; profile it.
2. **A continuous/sampling profiler in prod** (`py-spy record`/`py-spy dump`, or a hosted continuous profiler). Cheap enough to leave on; turns "the app is slow" into a flamegraph.
3. **GC pause frequency/duration** (`gc.get_stats()`) if you have large long-lived object graphs — the Instagram case.
4. **Per-process RSS** (Day 4). Python's object overhead (§1.3) makes memory the constraint surprisingly often; a dict of a million small objects is huge.

**Common mistakes, beginner → senior.**
- *Beginner:* rewriting a CRUD service in Go "for speed" (optimizes the 15% that isn't the bottleneck).
- *Intermediate:* optimizing the ugly-looking function instead of the frequently-called one (skipped the profiler).
- *Senior:* shipping a native extension that holds the GIL, or a process pool that pickles gigabytes — solving CPU-bound-ness while creating a new bottleneck.

**Scaling behaviour.** Python backends scale by *concurrency of waits* (async, more workers) far more than by interpreter speed. You add throughput by running more requests' I/O in parallel (Day 21) and by moving CPU hot paths to native code (§1.9), not by making bytecode dispatch faster. **Cost:** the dominant cost of a Python service is usually memory (object overhead) and the downstream systems it waits on, not CPU — size your containers for RSS + concurrency (Day 4), and cache/batch the I/O (Day 15/§1.6) before touching the language.

---

## 1.12 Failure modes & common misconceptions (Part 1)

| Misconception | Reality |
|---|---|
| "Python is interpreted, so it re-reads my source each line." | It **compiles to bytecode once** (cached in `.pyc`) and interprets the *bytecode*. §1.1–1.2. |
| "Python is slow, so Python backends are slow." | Backends are **I/O-bound**; the interpreter is often <15% of wall time and hidden behind the DB/network wait. §1.6. |
| "A Python `int` is 8 bytes like C." | It's a ~28-byte heap object: refcount + type pointer + size + digits. §1.3. |
| "`a is b` works for equal numbers." | Only within the cached −5..256 small-int range; `257 is 257` is often False. Use `==`. §1.3. |
| "The slow part of my code is the ugly-looking part." | The profiler disagrees ~70% of the time. Measure; the hotspot is where `ncalls × per-call` is largest. §1.5. |
| "Rewrite the whole service in Rust to go fast." | Rewrite the *profiled hot path* only (numpy/Cython/pyo3); the rest is I/O-bound and Rust wouldn't help. §1.9. |
| "Type hints make Python run faster." | In CPython they're (mostly) ignored at runtime — they help *tooling and humans*. (Meta's Static Python and specialization change this, §1.7; standard CPython does not.) |
| "The GIL makes Python useless for concurrency." | It's released during I/O and in native code, so I/O-bound concurrency works fine; it bites only pure-Python CPU parallelism. §1.6, Day 20. |
| "A native extension is automatically parallel." | Only if it *releases the GIL*. A C loop that holds it blocks every other thread — and the event loop. §1.9, Part 3. |

## 1.13 Interview & practice questions (Part 1)

1. Walk the five stages from `x = a + b` text to a running result, naming each. *(tokenizer → PEG parser → AST → compiler → bytecode → `ceval` loop.)*
2. Why is a Python `int` 28 bytes and a C `int` 8? Break down the 28. *(refcount 8 + type ptr 8 + ob_size 8 + one 30-bit digit 4.)*
3. `a = 257; b = 257; a is b` — True or False, and why? *(Usually False; 257 is outside the −5..256 small-int cache, so two separate objects.)*
4. What does `LOAD_FAST` do that `LOAD_GLOBAL` doesn't, and why does it matter in a loop? *(Array index vs dict lookup(s); hoist globals to locals in hot loops.)*
5. Your endpoint does a 5 ms DB query and 0.3 ms of Python. Where do you optimize, and why isn't it the language? *(The I/O — cache/index/batch; Python is ~6% and hidden behind the wait.)*
6. Name three things that make a single `a + b` cost ~100× a C add. *(Bytecode dispatch, runtime type discovery via type slots, boxing a new PyObject result.)*
7. You have a CPU-bound hot path in a Python service. Give the escape-hatch ladder, cheapest first. *(algorithm/stdlib → numpy vectorize → Cython/pyo3 native ext → process pool → sidecar service.)*
8. Pick a language for a microsecond-latency trading engine and defend it against Python. *(Rust: no GC pauses, no interpreter jitter, no GIL; Python's boxed objects and GC/refcount timing are disqualifying.)*
9. What did PEP 659 change, and how would you *see* it in `dis` output? *(Specializing adaptive interpreter; inline CACHE entries appear as offset-gaps / `show_caches=True`.)*
10. Why did Instagram disable the garbage collector, and what upstream feature later addressed the same problem? *(Refcount/GC writes dirtied shared COW pages across forked workers; immortal objects, PEP 683.)*

---

# PART 2 — AGENTIC AI

> Treating the backend (the interpreter, the network stack, the HTTP client) as a black box from Part 1, we ask one question of the agent: *where, if anywhere, does Python's per-operation cost actually show up in an LLM agent?* The answer sharpens §1.6 to its extreme — and finds the one place it bites.

## 2.1 Why "Python is slow" is a non-issue for an LLM agent

**Depth: [CORE]**

### Intuition

An LLM agent is the *most* I/O-bound program you will ever write. Its central act — call the model — is not a computation your process performs; it's a network request to a GPU cluster somewhere else that takes **hundreds of milliseconds to tens of seconds** to answer (Day 1's pyramid, bottom row: the LLM call is the slowest thing in this entire course, ~10⁷–10⁸× a bytecode op). Everything your Python code does around that call — build the messages list, JSON-encode the request, parse the response, decide the next step, format a tool result — is **microseconds** of interpreter work. So the interpreter's overhead isn't 15% here (the CRUD case, §1.6); it's a fraction of a *percent*. If a backend is a slow chef waiting on 2-day mail-order, an agent is a slow chef waiting on a spacecraft to return from Mars.

The reframe from §1.6 doesn't just hold for agents — it holds *harder*. "Should I write my agent in a faster language for performance?" is, in almost every case, a category error: you'd be optimizing the 0.1% while the 99.9% (the model call) is untouched and untouchable from your language choice.

### Analogy — the ludicrously fast waiter and the slow kitchen

You hire a waiter who's 100× faster than any human at taking orders and carrying plates. But the kitchen takes 30 minutes per dish, always. Your restaurant's throughput and the diner's wait are set *entirely* by the kitchen; the waiter's superhuman speed changes nothing about when food arrives. Rewriting your agent's orchestration in Rust is hiring an even faster waiter for a kitchen that still takes 30 minutes.

**Where the analogy breaks:** a fast waiter *does* matter in one way — they can serve *more tables* while the kitchen cooks. Translated: fast, non-blocking orchestration lets one process juggle *many concurrent agent sessions* during their model-call waits (async — Day 21). That concurrency win is real and important — but it comes from **not blocking during the wait** (an architecture question), not from **executing bytecode faster** (a language question). Python's async does this perfectly well. So even the one place "speed" helps an agent is about concurrency, not interpreter throughput — which is exactly §1.6's point restated.

### Worked example — the wall-clock budget of one agent turn

A single ReAct turn (Day 24): assemble request → call model → parse → run a tool → assemble again.

| Step | Nature | Time |
|---|---|---|
| Build `messages` list, serialize tool schemas to JSON | Python + C (json) | ~0.5–2 ms |
| **`client.messages.create(...)` — the model call** | **network + GPU inference** | **~1,000–8,000 ms** |
| Parse the JSON response, detect `stop_reason` | Python + C | ~0.2 ms |
| Run the tool (say, a DB lookup) | I/O wait | ~5 ms |
| Append `tool_result`, loop | Python | ~0.3 ms |
| **Total** | | **~1,010–8,010 ms** |
| **Python's share** | ~1–2.5 ms | **~0.02–0.25%** |

Rewrite the orchestration in the fastest language on earth and you save ~1–2 ms out of ~2,000+. **It is not measurable against the model call's variance.** The engineering effort that *actually* moves an agent's latency and cost is elsewhere entirely: fewer/shorter model calls (prompt design, caching, smaller models for easy steps), streaming so the user sees tokens sooner, and concurrency so one server handles many sessions during their waits. None of those is a language-speed problem.

### Under the hood — the SDK is mostly `await`

Peek at what an Anthropic/OpenAI SDK call *is* mechanically: it builds a dict, serializes it to JSON, opens (or reuses) an HTTPS connection via `httpx`, sends the bytes, and **`await`s the socket** — parking the coroutine on the event loop (Day 5's `epoll`, Day 21's loop) for the entire multi-second model wait, during which your process is free to do other sessions' work. The "agent" is a thin Python loop wrapped around a network client that spends ~99.9% of its time blocked on a socket. That's the whole reason Python — a "slow" language — is the *default* language of the entire AI ecosystem: the work isn't in Python, it's on the wire and on someone else's GPU. Cross-reference Part 1 §1.6: same physics, one more order of magnitude of wait.

---

## 2.2 Where Python overhead *does* bite an agent — token and embedding math

**Depth: [CORE]**

### Intuition

There is exactly one regime where an agent's Python code stops being noise: **tight loops over lots of data with no I/O to hide behind** — §1.6's "juice bar," now inside an agent. The usual suspects:

- **Embedding similarity for RAG (retrieval).** To answer "which of my 100,000 document chunks are most relevant?", you compute the cosine similarity between the query's embedding vector and every stored vector. Done as a pure-Python nested loop (`for each vector: for each of 1536 dims: multiply-add`), that's ~150 million boxed-float operations per query — each one paying the §1.4 PyObject tax. It can take **seconds**, and there's no network wait to shadow it: this is CPU, all yours.
- **Token processing / re-ranking / scoring.** Iterating over tokens to mask, weight, or score them; BM25 over a big corpus; reshaping large tensors by hand. Same shape: many small numeric ops in Python.
- **Chunking / parsing large documents** before embedding — regex and string ops over megabytes.

This is where you reach for **numpy (or a real vector store)**, and it matters for two reasons at once: numpy runs the loop in compiled C (no per-element PyObject overhead, ~50–200×), **and it releases the GIL** during the computation, so it doesn't block other coroutines/threads — which, as Part 3 shows, is the difference between one slow retrieval and *every concurrent agent session* stalling.

### Analogy — counting a warehouse by hand vs by forklift-scale

Pure-Python vector math is counting a warehouse of a million boxes one at a time, writing each tally on a fresh sticky note (a new PyObject per operation). numpy is a forklift-mounted scanner that sweeps a whole aisle in one motion and never writes sticky notes — the data lives in one dense C array, and one BLAS routine sweeps it. Same arithmetic, ~100× the throughput, because the *per-item ceremony* vanished.

**Where the analogy breaks:** the forklift only helps for *bulk, uniform* work laid out in dense rows. If your operation is irregular — branching per element, calling back into Python for each item (`np.vectorize` of a Python function, or a Python `for` over a numpy array) — you get numpy's *memory* layout but keep Python's *per-op* cost, and you've lost the win. The speed comes from staying in vectorized/compiled land for the *whole* loop; drop back into a Python callback per element and the forklift is idling while you count by hand anyway.

### Runnable example — cosine similarity over an embedding store: pure Python vs numpy

```python
# embed_similarity.py — any OS.  # pip install numpy
import time, random, math
import numpy as np

N, DIM = 50_000, 768                       # 50k chunks, 768-dim embeddings
random.seed(0)
store_py = [[random.random() for _ in range(DIM)] for _ in range(N)]  # list-of-lists
query_py = [random.random() for _ in range(DIM)]

# ---------- Pure Python: nested loops, boxed floats, PyObject tax per op ----------
def cosine_py(a, b):
    dot = na = nb = 0.0
    for x, y in zip(a, b):                 # DIM iterations, each a boxed-float multiply
        dot += x * y; na += x * x; nb += y * y
    return dot / (math.sqrt(na) * math.sqrt(nb) + 1e-9)

t = time.perf_counter()
top_py = max(range(N), key=lambda i: cosine_py(query_py, store_py[i]))
py_time = time.perf_counter() - t
print(f"pure python : best={top_py}  {py_time:6.3f}s")

# ---------- numpy: one dense C array, one vectorized BLAS sweep, GIL released ----------
store_np = np.array(store_py, dtype=np.float32)          # (50000, 768) contiguous C block
query_np = np.array(query_py, dtype=np.float32)          # (768,)
t = time.perf_counter()
# normalize once, then a single matrix-vector product = all 50k dot products at once
store_norm = store_np / (np.linalg.norm(store_np, axis=1, keepdims=True) + 1e-9)
query_norm = query_np / (np.linalg.norm(query_np) + 1e-9)
sims = store_norm @ query_norm                           # (50000,) — one BLAS call
top_np = int(np.argmax(sims))
np_time = time.perf_counter() - t
print(f"numpy       : best={top_np}  {np_time:6.3f}s")
print(f"speedup     : {py_time/np_time:6.1f}x")
```

Representative output (**run it yourself**; absolute times are machine-specific, the *ratio* is the lesson):

```
pure python :  best=41287   2.9xx s        # seconds — this would gate every RAG query
numpy       :  best=41287   0.0xx s        # tens of milliseconds
speedup     :   ~80–150x
```

**Why this works, line by line.**

- `cosine_py` is the honest pure-Python version: a Python `for` over 768 dims, called 50,000 times = ~38 million iterations, each doing boxed-float multiplies and adds — every one paying §1.4's dispatch + boxing tax. There is no I/O; this is 100% CPU, so §1.6's "hidden in the shadow of the wait" does **not** apply. It's slow and you feel it.
- `np.array(store_py, dtype=np.float32)` copies the data *once* into a single contiguous C block of raw floats — **no PyObjects per element** (§1.3's overhead paid once for the whole array, not per number). This memory layout is half the win.
- `store_norm @ query_norm` is a matrix-vector product: **all 50,000 dot products in one compiled BLAS call**, looping in C at near-hardware speed, **with the GIL released** for the duration. That's the other half — and the released GIL is why this scales under concurrency (Part 3).
- The ~100× speedup turns a per-query cost of seconds into tens of milliseconds — the difference between a usable RAG pipeline and one that times out. And note the fix was §1.9 **rung 2** (vectorize) — no native extension written, no service split out, just `pip install numpy` and thinking in arrays. In real systems you'd climb one more rung to a purpose-built vector store (FAISS, a vector DB — Phase 4), which is C++/Rust doing this with an index so it's sub-millisecond even at millions of vectors. Same ladder, same reasoning.

**The agent lesson.** The model call (§2.1) is untouchable and dominant, so don't optimize your orchestration's Python. But the *local numeric work an agent does around retrieval* — similarity, re-ranking, tokenization — is exactly §1.6's CPU-bound regime and it *will* bite if you write it as Python loops. Reach for numpy/native there, and nowhere else. This is the whole calibration: **model call → leave it; array math → vectorize it; everything in between → it's noise, ignore it.**

---

## 2.3 Reading the SDK source — the once-per-phase exercise

**Depth: [WORKING]** — you must be able to read a provider SDK and locate its request/retry/stream machinery; you don't reimplement it.

### Intuition

The 100-day method (plan, step 5) says: *once per phase, read the real source of a tool you use.* For the agentic phase, that tool is your LLM provider's SDK (`anthropic`, `openai`, or `google-genai`). The payoff is that it *proves* §2.1 with your own eyes: the SDK is not doing anything computationally heavy — it's a **typed, ergonomic wrapper around an HTTP client** (`httpx`) that builds a request, handles auth/retries/streaming, and `await`s a socket. Reading it dissolves the "magic" and confirms the agent's cost is on the wire.

### What to look for (a reading map, provider-agnostic)

When you `pip install anthropic` (or `openai`) and open the package source, hunt for these — the *shapes* are the same across providers; names differ, so **verify against the version you installed**:

1. **The client and its HTTP transport.** A base client (e.g. an internal `_base_client` module) constructs an `httpx.Client` / `AsyncClient` with a **connection pool**, base URL, timeout, and auth headers. This is the §1.7-of-Day-5 lesson resurfacing: *reuse one pooled client*, don't make a new one per request (new client = new TLS handshake + leaked sockets — the exact fd-leak bug from Day 5 §1.7, now with an SDK).
2. **Request construction.** A method like `messages.create(...)` (Anthropic) / `responses.create(...)` (OpenAI) builds a dict/typed model, serializes it to JSON, and POSTs it. Almost all the "code" is *validation and typing*, not computation. Notice how little CPU work happens before the network call.
3. **Retries and timeouts.** Look for exponential backoff with jitter on `429`/`5xx`/connection errors, and a max-retries setting. This is production HTTP hygiene (Day 11/Day 13) baked into the SDK — know the defaults so you can reason about tail latency and cost.
4. **Streaming.** The streaming path (`messages.stream(...)` / `stream=True`) parses **Server-Sent Events** off the socket incrementally and yields events. This is Day 5's "one thread, many descriptors" and an `async` generator — the tokens arrive over a long-lived read; your loop `await`s each chunk. It's the clearest demonstration that the SDK is I/O plumbing.
5. **The async surface.** Both providers ship `Async*` clients. That they exist at all is the §2.1 point: the whole design assumes you'll `await` the long model wait and do other work meanwhile.

### Worked example — inspect the installed SDK without leaving Python

```python
# read_sdk.py — # pip install anthropic   (or: openai)
# A harmless reconnaissance script: WHERE does the SDK live, and is it httpx underneath?
import anthropic, inspect, pathlib

print("version :", anthropic.__version__)                       # pin this in your notes
pkg = pathlib.Path(inspect.getfile(anthropic)).parent
print("source  :", pkg)                                         # go read these files
print("modules :", sorted(p.name for p in pkg.glob("*.py"))[:12])

# Is the transport really httpx? (confirms §2.1: it's an HTTP client under the types)
import httpx
print("httpx a dependency? ->", "httpx" in {d.split()[0] for d in
      __import__("importlib.metadata", fromlist=["requires"]).requires("anthropic") or []})
```

Representative output (**verify against your installed version — the module layout drifts across SDK releases; do not treat these filenames as stable**):

```
version : 0.x.y
source  : /…/site-packages/anthropic
modules : ['__init__.py', '_base_client.py', '_client.py', '_streaming.py', ...]
httpx a dependency? -> True
```

**Why this exercise works.** `inspect.getfile` locates the package on disk so you can *actually open and read it* (that's the assignment — read `_base_client.py` / the streaming module, not this script's output). Confirming **`httpx` is the dependency** is the whole point: the SDK's "intelligence" is types, retries, and stream parsing over a standard async HTTP client — there is no heavy computation, which is precisely why §2.1 holds and why Python's per-op speed is irrelevant to an agent's performance. **Honesty caveat:** exact module names (`_base_client`, `_streaming`) and the `requires()` output vary by SDK version and packaging; the reconnaissance *method* is stable, the specific filenames are not — run it and read what's actually there. And when you write real model calls (Day 23+), consult the `claude-api` skill for current model IDs and parameters rather than copying from memory (per the course's stack conventions).

---

## 2.4 System design (agentic) — where to spend the optimization budget on an agent service

**Depth: [WORKING]** (Part 2's tier for a design that leans on Part 1's framework.)

**The problem.** Your agentic RAG service is too slow and too expensive. p95 latency is 9 s; cost is $0.04/request. A junior engineer proposes rewriting the orchestration layer in Go "because Python is slow." Design the *actual* optimization plan, using this Part's calibration.

**Requirements → the key decision.** First, *measure the budget* (§2.1's table, for your service). Suppose profiling a live request (py-spy, Part 3) shows: model calls 7.8 s (three sequential calls), embedding+rerank 0.9 s (pure-Python loops), orchestration/serialization 0.05 s. The Go rewrite targets the **0.05 s** — 0.5% — and would save maybe 0.03 s while costing weeks and the Python ecosystem. **Reject it.** The budget lives in two places, and neither is interpreter speed:

1. **The model calls (7.8 s, and 100% of the $ cost).** Attack with: caching (identical/near-identical requests — prompt caching, semantic cache); fewer calls (collapse a 3-call chain into 1 where possible; use a smaller/cheaper model for easy steps — a router); shorter contexts (trim history — Day 43; you pay per token); streaming (doesn't cut total time but cuts *time-to-first-token*, the latency users feel); and concurrency (fire independent tool/model calls in parallel with `asyncio.gather` — Day 22 — instead of sequentially).
2. **The embedding/rerank loops (0.9 s).** This *is* the §1.6 CPU-bound regime (§2.2). Vectorize with numpy (§1.9 rung 2), or move to a real vector store (rung 4/5). This is the *only* place a "make Python faster" instinct is correct — and the fix is native math libraries, not a language rewrite.

**The trade-off made.** We spend the budget on I/O reduction (caching, fewer/parallel calls) and on the one genuine CPU hot path (vectorize retrieval) — and we *keep Python* for the orchestration, because its cost is 0.5% and its ecosystem (SDKs, RAG libs, async) is the reason the service exists. Cost drops via caching + smaller-model routing; latency drops via parallelism + streaming + vectorized retrieval. **Why this over the rewrite:** the rewrite optimizes the one thing that isn't the bottleneck, which is §1.8's classic failure mode wearing an AI hat.

**Failure modes.** Optimizing orchestration Python (wasted effort). Vectorizing retrieval but then blocking the event loop with the *synchronous* numpy call inside an async handler under high concurrency (Part 3 — run it in a thread so the released GIL actually buys parallelism). Caching model responses without a correctness story (stale/nondeterministic answers). Cross-reference: Day 22 (async patterns, gather/semaphore), Phase 4 (vectors/RAG), Day 43 (context management).

---

## 2.5 In production (Part 2, condensed)

- **Instrument the wall-clock budget, per request.** Break every agent turn into model-call time vs tool-I/O time vs local-CPU time (numpy/parsing) vs orchestration. You cannot make the right call without this split — and it almost always shows the model call dominating, which *frees* you from micro-optimizing Python.
- **Top failure mode (mirrors Part 1's fd-leak):** creating a new SDK/HTTP client per request instead of reusing one pooled async client. It re-does TLS handshakes (latency) and leaks sockets (`Too many open files` — Day 5 §1.7). Instantiate the client once at startup.
- **The one place to reach for native:** local numeric hot paths (embeddings, rerank, tokenization). Use fast native libraries (numpy, the Rust-based `tokenizers`, a vector store) and ensure they *release the GIL* so concurrency survives (Part 3). Everywhere else, Python's speed is a non-issue — spend the effort on prompt/caching/concurrency instead.
- **Cost note:** an agent's dollar cost is tokens, not CPU. The cheapest optimization is almost always "call the model less / with fewer tokens," never "run bytecode faster."

---

# PART 3 — THE BRIDGE

> Parts 1 and 2 are the same physics at two scales: a backend hides Python's cost behind a millisecond DB wait; an agent hides it behind a multi-second model wait. Part 3 is where the Python backend *serving* an agent and the agent's own numeric work become one process — and share one failure mode. No new concepts here; only the dependency map and the seam where they fail together.

## 3.1 The dependency map — native escape hatch vs premature optimization

The whole day reduces to one decision, and it's the same on both sides of the platform. A Python backend serving an agent is a stack of regimes; the escape hatch belongs in exactly one of them.

```
   AGENT REQUEST ──► Python orchestration (FastAPI + SDK loop)
                     │
                     │  regime A: I/O-bound glue  ── Python's cost = NOISE
                     │  (build request, route, serialize, await)   ← never optimize the language
                     │
                     ├─►  MODEL CALL (network + GPU) ───────────── dominates wall time & $
                     │    optimize: caching, fewer/parallel calls,   ← Part 2, Day 22/43
                     │              smaller models, streaming
                     │
                     ├─►  TOOL I/O (DB, HTTP, files) ───────────── Day 5 / Day 11 / Day 13
                     │    optimize: indexes, pools, async
                     │
                     └─►  LOCAL NUMERIC HOT PATH ───────────────── regime B: CPU-bound
                          (embedding similarity, rerank, tokenize)   ← THE escape hatch lives here
                          optimize: numpy / native / vector store     §1.9 rungs 2–5, §2.2
                          (and it must RELEASE THE GIL — see §3.3)
```

**The rule that unifies Part 1 and Part 2:** *reach for a native escape hatch only in regime B — a CPU-bound hot path the profiler has indicted — and treat any effort spent optimizing regime A (the I/O-bound glue) as premature optimization.* A CRUD backend (§1.6) and an agent (§2.1) are both ~99% regime A; their one difference is the *size* of the wait they hide behind (ms vs seconds). The escape hatch decision is identical: **profile → find regime B → climb the §1.9 ladder only as far as the data forces.**

- **When a Python backend serving an agent genuinely needs native code:** it has a real regime-B loop — similarity over a large in-process vector set, reranking a big candidate list, heavy tokenization/parsing, a numeric transform in a tool. Then: numpy (rung 2) → a vector store or a Rust/Cython kernel (rungs 3–5).
- **When it's premature:** the profiler shows the time is in the model call, tool I/O, or serialization. Rewriting the orchestration (or the whole service) in Go/Rust here optimizes 0.5% and is the §1.8 anti-pattern. The correct "make it faster" is Part 2's list (cache, fewer/parallel/shorter model calls), none of which is a language change.

## 3.2 Profiling applies identically — but use the right tool for a *live* agent

§1.5 taught `cProfile` for finding the real hotspot offline; the same discipline ("measure, don't guess — the bottleneck is never where you think") applies to a live agent service, but the *tool* changes because the setting changes.

| | `cProfile` (§1.5) | `py-spy` (for live services) |
|---|---|---|
| How it works | **deterministic** — instruments every call | **sampling** — peeks at the stack ~100×/sec |
| Overhead | high (rivals cheap functions' runtime) | ~negligible; safe on production traffic |
| Needs code change / restart? | yes (wrap or `-m cProfile`) | **no** — attaches to a running PID |
| Sees time spent in native/`await`? | poorly | yes (`--native`, and it shows all threads) |
| Best for | reproducible offline profiling, unit-level | *"prod is slow right now, why?"* → flamegraph |

For a live agent, `py-spy dump --pid <pid>` gives an instant stack of every thread (great for "why is it hung?" — usually a blocking call, §3.3), and `py-spy record -o out.svg --pid <pid>` produces a flamegraph over a window. Because it's a sampling profiler reading another process's memory, it needs no instrumentation and barely perturbs the target — the same reason you'd never leave `cProfile` on in production but *would* leave a sampling profiler running. The *question* is identical to §1.5 ("where does the time actually go?"); the answer for an agent is nearly always "the model call" (regime A — leave it) with the occasional genuine regime-B loop the flamegraph makes obvious. Cross-reference §1.5 and Part 2 §2.4's "measure the budget first."

## 3.3 The shared failure mode — a CPU-bound tool blocks the event loop

This is where Part 1 and Part 2 fail *together*, and it's the single most important operational lesson of the day. It previews **Day 21 (asyncio internals)** and reuses **Day 5 §1.6's analogy-break** ("a single un-served table freezes the whole restaurant") and **Day 20 (the GIL)**.

**The setup.** Your agent service is `async` (FastAPI, async SDK) so one process can juggle hundreds of concurrent agent sessions during their multi-second model waits (§2.1) — this is the *entire* reason it scales. The event loop (Day 5's `epoll`, Day 21's loop) runs on **one thread**, cooperatively: each coroutine runs until it hits an `await`, then yields so another can run. The contract is *nothing may run for long without awaiting.*

**The break.** One agent *tool* does regime-B work **synchronously** on the event loop thread — computes cosine similarity over 100k embeddings in a pure-Python loop (§2.2), or parses a 50 MB PDF, or does a numpy call that runs for 400 ms. For that whole time the coroutine **never awaits**, so the loop cannot switch. **Every other concurrent session freezes** — their model responses have arrived on their sockets, but the one thread is busy multiplying floats. p99 latency for *all* endpoints spikes to the length of the CPU stall. This is Day 5's "blocking the event loop," now caused by an agent's own numeric tool, and it is the exact postmortem shape the plan flags for Day 21 ("one sync call in an async handler → p99 to 30 s for everything").

```
  healthy async loop                     one CPU-bound tool on the loop thread
  ────────────────────────               ───────────────────────────────────────
  S1 await model ─┐                       S1 await model ─┐
  S2 await model  │ loop switches         S2 ─► cosine_py(100k) ── 400ms, NO await ──┐
  S3 await tool   │ among sessions        S3 await model  │  FROZEN                   │
  loop: 0% CPU, all progress              S4 await model  │  FROZEN — response ready, │
                                          loop: 100% CPU on ONE session, 0 progress ──┘
```

**Why the GIL makes it worse *and* the fix works.** Even if you spawn threads, the GIL (Day 20) means pure-Python CPU work in one thread still blocks Python bytecode in others — so naive threading doesn't save you for *pure-Python* hot loops. The fix has two parts, both from this day:
1. **Get the CPU work off the loop thread:** `await asyncio.to_thread(cpu_fn, ...)` (or a `ProcessPoolExecutor` via `loop.run_in_executor`) so the coroutine *yields* while the work runs elsewhere and the loop keeps serving other sessions (Day 21/§1.9 rung 4).
2. **Make the work *release the GIL*** so the offloaded thread runs truly in parallel: use numpy/native (§2.2), which drops the GIL during the C computation. Pure-Python-in-a-thread yields the loop (fixing the freeze) but still serializes on the GIL; native-in-a-thread both yields *and* parallelizes. **This is why §2.2 insisted the fix is numpy specifically, not just "move it to a thread"** — the two halves (yield the loop + release the GIL) are what make it work, and they're the through-line from Part 1's GIL (§1.6) to Part 2's embedding math (§2.2) to here.

**The dependency, stated once.** The agent (Part 2) depends on the backend's async event loop (Part 1 / Day 5) to hide model-call latency; the backend depends on the agent's tools *not* doing un-offloaded CPU-bound work on that loop. Break the contract from either side and *both* fail together — the agent's concurrency collapses and the backend's p99 explodes — for the exact reason this whole day taught: Python CPU work is slow (§1.4) and single-threaded under the GIL (§1.6), so it must never sit on the one thread that everything else is waiting on. Full treatment: Day 21.

---

# Topic-wide wrap-up

## Cheat Sheet — Python under the hood, at a glance

**The pipeline (§1.1–1.2):** source → **tokenizer** → **PEG parser** → **AST** → **compiler** → **bytecode** (cached in `.pyc`) → **`ceval` loop** (a stack machine, `Python/ceval.c`). Compiler runs once; the `ceval` loop is where runtime cost lives.

**The object model (§1.3):** *everything is a PyObject* reached by pointer. Header = `ob_refcnt` (8B) + `ob_type` (8B); var-objects add `ob_size` (8B). A small `int` = 24B header + 4B digit = **28 bytes** (`getsizeof(0)`=24, `getsizeof(1)`=28). Caches: small ints **−5..256** shared (`257 is 257`→False); string literals interned.

**The cost (§1.4):** dynamic typing ⇒ every op is a runtime lookup. `a+b` = type-slot dispatch + unbox + add + **box a new object**; the add is ~1% of the work. `LOAD_FAST` (array index) ≪ `LOAD_GLOBAL`/`LOAD_ATTR` (dict lookups) — hoist to locals in hot loops.

**Why "slow" rarely matters (§1.6):** backends are **I/O-bound**; interpreter is often <15% of wall time, hidden behind DB/network waits. It bites only in **CPU-bound loops with no I/O to hide behind** → reach for native code that *releases the GIL*.

**Tools:** `dis.dis(f)` (read bytecode), `dis.dis(f, show_caches=True)` (see PEP 659 inline caches), `sys.getsizeof(x)` (object bytes), `cProfile` (offline hotspot), **`py-spy`** (live/prod hotspot), `timeit` (micro-bench).

**Escape-hatch ladder (§1.9), cheapest first:** algorithm/stdlib → **numpy/vectorize** → Cython/Rust-via-pyo3 native ext → process pool → sidecar service. Climb only as far as the profiler forces.

**Language by workload (§1.8):** CRUD API → **Python** (I/O-bound); ML gateway → **Python** glue over native math; log processor → **Go/Rust** (CPU-bound); trading engine → **Rust** (CPU + latency tail, no GC pauses).

**Agent-specific (Part 2):** model call = ~10⁷–10⁸× a bytecode op, so orchestration Python is **noise** (§2.1) — optimize caching/parallel/shorter calls instead. The one CPU regime that bites: **embedding/rerank/token math** → numpy/vector store (§2.2). SDK = typed wrapper over `httpx` — reuse one pooled client (§2.3).

**Shared failure (§3.3):** a CPU-bound tool run **synchronously on the async loop** freezes every concurrent session. Fix = `asyncio.to_thread(...)` **and** native code that releases the GIL. (Day 21.)

## Build This — definition of done

Build a `python-under-the-hood/` folder (commit each part). **Done when you can, from memory, produce and *explain*:**

1. **`pipeline.py`** — print the tokens, AST, and bytecode for `total = a + b` (§1.2). *DoD:* you can name what each stage produced and why module-level code emits `LOAD_NAME` while a function emits `LOAD_FAST`.
2. **`sizes.py`** — print `sys.getsizeof` for `0, 1, 2**30, 2**100, "", "a"` and reproduce the 28-byte breakdown in a comment (§1.3). *DoD:* you can explain each number from header + digits, and demonstrate `256 is 256` vs `257 is 257`.
3. **`dis_read.py`** — `dis.dis()` a 3-line function and write the "read aloud" for every opcode (§1.5). *DoD:* you can point to the offset-gap and say "that's a hidden inline cache."
4. **`fast_vs_global.py`** — bytecode + `timeit` for `LOAD_GLOBAL` vs a hoisted local; report *your* ratio (§1.5). *DoD:* you can explain the ratio from array-index vs dict-lookup, and why 3.11+ narrowed it.
5. **`profile_me.py`** — `cProfile` a script whose real hotspot is *not* the obvious line; find it by `tottime` (§1.5). *DoD:* you predicted wrong first, then let the profiler correct you.
6. **`embed_similarity.py`** — pure-Python vs numpy cosine similarity over 50k×768; report *your* speedup (§2.2). *DoD:* you can explain the win as "dense C array + one BLAS sweep + GIL released," and connect it to §3.3.

**Stretch:** run `read_sdk.py` (§2.3), open the installed SDK's base-client and streaming modules, and confirm in one sentence that "the agent's cost is on the wire, not in Python."

## Active Recall & Self-Test (answer from memory)

1. Trace `x = a + b` through all five compiler/interpreter stages, naming each artifact.
2. Why is a Python `int` 28 bytes? Break it down. Why does `getsizeof(0)` differ?
3. Explain, mechanistically, why `a + b` costs ~100× a C add. Which part is the actual arithmetic?
4. Your endpoint = 5 ms DB + 0.3 ms Python. A teammate wants to rewrite it in Rust. Respond with numbers.
5. Give the escape-hatch ladder cheapest-to-heaviest and one sentence on when each is right.
6. Pick languages for the four §1.8 services and defend the trading engine's choice against Python.
7. For an LLM agent, where is Python's speed irrelevant and where does it bite? Give the ratio that settles it.
8. A CPU-bound agent tool spikes p99 for *every* session. Explain the mechanism and the two-part fix.

**60-second teach-back:** *"Python compiles your code to bytecode once, then a C loop walks it one instruction at a time, and every value — even the number 1 — is a 28-byte object it must dereference to know its type, so each operation is 10–100× a C one. That sounds fatal but almost never is, because backends and agents spend 99% of their time waiting on a database or a model — the interpreter runs in the shadow of that wait. It only bites when you have a tight CPU loop with nothing to wait on, like similarity math over embeddings — and there you drop to numpy, which runs compiled C and releases the GIL. The whole skill is knowing which regime you're in."*

## Spaced-Repetition Flashcards

- **Q:** What are CPython's five stages from source to running? → **A:** tokenizer → PEG parser → AST → compiler → bytecode → `ceval` loop.
- **Q:** Bytes in a small Python `int`, and the breakdown? → **A:** 28 = refcount 8 + type ptr 8 + ob_size 8 + one 30-bit digit 4.
- **Q:** `257 is 257`? Why? → **A:** Often False — outside the cached −5..256 small-int range. Use `==`.
- **Q:** `LOAD_FAST` vs `LOAD_GLOBAL`? → **A:** Array index into locals vs dict lookup (globals then builtins). Hoist to locals in hot loops.
- **Q:** Three costs that make a Python op slow? → **A:** Bytecode dispatch, runtime type discovery, boxing a new PyObject.
- **Q:** When does "Python is slow" actually matter? → **A:** CPU-bound loops with no I/O to hide behind; fix with native code that releases the GIL.
- **Q:** How slow is an LLM call vs a bytecode op? → **A:** ~10⁷–10⁸× — so agent orchestration Python is noise.
- **Q:** The one CPU regime that bites an agent, and the fix? → **A:** Embedding/rerank/token math → numpy / vector store.
- **Q:** `cProfile` vs `py-spy`? → **A:** Deterministic (offline, high overhead) vs sampling (live/prod, negligible overhead, no code change).
- **Q:** What did PEP 659 do? → **A:** Specializing adaptive interpreter (3.11) — opcodes self-specialize using inline caches.
- **Q:** Escape-hatch ladder? → **A:** algorithm/stdlib → numpy → Cython/pyo3 → process pool → sidecar service.
- **Q:** Why does a CPU-bound tool freeze an async agent, and the fix? → **A:** It never `await`s, so the one loop thread can't switch; offload with `to_thread` **and** use GIL-releasing native code.

## Primary Sources (verify against your version)

- **CPython docs:** `dis`, `ast`, `tokenize`, `sys.getsizeof`, `cProfile`/`profile`, `timeit`, `gc` — docs.python.org.
- **The data model:** `docs.python.org/3/reference/datamodel.html` (objects, types, special methods).
- **CPython source:** `Python/ceval.c` (the interpreter loop), `Python/compile.c`, `Objects/longobject.c` (int + `_PyLong_SmallInts`), `Objects/abstract.c` (`PyNumber_Add`), `Include/object.h` (`PyObject`/`PyVarObject`).
- **PEPs:** 617 (new PEG parser), 659 (specializing adaptive interpreter), 683 (immortal objects), 703 (no-GIL, experimental), 744 (JIT, experimental), 709 (comprehension inlining). **Statuses drift — check the "What's New" for your version.**
- **Books:** *Fluent Python* (Ramalho) — objects, the data model, and CPython internals (the plan's Python-internals anchor); *CPython Internals* (Shaw) for the source-code tour.
- **Case studies:** Dropbox eng blog, *"Rewriting the heart of our sync engine"* (2020); Instagram eng, *"Dismissing Python Garbage Collection at Instagram"* (2017); Meta's **Cinder** (`github.com/facebookincubator/cinder`); the "Faster CPython" project (`github.com/faster-cpython/ideas`).
- **Tooling:** `py-spy` (`github.com/benfred/py-spy`); numpy docs (vectorization, BLAS); `maturin`/`pyo3` (Rust extensions); Cython docs.
- **Latency context:** Day 1's "Latency Numbers Every Programmer Should Know" (Jeff Dean / Peter Norvig).

## Layered explanations

- **10-second:** Python compiles to bytecode and interprets it one PyObject at a time, so it's 10–100× slower per operation than C — but backends and agents spend 99% of their time *waiting* on I/O, so that rarely matters.
- **1-minute:** CPython turns your source into bytecode (tokenizer → AST → compiler) and runs it in a C stack-machine loop where every value, even `1`, is a 28-byte heap object it must dereference to learn its type. That runtime type discovery plus boxing is why a Python op costs ~100× a C one. But a server's wall time is dominated by database and network waits (and an agent's by the multi-second model call), so the interpreter runs in the shadow of the wait and its speed is a rounding error. The exception is CPU-bound loops with no I/O — log parsing, embedding math — where you drop the hot path to native code (numpy/Cython/Rust) that runs compiled and releases the GIL.
- **5-minute:** Add the mechanics: the `ceval` loop dispatches opcodes over a per-frame value stack; `LOAD_FAST` is an array index while `LOAD_GLOBAL`/`LOAD_ATTR` are dict lookups (hoist to locals in hot loops); small ints −5..256 and string literals are cached/interned; PEP 659 (3.11+) makes opcodes self-specialize via inline caches, narrowing old gaps. Profiling is mandatory because the hotspot is never where you guess (`cProfile` offline, `py-spy` live). Choose language *per service* from its physics: Python for I/O-bound CRUD/ML-glue, Go/Rust for CPU-bound log processing, Rust for latency-critical trading. For agents the model call is ~10⁷–10⁸× a bytecode op, so orchestration Python is noise — optimize caching/parallelism/shorter calls; the one place to go native is embedding/rerank/token math (numpy, releasing the GIL). The shared failure mode: a CPU-bound tool run synchronously on the async event loop freezes every concurrent session, fixed by offloading to a thread *and* using GIL-releasing native code.
- **Expert:** The reader can read a `dis` transcript including inline-cache offset-gaps and specialization opcodes; account for object memory from struct layout (`PyVarObject` + digits, immortal-object changes in 3.12) and reason about COW/refcount interactions across forked workers (Instagram/immortal objects); locate a hotspot with `cProfile`/`py-spy` and distinguish `tottime` from `cumtime`; apply the I/O-bound-vs-CPU-bound framework to pick a language per service and to place a native escape hatch at exactly the profiled regime-B loop, no further; and design an agent service that keeps Python for orchestration while vectorizing retrieval and offloading CPU-bound tools off the event loop with GIL-releasing native code — articulating throughout that interpreter overhead is only ever billed where there is no larger wait to hide behind, the single idea that unifies the backend and the agent.
