# Day P2 — Python Fundamentals II: Data Structures, Classes, Modules

> **What this covers:** the four workhorse collections (`list`, `dict`, `set`, `tuple`) and *when to use each*; comprehensions; classes and *why* they exist; modules & imports; exceptions; and the standard library you should never reinvent (`pathlib`, `json`, `collections`).
> **Prerequisite:** [Day P1](p1-python-fundamentals-i.md) — values, control flow, functions, f-strings, reading tracebacks.
> **The one idea that unites this day:** *every LLM/agent program you will ever write is made of exactly these parts — an API request is nested dicts and lists, a conversation is a list of dicts you append to, an agent is a class bundling state and behavior, tool results travel as JSON, and failures are exceptions you catch.* Learn the containers today; recognize them inside agent code tomorrow.
>
> **How to read this:** every concept runs the ladder **intuition → analogy (and where it breaks) → concrete worked example → diagram → under-the-hood**. When something feels abstract, jump to its "**Worked example**" or "**Runnable example**" block, run it, then re-read the theory.
>
> **Depth tiers:** **[CORE]** open every box · **[WORKING]** use it correctly, know the tradeoffs · **[AWARE]** know it exists and when to reach for it.
>
> *(P-days are prerequisite fundamentals: per the study plan, there are no `System design` / `Case studies` / `In production` blocks here — those begin at Day 1. Everything else — the full teaching ladder, runnable code, recall drills — applies in full.)*

---

# PART 1 — BACKEND: The Building Blocks of Every Python Program

## Overview & motivation — why "data structures" is the most important day of the prerequisite week

On Day P1 you worked with *single values*: one `int`, one `str`, one `bool` at a time. Real programs almost never deal with one thing. A web server holds *many* connected users. A word counter holds *many* words with *many* counts. An agent holds *many* conversation messages. The moment your program needs to hold "many," you need a **container** — a value whose job is to hold other values.

Before containers existed as language features, programmers managed raw blocks of memory by hand (this is what C programmers still do): "I have 400 bytes starting at address 0x7f3a; item number 7 lives at 0x7f3a + 7×4 bytes." That was fast but brutally error-prone — read one slot past the end and you corrupt memory or crash. Python's containers exist to solve that pain: they grow themselves, check their own bounds, and free their own memory, so you think about *your data* instead of *addresses*.

Python gives you four workhorse containers, and the single most valuable skill of this day is knowing **which one to reach for**:

| You need to store… | Reach for | One-line reason |
|---|---|---|
| Things **in order**, that will grow/shrink/change | `list` | ordered + mutable |
| A **lookup**: find a value *by name/key*, fast | `dict` | key → value, O(1) lookup — **the most important one** |
| "Have I seen this before?" / uniqueness | `set` | membership tests + automatic dedup |
| A small **fixed record** that never changes | `tuple` | immutable, unpackable, hashable |

The second half of the day gives you the tools to *organize* programs built from these containers: **classes** (bundle state + behavior into one object), **modules** (split a program across files), **exceptions** (handle failure without crashing), and the **standard library** (don't reinvent what ships with Python).

All examples verified on **CPython 3.12** (the standard Python interpreter, written in C — when we say "under the hood," we mean what CPython actually does). Exact byte sizes and hash values can differ across versions and machines; the *mechanisms* do not.

---

## 1. `list` — the ordered, growable sequence **[CORE]**

**Depth: [CORE]** — you will use lists in every program you ever write, and their performance behavior (what's fast, what's slow) matters constantly.

### Intuition — what problem it solves

You have several things, their **order matters**, and the collection will **change over time** (things get added, removed, replaced). Before growable lists, you had fixed-size arrays: you had to declare "this holds exactly 100 items" up front, and item 101 meant allocating a new bigger array and copying everything yourself. A Python `list` does all of that for you, invisibly.

```python
messages = ["hello", "how are you?", "goodbye"]   # order preserved
messages.append("wait, one more thing")            # grows itself
messages[0] = "HELLO"                              # mutable: replace in place
```

### Analogy — a numbered parking garage (and where it breaks)

A list is a **parking garage with numbered spots**: spot 0, spot 1, spot 2… You can drive straight to spot 7 without walking past spots 0–6 (that's *indexing*: `messages[7]`, instantly fast). New cars park in the next free spot at the end (`append`, fast).

*Where the analogy breaks:* if a car leaves spot 2 in a real garage, spot 2 just sits empty. In a Python list, removing item 2 (`del messages[2]`) makes **every car behind it shift forward one spot** — the list refuses to have gaps. That shifting is real work: removing from the *front* of a long list is slow, because everything behind it moves. This is the single most important performance fact about lists.

### Worked example — a conversation transcript, traced

Watch the exact state of the list after each operation:

```
messages = []                      # []
messages.append("Hi")              # ["Hi"]
messages.append("Hello!")          # ["Hi", "Hello!"]
messages.append("Bye")             # ["Hi", "Hello!", "Bye"]
messages[1]                        # "Hello!"        (index 1 = SECOND item; counting starts at 0)
messages[-1]                       # "Bye"           (negative index = count from the end)
len(messages)                      # 3
messages.insert(0, "SYSTEM: be nice")
                                   # ["SYSTEM: be nice", "Hi", "Hello!", "Bye"]
                                   #  ^ everything shifted right — O(n) work
messages.pop()                     # returns "Bye"; list is now
                                   # ["SYSTEM: be nice", "Hi", "Hello!"]
"Hi" in messages                   # True — but Python CHECKED EVERY ITEM to find out
```

That last line matters: `in` on a list is a front-to-back scan. On 10 items, fine. On 10 million, slow — and that is precisely the problem `set` (§3) and `dict` (§2) exist to solve.

### Visual — what a list actually is in memory

A CPython list does **not** store your objects inside itself. It stores an array of **pointers** (memory addresses) to objects that live elsewhere:

```
 the list object                       the actual objects (elsewhere in memory)
┌───────────────────────────┐
│ length: 3                 │
│ capacity: 4               │
│ pointer array:            │
│   [0] ──────────────────────────▶  str "Hi"
│   [1] ──────────────────────────▶  str "Hello!"
│   [2] ──────────────────────────▶  str "Bye"
│   [3] (spare slot)        │
└───────────────────────────┘
```

Two consequences fall out of this picture:

1. **A list can mix types** (`[1, "two", 3.0]`) — it only holds pointers, and a pointer to an `int` is the same size as a pointer to a `str`.
2. **Assignment copies the pointer, not the object.** `b = a` makes `b` point at *the same list*. Mutate through either name and both "see" it — the #1 beginner surprise, demonstrated in the runnable example below.

### Under the hood — dynamic array with over-allocation, and why `append` is "amortized O(1)"

The pointer array has a fixed size (its **capacity**). When you `append` and there's a spare slot, Python drops the pointer in — constant time. When the array is **full**, Python must allocate a *bigger* array and copy every pointer over. If it grew by exactly one slot each time, every append would trigger a copy — quadratic disaster. Instead, CPython **over-allocates**: it grows by roughly 12% extra headroom each time (the growth pattern is visible below), so copies become rarer and rarer as the list grows. Averaged over many appends, the occasional expensive copy is "paid off" by the many cheap ones — that's what **amortized O(1)** means: *usually* instant, *occasionally* a copy, *on average* constant time.

You can watch the over-allocation happen with `sys.getsizeof` (which reports the list object's own size, including its pointer array — not the pointed-to objects):

```
empty list:        56 bytes
after 1  append:   88 bytes   ← grew: room for 4 pointers
after 5  appends: 120 bytes   ← grew: room for 8
after 9  appends: 184 bytes   ← grew: room for 16
after 17 appends: 248 bytes   ← grew: room for 24
```

(Real output from CPython 3.12 on a 64-bit machine — exact numbers vary by version, the *staircase pattern* doesn't.) Notice the size only jumps at certain lengths: between jumps, appends are free because spare capacity already exists.

**Operation costs to memorize:**

| Operation | Cost | Why |
|---|---|---|
| `lst[i]` (index) | O(1) | pointer arithmetic: base address + i |
| `lst.append(x)` | O(1) amortized | spare slot usually exists |
| `lst.pop()` (from end) | O(1) | nothing shifts |
| `lst.insert(0, x)` / `lst.pop(0)` | O(n) | everything shifts |
| `x in lst` | O(n) | front-to-back scan |
| `lst.sort()` | O(n log n) | Timsort, in place |

### Runnable example — list operations, the shared-pointer trap, and the over-allocation staircase

```python
# lists.py — no installs needed. Run with:  python lists.py
import sys

# --- 1. the basics, on a transcript ---
messages = ["Hi", "Hello!"]
messages.append("Bye")
print(messages)              # order preserved, growable
print(messages[0], "/", messages[-1])
print(len(messages), "messages;", "Hi" in messages)

# --- 2. THE TRAP: assignment shares, it does not copy ---
a = [1, 2, 3]
b = a                        # b points at the SAME list object
b.append(4)
print("a is", a)             # a changed too!
c = a.copy()                 # .copy() (or a[:]) makes a NEW list
c.append(99)
print("a is still", a, "but c is", c)

# --- 3. watch over-allocation: size jumps in steps, not per-append ---
lst = []
prev = sys.getsizeof(lst)
print("empty:", prev, "bytes")
for i in range(20):
    lst.append(i)
    size = sys.getsizeof(lst)
    if size != prev:                      # only print when capacity grew
        print(f"len {len(lst)}: {size} bytes")
        prev = size
```

```bash
python lists.py
# -> ['Hi', 'Hello!', 'Bye']
# -> Hi / Bye
# -> 3 messages; True
# -> a is [1, 2, 3, 4]
# -> a is still [1, 2, 3, 4] but c is [1, 2, 3, 4, 99]
# -> empty: 56 bytes
# -> len 1: 88 bytes
# -> len 5: 120 bytes
# -> len 9: 184 bytes
# -> len 17: 248 bytes
```

**Why this works, line by line.** `messages.append("Bye")` drops a pointer into the next spare slot — no copying, which is why it's the list's signature cheap operation. `messages[-1]` works because negative indexes are translated to `len(lst) + i` before the pointer lookup. In section 2, `b = a` copies only the *reference* — both names point at one list object in memory, so `b.append(4)` is visible through `a`; `a.copy()` allocates a fresh pointer array and copies the pointers (a "shallow copy" — the *items* are still shared, which is fine for immutable items like ints). In section 3, `sys.getsizeof` measures the list object itself; the printout shows size changing only at lengths 1, 5, 9, 17 — those are precisely the appends where the capacity was exhausted and CPython allocated a bigger array with headroom. Every append *between* the jumps hit a spare slot: that's amortized O(1) made visible.

**When to use a list:** ordered data that changes — transcripts, queues of work (append + pop from the end), rows read from a file, results you're accumulating. **When not to:** membership testing on large data (use `set`), lookup by name (use `dict`), fixed records (use `tuple`).

---

## 2. `dict` — key → value, the most important structure in Python **[CORE]**

**Depth: [CORE]** — dicts are not just *a* data structure in Python; they are *the* data structure. Modules, classes, and objects are all dicts under the hood. Every JSON payload you'll ever touch becomes a dict. Learn this one to the metal.

### Intuition — what problem it solves

A list answers "give me item **number 3**." But most real questions are "give me the balance **for account 'vinay'**" or "what's the value **for the key `'model'`**?" — lookup **by name**, not by position. With only lists you'd store pairs and scan: check item 0's name, check item 1's name… O(n), painfully slow at scale. Before hash-based dictionaries were mainstream, that scan (or maintaining hand-sorted arrays with binary search) *was* how lookup tables worked. A `dict` gives you lookup by key in **O(1)** — constant time, whether it holds 10 entries or 10 million.

```python
balances = {"vinay": 120.0, "alice": 300.0}
balances["vinay"]          # 120.0 — instant, no scanning
balances["bob"] = 50.0     # add a new key → value pair
```

### Analogy — a coat check (and where it breaks)

A dict is a **coat check**: you hand over your coat, get ticket #47, and later the attendant goes *straight to hook 47* — no searching through coats. The ticket is the key; the coat is the value; the numbered hook is where hashing comes in (below).

*Where the analogy breaks:* a coat check issues you an arbitrary ticket. In a dict **you choose the key**, and the same key always leads to the same slot — hand in a new coat with ticket #47 and it *replaces* the old one (keys are unique; assigning to an existing key overwrites). Also, real coat checks don't care what a ticket is made of; dict keys **must be hashable** (immutable, in practice) — strings, numbers, tuples yes; lists no. Why becomes clear under the hood.

### Worked example — an API request is a dict (trace every access)

This is a real Anthropic API request body, and it is *nothing but* dicts and lists — Part 2 will send it for real:

```python
request = {
    "model": "claude-opus-4-8",
    "max_tokens": 1000,
    "messages": [
        {"role": "user", "content": "What is a dict?"},
    ],
}

request["model"]                     # "claude-opus-4-8"
request["messages"]                  # a LIST of dicts
request["messages"][0]               # {"role": "user", "content": "What is a dict?"}
request["messages"][0]["role"]       # "user"   ← dict inside list inside dict
request["temperature"]               # KeyError: 'temperature'  ← missing key CRASHES
request.get("temperature")           # None                     ← .get returns None instead
request.get("temperature", 1.0)      # 1.0                      ← or a default you choose
```

Read `request["messages"][0]["role"]` right-to-left as instructions: take `request`, look up key `"messages"` (get a list), take index `0` (get a dict), look up key `"role"`. Chained access on nested containers is 80% of working with API data.

### Visual — hashing, buckets, and what a lookup actually does

```
  key "vinay"
      │
      ▼
  hash("vinay")  ──▶  e.g. -8829424373894...   (a big deterministic number)
      │
      ▼  hash % table_size (take last few bits)
  slot 3
      │
      ▼
 ┌────────────────────────────────────────────┐
 │ slot 0:  <empty>                           │
 │ slot 1:  hash / "alice" / 300.0            │
 │ slot 2:  <empty>                           │
 │ slot 3:  hash / "vinay" / 120.0   ◀── here │
 │ ...                                        │
 └────────────────────────────────────────────┘
```

`balances["vinay"]` never scans. It computes a number from the key, jumps straight to that slot, confirms the key matches, returns the value. That's the whole O(1) trick.

### Under the hood — hash tables: hashing, collisions, resizing, and the 3.7 ordering guarantee

Open the box fully — dicts are [CORE]:

1. **Hashing.** `hash(key)` maps any hashable object to a fixed-size integer, *deterministically within one run*: same key → same hash, always. `hash("vinay")` and `hash("alice")` are (almost certainly) different numbers. (For strings, the hash is randomized per *process* for security — so don't persist raw hash values across runs.)
2. **Buckets (slots).** The dict owns an array of slots. The slot index is derived from the hash (roughly `hash % table_size` — really the low bits). Insert = compute slot, store `(hash, key_pointer, value_pointer)` there. Lookup = compute slot, compare, return.
3. **Collisions.** Two different keys can land on the same slot (there are infinitely many keys, finitely many slots). CPython uses **open addressing**: on a collision it *probes* — deterministically visits a sequence of alternative slots until it finds the key or an empty slot. A few probes are cheap; many probes are slow, which leads to…
4. **Resizing.** CPython keeps the table at most ~2/3 full. Past that, it allocates a bigger table (~4× for small dicts) and **re-inserts every entry** (slot positions depend on table size, so everything must be re-placed). Like list over-allocation, this occasional O(n) rebuild amortizes away: dict insert is amortized O(1).
5. **Why keys must be hashable/immutable.** The key's hash decides *where it lives*. If you could use a list as a key and then mutate that list, its hash would change — the entry would still sit in the *old* slot, and every future lookup would probe the *new* slot and find nothing. Your data would be lost inside the dict. Python forbids the whole category: unhashable types (`list`, `dict`, `set`) can't be keys. Tuples can (if their contents are hashable) — that's one big reason tuples exist (§4).
6. **Insertion order is guaranteed — since Python 3.7.** Modern CPython dicts are *compact*: entries live in a dense array **in insertion order**, and the hash table stores only small indices into that array. Iterating a dict yields keys in the order you inserted them. This started as a CPython 3.6 implementation detail and became a *language guarantee* in 3.7 (announced by the 3.7 release notes; before 3.6, dict order was effectively arbitrary and changed between runs). Consequence you'll rely on constantly: `json.dumps(d)` writes keys in insertion order, and a conversation history dict-of-turns keeps its shape.

### Runnable example — dict mechanics: O(1) lookup, overwrite, ordering, safe access, nesting

```python
# dicts.py — run with:  python dicts.py

# --- 1. build, read, overwrite, add ---
balances = {"vinay": 120.0, "alice": 300.0}
print(balances["vinay"])
balances["vinay"] = 999.0            # same key → OVERWRITES (keys are unique)
balances["bob"] = 50.0               # new key → adds
print(balances)

# --- 2. insertion order is guaranteed (Python 3.7+) ---
d = {"b": 2, "a": 1}
d["c"] = 3
print(list(d.keys()))                # b, a, c — insertion order, NOT sorted

# --- 3. missing keys: [] crashes, .get() doesn't ---
print(balances.get("zoe"))           # None
print(balances.get("zoe", 0.0))      # default of your choosing
try:
    balances["zoe"]
except KeyError as e:
    print("KeyError for key:", e)

# --- 4. iterate: keys, values, and pairs ---
for name, amount in balances.items():
    print(f"  {name}: {amount}")

# --- 5. the nested shape every API payload has ---
request = {
    "model": "claude-opus-4-8",
    "messages": [{"role": "user", "content": "hi"}],
}
print(request["messages"][0]["role"])

# --- 6. unhashable keys are rejected up front ---
try:
    bad = {["a", "list"]: 1}
except TypeError as e:
    print("TypeError:", e)
```

```bash
python dicts.py
# -> 120.0
# -> {'vinay': 999.0, 'alice': 300.0, 'bob': 50.0}
# -> ['b', 'a', 'c']
# -> None
# -> 0.0
# -> KeyError for key: 'zoe'
# ->   vinay: 999.0
# ->   alice: 300.0
# ->   bob: 50.0
# -> user
# -> TypeError: unhashable type: 'list'
```

**Why this works, line by line.** `balances["vinay"] = 999.0` hashes `"vinay"`, jumps to its slot, finds the key already present, and swaps the value pointer — which is why the printout shows one `vinay`, not two: a dict *is* the guarantee "one value per key." Section 2 proves the 3.7 ordering rule: `b, a, c` is insertion order, and Python did not helpfully sort it. Section 3 shows the two access styles — `[]` says "this key *must* exist; crashing is correct if it doesn't" (right for required fields), while `.get()` says "this key is optional" (right for optional config); choosing between them is an API-design decision you'll make daily. `.items()` in section 4 yields `(key, value)` tuples, unpacked into two loop variables — tuple unpacking previewing §4. Section 6 shows Python protecting you from the mutating-key disaster described under the hood: the `TypeError` fires at insert time, not as silent data loss later.

**When to use a dict:** any time you look things up by name/id — configs, JSON payloads, counters, caches, registries mapping name→function (Part 2 §3 builds exactly that). **When not to:** data whose *position* is the meaning (list), or when you only care about membership, not an associated value (set).

---

## 3. `set` — uniqueness and instant membership **[WORKING]**

**Depth: [WORKING]** — use it correctly and know why it's fast; the internals are "a dict with no values," so §2 already opened this box. We name the black box and move on.

### Intuition — what problem it solves

Two recurring needs: (1) **"Have I seen this before?"** and (2) **"Remove the duplicates."** With a list, both mean scanning — O(n) per check, O(n²) to dedup. Before sets, programmers abused dicts for this (keys with dummy values — `seen[url] = True` — which works, and is literally what a set is, minus the noise). A `set` stores *only keys*: unordered, unique, with O(1) membership tests.

```python
seen = {"url-a", "url-b"}
"url-a" in seen        # True — hashed, not scanned
seen.add("url-a")      # already present → nothing happens (no duplicates, no error)
```

### Analogy — a nightclub guest list (and where it breaks)

The bouncer's **guest list** only answers one question: *is this name on it?* Nobody cares what position your name occupies, and writing your name twice changes nothing. *Where it breaks:* a real guest list is on paper, so the bouncer scans it — a `set` doesn't scan, it hashes your name and jumps to the slot. Also, a paper list preserves the order names were added; a set **has no reliable order at all** (iteration order depends on hash values — never depend on it).

### Worked example — deduplication and set algebra, traced

```
tags = {"python", "backend", "python", "agents", "backend"}
                       # {"python", "backend", "agents"}  ← dupes silently dropped at creation

week1 = {"http", "tcp", "dns"}
week2 = {"tcp", "tls", "http2"}

week1 & week2          # {"tcp"}                    intersection: in BOTH
week1 | week2          # {"http","tcp","dns","tls","http2"}   union: in EITHER
week1 - week2          # {"http", "dns"}            difference: in week1 but NOT week2
```

The algebra operators are the underrated half of sets: "which users are in group A but not group B" is one line instead of nested loops.

### Visual — list membership vs set membership

```
"needle" in list_of_1_000_000:            "needle" in set_of_1_000_000:
  [0] no  [1] no  [2] no ... [999999]?      hash("needle") → slot 4187 → yes/no
  up to 1,000,000 comparisons               ~1 comparison
```

### Under the hood (interface-level — box named, not fully opened)

A set is a hash table storing only `(hash, key)` — a dict minus the value column. Everything from §2 carries over: members must be hashable (no lists/dicts in a set), collisions are probed, the table resizes, and there is **no order guarantee** (sets did *not* get the 3.7 dict ordering — the compact-layout redesign was applied to dicts only). Empty-set gotcha: `{}` creates an empty **dict** (dicts had the braces first, historically); an empty set is `set()`.

### Runnable example — dedup, membership speed intuition, and set algebra

```python
# sets.py — run with:  python sets.py

# --- 1. dedup for free ---
raw = [1, 2, 2, 3, 3, 3]
unique = set(raw)
print(unique)                          # duplicates gone
print(sorted(unique))                  # need order back? sort into a list

# --- 2. membership: the seen-before pattern ---
seen = set()
for url in ["a.com", "b.com", "a.com", "c.com", "b.com"]:
    if url in seen:                    # O(1) check
        print("skip duplicate:", url)
    else:
        seen.add(url)
        print("processing:", url)

# --- 3. set algebra ---
week1 = {"http", "tcp", "dns"}
week2 = {"tcp", "tls", "http2"}
print("both weeks:   ", week1 & week2)
print("only week 1:  ", week1 - week2)
print("all topics:   ", sorted(week1 | week2))

# --- 4. the empty-set gotcha ---
print(type({}), type(set()))
```

```bash
python sets.py
# -> {1, 2, 3}
# -> [1, 2, 3]
# -> processing: a.com
# -> processing: b.com
# -> skip duplicate: a.com
# -> processing: c.com
# -> skip duplicate: b.com
# -> both weeks:    {'tcp'}
# -> only week 1:   {'dns', 'http'}   (order may differ on your machine — sets are unordered)
# -> all topics:    ['dns', 'http', 'http2', 'tcp', 'tls']
# -> <class 'dict'> <class 'set'>
```

**Why this works, line by line.** `set(raw)` walks the list once, hashing each item into the table; the second `2` and the extra `3`s hash to slots already occupied by equal keys, so they're no-ops — dedup is a *side effect of how insertion works*, not a special feature. The `seen` loop is the canonical pattern you'll write dozens of times (crawlers, log dedup, retry tracking): check-then-add, both O(1), so the whole loop is O(n) instead of the O(n²) a list would cost. The algebra operators map to their math meanings (`&` ∩, `|` ∪, `-` \) and are computed by hashing the smaller set against the larger. The final line proves the `{}` gotcha — memorize it now, it *will* bite otherwise.

**When to use a set:** seen-before checks, dedup, allowed-values checks (`if role in {"user", "assistant", "system"}`), comparing two collections. **When not to:** you need order (list), you need an associated value (dict), items are unhashable.

---

## 4. `tuple` — the fixed record **[WORKING]**

**Depth: [WORKING]** — small surface, two important properties (immutability, hashability), one signature move (unpacking).

### Intuition — what problem it solves

Some groups of values belong together and **never change**: a coordinate `(48.86, 2.35)`, an RGB color `(255, 0, 0)`, a date `(2026, 7, 22)`. Putting them in a list *works* but sends the wrong message ("this may grow and mutate") and forfeits two abilities: lists can't be dict keys or set members. A **tuple** is a frozen sequence: ordered like a list, but immutable — created once, never modified.

```python
point = ("paris", 48.86, 2.35)
point[1]          # 48.86 — indexing works like a list
point[1] = 0      # TypeError — tuples don't change. Ever.
```

The deeper idea: a tuple is a **record** — position carries *meaning*. Slot 0 is *the city*, slot 1 is *latitude*, slot 2 is *longitude*. A list of three floats is "three floats"; a tuple of three fields is "one thing with three parts."

### Analogy — a laminated ID card (and where it breaks)

A tuple is a **laminated ID card**: name, date of birth, ID number sealed in fixed positions. You can't scratch out a field — you'd issue a *new card*. Because it can't change, institutions trust it as a *key* (it identifies you in their files) — exactly why tuples can be dict keys. *Where it breaks:* lamination protects the ink, but a tuple only freezes its *slots*, not what the slots point to: `(1, [2, 3])` is a tuple whose second slot points at a mutable list — you can't repoint the slot, but the list itself can still grow. (And such a tuple is unhashable, since its hash would depend on mutable content.)

### Worked example — unpacking and tuple keys, traced

```
point = ("paris", 48.86, 2.35)
city, lat, lon = point         # UNPACKING: three names bound in one line
                               # city="paris"  lat=48.86  lon=2.35

def min_max(nums):
    return min(nums), max(nums)     # returning two values? really ONE tuple
lo, hi = min_max([3, 1, 4])         # lo=1, hi=4  ← unpacked at the call site

distances = {("paris", "london"): 344, ("paris", "berlin"): 878}
distances[("paris", "london")]      # 344 — a COMPOSITE key, impossible with a list
```

That last pattern — tuple as compound dict key — is the payoff of hashability, and you saw the machinery for it in §2's under-the-hood.

### Visual — list vs tuple, side by side

```
list  ["paris", 48.86, 2.35]      tuple ("paris", 48.86, 2.35)
  ├─ ordered            ✔           ├─ ordered            ✔
  ├─ indexable          ✔           ├─ indexable          ✔
  ├─ mutable            ✔           ├─ mutable            ✘  (frozen at creation)
  ├─ dict key / in set  ✘           ├─ dict key / in set  ✔  (if contents hashable)
  └─ says "collection"              └─ says "record: positions have meaning"
```

### Under the hood (interface-level)

A tuple is a fixed-length pointer array allocated in one shot — no capacity field, no over-allocation, because it will never grow. That makes tuples slightly smaller and faster to create than lists, and lets CPython cache and reuse small ones. Its hash is computed from the hashes of its items, which is why every item must itself be hashable. One syntax wart: a one-item tuple needs a trailing comma — `(42,)` — because `(42)` is just the number 42 in parentheses; it's the *comma* that makes a tuple, not the parens.

### Runnable example — records, unpacking, multi-return, tuple keys

```python
# tuples.py — run with:  python tuples.py

# --- 1. a record, and the immutability guarantee ---
point = ("paris", 48.86, 2.35)
city, lat, lon = point                 # unpack into named variables
print(f"{city}: lat={lat}, lon={lon}")
try:
    point[1] = 0.0
except TypeError as e:
    print("TypeError:", e)

# --- 2. functions "returning two values" return one tuple ---
def min_max(nums):
    return min(nums), max(nums)        # comma builds the tuple

lo, hi = min_max([3, 1, 4, 1, 5])
print(f"lo={lo}, hi={hi}")

# --- 3. tuples as compound dict keys ---
distances = {
    ("paris", "london"): 344,
    ("paris", "berlin"): 878,
}
print(distances[("paris", "london")], "km")

# --- 4. swapping without a temp variable (unpacking again) ---
a, b = 1, 2
a, b = b, a
print(a, b)
```

```bash
python tuples.py
# -> paris: lat=48.86, lon=2.35
# -> TypeError: 'tuple' object does not support item assignment
# -> lo=1, hi=4
# -> 344 km
# -> 2 1
```

**Why this works, line by line.** `city, lat, lon = point` is *tuple unpacking*: Python checks the lengths match (3 names, 3 slots) and binds positionally — a mismatch raises `ValueError` immediately, which is a feature: your record's shape is being validated. The `TypeError` in section 1 is the immutability contract enforced at runtime. In `min_max`, the `return` line's comma constructs a tuple (`(1, 4)`), and the caller unpacks it — Python's famously clean "multiple return values" are just tuples both ways. The tuple keys in section 3 work because `hash(("paris", "london"))` combines the string hashes into one stable number — the same mechanism as any dict key. The swap in section 4 builds the tuple `(2, 1)` on the right *before* any assignment happens on the left, which is why no temporary variable is needed.

**When to use a tuple:** fixed records, multiple return values, dict keys, data that must not be accidentally mutated. **When not to:** the collection grows or changes (list).

### The four workhorses — one decision table **[CORE]**

Commit this to memory; it's the day's central skill:

| | `list` | `dict` | `set` | `tuple` |
|---|---|---|---|---|
| Literal | `[1, 2]` | `{"a": 1}` | `{1, 2}` | `(1, 2)` |
| Ordered? | ✔ | ✔ insertion (3.7+) | ✘ | ✔ |
| Mutable? | ✔ | ✔ | ✔ | ✘ |
| Duplicates? | ✔ | keys ✘ | ✘ | ✔ |
| Lookup style | by position O(1) | **by key O(1)** | membership O(1) | by position O(1) |
| `x in c` cost | O(n) scan | O(1) keys | O(1) | O(n) scan |
| Killer use | ordered, changing data | **name → value** | seen-before / dedup | fixed record, dict key |

**Worked micro-example — one dataset, four containers.** A log of API calls: as a **list** `["GET /a", "GET /b", "GET /a"]` (the full ordered history); as a **set** `{"GET /a", "GET /b"}` (which distinct endpoints were hit); as a **dict** `{"GET /a": 2, "GET /b": 1}` (hit counts per endpoint); each entry itself best as a **tuple** `("GET", "/a", 200)` (method, path, status — a fixed record). Same data, four questions, four structures.

---

## 5. Comprehensions — build collections from loops, in one expression **[WORKING]**

**Depth: [WORKING]** — syntax sugar, but sugar you'll read and write daily; every Python codebase (and every AI codebase) is full of them.

### Intuition — what problem it solves

The pattern "make an empty list, loop, append a transformed item" is so common it deserved its own syntax:

```python
# the long way                          # the comprehension
squares = []                            squares = [n * n for n in range(1, 6)]
for n in range(1, 6):
    squares.append(n * n)
```

Both produce `[1, 4, 9, 16, 25]`. The comprehension states *what the result is* ("n² for each n in 1..5") rather than *how to assemble it* — closer to how you'd describe it in English or math (it's borrowed from mathematical set-builder notation: {n² | n ∈ 1..5}).

### Analogy — a factory line order form (and where it breaks)

A for-loop is **standing at the conveyor belt yourself**, picking up each part, machining it, placing it in the bin. A comprehension is the **order form**: "one bin of: part², for every part in tray A, skipping the rusty ones." You describe the output; the machinery is implied. *Where it breaks:* an order form can express arbitrarily complex builds; a comprehension should stay simple. Multiple nested conditions and transformations belong in a normal loop — a comprehension nobody can read in one glance has failed at its only job.

### Worked example — the anatomy, traced

```python
[expression  for item in iterable  if condition]
 ─────────   ────────────────────  ────────────
 what each   where items           keep only items
 output      come from             passing the test
 item is                           (optional)

[n * n for n in range(10) if n % 2 == 0]
# range(10) yields 0..9
# filter keeps 0, 2, 4, 6, 8
# expression squares them → [0, 4, 16, 36, 64]
```

And the same idea with braces builds dicts and sets:

```python
{w: len(w) for w in ["api", "agent", "tool"]}   # dict: {'api': 3, 'agent': 5, 'tool': 4}
{len(w) for w in ["api", "agent", "tool"]}      # set:  {3, 4, 5}
```

### Runnable example — list/dict/set comprehensions on realistic data

```python
# comprehensions.py — run with:  python comprehensions.py

# --- 1. transform: list comprehension ---
prices = [19.99, 5.49, 3.25]
with_tax = [round(p * 1.18, 2) for p in prices]
print(with_tax)

# --- 2. filter + transform ---
evens_squared = [n * n for n in range(10) if n % 2 == 0]
print(evens_squared)

# --- 3. dict comprehension: build a lookup ---
words = ["api", "agent", "tool"]
lengths = {w: len(w) for w in words}
print(lengths)

# --- 4. set comprehension: unique first letters ---
firsts = {w[0] for w in ["agent", "api", "tool", "tuple"]}
print(sorted(firsts))

# --- 5. the pattern you'll use in Part 2: extract from a list of dicts ---
messages = [
    {"role": "user", "content": "hi"},
    {"role": "assistant", "content": "hello!"},
    {"role": "user", "content": "bye"},
]
user_texts = [m["content"] for m in messages if m["role"] == "user"]
print(user_texts)
```

```bash
python comprehensions.py
# -> [23.59, 6.48, 3.84]
# -> [0, 4, 16, 36, 64]
# -> {'api': 3, 'agent': 5, 'tool': 4}
# -> ['a', 't']
# -> ['hi', 'bye']
```

**Why this works, line by line.** Each comprehension desugars to exactly the loop-and-append you already know — same work, same order, one expression. Section 3's `{w: len(w) ...}` has a `key: value` pair before the `for`, which is what makes it a *dict* comprehension; section 4 has a bare expression in braces, making it a *set* (and dedup happens by the §3 insertion mechanics — two words starting with `t` produce one `'t'`). Section 5 is the single most useful comprehension in this whole course: *filter a list of dicts by one field, extract another field* — that is verbatim how you'll pull the user's messages out of a conversation history or the text blocks out of an API response in Part 2. Rule of thumb: one `for`, at most one `if`, a readable expression — beyond that, write the loop.

---

## 6. Classes — bundling state and behavior **[CORE]**

**Depth: [CORE]** — classes are how every real Python codebase (and every SDK object you'll touch — `client`, `response`, `stream`) is organized. Open the box fully: what `self` is, where attributes live, what a method call actually does.

### Intuition — WHY classes exist (the problem before them)

Model a bank account with what you know so far: a dict for the **state** (`{"owner": "vinay", "balance": 100.0}`) and free-floating functions for the **behavior** (`deposit(account, amount)`, `withdraw(account, amount)`). It works — until it doesn't:

- Nothing *connects* the functions to the data. `deposit(some_random_dict, 50)` happily "deposits" into a dict that isn't an account at all.
- Nothing *protects* the state: any code anywhere can do `account["balance"] = -1_000_000` directly, bypassing every rule.
- With ten account operations and ten other dict-shaped things in the program, you get a soup of functions and dicts with only naming conventions holding it together.

Early programs were exactly this soup ("procedural programming": data over here, functions over there), and it collapses under scale. The fix, dating to Simula and Smalltalk in the 1960s–70s and adopted by nearly every language since: **put the data and the functions that operate on it in one unit.** That unit is an **object**; the blueprint that stamps out objects is a **class**.

```python
acct = BankAccount("vinay", 100.0)   # an OBJECT: state + behavior together
acct.deposit(50.0)                   # behavior that belongs to the state it changes
acct.balance                         # 150.0
```

One sentence to keep: **a class bundles state (attributes) with the behavior (methods) that is allowed to touch that state.**

### The vocabulary, defined before use

- **class** — the blueprint. `class BankAccount:` defines what every account *has* and *can do*. Defining a class creates no accounts.
- **instance (object)** — one concrete thing stamped from the blueprint. `BankAccount("vinay", 100.0)` creates one. A thousand instances share one class.
- **attribute** — a variable living *on* an instance: `acct.balance`. Each instance has its own.
- **method** — a function defined *inside* a class, called *on* an instance: `acct.deposit(50)`.
- **`__init__`** — the initializer, called automatically when an instance is created; its job is to set up the instance's starting attributes. (Double-underscore names — "dunders" — are hooks Python itself calls; you'll meet more later, `__init__` is the one that matters today.)
- **`self`** — the instance currently being operated on. Every method's first parameter. Explained mechanically below, because it's the #1 confusion.

### Analogy — cookie cutter and cookies (and where it breaks)

The class is the **cookie cutter**; instances are the **cookies**. One cutter, many cookies; each cookie decorated independently (each instance has its own attribute values); reshape the cutter and future cookies change, existing ones don't. *Where it breaks:* cookies are passive — they don't *do* anything. Objects carry behavior: each cookie would come with its own "eat me" and "decorate me" operations that know *which cookie* they belong to. That self-knowledge is literally the `self` parameter.

### Worked example — a class, traced line by line

```python
class BankAccount:                         # 1. define the blueprint
    def __init__(self, owner, balance=0.0):
        self.owner = owner                 # 2. attach attributes to THIS instance
        self.balance = balance

    def deposit(self, amount):             # 3. behavior that belongs to the data
        self.balance += amount
        return self.balance

acct = BankAccount("vinay", 100.0)
# What that line ACTUALLY does:
#   a. Python creates a blank BankAccount instance
#   b. calls BankAccount.__init__(<that instance>, "vinay", 100.0)
#      └── inside, self IS that blank instance
#   c. __init__ sets self.owner="vinay", self.balance=100.0
#   d. the now-initialized instance is assigned to the name acct

acct.deposit(50.0)
# What THAT line actually does:
#   BankAccount.deposit(acct, 50.0)
#   └── self = acct, amount = 50.0 → acct.balance becomes 150.0

b = BankAccount("alice")        # a SECOND instance; balance defaults to 0.0
b.balance                       # 0.0  — completely independent of acct's 150.0
```

**`self`, demystified in one line:** `acct.deposit(50.0)` is exactly `BankAccount.deposit(acct, 50.0)`. The instance to the left of the dot is silently passed as the first argument. `self` isn't magic or a keyword — it's a plain parameter name (by unbreakable convention called `self`) that receives the instance. That's why every method signature starts with it, and why forgetting it produces the classic error you'll see in the runnable example.

### Visual — one class, two instances

```
        class BankAccount              (one blueprint, holds the methods)
        ├── __init__
        ├── deposit
        └── withdraw
              ▲            ▲
   ┌──────────┘            └──────────┐
   instance acct                instance b
   .owner   = "vinay"           .owner   = "alice"
   .balance = 150.0             .balance = 0.0
   (own state)                  (own, independent state)
```

Methods live once, on the class. State lives per-instance. A method call = look up the function on the class, hand it the instance as `self`.

### Under the hood — objects are dicts (it really is that literal)

- Each instance carries a plain dict of its attributes: `acct.__dict__` is `{'owner': 'vinay', 'balance': 150.0}`. `self.balance = x` is, mechanically, `acct.__dict__['balance'] = x`. Everything §2 taught about dicts applies to your objects.
- Attribute lookup `acct.balance` checks the instance `__dict__` first, then the class (that's how methods are found — they're in the *class's* dict), then parent classes. Miss everywhere → `AttributeError`.
- The class itself is an object too (`type(acct)` is the class; `type(BankAccount)` is `type`). File that away — it explains a lot of Python later; today you just need instance vs class.

### Runnable example — the class mechanics, including the classic `self` mistake

```python
# classes.py — run with:  python classes.py

class BankAccount:
    """A tiny account: state (owner, balance) + the behavior allowed to touch it."""

    def __init__(self, owner, balance=0.0):
        self.owner = owner
        self.balance = balance

    def deposit(self, amount):
        self.balance += amount
        return self.balance

    def describe(self):
        return f"{self.owner} has {self.balance:.2f}"


# --- 1. two instances, independent state ---
acct = BankAccount("vinay", 100.0)
b = BankAccount("alice")
acct.deposit(50.0)
print(acct.describe())
print(b.describe())                       # untouched by acct's deposit

# --- 2. proof: acct.deposit(50) IS BankAccount.deposit(acct, 50) ---
BankAccount.deposit(acct, 25.0)           # calling through the class, passing self by hand
print(acct.balance)

# --- 3. proof: attributes live in a plain dict on the instance ---
print(acct.__dict__)

# --- 4. the classic beginner error, on purpose ---
class Broken:
    def hello():                          # forgot self!
        return "hi"

try:
    Broken().hello()
except TypeError as e:
    print("TypeError:", e)
```

```bash
python classes.py
# -> vinay has 150.00
# -> alice has 0.00
# -> 175.0
# -> {'owner': 'vinay', 'balance': 175.0}
# -> TypeError: Broken.hello() takes 0 positional arguments but 1 was given
```

**Why this works, line by line.** `BankAccount("vinay", 100.0)` triggers instance creation + `__init__`, whose two `self.x = ...` lines write into that instance's private attribute dict — which section 3 then prints raw, proving objects are dicts underneath. Section 1 shows the whole point of instances: `acct` and `b` were stamped from one blueprint but their `.balance` values are independent, because each has its own `__dict__`. Section 2 is the `self` proof: calling `BankAccount.deposit(acct, 25.0)` explicitly does exactly what `acct.deposit(25.0)` does implicitly — the dot-call is *sugar* for "pass the left-hand object as the first argument." Section 4's error message is worth reading slowly: *"takes 0 positional arguments but 1 was given"* — the mysterious 1 extra argument **is the instance**, auto-passed as always; `hello()` declared no parameter to receive it. Now that error message will never confuse you again.

*(Classes go much deeper — inheritance, properties, dataclasses, dunder protocols. Deliberately out of scope for P2; you have exactly what Pydantic and the Anthropic SDK require you to recognize: `__init__`, `self`, attributes, methods, instances.)*

---

## 7. Modules & imports — programs that span files **[CORE]**

**Depth: [CORE]** — every program beyond ~100 lines is multiple files, and every library you'll ever use (`anthropic`, `fastapi`, `json`) arrives through `import`. You must know what actually happens when that word executes.

### Intuition — what problem it solves

One giant file doesn't scale: you can't find anything, two features tangle together, and *nothing can be reused* — your `BankAccount` class is trapped inside the script that uses it. Before module systems, code reuse meant literally copy-pasting source between files (and every copy drifting out of sync). The fix: make **each file a self-contained unit of code** that other files can *load and use by name*.

- A **module** is simply a `.py` file. `bank.py` is a module named `bank`. That's the entire definition.
- A **package** is a directory of modules (traditionally marked by an `__init__.py` file inside it), so related modules can nest: `myapp/db.py`, `myapp/api.py` → `import myapp.db`. When you `pip install anthropic`, you're downloading a package into a place Python can find.
- **`import bank`** loads the file and gives you the name `bank`; you use its contents as `bank.BankAccount`.
- **`from bank import BankAccount`** loads the same file but copies one name directly into your namespace: `BankAccount`, no prefix.

### Analogy — a toolbox wall (and where it breaks)

Modules are **labeled toolboxes on a workshop wall**. `import bank` brings the whole *bank* toolbox to your bench — tools stay inside, you reach in with `bank.deposit`. `from bank import BankAccount` takes *one tool out* and puts it on your bench directly. *Where it breaks:* taking a tool out of a toolbox doesn't run anything — but **importing a module executes the whole file, top to bottom, once**. Any `print` at a module's top level fires at import time. That surprising fact is exactly why the `if __name__ == "__main__":` idiom exists (below).

### Worked example — what `import` actually does, step by step

```
main.py runs:  from bank import BankAccount

1. Python checks sys.modules (the ALREADY-LOADED cache, itself a dict:
   name → module object). "bank" not there? Continue. There? Skip to 4.
2. Python searches sys.path (a list of directories: the script's own folder
   first, then installed packages) for a file called bank.py.
3. Found → Python EXECUTES bank.py top to bottom, once. Every def and class
   statement in it runs, creating those objects inside a new module object.
   The module object's attributes ARE its top-level names (yes — module
   attributes live in a dict too; __dict__ all the way down).
4. The module object is cached in sys.modules — a second import anywhere
   in the program is a dict lookup, not a re-execution.
5. "from ... import BankAccount" copies that one attribute into main.py's
   namespace.
```

Two consequences worth pinning: imports are **cached** (import the same module from ten files, it executes once), and import is **execution** (side effects at a module's top level happen at import time — keep module top levels to defs, classes, and constants).

### The `if __name__ == "__main__":` idiom

Every module gets an automatic variable `__name__`. When the file is **run directly** (`python bank.py`), `__name__` is the string `"__main__"`. When the file is **imported** (`import bank`), `__name__` is `"bank"`. So this guard means *"only run this when I'm the program, not when I'm a library"*:

```python
if __name__ == "__main__":
    # demo / test code — runs on `python bank.py`, silent on `import bank`
    acct = BankAccount("test", 10)
```

It's how one file can be both an importable library *and* a runnable script — you'll see it in virtually every Python file in the wild.

### Runnable example — a two-module program (the exact shape of the Build task)

Create **two files in the same folder**:

```python
# bank.py — the library module: defines things, runs nothing loudly.
print("(bank.py is being executed — this proves import runs the file)")

class BankAccount:
    def __init__(self, owner, balance=0.0):
        self.owner = owner
        self.balance = balance

    def deposit(self, amount):
        self.balance += amount
        return self.balance

if __name__ == "__main__":
    print("bank.py run directly — quick self-test:")
    print(BankAccount("test", 1.0).deposit(2.0))
```

```python
# main.py — the entry point: imports the library and uses it.
from bank import BankAccount     # executes bank.py (once), copies one name in

acct = BankAccount("vinay", 100.0)
acct.deposit(50.0)
print(f"{acct.owner}: {acct.balance}")

import bank                      # SECOND import: cache hit, bank.py does NOT rerun
print(type(bank))                # a module is an object like any other
```

```bash
python main.py
# -> (bank.py is being executed — this proves import runs the file)
# -> vinay: 150.0
# -> <class 'module'>

python bank.py
# -> (bank.py is being executed — this proves import runs the file)
# -> bank.py run directly — quick self-test:
# -> 3.0
```

**Why this works, line by line.** Running `main.py`, the very first output line comes from *bank.py*, not main.py — proof that `from bank import BankAccount` executes the imported file. The class statement inside bank.py runs during that execution, creating the `BankAccount` object that the `from ... import` then copies into main.py's namespace. The later `import bank` prints nothing: `sys.modules` already holds `"bank"`, so Python hands back the cached module object — imports are load-once. Note what `python bank.py` prints *extra*: the self-test guarded by `__name__ == "__main__"` fires only in direct-run mode, and stayed silent during the import from main.py — one file, two roles. When you `pip install anthropic` and write `import anthropic`, this identical machinery runs; the only difference is *where* on `sys.path` the file is found.

---

## 8. Exceptions — handling failure without crashing **[CORE]**

**Depth: [CORE]** — every network call, file read, and JSON parse can fail. Programs that survive failure are built from this section; every later day (HTTP errors, API retries, tool errors) stands on it.

### Intuition — what problem it solves

Things go wrong at runtime: the file isn't there, the network drops, the user typed `"abc"` where a number goes. Before exceptions, functions signaled failure through **return codes** — return `-1` or `null` for "it failed" — and C code still works this way. The flaw: return codes are *ignorable*. Forget one `if result == -1` check and your program marches on with garbage, failing much later, far from the cause (the worst kind of bug). **Exceptions** make failure *impossible to ignore*: when something goes wrong, an exception object is **raised**, and it forcibly interrupts execution and travels **up the call stack** until something **catches** it — or nobody does, and Python prints the traceback you learned to read on P1 and exits.

```python
try:
    data = json.loads(text)      # might fail
except json.JSONDecodeError:
    data = {}                    # we chose what failure means here
```

Four keywords, four jobs: **`try`** marks code that might fail, **`except`** handles a failure, **`raise`** *signals* a failure yourself, **`finally`** runs cleanup no matter what happened.

### Analogy — a fire alarm (and where it breaks)

An exception is a **fire alarm**. It doesn't whisper into a logbook you might read later (a return code) — it interrupts *everything*, and it keeps escalating (up the call stack) until someone competent responds (an `except` on the right floor) or the whole building is evacuated (the program exits with a traceback). *Where it breaks:* fire alarms are for fires only; exceptions cover *everything* abnormal, including mundane, expected events like "file not found" — routine happenings that you often *plan* for with `except` rather than treat as disasters. Also: firefighters handle every fire the same way, but `except FileNotFoundError:` responds only to its declared type and lets other alarms keep climbing — precision the analogy lacks.

### Worked example — an exception climbing the call stack, traced

```python
def read_config(path):
    return open(path).read()        # ← FileNotFoundError raised HERE

def load_app():
    return read_config("settings.json")

def main():
    try:
        load_app()
    except FileNotFoundError:       # ← ...and CAUGHT here, two frames up
        print("no config, using defaults")

main()
```

Trace: `open()` fails inside `read_config` → an exception object is created and starts climbing → `read_config` has no `try`, so that frame is abandoned mid-function → `load_app` has none either, abandoned → `main`'s `try` block is enclosing the call, its `except FileNotFoundError` matches the type → execution resumes *in the handler*. Every frame between raise-point and catch-point simply stops. And now the P1 lesson pays off: a traceback is the *printed record of exactly that climb* — deepest frame last, which is why you read it bottom-up.

### `raise` — you signal failures too

Exceptions aren't only received — you raise them when *your* function's contract is violated. That is precisely the Build task's rule: withdrawing more than the balance is an error, and the account must say so *loudly*:

```python
def withdraw(self, amount):
    if amount > self.balance:
        raise InsufficientFunds(f"cannot withdraw {amount}: balance is {self.balance}")
    self.balance -= amount
```

Why raise instead of returning `None` or `False`? Because a return value can be silently ignored, and an ignored overdraft is corrupted money. Raising forces the caller to *decide* — catch it and handle it, or crash. And defining your own exception type is nothing but a tiny class (§6 again!):

```python
class InsufficientFunds(Exception):   # inherit from Exception — that's all it takes
    pass
```

A custom type lets callers catch *your* failure specifically (`except InsufficientFunds:`) without accidentally swallowing unrelated bugs — which is also the rule for catching: **catch the most specific type that you can actually handle**. Bare `except:` (catch absolutely everything, including typos in your own code) is the classic way to hide bugs.

### `finally` — cleanup that always runs

`finally` runs whether the `try` succeeded, raised, or even `return`ed — its job is releasing resources (close the file, release the lock) on *every* exit path. (Preview, on faith until P3: the `with` statement packages this pattern so well that you'll rarely write `finally` by hand for files.)

### Visual — the four keywords in one flow

```
        try:  risky code
          │
   ┌──────┴───────────┐
   no exception    exception raised
   │                    │
   │             matching except?
   │             ┌──────┴──────┐
   │            yes            no
   │             │              │
   │        handler runs   climbs to caller
   └──────┬──────┘              │
          ▼                     ▼
      finally: ALWAYS runs   finally still runs,
          │                  then keeps climbing
          ▼
      program continues
```

### Runnable example — try/except/raise/finally, plus a custom exception

```python
# exceptions.py — run with:  python exceptions.py

class InsufficientFunds(Exception):
    """Raised when a withdrawal exceeds the balance."""

def withdraw(balance, amount):
    if amount > balance:
        raise InsufficientFunds(f"cannot withdraw {amount}: balance is {balance}")
    return balance - amount

# --- 1. catch a specific, expected failure ---
try:
    withdraw(100.0, 500.0)
except InsufficientFunds as e:        # `as e` binds the exception OBJECT
    print("blocked:", e)

# --- 2. multiple except clauses: most specific handling wins ---
for raw in ["50", "abc"]:
    try:
        amount = float(raw)           # "abc" → ValueError
        print("parsed:", withdraw(100.0, amount))
    except ValueError:
        print(f"not a number: {raw!r}")
    except InsufficientFunds as e:
        print("blocked:", e)

# --- 3. finally always runs — even when the try raises ---
def read_first_line(path):
    f = None
    try:
        f = open(path)
        return f.readline()
    except FileNotFoundError:
        return "<no such file>"
    finally:
        print("  (finally: cleaning up)")
        if f is not None:
            f.close()

print(read_first_line("does_not_exist.txt"))

# --- 4. uncaught = climb all the way up = traceback (uncomment to see) ---
# withdraw(0, 1)
```

```bash
python exceptions.py
# -> blocked: cannot withdraw 500.0: balance is 100.0
# -> parsed: 50.0
# -> not a number: 'abc'
# ->   (finally: cleaning up)
# -> <no such file>
```

**Why this works, line by line.** `raise InsufficientFunds(...)` creates an instance of our exception class (an object like any other — §6) and launches it up the stack; the `except InsufficientFunds as e:` clause matches by type and binds the object to `e`, whose `str()` is the message we packed in — that's why `print("blocked:", e)` shows the full sentence. In section 2, the *same* `try` block has two different failure modes, and Python routes each to its matching clause by type: `"abc"` explodes at `float()` with `ValueError` before `withdraw` is ever called, so specific handlers cleanly separate "bad input" from "business rule violated." Section 3 demonstrates the `finally` guarantee: the function *returns from inside `try`*, and the cleanup line still prints first — `finally` intercepts every exit path, which is what makes it safe for releasing resources. The commented-out line in section 4 is your homework: uncomment it, run again, and read the traceback bottom-up like you learned on P1 — bottom line names `InsufficientFunds` and the message, lines above show the climb.

---

## 9. The standard library exists — don't reinvent it

**The idea [CORE], each module [WORKING]/[AWARE]:** Python ships with ~200 modules of pre-built, battle-tested code — "batteries included" has been Python's slogan since the 90s. The problem it solves: before rich standard libraries, every project hand-rolled its own path handling, its own config parser, its own date math — each subtly buggy in its own way. The professional habit to build *today*: **when a task feels generic (paths, JSON, counting, dates, randomness), assume the standard library has it and look before writing it.** You import these exactly like your own modules (§7) — same machinery, they're just files that ship with Python.

Three you need this week:

### 9a. `pathlib` — file paths as objects **[WORKING]**

**Intuition & the problem before it:** file paths used to be raw strings, glued together with `+` — which breaks across operating systems (Windows `\` vs Linux `/`) and scatters path logic everywhere. `pathlib.Path` makes a path an *object* (§6 in action: state = the path, methods = the operations) with cross-platform behavior built in.

**Analogy:** a `str` path is an address scribbled on paper; a `Path` is that address loaded into a navigation app — same information, but now "does it exist?", "what folder is it in?", "what's next door?" are one method away. *Where it breaks:* the nav app metaphor implies the destination exists; a `Path` object is just a description — `Path("nope.txt")` constructs happily, and only `.exists()` or an actual read touches the disk.

**Worked example:**

```python
from pathlib import Path
p = Path("data") / "account.json"   # the / operator JOINS paths (any OS)
p.name                              # "account.json"
p.suffix                            # ".json"
p.exists()                          # False (nothing on disk yet — just a description)
p.write_text("hello")               # create + write + close, one call
p.read_text()                       # "hello"
```

`write_text` / `read_text` handle open-write-close in one call — no `finally: f.close()` needed. This pair is exactly what the Build task uses for its JSON file.

### 9b. `json` — the data language of the internet **[WORKING]**

**Intuition & the problem before it:** your dicts and lists live in memory and die with the process. To *save* them to disk or *send* them over a network, they must become text; the receiving program (possibly written in Go or JavaScript) must parse that text back. Every pre-JSON era solved this with ad-hoc formats or heavyweight XML; JSON won because it's minimal and maps almost 1:1 onto Python's containers:

| Python | JSON | | Python | JSON |
|---|---|---|---|---|
| `dict` | object `{}` | | `str` | string |
| `list` | array `[]` | | `True`/`False` | `true`/`false` |
| `int`/`float` | number | | `None` | `null` |

Two functions to know (the `s` means *string*): **`json.dumps(obj)`** dumps a Python object *to* a JSON string; **`json.loads(text)`** loads a JSON string back into Python objects. Every API request you send in Part 2 is `dumps`-ed on the way out and every response `loads`-ed on the way in — the SDK does it for you, but *this* is what it's doing. *Round-trip caveats:* tuples come back as lists, dict keys come back as strings, and sets can't be serialized at all — JSON only has the types in the table.

### 9c. `collections` — upgraded containers **[AWARE]**

One paragraph, one example, then treat as a black box until needed: the `collections` module ships specialized containers so you don't hand-roll them. The one to remember is **`Counter`** — the P1 word-count exercise, industrialized: feed it any iterable, get a dict of item → count with a `.most_common(n)` method. Also in the box: `defaultdict` (a dict that auto-creates missing values) and `deque` (a list that's O(1) at *both* ends — the fix for §1's slow `pop(0)`). Know they exist; reach for them when a project forces you deeper.

### Runnable example — pathlib + json + Counter working together

```python
# stdlib_tour.py — run with:  python stdlib_tour.py
import json
from pathlib import Path
from collections import Counter

# --- 1. json round-trip: memory → text → memory ---
profile = {"name": "vinay", "scores": [88, 92], "active": True, "nickname": None}
text = json.dumps(profile)
print(text)                              # note: true/null, not True/None — this is JSON
restored = json.loads(text)
print(restored == profile)               # perfect round-trip (for these types)

# --- 2. pathlib: write the JSON to disk and read it back ---
path = Path("profile.json")
path.write_text(json.dumps(profile, indent=2))   # indent=2 → human-readable file
print(path.exists(), "-", path.name, "-", path.stat().st_size, "bytes")
from_disk = json.loads(path.read_text())
print(from_disk["scores"])

# --- 3. Counter: P1's word count in two lines ---
words = ["agent", "tool", "agent", "loop", "tool", "agent"]
counts = Counter(words)
print(counts)
print(counts.most_common(2))

path.unlink()                            # clean up the demo file
```

```bash
python stdlib_tour.py
# -> {"name": "vinay", "scores": [88, 92], "active": true, "nickname": null}
# -> True
# -> True - profile.json - 101 bytes
# -> [88, 92]
# -> Counter({'agent': 3, 'tool': 2, 'loop': 1})
# -> [('agent', 3), ('tool', 2)]
```

**Why this works, line by line.** `json.dumps(profile)` walks the dict using the mapping table above — watch the output: Python's `True` became JSON's `true`, `None` became `null`; this string is now language-neutral and any program anywhere can parse it. `json.loads` reverses the walk, and equality holds because every type in `profile` survives the round-trip. `path.write_text(...)` opens, writes, and closes in one call (the `finally` from §8, packaged for you), and `indent=2` pretty-prints — open `profile.json` in your editor mid-run and it's readable. `Counter(words)` iterates the list once, doing `counts[word] += 1` with a dict underneath (§2's machinery — `Counter` *is* a dict subclass), and `.most_common(2)` returns a **list of tuples** — all four of today's ideas in one return value.

---

## Part 1 — practice questions

**Easy.** (1) Which container for: a to-do list? unique visitor IPs? HTTP-status→meaning lookup? an (x, y) coordinate? (2) Why does `{}` not create an empty set? (3) What does `acct.deposit(50)` pass as `self`?

**Medium.** (4) Why is `lst.insert(0, x)` O(n) but `lst.append(x)` amortized O(1)? (5) Why can a tuple be a dict key but a list can't — what would actually go wrong? (6) `import bank` appears in five files of one program: how many times does `bank.py` execute, and what makes that true? (7) When should a function `raise` instead of returning `None`?

**Hard.** (8) Explain a dict lookup end-to-end: hash → slot → collision probing → the 2/3-full resize — and where insertion order survives all of that (the compact entries array). (9) `def f(x, items=[])` with `items.append(x); return items` returns `[1]` then `[1, 2]` on successive calls `f(1)`, `f(2)` — explain using "a list is a pointer to one mutable object" (§1) and the fact that defaults are evaluated once at `def` time. (10) Sketch what Python does, step by step, for `from bank import BankAccount` when `bank` is *already* in `sys.modules`.

---

# PART 2 — AGENTIC AI: Where Every P2 Construct Shows Up in Real LLM Code

## Overview & motivation — you already know the shape of agent code

Here's the reveal this Part exists for: **calling a frontier LLM requires no new data structures whatsoever.** An Anthropic API request is a dict. The messages in it are a list of dicts. The response is an object (a class instance) holding a list of objects. A tool definition is a dict containing a schema dict. A tool registry is a dict mapping names to functions. An agent is a class with a history attribute. Failures are exceptions. Persistence is JSON. Part 1 *was* the agentic AI lesson — this Part just shows you the same containers wearing production clothes.

**Ground rules for this Part.** The code is real and runnable against the Claude API via the official SDK. Setup once:

```bash
pip install anthropic                       # the official Anthropic Python SDK
export ANTHROPIC_API_KEY="sk-ant-..."       # your key, as an environment variable
# (Windows PowerShell:  $env:ANTHROPIC_API_KEY = "sk-ant-...")
```

Anything beyond P2 knowledge is flagged **(preview — taken on faith until Day 22)**: today you read these programs as *exercises in dicts, lists, classes, and exceptions*; the *meaning* of models, tokens, and tool use gets its own days. LLM outputs vary run to run, so the `# ->` comments show one real-shaped run, not a guaranteed byte-for-byte transcript.

---

## 1. An API request and response are nested dicts and lists **[CORE]**

### Intuition

You talk to Claude by sending a **dict** over the network and getting structured data back. The SDK (`anthropic` — a package you import, §7) wraps the networking; your entire job is *building the right dict shape* and *navigating the response*. Both skills are Part 1 §2 verbatim.

### Worked example — the request dict, annotated field by field

```python
{
    "model": "claude-opus-4-8",       # str  — which model (preview: model IDs, Day 22)
    "max_tokens": 1000,               # int  — response length cap (preview: tokens, Day 22)
    "messages": [                     # LIST of message dicts — the conversation
        {"role": "user", "content": "..."}    # each message: exactly this dict shape
    ],
}
```

Every key is a plain string key in a plain dict; `messages` is a list you could `append` to (that's §2 of this Part). The SDK turns this dict into JSON (§9b — `json.dumps` under the hood), sends it, and parses the JSON reply back into Python objects.

### Runnable example — first API call: build the dict, navigate the response

```python
# first_call.py — pip install anthropic; set ANTHROPIC_API_KEY. Run: python first_call.py
import anthropic

client = anthropic.Anthropic()        # a class instance (§6) — reads the key from the env

response = client.messages.create(    # a METHOD call on the client object
    model="claude-opus-4-8",
    max_tokens=1000,
    messages=[
        {"role": "user", "content": "In ONE sentence: why is a dict lookup O(1)?"}
    ],
)

# The response is an object (class instance). Its .content attribute is a LIST
# of content blocks; each block is an object with a .type. Filter by type —
# the comprehension pattern from Part 1 §5:
texts = [block.text for block in response.content if block.type == "text"]
print(texts[0])

# usage is another object with int attributes (preview: tokens, Day 22)
print("tokens in/out:", response.usage.input_tokens, "/", response.usage.output_tokens)
```

```bash
python first_call.py
# -> A dict lookup is O(1) because the key is hashed to compute the exact
#    location where its value is stored, so no scanning is needed.   (wording varies per run)
# -> tokens in/out: 24 / 31   (numbers vary per run)
```

**Why this works, line by line.** `anthropic.Anthropic()` instantiates the SDK's client class — the same `ClassName(...)` construction as `BankAccount("vinay")`, and its `__init__` reads your API key from the environment. `client.messages.create(...)` is a method call whose keyword arguments *are* the request dict's fields; `messages=` receives a real Python list holding a real dict — build it wrong (typo `"roel"`) and the API rejects it, which §3's error handling will catch. The response comes back as an object, not raw JSON: `response.content` is a list (indexable, iterable, comprehension-able — everything §1 taught), and each block carries a `.type` attribute you filter on — the exact "extract a field from a list of things, filtered by another field" comprehension from Part 1 §5. The habit of *checking `.type` before touching `.text`* matters because responses can contain other block kinds (preview — thinking blocks and tool-use blocks, Days 22+).

**Sidebar — asking "how big is this prompt?"** The SDK exposes a token counter, and it's the same dict/list shapes going in:

```python
count = client.messages.count_tokens(
    model="claude-opus-4-8",
    messages=[{"role": "user", "content": "In ONE sentence: why is a dict lookup O(1)?"}],
)
print(count.input_tokens)     # -> 24    (an int attribute on a response object)
```

*(What a token actually is: preview — Day 22. Today it's "an object with an int attribute.")*

---

## 2. A conversation is a list of dicts you append to **[CORE]**

### Intuition

The API has no memory between calls **(preview — statelessness gets a full treatment later; today, take it on faith)**. If you want Claude to remember your name from the last message, *you* must send the whole conversation again — which means the conversation lives in your program as a data structure. Which one? Ordered (turn order is meaning), growing (every exchange adds turns), items are role+content pairs → **a list of dicts**. Multi-turn chat is `list.append` in a loop; that's the entire trick behind every chatbot you've used.

### Visual — the history growing turn by turn

```
history = []
  └─ append user msg      [{"role":"user", ...}]
  └─ call API, append reply
                           [{"role":"user",...}, {"role":"assistant",...}]
  └─ append next user msg  [{"role":"user",...}, {"role":"assistant",...}, {"role":"user",...}]
  └─ call API (sends ALL THREE), append reply ...
```

Each API call receives the full list — the model "remembers" only because the list re-tells it everything.

### Runnable example — memory across turns, built from `append`

```python
# multiturn.py — pip install anthropic; set ANTHROPIC_API_KEY. Run: python multiturn.py
import anthropic

client = anthropic.Anthropic()
history = []                                        # THE conversation: a plain list

def ask(user_text):
    history.append({"role": "user", "content": user_text})       # grow: user turn
    response = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=1000,
        messages=history,                            # send the WHOLE list every time
    )
    reply = next(b.text for b in response.content if b.type == "text")
    history.append({"role": "assistant", "content": reply})      # grow: assistant turn
    return reply

print(ask("My name is Vinay. Reply with just: noted."))
print(ask("What is my name? Answer in one word."))
print("turns stored:", len(history))
```

```bash
python multiturn.py
# -> Noted.
# -> Vinay.        ← memory! ...except it's OUR list doing the remembering
# -> turns stored: 4
```

**Why this works, line by line.** `history` is module-level state; `ask` mutates it — two `append`s per exchange, so two calls leave `len(history) == 4`. The second API call answers "Vinay" *only* because `messages=history` re-sent the first exchange: delete the first two entries before the second call and the model cannot know your name — run that experiment, it's the most instructive five minutes of the day. The `next(... if b.type == "text")` line is the §5 comprehension pattern in generator form: take the first text block. Note the dict shape discipline: every entry is exactly `{"role": ..., "content": ...}`, roles alternating user/assistant — the API validates this shape and rejects malformed history (a `KeyError` in *their* validation instead of yours). Also note the design smell you should already feel: a global list mutated by a function is the "dict + free functions" soup from Part 1 §6's motivation. State that belongs with behavior… wants a class. That's §4.

---

## 3. Tools: schemas are dicts, the registry is a dict, the loop is a while-loop **[CORE]** *(preview — taken on faith until Day 22)*

### Intuition

A model can't check your bank balance — it only produces text. **Tool use** bridges that: you *describe* your functions to the model (name, purpose, parameters), and instead of prose the model can reply "please run `get_balance` with `{"owner": "vinay"}`." **Your code** runs the real function and sends the result back; the model then writes its answer using it. The deep mechanics (why the model emits this, how it decides) are Day 22 material — but the *plumbing* is 100% Part 1:

- a **tool description** = a dict, whose `"input_schema"` key holds a dict describing parameters (this is JSON Schema — preview);
- the **dispatch table** = a dict mapping tool name → actual Python function (functions are objects; they sit in dicts as happily as ints do);
- the **loop** = `while` + an exception-guarded function call + `list.append`.

### Analogy — the model as a chef with no hands (and where it breaks)

The model is a **brilliant chef who cannot touch the kitchen**: it reads the menu of appliances you've published (tool schemas) and shouts precise instructions — "microwave, 90 seconds, this bowl" (a tool-use request with arguments). *You* press the buttons and report what happened (the tool result); the chef continues. *Where it breaks:* a chef trusts their own eyes; the model knows **only what you send back**. If your tool returns a wrong number, the model confidently serves a wrong answer — and if your code never sends a result at all, the conversation just stalls. The kitchen is entirely your responsibility.

### Visual — one full tool round-trip

```
 you: "What's vinay's balance?"          (history: 1 msg)
   │  messages.create(tools=[...])
   ▼
 model replies: stop_reason="tool_use"
   content includes: tool_use block {name:"get_balance", input:{"owner":"vinay"}, id:"toolu_x"}
   │
   │  YOUR code: fn = registry["get_balance"]; result = fn(owner="vinay")
   ▼
 append assistant msg (its content), then user msg with:
   {"type":"tool_result", "tool_use_id":"toolu_x", "content":"120.0"}
   │  messages.create(...) again
   ▼
 model replies: stop_reason="end_turn" → final text: "Vinay's balance is 120.0."
```

### Runnable example — a working tool-using agent in ~50 lines of P2 Python

```python
# tool_loop.py — pip install anthropic; set ANTHROPIC_API_KEY. Run: python tool_loop.py
# (preview — the tool-use PROTOCOL is Day 22; every STRUCTURE here is Day P2)
import json
import anthropic

client = anthropic.Anthropic()

# --- the real Python functions the model may ask us to run ---
BALANCES = {"vinay": 120.0, "alice": 300.0}          # demo stand-in for a database

def get_balance(owner):
    if owner not in BALANCES:
        raise KeyError(f"no account for {owner!r}")
    return BALANCES[owner]

# --- the registry: a dict mapping tool NAME → FUNCTION (functions are values!) ---
TOOL_FUNCTIONS = {"get_balance": get_balance}

# --- the schema: a dict describing the tool to the model (JSON Schema — preview) ---
TOOLS = [
    {
        "name": "get_balance",
        "description": "Look up a customer's current account balance by owner name.",
        "input_schema": {
            "type": "object",
            "properties": {"owner": {"type": "string", "description": "account owner"}},
            "required": ["owner"],
        },
    }
]

messages = [{"role": "user", "content": "What is vinay's balance? One short sentence."}]

MAX_ITERS = 5                                        # never loop unbounded
for _ in range(MAX_ITERS):
    response = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=1000,
        tools=TOOLS,
        messages=messages,
    )
    if response.stop_reason != "tool_use":           # model is done — no tool wanted
        break

    messages.append({"role": "assistant", "content": response.content})

    tool_results = []                                # one result per requested call
    for block in response.content:
        if block.type == "tool_use":
            print(f"model requests: {block.name}({json.dumps(block.input)})")
            fn = TOOL_FUNCTIONS[block.name]          # dict lookup → a function
            try:
                result = fn(**block.input)           # run it with the model's args
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,         # ties result to request
                    "content": json.dumps(result),   # results travel as JSON text
                })
            except Exception as e:                   # tool failed? TELL the model
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": f"{type(e).__name__}: {e}",
                    "is_error": True,
                })
    messages.append({"role": "user", "content": tool_results})

print(next(b.text for b in response.content if b.type == "text"))
```

```bash
python tool_loop.py
# -> model requests: get_balance({"owner": "vinay"})
# -> Vinay's balance is 120.0.        (wording varies per run)
```

**Why this works, line by line.** `TOOL_FUNCTIONS` is the day's quietest big idea: **functions are objects**, so a dict can map strings to them, and `TOOL_FUNCTIONS[block.name]` turns the model's *text* choice of tool into an actual callable — that's a dispatch table, the pattern behind every plugin system and web-framework router you'll ever meet. `TOOLS` is dicts-in-dicts, nothing more; read `input_schema` as "a dict that documents my function's parameters." The loop is `for _ in range(MAX_ITERS)` rather than `while True` because a model may keep requesting tools — an unbounded loop against a paid API is a self-inflicted outage, so the guard is non-negotiable. `response.stop_reason != "tool_use"` is a plain string comparison steering plain control flow (P1's `if`/`break`). When a tool *is* requested: we append the assistant's content (the model must see its own request in history), execute via `fn(**block.input)` — the `**` unpacks the model's argument dict `{"owner": "vinay"}` into `owner="vinay"`, dict-to-keyword-arguments — and package the result with `json.dumps` (§9b: results travel as text). The `except Exception` branch is Part 1 §8 earning its keep in production: a failing tool must not crash the loop — it becomes a `tool_result` with `"is_error": True`, and the *model* gets to read the exception message and adapt (ask a different way, apologize, try another owner). Failure as data, not as a crash — remember this line; it comes back in Part 3.

---

## 4. An agent is a class; failures are exceptions; persistence is JSON **[WORKING]**

### Intuition

Look at what §2 and §3 accumulated: a client object, a history list, tool dicts, error handling — state and behavior, sprawled across module-level globals and loose functions. Part 1 §6 named this exact smell and its exact fix: **a class**. `history` becomes `self.history`, `ask()` becomes a method, and suddenly you can run two independent conversations (`support = Agent(...)`, `research = Agent(...)`) just by making two instances — the cookie cutter paying off. Add §8 (catch the SDK's exception types) and §9 (save/load history as JSON via pathlib) and you have, honestly, the skeleton of every agent framework you'll ever read.

### Worked example — the SDK's exceptions are a class hierarchy you already understand

The `anthropic` package defines exception classes for each failure mode, and you catch them with §8's most-specific-first rule:

```
anthropic.RateLimitError        → you're calling too fast: wait, retry   (preview: Day 23)
anthropic.AuthenticationError   → bad/missing API key: fix config, no retry
anthropic.APIConnectionError    → network problem: retry later
anthropic.APIStatusError        → other API-level failures (has .status_code)
```

They're just classes inheriting from `Exception` — the same one-line pattern as `InsufficientFunds` in Part 1, written by Anthropic's engineers instead of you.

### Runnable example — a persistent, failure-tolerant Agent class

```python
# agent.py — pip install anthropic; set ANTHROPIC_API_KEY. Run: python agent.py
import json
from pathlib import Path
import anthropic

class Agent:
    """State (client + history) bundled with behavior (ask/save/load). Pure Part 1."""

    def __init__(self, system_prompt, history_file="history.json"):
        self.client = anthropic.Anthropic()      # attribute: the SDK client object
        self.system_prompt = system_prompt       # attribute: fixed instructions
        self.history = []                        # attribute: list of message dicts
        self.history_file = Path(history_file)   # attribute: a pathlib.Path

    def ask(self, user_text):
        self.history.append({"role": "user", "content": user_text})
        try:
            response = self.client.messages.create(
                model="claude-opus-4-8",
                max_tokens=1000,
                system=self.system_prompt,        # (preview: system prompts, Day 22)
                messages=self.history,
            )
        except anthropic.AuthenticationError:
            self.history.pop()                    # undo the append — call never happened
            return "[config error: check ANTHROPIC_API_KEY]"
        except anthropic.RateLimitError:
            self.history.pop()
            return "[rate limited: try again shortly]"     # (retries done right: Day 23)
        reply = next(b.text for b in response.content if b.type == "text")
        self.history.append({"role": "assistant", "content": reply})
        return reply

    def save(self):
        self.history_file.write_text(json.dumps(self.history, indent=2))

    def load(self):
        if self.history_file.exists():
            self.history = json.loads(self.history_file.read_text())


agent = Agent("You are terse. Answer in at most one sentence.")
agent.load()                                     # resume, if a previous run saved
print(agent.ask("Remember: my project is called AgenticAI-learning. Say noted."))
print(agent.ask("What is my project called?"))
agent.save()
print(f"saved {len(agent.history)} turns to {agent.history_file}")
```

```bash
python agent.py
# -> Noted.
# -> Your project is called AgenticAI-learning.     (wording varies)
# -> saved 4 turns to history.json

python agent.py     # run it AGAIN — it loads history.json and keeps the memory
# -> Noted.
# -> Your project is called AgenticAI-learning.
# -> saved 8 turns to history.json
```

**Why this works, line by line.** `__init__` gathers all four pieces of agent state as instance attributes, so *everything about one conversation lives on one object* — construct a second `Agent("You are verbose.")` and it gets its own client, its own history, its own file; that isolation is exactly what §6's "independent instances" section promised. In `ask`, the `try/except` chain catches the SDK's typed exceptions most-specific-first, and each handler does the honest state repair: `self.history.pop()` removes the user turn that never got answered, keeping the list's user/assistant alternation intact — a small move that separates toy code from code that survives failure. `save` is Part 1's §9 pair in two calls: history (a list of dicts) is JSON's native shape, so `json.dumps` serializes it losslessly and `Path.write_text` puts it on disk; `load` reverses it, guarded by `.exists()` so first runs don't crash — which is why the *second* `python agent.py` still knows your project name: the "memory" survived the process because *you* persisted the list. Between runs, open `history.json` — it's your conversation, human-readable, because `indent=2`.

---

## Part 2 — practice questions

(1) In the request dict, what are the exact types of `messages`, `messages[0]`, and `messages[0]["role"]`? (2) Why must the whole history list be sent on every call — where does the "memory" actually live? (3) In `TOOL_FUNCTIONS[block.name]`, what is being looked up, and what type comes back? (4) Why does the tool loop use `MAX_ITERS` instead of `while True`? (5) Why does a failing tool become a `tool_result` with `is_error: True` instead of crashing the loop — who is the error message *for*? (6) What does `fn(**block.input)` do, mechanically, when `block.input` is `{"owner": "vinay"}`? (7) Why does `ask` call `self.history.pop()` inside its `except` handlers — what invariant is it protecting?

---

# PART 3 — THE BRIDGE: One Set of Constructs, Two Worlds

## The payoff of the framing note

The opening claimed that *every agent program is made of exactly today's parts*. Parts 1 and 2 have now shown both sides; the bridge is a straight mapping — every row below points **left at a Part 1 section, right at the Part 2 code that used it**:

| Part 1 construct | §  | Where it appeared in Part 2 | § |
|---|---|---|---|
| `dict` (key→value, nesting, `.get` vs `[]`) | 1.2 | the request body; each message; tool schemas; `block.input` | 2.1, 2.3 |
| `list` (ordered, `append`, comprehensions over it) | 1.1, 1.5 | `messages` history; `response.content`; `tool_results` | 2.1–2.3 |
| dict as **dispatch table** | 1.2 | `TOOL_FUNCTIONS`: name → function | 2.3 |
| `class` (state + behavior, `self`, instances) | 1.6 | `anthropic.Anthropic()`, `response`, and your own `Agent` | 2.1, 2.4 |
| exceptions (`try/except/raise`, custom types) | 1.8 | SDK error classes; tool failure → `is_error` result | 2.3, 2.4 |
| modules & imports | 1.7 | `import anthropic` — same machinery as `import bank` | all |
| `json` + `pathlib` | 1.9 | tool results as JSON text; `history.json` persistence | 2.3, 2.4 |
| `tuple` | 1.4 | quietly everywhere: `.items()` pairs, `most_common` results | — |

Three bridge observations worth internalizing, because they'll recur for the next 98 days:

**1. `BankAccount` and `Agent` are the same design.** Compare them field by field: protected state (`balance` / `history`), an initializer that sets it up, methods that are the *only* sanctioned way to change it (`deposit`/`withdraw` / `ask`), and rule-enforcement at the boundary (`raise InsufficientFunds` on overdraft / `pop()`-and-report on API failure). When Day 22+ hands you agent frameworks, read their `Agent` classes as bank accounts with fancier vaults — the design instinct transfers 1:1.

**2. Failure-as-data is the bridge's deepest idea.** Part 1 §8 taught exceptions as *interruptions that climb*. Part 2 §3 did something subtler: it **caught** the exception and turned it into *data* (`is_error: True`) sent to another reasoner — because the model can't catch your exceptions, it can only read messages. Handle-or-forward is a boundary decision you'll make constantly: inside your process, raise and catch; across a boundary (an API response, a tool result, a queue message), convert failure into structured data the other side can act on. Week 3's HTTP status codes are exactly this idea again, standardized.

**3. Persistence is just serialization of the structures you chose.** The Build task saves an account as JSON; the `Agent` saves its history as JSON. Both work *because* the in-memory structures (dicts, lists, strings, numbers) were chosen from JSON's native palette. Choose your Part 1 container well and persistence, logging, and API transport come almost free; choose exotic in-memory shapes and every boundary becomes a translation chore.

### Runnable example — the bridge in one program: a persisted, module-split, exception-guarded conversation store

Everything below was taught above; nothing new. Two files:

```python
# convo_store.py — the library module (compare: bank.py)
import json
from pathlib import Path

class CorruptHistory(Exception):
    """Raised when the history file exists but isn't valid JSON."""

class ConversationStore:
    """Owns a message-history list and its JSON file. State + behavior, bundled."""

    def __init__(self, path):
        self.path = Path(path)
        self.messages = []

    def add(self, role, content):
        if role not in {"user", "assistant"}:          # set membership check (§1.3)
            raise ValueError(f"bad role: {role!r}")
        self.messages.append({"role": role, "content": content})

    def user_turns(self):
        return [m["content"] for m in self.messages if m["role"] == "user"]

    def save(self):
        self.path.write_text(json.dumps(self.messages, indent=2))

    def load(self):
        if not self.path.exists():
            return 0
        try:
            self.messages = json.loads(self.path.read_text())
        except json.JSONDecodeError as e:
            raise CorruptHistory(f"{self.path} is not valid JSON: {e}")
        return len(self.messages)
```

```python
# use_store.py — the entry point (compare: main.py)
from convo_store import ConversationStore, CorruptHistory

store = ConversationStore("demo_history.json")
try:
    n = store.load()
    print(f"loaded {n} prior messages")
except CorruptHistory as e:
    print(f"starting fresh: {e}")
    store.messages = []

store.add("user", "What is a dispatch table?")
store.add("assistant", "A dict mapping names to functions.")
store.save()
print("user turns so far:", store.user_turns())
```

```bash
python use_store.py
# -> loaded 0 prior messages
# -> user turns so far: ['What is a dispatch table?']

python use_store.py        # second run: the file now exists
# -> loaded 2 prior messages
# -> user turns so far: ['What is a dispatch table?', 'What is a dispatch table?']
```

**Why this works, line by line.** The module split mirrors §1.7's bank exactly: `convo_store.py` defines and stays quiet; `use_store.py` imports and drives. `ConversationStore` is the `Agent` class minus the API call — proof that the *data-management* half of an agent is pure P2 material. `add` validates with a set-membership check and enforces its contract by raising (§1.8's "raise on contract violation," same as the overdraft). `user_turns` is the §1.5 filter-and-extract comprehension. `load` shows layered exceptions: it *catches* the library's `json.JSONDecodeError` and *re-raises* it as the domain-specific `CorruptHistory` — callers of your class handle *your* vocabulary, not the internals' (corrupt the JSON file by hand and rerun to watch the `except CorruptHistory` branch fire). Swap `add()` for a real `messages.create()` call and this *is* Part 2 §4 — that's the bridge.

---

# Wrap-Up

## Cheat Sheet

```python
# ── COLLECTIONS ──────────────────────────────────────────────
lst = [1, 2]; lst.append(3); lst[0]; lst[-1]; 3 in lst    # ordered, mutable; `in` is O(n)
d = {"k": 1}; d["k"]; d.get("x", 0); d["new"] = 2         # key→value, O(1); insertion-ordered (3.7+)
for k, v in d.items(): ...                                # iterate pairs (as tuples)
s = {1, 2}; s.add(2); 2 in s; a & b; a | b; a - b         # unique, unordered, O(1) membership; set() not {}
t = (1, 2); x, y = t                                      # frozen record; unpacking; dict-key-able
# choose:  order+change→list | name→value→dict | seen?/dedup→set | fixed record→tuple

# ── COMPREHENSIONS ───────────────────────────────────────────
[x*2 for x in xs if x > 0]                                # list: transform + filter
{k: f(k) for k in ks}   /   {f(x) for x in xs}            # dict / set
[m["content"] for m in msgs if m["role"] == "user"]       # THE agent-code pattern

# ── CLASSES ──────────────────────────────────────────────────
class Account:
    def __init__(self, owner, balance=0.0):              # runs at Account(...)
        self.owner, self.balance = owner, balance         # attributes on THIS instance
    def deposit(self, amount):                            # a.deposit(x) == Account.deposit(a, x)
        self.balance += amount

# ── MODULES ──────────────────────────────────────────────────
import bank; bank.BankAccount                             # load file once (cached), use with prefix
from bank import BankAccount                              # copy one name in
if __name__ == "__main__": ...                            # run-directly-only block

# ── EXCEPTIONS ───────────────────────────────────────────────
class MyError(Exception): pass                            # custom type: one line
try: risky()
except MyError as e: handle(e)                            # most specific first; never bare except:
finally: cleanup()                                        # every exit path
raise MyError("what went wrong, with values")             # enforce your contract

# ── STDLIB ───────────────────────────────────────────────────
from pathlib import Path; p = Path("d")/"f.json"; p.write_text(s); p.read_text(); p.exists()
import json; text = json.dumps(obj); obj = json.loads(text)      # memory ↔ text
from collections import Counter; Counter(words).most_common(3)

# ── THE AGENT SHAPES (Part 2) ────────────────────────────────
history = [{"role": "user", "content": "..."}]            # conversation = list of dicts
registry = {"tool_name": python_function}                 # dispatch table = dict
# loop: for _ in range(MAX_ITERS): call → if stop_reason != "tool_use": break
#       → run registry[name](**block.input) in try/except → append tool_result (is_error on failure)
```

## Build This — the Day P2 build task

**Task (from the study plan):** model a tiny domain in classes — a `BankAccount` with `deposit`/`withdraw` that **raises on overdraft** — split across **two modules** with a `main.py` that imports and uses it; **write to and read back a `.json` file**.

Concretely, produce:

1. **`bank.py`** — a custom `InsufficientFunds(Exception)`; a `BankAccount` class with `__init__(self, owner, balance=0.0)`, `deposit(amount)` (raise `ValueError` if `amount <= 0`), and `withdraw(amount)` (raise `InsufficientFunds` — with a message containing both numbers — if `amount > self.balance`).
2. **`main.py`** — imports from `bank`; creates an account; deposits; attempts an overdraft inside `try/except InsufficientFunds` and prints the caught message; withdraws a valid amount; saves `{"owner": ..., "balance": ...}` to `account.json` with `json.dumps` + `pathlib.Path.write_text`; reads it back with `read_text` + `json.loads` and constructs a **new** `BankAccount` from the loaded dict.

**Definition of done — this exact behavior, runnable:**

```bash
python main.py
# -> after deposit: 150.0
# -> blocked: cannot withdraw 500.0: balance is 150.0
# -> after withdraw: 120.0
# -> loaded from disk: {'owner': 'vinay', 'balance': 120.0}
# -> restored object: vinay has 120.0
```

(That transcript is real — the reference solution produces it byte-for-byte.) You're done when: the overdraft **raises and is caught** (not an `if` that prints), the two-module split works (`python main.py` runs; `bank.py` contains no top-level demo output), `account.json` exists on disk and is valid JSON, and a *second* run restores state from the file. **Stretch:** add a `history` list attribute recording each operation as a tuple `("deposit", 50.0)`, and persist it too.

## Active Recall & Self-Test

Answer from memory — no scrolling up. Retrieval is what builds durable memory.

1. Name the four workhorse collections and the one-line "when to use" for each.
2. Why is `x in my_list` O(n) but `x in my_set` O(1)? What does the set *do* with `x`?
3. Walk a dict lookup: what happens between `d["key"]` and the value coming back? What's a collision, and how does CPython handle it?
4. Since which Python version is dict insertion order guaranteed, and what implementation change made it possible?
5. Why can't a list be a dict key? What specifically would break?
6. What does `self` receive, and what is `acct.deposit(50)` sugar for?
7. What happens — all five steps — when Python executes `from bank import BankAccount` for the first time? And the second time?
8. What is `__name__` when a file is run directly vs imported, and what idiom exploits that?
9. When should a function `raise` rather than return a sentinel like `None`? What did the Build task raise, and why?
10. `json.dumps` vs `json.loads` — which direction is which, and what happens to a tuple on a round-trip?
11. In agent code: what container is a conversation history, a tool registry, a tool schema? Why does a failing tool become `is_error: True` data instead of a crash?

**Teach-back prompt (60 seconds, out loud, to an imaginary colleague):** *"Explain why classes exist — start from the pain of dicts-plus-loose-functions, define class/instance/attribute/method/`__init__`, demystify `self` with the `ClassName.method(instance, ...)` equivalence, and land on why an AI agent is naturally a class."* If you stumble, reread §1.6 and try again — this exact question is also a real interview staple.

## Spaced-Repetition Flashcards

| Q | A |
|---|---|
| Container for name→value lookup? | `dict` — hashed keys, O(1) |
| Container for dedup / seen-before? | `set` — hash table of keys only, O(1) membership |
| Container for a fixed record / dict key? | `tuple` — immutable, hashable |
| Why is `list.append` fast on average? | Over-allocation: spare capacity absorbs most appends; rare copies amortize to O(1) |
| Dict insertion order guaranteed since…? | Python 3.7 (compact dict: dense entries array + index table) |
| Why must dict keys be immutable? | Hash decides the slot; a mutated key's entry becomes unfindable |
| `{}` creates a…? | dict. Empty set is `set()` |
| `self` is…? | The instance, auto-passed as the first argument: `a.m(x)` ≡ `Class.m(a, x)` |
| Where do instance attributes live? | In the instance's own dict: `obj.__dict__` |
| What does `import mod` do first time / second time? | Execute the file once, cache in `sys.modules` / return the cached module |
| `if __name__ == "__main__":` means? | Run only when executed directly, not when imported |
| `raise` vs return `None`? | Raise: unignorable, carries type+message, climbs to a handler. `None`: silently ignorable |
| `finally` runs when? | On every exit path — success, exception, even `return` |
| `json.dumps` / `json.loads`? | object → JSON string / JSON string → object |
| Conversation history in agent code? | A list of `{"role": ..., "content": ...}` dicts you `append` to and resend in full |
| Tool registry pattern? | Dict mapping tool name (str) → Python function; dispatch = one lookup |
| Why `MAX_ITERS` in a tool loop? | The model may keep requesting tools; unbounded loops against a paid API must be impossible |

## Primary Sources

Verify against these; they're the ground truth this note was checked against:

- **Python Tutorial §5 — Data Structures** (lists, dicts, sets, tuples, comprehensions): docs.python.org/3/tutorial/datastructures.html
- **Python Tutorial §9 — Classes**: docs.python.org/3/tutorial/classes.html
- **Python Tutorial §6 — Modules** (imports, packages, `__name__`, the module search path): docs.python.org/3/tutorial/modules.html
- **Python Tutorial §8 — Errors and Exceptions**: docs.python.org/3/tutorial/errors.html
- **Library reference:** `pathlib` (docs.python.org/3/library/pathlib.html) · `json` (…/json.html) · `collections` (…/collections.html)
- **Dict ordering guarantee:** the "What's New in Python 3.7" notes declare insertion order an official language feature (docs.python.org/3/whatsnew/3.7.html). The compact-dict design is documented in the CPython source (`Objects/dictobject.c` and `Objects/dictnotes.txt`).
- **List over-allocation:** CPython source, `Objects/listobject.c` (`list_resize` — the growth-factor comment). Exact growth factors change between versions — verify against your version if it ever matters.
- **Anthropic SDK (Part 2):** the official Python SDK repo, github.com/anthropics/anthropic-sdk-python, and platform.claude.com/docs. Model IDs and API parameters drift fast — **verify before relying on them**.

## Key Takeaways & Summary

**10-second explanation.** Python gives you four containers — list (ordered, changing), dict (name→value, the important one), set (uniqueness), tuple (fixed record) — plus classes to bundle state with behavior, modules to span files, exceptions to survive failure, and a standard library so you don't reinvent paths/JSON/counting. Every AI agent is built from exactly these parts.

**1-minute explanation.** Choosing the right container is the day's core skill: order-that-changes → list; lookup-by-name → dict (a hash table: keys are hashed to slots, so lookup is O(1), which is also why keys must be immutable — and since 3.7, insertion order is guaranteed); have-I-seen-it / dedup → set (a dict with no values); fixed record → tuple (immutable, so it can be a dict key). Comprehensions build these from loops in one readable expression. Classes fix the "dicts + loose functions" soup by bundling state (attributes, set in `__init__`) with behavior (methods, where `self` is just the instance auto-passed first). Modules make every `.py` file a reusable unit — `import` executes the file once and caches it. Exceptions make failure unignorable: `raise` on contract violations, catch specifically, `finally` for cleanup. And `pathlib`/`json`/`collections` cover files, serialization, and counting so you never hand-roll them.

**5-minute explanation.** Add the mechanisms: a list is a dynamic array of pointers with ~12% over-allocation, giving amortized-O(1) append but O(n) front-insertion and O(n) membership scans — the scan being precisely what hash-based dict/set eliminate by computing each key's slot from `hash(key)`, probing on collisions, and resizing at ~2/3 full; the compact-dict layout (dense entries array + small index table) is what made insertion order cheap enough to guarantee in 3.7. Mutability draws the map: mutable things (list/dict/set) can't be hashed because their slot would go stale; immutable tuples can, which is why they serve as compound keys and as the shape of "multiple return values." Classes are dicts underneath (`obj.__dict__`), and a method call is sugar — `a.m(x)` is `Class.m(a, x)` — which demystifies `self` completely. Imports execute-once-then-cache via `sys.modules`, and `__name__ == "__main__"` distinguishes script-mode from library-mode. Exceptions are objects that climb the call stack, abandoning frames until a type-matching `except` catches them; you define domain types in one line and raise them to make contract violations unignorable. Then the punchline: an LLM API request is a dict whose `messages` is a list of dicts; multi-turn memory is *your* list, re-sent whole; tools are schema dicts plus a name→function registry dict driven by a bounded loop; a failing tool is *caught* and forwarded to the model as `is_error` data; an agent is a class holding client + history; persistence is `json.dumps` into a `Path`. Same constructs, production clothes.

**Expert summary.** P2 is CPython's object model in miniature: everything is a pointer to a heap object; sequences are pointer arrays (contiguous, over-allocated) while mappings are open-addressed hash tables (randomized string hashing, perturbed probing, 2/3 load factor, compact ordered layout post-3.6); hashability ≡ stable-identity-under-equality, which partitions the built-ins into dict-key-able and not. Namespaces are dicts at every level — instance `__dict__`, class `__dict__`, module `__dict__`, `sys.modules` — so attribute access, method resolution, and imports are all dict lookups with caching and precedence rules layered on. Exceptions implement non-local exit with typed, hierarchical dispatch. The agentic mapping is structural, not metaphorical: conversation state is an append-only list of role-tagged dicts (the wire format is JSON, so the in-memory and serialized shapes coincide); tool dispatch is a first-class-function table; the agent loop is bounded iteration over a stop-reason state machine; and cross-boundary failure is reified into data (`is_error`) because exceptions don't cross process boundaries — the same insight that HTTP status codes institutionalize.

## Mental Models

- **Parking garage (list):** numbered spots, drive straight to any number; no gaps allowed, so removals shift everyone behind.
- **Coat check (dict):** ticket → hook, no searching; same ticket replaces the coat; tickets must never change shape (hashable keys).
- **Guest list (set):** one question only — "on the list?"; writing a name twice changes nothing; no order.
- **Laminated ID (tuple):** fields sealed in fixed positions; trusted as a key *because* it can't change.
- **Cookie cutter (class):** one blueprint, many independent cookies; the cutter holds the *behavior*, each cookie holds its own *state*.
- **Toolbox wall (modules):** import = bring a labeled box to your bench; the box is stocked once (cached) no matter how many benches request it.
- **Fire alarm (exceptions):** unignorable, escalates floor by floor until someone competent responds, or the building evacuates (traceback).
- **Chef with no hands (tool use):** the model shouts precise instructions; your code presses the buttons and reports back; it knows only what you tell it.

## Mnemonics

- **L-D-S-T = "Lists Do Sequence, Tables (dicts) Do Lookup, Sets Do Uniqueness, Tuples Don't change."**
- Container chooser — ask in order: **"Order? Name? Seen? Sealed?"** → list, dict, set, tuple.
- `self`: **"the thing left of the dot arrives first."**
- Import: **"import = execute once, cache forever."**
- JSON directions: dump**s** = **s**tring out (object→text), load**s** = **s**tring in (text→object).
- Exceptions: **"raise to refuse, except to expect, finally to free."**
- Empty containers: **"braces alone belong to dict"** — `{}` is a dict; `set()` spells itself out.

## Common Confusions

| Confusion | Resolution |
|---|---|
| `b = a` copies my list | It copies the *pointer*. Both names, one list. Use `a.copy()` for a new one. |
| `{}` is an empty set | It's an empty **dict** (dicts owned the braces first). Empty set: `set()`. |
| Dicts are unordered | Pre-3.7 folklore. Since 3.7, insertion order is a language guarantee. (Sets remain unordered.) |
| `d["missing"]` and `d.get("missing")` are interchangeable | `[]` crashes (right for *required* keys — fail loudly); `.get` defaults (right for *optional* keys). It's a contract choice. |
| `(42)` is a one-item tuple | It's the int 42 in parens. The **comma** makes tuples: `(42,)`. |
| `self` is a magic keyword | It's a plain parameter receiving the instance: `a.m(x)` ≡ `Class.m(a, x)`. The convention is universal; the mechanism is ordinary. |
| Defining a class creates an object | The class is the blueprint; `ClassName(...)` creates instances (and triggers `__init__`). |
| `import` just "makes names available" | It **executes the entire file** (once), then caches it. Top-level side effects fire at import time. |
| `except:` bare is a safe catch-all | It swallows *everything*, including typos (`NameError`) in your own code. Catch specific types. |
| `finally` is skipped if `try` returns | `finally` runs on **every** exit path — return, raise, or success. |
| `json.dumps` writes to a file | It returns a **string** (`s` = string). File writing is separate (`Path.write_text`) — or use `json.dump` (no `s`) with a file object. |
| Tuples survive a JSON round-trip | JSON has no tuple type — they come back as **lists**. (And dict keys come back as strings; sets don't serialize at all.) |
| The model runs my tools | Never. The model *asks*; **your code** looks up the function (registry dict), runs it, and reports back. |
| The API remembers my conversation | Your **list** remembers. You re-send the whole history every call; drop an entry and the model never knew it. |
