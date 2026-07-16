# Phase 1 — Week 3: HTTP Semantics & Tool-Call Internals

> **What this covers:** HTTP done *properly* — idempotency, the real status-code taxonomy, caching, content negotiation (backend) — and the internals of function calling: the tool-definition contract, constrained decoding, and token economics (agentic). Then: **a tool is an API, and its result is a status code the model reasons over.**
> **Prerequisite:** [Week 2](phase1-week2.md) — the event loop and the raw ReAct loop.
> **The one idea that unites this week:** *a tool is an API call, so the HTTP contract — which calls are safe to retry, and what a status code *means* — is exactly how a tool should tell the model what to do next.*
>
> **How to read this:** every concept runs the ladder **intuition → analogy → concrete worked example → diagram → under-the-hood**. When something feels abstract, jump to its "**Example**" block.
>
> **Depth tiers:** **[CORE]** open every box · **[WORKING]** use it, know tradeoffs · **[AWARE]** know it exists.

---

# PART 1 — BACKEND: HTTP Semantics Done Properly

## Overview & motivation — HTTP is a contract, not plumbing

**The intuition.** Most people picture HTTP as invisible pipes: "it just moves data." But HTTP is really a **contract written in a shared vocabulary**, so that a client and a server who have never met can agree on what an operation *means* and what to do when it goes sideways. When you send `DELETE`, the server knows it's safe to run twice. When it answers `429`, you know to wait and retry rather than give up. Get the vocabulary right and clients behave correctly *automatically*; get it wrong and a retry after a timeout silently double-charges a customer.

**Why it matters for this course.** Every action an agent takes is an HTTP request under the hood ([Week 2](phase1-week2.md); Part 3 below). So the HTTP contract *is* the agent's contract: the rules that decide whether a tool is safe to retry, and how a tool tells the model "fix your input" vs. "try again later," are HTTP rules. Learn them here as backend; wield them as agent design in Part 3.

## First principles & terminology **[CORE]**

- **HTTP** (HyperText Transfer Protocol) — a request/response protocol. The client sends a **request** (method + path + headers + optional body); the server returns a **response** (status code + headers + optional body).
- **Method (verb)** — the *intent*: `GET` (read), `POST` (create/act), `PUT` (replace), `PATCH` (partial update), `DELETE` (remove).
- **Header** — a key/value metadata line (e.g. `Content-Type: application/json`) carried alongside the body.
- **Status code** — a 3-digit result class: `1xx` info · `2xx` success · `3xx` redirect/cache · `4xx` **client** error (you sent something wrong) · `5xx` **server** error (the server failed).
- **Safe method** — no side effects, read-only (`GET`, `HEAD`). **Idempotent method** — repeating it equals doing it once (below).

---

## 1. Idempotency — the property that governs safe retries **[CORE]**

### Intuition & first principles

An operation is **idempotent** if doing it *N* times leaves the server in the **same state** as doing it once. Crucially, this is about the *effect on the world*, not about getting an identical response body back.

- **Idempotent by contract:** `GET` (reading changes nothing), `PUT` (set the resource to value X — setting it to X twice = X), `DELETE` (delete it — deleting an already-deleted thing leaves it deleted).
- **NOT idempotent:** `POST` (two `POST /orders` = two orders), and usually `PATCH`.

### Analogy (and where it breaks)

An **elevator call button**: pressing it once or five times summons the elevator exactly once — idempotent. A **vending machine**: each press of "buy" dispenses another snack and charges again — not idempotent. *Where it breaks:* the elevator "remembers" it's already lit; a raw `POST` endpoint has no such memory *unless you build it* (that's the idempotency-key work in [Week 4](phase1-week4.md)).

### Why it matters so much — the concrete example (worked)

Networks lose responses. Here is the exact failure that idempotency exists to prevent:

```
t0  client → POST /charge {amount: 10}
t1  server charges the card $10, prepares "200 OK, charge #9"
t2  ✗ the RESPONSE packet is lost on the flaky network
t3  client waits... times out... assumes it failed... RETRIES:
    client → POST /charge {amount: 10}
t4  server (no memory of t1) charges the card $10 AGAIN → "200 OK, charge #10"
    RESULT: customer charged $20 for one purchase.
```

The client did nothing wrong — retrying after a timeout is *correct* behavior. The bug is that `POST /charge` is not idempotent. This single scenario drives the idempotency-key pattern ([Week 4](phase1-week4.md)) and, in the agent world, safe tool retries (Part 3). Note that a `GET /balance` retried at t3 would be harmless — that's the power of idempotency.

### Runnable example — non-idempotent charge vs. idempotent update (FastAPI)

The double-charge above isn't hypothetical. Here it is as code you can actually run.

```python
# app.py — run with:  uvicorn app:app --reload
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

# Tiny in-memory "databases". In real life these are Postgres/Redis.
charges: list[dict] = []          # every POST appends one row
profiles: dict[str, dict] = {}    # keyed by user id


class ChargeIn(BaseModel):
    amount: int


@app.post("/charge")                       # NOT idempotent — appends
def charge(body: ChargeIn):
    charge_id = len(charges) + 1
    charges.append({"id": charge_id, "amount": body.amount})
    return {"charge_id": charge_id, "total_charges": len(charges)}


class ProfileIn(BaseModel):
    display_name: str


@app.put("/profile/{user_id}")             # idempotent — assigns
def set_profile(user_id: str, body: ProfileIn):
    profiles[user_id] = {"display_name": body.display_name}
    return {"user_id": user_id, **profiles[user_id]}
```

Reproduce the failure and the safe case from a second terminal:

```bash
# POST is NOT idempotent — "retrying" creates a second charge
curl -s -X POST localhost:8000/charge -H 'content-type: application/json' -d '{"amount":10}'
# -> {"charge_id":1,"total_charges":1}
curl -s -X POST localhost:8000/charge -H 'content-type: application/json' -d '{"amount":10}'
# -> {"charge_id":2,"total_charges":2}   ← the double charge, exactly as in the timeline above

# PUT IS idempotent — "retrying" leaves the same single state
curl -s -X PUT localhost:8000/profile/u1 -H 'content-type: application/json' -d '{"display_name":"Vinay"}'
# -> {"user_id":"u1","display_name":"Vinay"}
curl -s -X PUT localhost:8000/profile/u1 -H 'content-type: application/json' -d '{"display_name":"Vinay"}'
# -> {"user_id":"u1","display_name":"Vinay"}   ← identical state; twice == once
```

**Why this proves the point, line by line.** `charges` is a **list**, so `charge()` *appends* on every call — two identical `POST`s leave **two** rows, doubling the effect on the world, which is the literal definition of *not* idempotent. `profiles` is a **dict keyed by `user_id`**, so `set_profile()` *assigns* (`profiles[user_id] = ...`) — two identical `PUT`s overwrite the same key with the same value, leaving **one** row in the same state. The property lives in **what the handler does to state** (append vs. assign), *not* in the verb label: a `POST` handler can be made idempotent (the idempotency-key work in [Week 4](phase1-week4.md)), and a `PUT` handler that appended to a list would *not* be. The verb is a promise to the caller; your handler code is what keeps that promise.

---

## 2. The status-code taxonomy — a precise vocabulary **[CORE]**

### Intuition

A status code is not a label; it is an **instruction to the caller about what to do next.** Return the wrong one and the caller reacts wrongly — retries a request that will never succeed, or gives up on one that just needed a moment. Think of them as traffic signals: red/yellow/green mean specific things, and painting a red light green causes crashes.

### The codes that matter, and their exact distinctions

| Code | Name | Precise meaning | Caller's correct action |
|---|---|---|---|
| `200/201/204` | OK / Created / No Content | success / created a resource / success, no body | proceed |
| `301/304` | Moved / Not Modified | redirect / your cached copy is still valid | follow / use cache |
| `400` | Bad Request | **syntactically** malformed — server can't even parse it | fix the request; don't retry as-is |
| `401` | Unauthorized | not authenticated (missing/invalid credentials) | authenticate, then retry |
| `403` | Forbidden | authenticated but not allowed | give up (or get permission) |
| `404` | Not Found | no such resource | don't retry; try a different target |
| `409` | Conflict | valid request, but conflicts with **current state** | re-read state, resolve, retry |
| `422` | Unprocessable Entity | parsed fine, but the **values are invalid** | fix the *data* |
| `429` | Too Many Requests | rate-limited | back off (honor `Retry-After`), retry later |
| `500/502/503` | Server errors | the server failed | retry with backoff **if the op is safe** |

### The classic trap: `400` vs `422` vs `409` — worked examples **[CORE]**

Take one endpoint, `POST /users`, and three different bad requests:

```
1. Body is broken JSON:      {"email": "a@b.com"           ← missing brace
   → 400 Bad Request         "I couldn't PARSE your request at all."

2. Body parses, email invalid: {"email": "not-an-email"}
   → 422 Unprocessable       "I parsed it fine, but the VALUE is invalid."

3. Body valid, username taken: {"email":"x@y.com","user":"vinay"} (already exists)
   → 409 Conflict            "The values are fine, but they CONFLICT with current state."
```

Collapsing all three into a generic `400` destroys the caller's ability to react correctly: on `422` it should fix the field; on `409` it should pick a different username or re-read; on `400` it should fix its serializer. This distinction becomes vivid in Part 3, where the "caller" is an LLM deciding what to do next.

### `4xx` vs `5xx` and retry safety — worked example **[CORE]**

- `4xx` = **you** did something wrong; retrying the *identical* request won't help (except `429` "wait" and `401` "authenticate").
- `5xx` = the **server** failed; retrying (with backoff) may succeed.

**The danger example:** you `POST /charge`, the server charges the card, then crashes *before* sending the response → the client sees `500`. If the client "safely retries the 5xx," it **double-charges** — because the operation wasn't idempotent and actually *did* complete. This is the exact reason `5xx` retries on non-idempotent operations require an idempotency key ([Week 4](phase1-week4.md)).

### Runnable example — `POST /users` returning the *correct* code for each failure (FastAPI)

This is the "Build This" endpoint. It returns `400` / `422` / `409` / `201`, each for its precise reason.

```python
# pip install "fastapi[standard]" "pydantic[email]"
from fastapi import FastAPI, HTTPException, Request
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse
from pydantic import BaseModel, EmailStr

app = FastAPI()
users: dict[str, dict] = {}          # keyed by username


class UserIn(BaseModel):
    username: str
    email: EmailStr                  # Pydantic checks this is a real email SHAPE


@app.post("/users", status_code=201)
def create_user(body: UserIn):
    if body.username in users:                              # values fine, but CONFLICT with state
        raise HTTPException(status_code=409, detail="username already taken")
    users[body.username] = {"email": body.email}
    return {"username": body.username, "email": body.email}   # 201 Created


# FastAPI's DEFAULT returns 422 even when the body can't be PARSED at all.
# To honor "400 = can't parse, 422 = parsed but value invalid", we downgrade
# the specific json_invalid error to 400 and leave every other case as 422.
@app.exception_handler(RequestValidationError)
async def on_validation_error(request: Request, exc: RequestValidationError):
    if any(err["type"] == "json_invalid" for err in exc.errors()):
        return JSONResponse(status_code=400, content={"error": "malformed JSON"})
    return JSONResponse(status_code=422,
                        content={"error": "invalid values", "detail": exc.errors()})
```

Drive all four outcomes (`-w '%{http_code}'` prints just the status code):

```bash
# 400 — body is not JSON (missing closing brace) → we can't PARSE it
curl -s -o /dev/null -w '%{http_code}\n' -X POST localhost:8000/users \
  -H 'content-type: application/json' -d '{"username":"a","email":"a@b.com"'
# -> 400

# 422 — parses fine, but the email VALUE is invalid
curl -s -o /dev/null -w '%{http_code}\n' -X POST localhost:8000/users \
  -H 'content-type: application/json' -d '{"username":"a","email":"not-an-email"}'
# -> 422

# 201 — created
curl -s -o /dev/null -w '%{http_code}\n' -X POST localhost:8000/users \
  -H 'content-type: application/json' -d '{"username":"vinay","email":"v@x.com"}'
# -> 201

# 409 — values valid, but username CONFLICTS with current state
curl -s -o /dev/null -w '%{http_code}\n' -X POST localhost:8000/users \
  -H 'content-type: application/json' -d '{"username":"vinay","email":"v2@x.com"}'
# -> 409
```

**What each mechanism is doing.** The `422` is *free*: FastAPI runs the `UserIn` Pydantic model against the parsed body, and `EmailStr` rejects `"not-an-email"` before your function body ever runs — that's the "I parsed it but the value is wrong" case. The `409` is *yours to raise*: only your code knows the username is already taken (a fact about **current state**, not about the request's shape), so you `raise HTTPException(409, ...)`. The `400` needs the custom handler because FastAPI/Starlette treat an unparseable body as *just another validation error* and would return `422` — but "I couldn't even parse your request" and "I parsed it but a field is wrong" are different instructions to the caller, so we split them by inspecting `err["type"] == "json_invalid"`. This distinction is exactly what Part 3 exploits: when the "caller" is an LLM, a `422` means *fix the argument* and a `409` means *re-read state and retry* — collapsing them into a generic `400` destroys the model's ability to recover.

---

## 3. Caching headers **[WORKING]**

### Intuition & worked example

Re-downloading something that hasn't changed wastes bandwidth and time. An **`ETag`** is a fingerprint (version tag) of a resource's current content; the client stores it and, next time, asks "only send it if it changed":

```
1st request:  GET /profile
   ← 200 OK, ETag: "v7", <body>              client caches body + "v7"

2nd request:  GET /profile   If-None-Match: "v7"
   (nothing changed)
   ← 304 Not Modified   (NO body!)           client reuses its cached copy
   (something changed)
   ← 200 OK, ETag: "v8", <new body>
```

The `304` saves sending the whole body. `Cache-Control` directives (`max-age=60`, `no-store`, `private`) state who may cache and for how long. Know the interface and when to reach for it; cache *invalidation strategy* is a Phase 2 topic. **[WORKING].**

### Runnable example — `ETag` / `If-None-Match` → `304` (FastAPI)

```python
import hashlib
from fastapi import FastAPI, Response, Header
from fastapi.responses import JSONResponse

app = FastAPI()
profile = {"name": "Vinay", "bio": "backend + agents"}


def etag_for(obj: dict) -> str:
    # A stable fingerprint of the CURRENT content. sorted() so key order
    # never changes the tag; slice keeps the header short.
    raw = repr(sorted(obj.items())).encode()
    return '"' + hashlib.sha256(raw).hexdigest()[:8] + '"'


@app.get("/profile")
def get_profile(if_none_match: str | None = Header(default=None)):
    current = etag_for(profile)
    if if_none_match == current:            # client's cached copy is still valid
        return Response(status_code=304)    # 304 Not Modified — NO body
    resp = JSONResponse(content=profile)
    resp.headers["ETag"] = current          # hand the client a tag to send back next time
    return resp
```

```bash
# 1st request: full body + the ETag
curl -si localhost:8000/profile | grep -E 'HTTP|etag'
# -> HTTP/1.1 200 OK
# -> etag: "3f8a1c9d"

# 2nd request: echo the tag back — nothing changed, so no body is sent
curl -si localhost:8000/profile -H 'If-None-Match: "3f8a1c9d"' | head -1
# -> HTTP/1.1 304 Not Modified
```

**How the exchange works.** `etag_for()` derives a version tag from the resource's *content* — change `profile` and the tag changes automatically. FastAPI maps the incoming `If-None-Match` request header onto the `if_none_match` parameter (header names are case-insensitive and dashes become underscores). If the client's tag equals the current one, the content hasn't changed, so we return a bare `304` with **no body** — the client reuses its cached copy and we save the bandwidth of re-sending it. Only when the tag differs do we send the full `200` plus the new `ETag`. This is the same "avoid re-doing work whose inputs didn't change" instinct that prompt caching applies to token prefixes in Phase 2.

---

## 4. Content negotiation **[WORKING]**

The client states what representation it wants via headers, and **one URL** serves the best match:

```
GET /report   Accept: application/json   → the JSON representation
GET /report   Accept: text/html          → the HTML page — SAME url
GET /report   Accept-Encoding: gzip      → a compressed body
```

The server picks and echoes its choice in `Content-Type` / `Content-Encoding`. One resource, many representations, chosen by the caller's stated preference. **[WORKING].**

### Runnable example — one URL, two representations (FastAPI)

```python
from fastapi import FastAPI, Header
from fastapi.responses import JSONResponse, HTMLResponse

app = FastAPI()
report = {"title": "Q3", "revenue": 42}


@app.get("/report")
def get_report(accept: str = Header(default="application/json")):
    if "text/html" in accept:                       # caller asked for a page
        return HTMLResponse(f"<h1>{report['title']}</h1><p>Revenue: {report['revenue']}</p>")
    return JSONResponse(report)                      # default representation
```

```bash
curl -s localhost:8000/report -H 'Accept: application/json'
# -> {"title":"Q3","revenue":42}

curl -s localhost:8000/report -H 'Accept: text/html'
# -> <h1>Q3</h1><p>Revenue: 42</p>
```

**The point.** Same path `/report`, same underlying `report` data — the `Accept` header alone selects which representation the caller gets, and the response's `Content-Type` (set automatically by `JSONResponse` vs. `HTMLResponse`) echoes the choice back. This is one resource with many faces, negotiated by the client's stated preference rather than by inventing separate `/report.json` and `/report.html` URLs.

---

## Data flow, trade-offs, failure modes, common mistakes (Part 1)

**Request/response lifecycle:**
```
client → method + path + headers (+ body) → server parses → validates → acts
       → status code + headers (+ body) → client reads the STATUS to decide the next step
```

| Discipline | Cost | Payoff |
|---|---|---|
| Correct status codes | thoughtfulness | callers react correctly automatically |
| Idempotent design | up-front effort | retries are safe (survive lost responses) |
| Caching (`ETag`) | version bookkeeping | huge bandwidth/latency savings |

- **Failure modes:** wrong status code → destructive retry or wrongful give-up • retrying a non-idempotent `5xx` → duplicate side effects • no caching → needless load.
- **Common mistakes:** *beginner* — returning `200` with an `{"error": ...}` body (lies to the caller; use the real code); *intermediate* — `400` for validation errors that should be `422`; *senior* — retrying non-idempotent operations on `5xx` without an idempotency key.

## Interview & practice (Part 1)

1. Which methods are idempotent, and why does idempotency govern *safe retries* specifically?
2. Give a precise `POST /users` scenario for `400`, `422`, and `409` each.
3. When is retrying a `5xx` dangerous, and what makes it safe? Walk the double-charge case.
4. Show the `ETag` / `If-None-Match` exchange that yields a `304`, and what it saves.

*Practice — Easy:* what does `429` tell the caller to do? • *Medium:* design the status codes for "create user" covering malformed body, invalid email, and duplicate username. • *Hard:* a client double-creates resources under flaky mobile networks — trace exactly why and outline the fix (preview of [Week 4](phase1-week4.md)).

---

# PART 2 — AGENTIC AI: Function-Calling Internals

> Backend is a black box: "your code parses/executes" = the machinery of [Week 2](phase1-week2.md).

## Overview & motivation

[Week 2](phase1-week2.md) showed the *shape* of function calling (send tool defs → model emits a tool-call → you execute). This week opens the box: **how** the model is constrained to emit valid, parseable tool calls, and why the *format* of your tool definitions and outputs quietly decides both reliability and cost. Two agents with identical logic can differ 10× in reliability purely from how their tools are described and how their outputs are shaped.

## 1. The tool-definition contract **[CORE]**

### First principles & worked example

You send, alongside the prompt, a list of tool definitions. Each has three parts, and each part does a specific job:

```json
{
  "name": "get_weather",                                          // (a) the handle
  "description": "Get current weather for a city. Use when the user asks about weather.", // (b) WHEN to use it
  "parameters": {                                                 // (c) the JSON-Schema contract
    "type": "object",
    "properties": {
      "city":  {"type": "string", "description": "City name, e.g. 'Austin'"},
      "units": {"type": "string", "enum": ["celsius", "fahrenheit"]}
    },
    "required": ["city"]
  }
}
```

- **(a) `name`** — how the model refers to the tool.
- **(b) `description`** — natural-language *when-to-use* guidance. This is the model's *only* clue about **whether** to call the tool. **A vague description is a bug**, not cosmetics — it causes missed calls ("I didn't know I could look up weather") and spurious calls (using it when it shouldn't).
- **(c) `parameters` (JSON Schema)** — the exact shape of allowed arguments: `type`s, `enum` (a fixed set of allowed values), `required` fields, and per-field `description`s. This is a machine-readable *contract*, the same idea as a typed API endpoint (Part 3).

### Worked example — how the description changes behavior

```
Vague:  "description": "weather tool"
  User: "Should I bring a jacket to my meeting?"  → model may NOT call it (doesn't connect jacket↔weather)

Rich:   "description": "Get current weather for a city. Use for ANY question about
         temperature, rain, or what to wear."
  User: "Should I bring a jacket to my meeting?"  → model calls get_weather, then advises
```

Same tool, same code — the description alone changed whether the agent worked. **Tool descriptions are prompt engineering.**

### Runnable example — sending a tool and watching the model emit a tool-call (Anthropic SDK)

The examples below use Claude (the model this course is built around). Install and set your key first:

```bash
pip install anthropic
export ANTHROPIC_API_KEY=sk-ant-...
```

```python
import anthropic

client = anthropic.Anthropic()   # reads ANTHROPIC_API_KEY from the environment

TOOLS = [{
    "name": "get_weather",
    "description": (
        "Get current weather for a city. Use for ANY question about "
        "temperature, rain, or what to wear."          # the WHEN-to-use clue
    ),
    "input_schema": {                                   # JSON-Schema contract
        "type": "object",
        "properties": {
            "city":  {"type": "string", "description": "City name, e.g. 'Austin'"},
            "units": {"type": "string", "enum": ["celsius", "fahrenheit"]},
        },
        "required": ["city"],
    },
}]

resp = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    tools=TOOLS,
    # phrased INDIRECTLY — never mentions "weather"
    messages=[{"role": "user", "content": "Should I bring a jacket to my meeting in Austin?"}],
)

print(resp.stop_reason)                       # -> "tool_use"
for block in resp.content:
    if block.type == "tool_use":
        print(block.name, block.input)        # -> get_weather {'city': 'Austin'}
```

**What just happened.** We never told the model "call `get_weather`." The rich `description` ("...for ANY question about ... what to wear") is the *only* thing that let it connect "jacket" → weather and decide to call the tool. The response comes back with `stop_reason == "tool_use"` — the model's way of saying *"I'm not answering yet; run this tool and give me the result."* The `tool_use` content block carries the `name` it chose and the `input` it filled in against your schema (`{'city': 'Austin'}` — note it left out the optional `units`). Blank out the description to `"weather tool"` and re-run: the model often answers from guesswork instead of calling the tool. Same code, different behavior — the description *is* the program.

## 2. Constrained / structured decoding — how valid JSON is guaranteed **[CORE]**

### The problem (first principles)

A model generates text **one token at a time**, sampling each next token from a probability distribution over its vocabulary. Left completely free, it might produce *almost*-valid JSON — a trailing comma, a missing quote, a stray word — that your code cannot parse. That is fatal when your code must *machine-read* the tool call.

### The mechanism — token masking (worked example)

**Constrained decoding** (a.k.a. grammar-constrained / structured decoding; "JSON mode" is one flavor) enforces a grammar *during* generation. At each step, **before sampling, it sets the probability of any token that would break the required grammar to zero** — so only tokens that keep the output valid JSON (matching your schema) are even eligible.

```
Generating arguments for get_weather, so far produced:  {"city": "Austin"
  Next token — the JSON grammar allows only:  }  or  ,
  → the decoder MASKS every other token (letters, spaces, quotes) to probability 0
  → whatever it samples is GUARANTEED to keep the JSON valid
Continue:  {"city": "Austin"}   ← parseable, by construction
```

### The nuance that trips people up **[CORE]**

Constrained decoding guarantees **structural** validity (valid JSON, right types) — it does **NOT** guarantee **semantic** correctness. The model can still place a well-typed but nonsensical value in a field:

```
Valid JSON, wrong value:  {"city": "Austinnn", "units": "celsius"}   ← "Austinnn" isn't a city
```

So you **still validate values** with Pydantic ([Phase 0](phase0-week1.md); Part 3). Constrained decoding gets you a parseable object; validation gets you a *sensible* one.

### Runnable example — the layer constrained decoding *can't* give you (Pydantic)

Constrained decoding guarantees the arguments arrive as valid JSON of the right types. It cannot know that `"Austinnn"` is not a real city. That's your job:

```python
from pydantic import BaseModel, field_validator

KNOWN_CITIES = {"Austin", "Seattle", "Denver"}


class WeatherArgs(BaseModel):
    city: str                       # constrained decoding guarantees this is a STRING
    units: str = "celsius"

    @field_validator("city")
    @classmethod
    def city_must_exist(cls, v: str) -> str:
        if v not in KNOWN_CITIES:   # ...but only YOU can check it's a REAL city
            raise ValueError(f"unknown city: {v!r}")
        return v


# What the model emitted — structurally perfect JSON, semantically wrong:
raw_args = {"city": "Austinnn", "units": "celsius"}

try:
    args = WeatherArgs(**raw_args)
except ValueError as e:
    print("rejected:", e)           # -> rejected: unknown city: 'Austinnn'
```

**Why both layers are needed.** By the time `raw_args` reaches your code, constrained decoding has already guaranteed it parses, that `city` is a string, and that `units` (if present) is one of the `enum` values — you will never get a trailing comma or a number where a string belongs. What it *can't* guarantee is meaning: `"Austinnn"` is a perfectly valid string and a nonsense city. The Pydantic `field_validator` is the second gate — it turns a well-typed-but-wrong argument into a clean `ValueError` you can catch. In Part 3 you'll return that failure to the model as a `422`-style result so it *fixes the argument and retries*, instead of executing a lookup that would blow up on garbage input.

## 3. Token economics of tools **[WORKING]**

### Why tool data is expensive — worked comparison

**Tokenization** splits text into chunks (tokens) the model processes; JSON and structured data fragment into **many more tokens per character** than prose, because braces, quotes, field names, and punctuation each cost tokens.

```
Prose:  "Austin is 34°C and clear."                    ≈ 9 tokens
Raw JSON: {"location":{"city":"Austin","region":"TX"}, ≈ 40+ tokens for the SAME facts
           "current":{"temp_c":34,"condition":"clear","humidity":41,...}}
```

### Why it compounds — the loop multiplier

Every tool result you append is **permanent context**, re-sent on *every* subsequent turn ([Phase 0](phase0-week1.md), Model 1: statelessness ⇒ resend everything). So a 2,000-token raw API response dumped into history is re-paid for on **every remaining step** of the loop.

```
5-step loop, raw 2,000-token tool output appended at step 1:
  it is re-sent at steps 2,3,4,5  → ~2,000 × 4 = 8,000 extra tokens paid
Trim it to 100 useful tokens:
  re-sent 4 times → 400 tokens.  A 20× saving, just from shaping the output.
```

**Lean schemas and compact, model-readable tool outputs are a performance feature**, not polish. Return the 3 fields the model needs, not the raw 40-field API blob.

### Runnable example — measuring the loop multiplier with real token counts

Don't estimate with `tiktoken` (that's OpenAI's tokenizer and undercounts Claude). Use the model's own counter:

```python
import anthropic, json

client = anthropic.Anthropic()

raw = {                                   # what a weather API actually returns
    "location": {"city": "Austin", "region": "TX", "country": "US",
                 "lat": 30.27, "lon": -97.74, "tz": "America/Chicago"},
    "current": {"temp_c": 34, "temp_f": 93, "condition": "clear", "humidity": 41,
                "wind_kph": 12, "wind_dir": "SSE", "pressure_mb": 1012,
                "uv": 8, "visibility_km": 16, "feelslike_c": 36},
}
lean = {"city": "Austin", "temp_c": 34, "condition": "clear"}   # what the model needs


def toks(obj: dict) -> int:
    r = client.messages.count_tokens(
        model="claude-opus-4-8",
        messages=[{"role": "user", "content": json.dumps(obj)}],
    )
    return r.input_tokens


raw_t, lean_t = toks(raw), toks(lean)
print(f"raw={raw_t}  lean={lean_t}")
# A tool result appended at step 1 of a 5-step loop is RE-SENT at steps 2,3,4,5:
print(f"raw re-sent cost = {raw_t * 4}   lean re-sent cost = {lean_t * 4}")
# e.g. raw=120  lean=22  ->  raw re-sent cost = 480   lean re-sent cost = 88
```

**Reading the numbers.** `count_tokens` calls the same tokenizer the model uses to bill you, so these counts are exact, not approximate. The raw blob costs several times what the lean version does *per occurrence* — and because every appended tool result is permanent context re-sent on every later turn ([Phase 0](phase0-week1.md): statelessness ⇒ resend everything), that per-occurrence gap multiplies by the number of remaining steps. Trimming the 40-field blob to the 3 fields the model actually reads isn't cosmetic; it's a multiplicative cost and latency win that compounds across the whole agent loop.

## Failure modes & common mistakes (Part 2)

- **Vague `description` → wrong/missing tool usage.** *Fix:* write when-to-use guidance, including edge phrasings (the jacket example).
- **Over-permissive schema → well-typed nonsense args.** Constrained decoding won't save you; validate values.
- **Dumping raw outputs → context bloat + confusion** (and the loop multiplier above). *Fix:* trim to what the model needs.
- **Too many tools → decision paralysis.** Large menus degrade selection accuracy; expose only what's relevant to the task.

## Interview & practice (Part 2)

1. What are the three parts of a tool definition, and what job does each do?
2. What does constrained decoding do at the token level, and what does it guarantee vs. *not* guarantee? Give a valid-JSON-wrong-value example.
3. With numbers, why do tool outputs eat context faster than prose, and why does it compound over a loop?

*Practice — Easy:* which field tells the model *when* to use a tool? • *Medium:* write a JSON-Schema for `search_orders(status, limit)` using `enum` and `required`. • *Hard:* your agent emits valid-JSON tool calls with absurd argument values — explain why constrained decoding didn't prevent it and how you'd fix it.

---

# PART 3 — THE BRIDGE: A Tool Is an API, Its Result Is an HTTP-Style Signal

*References only Part 1 and Part 2 — no new concepts.*

## The core realization

A tool ([Week 2](phase1-week2.md), Part 2) is executed by your code as an API call (Part 1). Two consequences follow immediately:

> **(1)** Designing a tool = designing an API endpoint (schema, semantics, errors). **(2)** A tool's *result* is the model's equivalent of an HTTP *status code* — it's how the tool tells the model what happened and what to do next.

## The dependency map

```
      AGENTIC (Part 2)                        BACKEND (Part 1)
 ┌───────────────────────────┐      ┌────────────────────────────────┐
 │ tool definition (schema)  ┼─────►│ = a typed API endpoint contract │
 │ tool CALL (model → you)   ┼─────►│ = an HTTP request               │
 │ tool RESULT (you → model) ┼─────►│ = an HTTP STATUS + body         │
 │ model decides: retry?     ┼─────►│ the status taxonomy guides it   │
 │ model retries a POST-like ┼─────►│ idempotency needed (→ Week 4)   │
 └───────────────────────────┘      └────────────────────────────────┘
```

## Interlink 1 — a good tool is a good API endpoint **[CORE]**

The JSON-Schema tool definition (Part 2) is the same contract as a typed, documented endpoint (Part 1): clear names, precise types, `required` fields, correct semantics. **Worked parallel:**

```
Good API endpoint                        Good tool definition
POST /orders {items:[...], required}  ≈  create_order(items: array, required)
returns 201 + {id} / 422 on bad data     returns {order_id} / {"error":"invalid","field":...}
```

Everything that makes an API pleasant to consume — predictable shapes, good errors — makes a tool reliable for the model. The skill transfers verbatim.

## Interlink 2 — map tool outcomes onto the HTTP status vocabulary **[CORE]**

This is the heart of the week. The model decides whether to retry, fix its arguments, or give up **based only on what the tool result says** (Part 2 failure modes). So the tool should signal outcomes the way HTTP does (Part 1 taxonomy) — *not* as raw stack traces:

| Tool outcome | Return this (HTTP analogue) | Model's correct reaction |
|---|---|---|
| bad/invalid arguments | `422`-style: `{"error":"invalid_argument","field":"city","detail":"required"}` | fix the argument, retry |
| transient failure / rate-limited | `429`/`503`-style: `{"error":"temporarily_unavailable","retry":true}` | back off, retry |
| target doesn't exist | `404`-style: `{"error":"not_found"}` | stop retrying; try another approach |
| state conflict | `409`-style: `{"error":"conflict","detail":"record changed"}` | re-read, then retry |
| success | compact result data | continue reasoning |

### Worked example — why raw errors cause runaway loops

```
BAD: tool crashes, you return the raw stack trace as the tool result:
  "Traceback (most recent call last): File "app.py", line 42, in run_tool ..."
  → the model can't tell "fix my input" from "retry later" from "give up"
  → it re-calls the SAME tool with the SAME args → same crash → repeat
  → runaway loop until MAX_ITERS trips (Week 2), burning tokens the whole way

GOOD: you catch it and return a status-style result:
  {"error":"invalid_argument","field":"date","detail":"expected YYYY-MM-DD"}
  → the model reads it, FIXES the date format, retries once, succeeds.
```

A raw stack trace is the agentic equivalent of returning a naked `500` with a wall of text — the caller has nothing structured to act on. Status-style results turn tool failures into *recoverable* signals.

### Runnable example — a ReAct loop whose tool speaks in HTTP-style signals (Anthropic SDK)

This is the Week-2 loop with the Week-3 upgrade: the tool executor maps every outcome onto a `422`/`404`/success-style result, and marks failures with `is_error` so the model can classify and recover.

```python
import anthropic, json

client = anthropic.Anthropic()
ORDERS = {"abc": {"id": "abc", "status": "shipped"}}   # our "database"

TOOLS = [{
    "name": "get_order",
    "description": "Look up an order by id. Use when the user asks about an order's status.",
    "input_schema": {
        "type": "object",
        "properties": {"id": {"type": "string", "description": "order id"}},
        "required": ["id"],
    },
}]


def run_tool(name: str, args: dict) -> dict:
    """Execute the tool and map the outcome onto an HTTP-style signal."""
    if name != "get_order":
        return {"error": "unknown_tool"}                        # 404-style
    order_id = args.get("id")
    if not order_id:
        return {"error": "invalid_argument", "field": "id",     # 422-style
                "detail": "id is required"}                     #  -> model fixes its args
    order = ORDERS.get(order_id)
    if order is None:
        return {"error": "not_found", "id": order_id}           # 404-style
    return {"order": order}                                     # success


messages = [{"role": "user", "content": "What's the status of order abc?"}]

for _ in range(6):                                              # MAX_ITERS guard (Week 2)
    resp = client.messages.create(
        model="claude-opus-4-8", max_tokens=1024, tools=TOOLS, messages=messages,
    )
    if resp.stop_reason != "tool_use":                         # model is done — print answer
        print(next(b.text for b in resp.content if b.type == "text"))
        break

    messages.append({"role": "assistant", "content": resp.content})   # keep the tool_use blocks
    results = []
    for block in resp.content:
        if block.type == "tool_use":
            out = run_tool(block.name, block.input)
            results.append({
                "type": "tool_result",
                "tool_use_id": block.id,                        # MUST match the tool_use id
                "content": json.dumps(out),
                "is_error": "error" in out,                    # flag failures as failures
            })
    messages.append({"role": "user", "content": results})      # feed results back, loop again
```

Now the version that causes the runaway loop — the *only* change is the tool result:

```python
import traceback

def run_tool_bad(name: str, args: dict) -> str:
    try:
        return json.dumps({"order": ORDERS[args["id"]]})       # KeyError on a missing id
    except Exception:
        # Returning the raw traceback gives the model nothing to CLASSIFY.
        return traceback.format_exc()   # "Traceback (most recent call last): ... KeyError: 'xyz'"
```

**Trace both paths.** With `run_tool`, ask about a missing order (`"order xyz"`): the tool returns `{"error": "not_found", "id": "xyz"}` with `is_error=True`. The model reads that structured signal, recognizes retrying won't help, and tells the user the order doesn't exist — the loop ends cleanly. Swap in `run_tool_bad` and the tool returns a wall of traceback text; the model can't tell *"fix my input"* from *"retry later"* from *"give up,"* so it re-calls `get_order` with the same bad id, gets the same traceback, and repeats until the `range(6)` guard trips — six wasted API calls and a dead end. The mechanical pieces that make the good path work: append the assistant's `content` (with its `tool_use` blocks) **before** the results; return exactly one `tool_result` per `tool_use`, keyed by the matching `tool_use_id`; and set `is_error` so a failure reads as a failure. That is the whole thesis of the week in code — **a tool is an API call, and its result is the status code the model reasons over.**

## Interlink 3 — retry safety previews Week 4 **[CORE]**

Because a tool result can say "retry" (`429`/`5xx`-style) and the model *will* retry, any **side-effecting** tool ("create order," "send email") hits the exact `POST`-retry duplication problem from Part 1 (the double-charge). **Worked preview:**

```
model calls send_email(...)  → tool succeeds but result is slow/lost
model (unsure) calls send_email(...) again  → WITHOUT idempotency: two emails sent
```

The fix — idempotency keys — is [Week 4](phase1-week4.md). Note here only the rule: **the moment a tool can be retried, it must be made idempotent.**

## Under the hood — one turn, HTTP-style **[CORE]**

```
1. [MODEL] emits tool-call {name:"get_order", args:{id:"abc"}}  (constrained decoding, Part 2)
2. [YOU]   validate args (Pydantic) — invalid? → return 422-style → model fixes  (Part 1 §422 / Part 2)
3. [YOU]   execute = HTTP GET /orders/abc                       (Part 1)
4. [YOU]   map the outcome:
             200 → compact JSON result
             404 → {"error":"not_found"}            → model stops retrying
             503 → {"error":"unavailable","retry":true} → model backs off and retries
5. [MODEL] reads the status-style result and reacts correctly  (Part 1 taxonomy)
```

## Coupled failure modes

- **Raw stack trace as tool result → runaway loop** (Part 1 taxonomy ignored × [Week 2](phase1-week2.md) guard) — the model can't classify the failure, so it retries blindly until the guard trips.
- **Non-idempotent tool + "retry"-style result → duplicate side effects** (Part 1 idempotency × Part 2 retries) — the double-charge, fixed in [Week 4](phase1-week4.md).
- **Bloated raw output → context overflow** (Part 2 token economics × [Phase 0](phase0-week1.md) statelessness) — re-sent every turn until the window fills.

## Interview & practice (Part 3)

1. Why is "a tool is an API call" the unifying insight, and what two consequences follow?
2. Map each HTTP status class to how a tool should signal the model, with an example each.
3. Walk the worked example of why a raw stack trace causes a runaway loop, and how a status-style result fixes it.
4. Why must any retryable tool be idempotent?

*Practice — Medium:* rewrite a tool that currently returns raw exceptions so it returns `422`/`429`/`404`-style structured errors. • *Hard:* an agent loops forever on a failing tool — explain the failure using the status taxonomy and fix it.

---

# Cheat Sheet

**Part 1:** idempotent = `GET/PUT/DELETE`, not `POST` (governs safe retries) · status codes are a precise vocabulary: `400`(unparseable) ≠ `422`(invalid values) ≠ `409`(state conflict); `4xx`=you, `5xx`=server (retry w/ backoff if safe); `429`=back off · `ETag`+`If-None-Match`→`304` (no body) · content negotiation via `Accept`.

**Part 2:** tool def = name + **description (when-to-use = prompt engineering)** + JSON-Schema · constrained decoding masks invalid tokens → **structurally** valid JSON (not semantically correct — still validate) · tool outputs tokenize worse than prose and are re-sent every turn → keep lean (the loop multiplier).

**Part 3:** a tool = an API endpoint; a tool result = an HTTP status; map outcomes to `422`/`429`/`404`/`409`-style signals so the model fixes/retries/gives-up correctly; raw stack trace = naked `500` = runaway loop; retryable tool ⇒ must be idempotent.

**Mnemonics:** "`400` can't parse, `422` won't accept, `409` conflicts" · "a tool result is a status code" · "valid JSON ≠ sensible values" · "retry ⇒ idempotency."

---

# Build This

**Definition of done:**
1. **(Part 1)** Build a FastAPI `POST /users` returning the *correct* codes: `400` for broken JSON, `422` for an invalid email, `409` for a duplicate username, `201` on success. Add an `ETag` to a `GET` and prove a conditional request returns `304` with no body.
2. **(Part 2)** Define a tool with a rich JSON-Schema (`enum`, `required`, good descriptions). Ask a question phrased indirectly (the "jacket" example) and confirm the model still calls it. Then blank out the description and watch it fail — measure how much the description matters.
3. **(Part 2)** Log token counts for a raw 2KB JSON tool output vs. a trimmed compact one over a 5-step loop; quantify the multiplier.
4. **(Bridge)** Take the [Week 2](phase1-week2.md) ReAct agent and make its tools return **status-style structured results**: `422`-style for bad args (validated with Pydantic), `429`-style retryable for transient failures, `404`-style for missing targets. Verify the model *fixes* a `422`, *retries* a `429`, and *stops* on a `404`. Then feed it a raw stack trace instead and watch it loop — the aha of the week.

---

# Active Recall & Self-Test

**From memory:** (1) Which methods are idempotent and why does it matter for retries? (2) Distinguish `400`/`422`/`409` with a `POST /users` scenario each. (3) When is retrying `5xx` unsafe — walk the double-charge. (4) What does constrained decoding guarantee and *not* guarantee? Give a valid-JSON-wrong-value example. (5) With numbers, why do tool outputs eat context faster than prose over a loop? (6) Map tool outcomes to HTTP-style signals and explain why raw stack traces break the model.

**Teach-back (60s, aloud):** explain "a tool is an API and its result is a status code," and why that makes the HTTP taxonomy an agent-reliability tool. Stumble → re-read Part 3.

---

# Primary Sources

- **HTTP semantics** — RFC 9110 (methods, status codes, idempotency, safe methods). **Caching** — RFC 9111 (`ETag`, `Cache-Control`, conditional requests). **Content negotiation** — RFC 9110 §12.
- **Status-code reference** — MDN Web Docs, "HTTP response status codes" (developer.mozilla.org).
- **JSON Schema** — `json-schema.org` (Draft 2020-12).
- **Constrained/structured decoding & JSON mode** — your model provider's tool-use / structured-output docs (Anthropic/OpenAI); **verify behavior against current docs — it drifts.** Background: grammar-constrained decoding literature.

---

# Key Takeaways & Summary

- **Part 1:** HTTP is a contract; idempotency governs safe retries; status codes are a precise instruction to the caller (`400`≠`422`≠`409`); caching and negotiation are interface-level tools.
- **Part 2:** Tool definitions are a JSON-Schema contract whose *description* is prompt engineering; constrained decoding guarantees structurally valid JSON (not sensible values); tool outputs are costly context re-sent every turn.
- **Part 3:** A tool is an API and its result is a status code — map outcomes to HTTP-style signals so the model reacts correctly; raw errors cause runaway loops; retryable tools must be idempotent.

**10-second:** Learn HTTP as a contract (idempotency + precise status codes), learn how tool calls are constrained to valid JSON, and see that a tool is an API whose result is a status code the model reasons over.

**1-minute:** HTTP methods carry idempotency guarantees that make retries safe or unsafe (a lost response makes a client retry, so a non-idempotent `POST` double-charges), and status codes are a precise vocabulary — `400` can't-parse, `422` invalid-values, `409` state-conflict; `4xx` is your fault, `5xx` is the server's (retryable with backoff if safe). On the agent side, tool definitions are JSON-Schema contracts whose description decides *whether* the model calls the tool, constrained decoding masks invalid tokens so arguments are structurally valid JSON (though not necessarily sensible — still validate), and tool outputs tokenize worse than prose and are re-sent every turn, so keep them lean. The bridge: a tool is an API call and its result is the model's status code — return `422`/`429`/`404`/`409`-style results so the model fixes, retries, or gives up correctly; a raw stack trace is a naked `500` that makes it loop.

**5-minute:** read Part 1 §1 (idempotency + double-charge), §2 (the `400`/`422`/`409` worked examples), Part 2 §2 (token-masking) + §3 (token multiplier), then Part 3's outcome-mapping table and the raw-error-loop worked example.

**Expert summary:** the agent's tool layer is an HTTP client/server relationship in disguise. Idempotency decides retry safety; the status taxonomy is the language a tool uses to steer the model's next move. Design tools like good endpoints and return status-style results, and the model's error recovery becomes principled instead of accidental.

---

# Next: [Week 4 — API design details & reasoning strategies](phase1-week4.md) · Prev: [Week 2](phase1-week2.md)
