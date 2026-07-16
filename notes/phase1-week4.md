# Phase 1 — Week 4: API Design Details & Reasoning Strategies

> **What this covers:** senior-level API design — idempotency keys, cursor pagination, standard error formats, versioning (backend) — and the family of LLM reasoning strategies: CoT, ReAct, Reflexion, Plan-and-Execute, Tree-of-Thought, self-consistency, temperature (agentic). Then: **the idempotency-key discipline is exactly what makes an agent's constant retries safe.**
> **Prerequisite:** [Week 3](phase1-week3.md) — HTTP semantics (idempotency, status codes) and function-calling internals.
> **The one idea that unites this week:** *agents retry constantly (failed tools, re-planning, N-sample voting), and idempotency keys stop a retried side-effecting action from happening twice — the same pattern that stops a double charge in any API.*
>
> **How to read this:** every concept runs the ladder **intuition → analogy → concrete worked example → diagram → under-the-hood**. When something feels abstract, jump to its "**Example**" block.
>
> **Depth tiers:** **[CORE]** open every box · **[WORKING]** use it, know tradeoffs · **[AWARE]** know it exists.

---

# PART 1 — BACKEND: Senior-Level API Design

## Overview & motivation — the gap between "works" and "production"

**The intuition.** A beginner's API works in the *happy path*: valid input, stable network, one request at a time. Production is defined entirely by the *unhappy* path — the client that retries after a timeout, the data that changes while you're paging through it, the consumer that must parse every error the same way, and the future where you must change the API without breaking callers who wrote code against the old one. The disciplines in this Part are what separate a demo from a system people can build on.

**Why it matters for this course.** An agent is the *most* unhappy-path client imaginable: it is autonomous, impatient, and retries far more than a human ever would ([Week 2](phase1-week2.md)'s loop, [Week 3](phase1-week3.md)'s retry signals). So every discipline here maps straight onto an agentic failure in Part 3.

## First principles & terminology **[CORE]**

- **API (Application Programming Interface)** — the contract by which one program calls another (here, over HTTP).
- **Side effect** — a change to state in the world (a charge, an email, a row inserted). Operations *with* side effects are the dangerous ones to retry.
- **Cursor** — an opaque pointer to a position in a result set.
- **Breaking change** — an API change that makes existing client code stop working.

---

## 1. Idempotency keys — making unsafe operations safely retryable **[CORE]**

### The problem, recapped and deepened from [Week 3](phase1-week3.md)

`POST` is not idempotent, but networks *force* retries: a lost response ([Week 3](phase1-week3.md)'s double-charge) means the client must retry to survive, yet retrying a raw `POST` duplicates the side effect. You cannot remove retries (they're how you survive lost responses), so you must make the *operation* safe to repeat.

### The solution — first principles

The client generates a **unique key per logical operation** (a UUID) and sends it in a header:
```
POST /charge   Idempotency-Key: 7f3a-9c...-91   {amount: 10}
```
The server implements this contract:
```
1. look up the key in an idempotency store (Redis / a DB table)
2. key EXISTS and completed   → return the STORED response (do NOT re-execute)
3. key EXISTS but in-flight   → return 409 (a retry arrived while the first is still running)
4. key is NEW                 → mark in-flight → execute → store {key → response} → return it
```

### Analogy (and where it breaks)

A **coat check**: you hand over your coat and get a numbered ticket. Show the ticket once or ten times, you get back the *same one coat* — never a second coat, never someone else's. The ticket (key) makes repeated claims return the same result. *Where it breaks:* a coat check clerk remembers naturally; your server only "remembers" because you *built* the store — and if you store the ticket *after* handing back the coat and crash in between, the memory is lost (the atomicity point below).

### Worked example — the fix in action

```
1st attempt:  POST /charge  Idempotency-Key: 7f3a   → key NEW
              charge $10, store 7f3a → {charge_id: 9, status: 200}, return 200
              ✗ response lost on the network
retry:        POST /charge  Idempotency-Key: 7f3a   → key EXISTS + completed
              return the STORED {charge_id: 9, status: 200}  — NO second charge
RESULT: customer charged exactly once, despite the retry.
```

### Under the hood — the atomicity requirement **[CORE]**

The store write must be **atomic with (or transactionally bound to)** the side effect. Otherwise a crash *between* "charge the card" and "store the key" reintroduces the exact duplicate you were preventing:

```
DANGEROUS ordering:            SAFE ordering:
 1. charge card                 begin transaction
 2. ✗ crash                       reserve key (in-flight)
 3. store key  ← never runs        charge card
 → retry sees no key →             store {key → result}
   charges AGAIN                  commit   (all-or-nothing)
```

This is *why* idempotency keys pair with a durable store and, later, the **outbox pattern** (Phase 3). Give keys a **TTL** (time to live) long enough to cover realistic retry windows (hours) so the store doesn't grow forever. And the key is **client-generated** — only the client knows that two physical requests are "the same logical operation."

---

## 2. Cursor-based pagination **[WORKING]**

### The problem with offset — worked example

`GET /items?offset=40&limit=20` tells the DB "skip 40, return 20." If rows are inserted/deleted *between* page fetches, the offset shifts:

```
Page 1: offset=0, limit=3   → [A, B, C]          (you've now seen A,B,C)
  ...meanwhile someone inserts row Z at the top...
Page 2: offset=3, limit=3   → [C, D, E]          ← C appears AGAIN (duplicate!)

Or with a DELETE at the top instead:
Page 2: offset=3, limit=3   → [E, F, G]          ← D was SKIPPED (missed!)
```
Under any concurrent write load, offset pagination is subtly wrong. It's also slow at depth (the DB must scan past all skipped rows to count them).

### The fix — a stable pointer

A **cursor** encodes a *position* (usually the last item's sort-key + id): `GET /items?after=<cursor>&limit=3` = "give me the 3 items *after* this stable point." Because it anchors to a value, not a count, inserts/deletes elsewhere don't shift the window:

```
Page 1: after=∅          → [A, B, C]   next=cursor(C)
  ...someone inserts Z at the top...
Page 2: after=cursor(C)  → [D, E, F]   ← no skip, no dupe: it resumes AFTER C
```

Prefer cursors for anything that mutates; offset is acceptable only for small static datasets or when the user genuinely needs "jump to page N."

---

## 3. Consistent error format **[WORKING]**

Every consumer needs to parse errors the same way. Adopt **RFC 9457 "Problem Details"** (obsoletes RFC 7807) — a standard JSON error envelope:
```json
{ "type": "https://api.example.com/errors/insufficient-funds",
  "title": "Insufficient funds",
  "status": 402,
  "detail": "Your balance is $3, this charge is $10.",
  "instance": "/charges/abc123" }
```
- **`type`** — a stable URI identifying the error *class*; clients branch on this, not on the human prose (which may change).
- **`title`** — a human summary; **`status`** — matches the HTTP code; **`detail`** — this specific occurrence; **`instance`** — which resource.

**Why it matters (worked):** without a standard shape, one endpoint returns `{"error":"x"}`, another `{"message":"y"}`, another `{"errors":[...]}` — and every consumer writes bespoke parsing that breaks on the next endpoint. One envelope everywhere = one parser everywhere.

---

## 4. Versioning **[WORKING]**

APIs must evolve without breaking existing clients. Decide the strategy **up front**:
- **URL versioning** — `/v1/orders`, `/v2/orders`. Explicit, cache-friendly, most common.
- **Header versioning** — `Accept: application/vnd.api+json; version=2`. Cleaner URLs, less visible.

**The rule, with examples:**
```
SAFE (additive):     add a new OPTIONAL field to the response       → old clients ignore it
BREAKING:            rename `user_name` → `username`                → old clients read null/crash
BREAKING:            change `amount` from string "10" to number 10  → old clients mis-parse
BREAKING:            remove a field                                  → old clients get nothing
```
Breaking changes require a new version so old clients keep hitting `/v1`. Cheap to plan now, brutally expensive to retrofit onto live consumers.

---

## Trade-offs, failure modes, common mistakes (Part 1)

| Discipline | Cost | Payoff |
|---|---|---|
| Idempotency keys | a store + key logic + atomicity care | safe retries; no duplicate side effects |
| Cursor pagination | opaque cursors, no "jump to page N" | correctness under concurrent writes |
| RFC 9457 errors | a small envelope everywhere | uniform machine-parseable errors |
| Versioning up front | discipline about breaking changes | evolve without breaking clients |

- **Failure modes:** no idempotency → duplicate charges on retry • offset paging → skipped/duplicated rows • ad-hoc error shapes → brittle consumers • no versioning plan → one breaking change takes down every client at once.
- **Common mistakes:** *beginner* — offset pagination everywhere; *intermediate* — server-generated idempotency keys (only the client knows two calls are "the same"); *senior* — storing the key *after* the side effect instead of atomically with it.

## Interview & practice (Part 1)

1. Walk the idempotency-key server contract (all four cases). Why must the key be client-generated and stored atomically with the side effect?
2. Show, with a worked insert, exactly how offset pagination skips/duplicates rows, and how a cursor fixes it.
3. Give one additive and two breaking changes, and explain how versioning contains the breaking ones.

*Practice — Easy:* which RFC standardizes error bodies? • *Medium:* design an idempotency layer for `POST /transfers` including the in-flight and TTL cases. • *Hard:* a payments API double-charges ~0.1% of the time under mobile networks — diagnose and fix end to end, including the atomicity ordering.

---

# PART 2 — AGENTIC AI: Reasoning Strategies

> Backend is a black box: "each call" = the network round-trip and loop of [Weeks 2–3](phase1-week2.md).

## Overview & motivation — why "how it thinks" is a tunable knob

**The intuition.** A raw model call gives the model *one shot* to produce an answer — no chance to work through steps, check itself, plan ahead, or reconsider. **Reasoning strategies** are the ways you *structure* the model's thinking to get better answers: make it reason step by step, let it critique its own draft, plan before acting, or answer several times and vote. Each buys quality at a cost in tokens and latency. The senior skill is matching the strategy to the task, because every strategy multiplies cost.

## The mechanical foundation: tokens *are* the computation **[CORE]**

Before the strategies, the one fact that explains most of them. A transformer does a **fixed amount of computation per token generated**, and it has **no hidden scratchpad** — the *only* place it can "work through" a problem is in the tokens it actually emits. So **making the model produce intermediate tokens literally gives it more computation to reach the answer.**

### Worked example — why "show your work" changes the answer

```
Q: "A shirt costs $80, is discounted 25%, then taxed 10%. Final price?"

No CoT (answer immediately):  "$70"   ← wrong; it guessed in one forward pass
With CoT ("think step by step"):
   "80 × 0.75 = 60 (after discount)
    60 × 1.10 = 66 (after tax)
    Final price: $66"                 ← correct; each step was a chunk of computation
```

The intermediate tokens (`80 × 0.75 = 60 …`) *are* the arithmetic being carried out. This is why "think step by step" isn't a trick — it's buying the model more forward passes. Hold onto this; it explains CoT, ReAct, and why "reasoning" models emit long thought traces.

## The strategies **[CORE for CoT/ReAct; WORKING otherwise]**

### Chain-of-Thought (CoT) **[CORE]**
Prompt the model to emit intermediate reasoning before the final answer ("Let's think step by step"). *Why it works:* the intermediate tokens are the computation (above). *When:* any multi-step reasoning (math, logic, planning). *Cost:* modest extra tokens. *Failure mode:* it can produce a fluent, confident, **wrong** rationale — a reasoning trace is not a proof; verify outcomes, not eloquence.

### ReAct **[CORE]**
The reason→act→observe loop you built in [Week 2](phase1-week2.md): CoT interleaved with real tool calls, so reasoning is grounded in observations from the world. *When:* tasks needing external info or actions. *Cost:* a network round-trip per tool step.
```
Thought: I need the weather to answer.   Act: get_weather("Austin")
Observation: 34°C, clear.                 Thought: warm — no jacket needed. Answer: ...
```

### Reflexion / self-critique **[WORKING]**
The model produces a draft, then critiques and revises it (optionally remembering what failed across attempts).
```
Draft → "Critique your draft for errors." → "Now revise given the critique." → improved answer
```
*When:* quality-critical outputs where a second pass pays off. *Cost:* ~2×+ calls.

### Plan-and-Execute **[WORKING]**
Plan the *entire* task as a step list **once** up front, then execute the steps (often with a cheaper model), replanning only if a step surprises you — versus ReAct's decide-one-step-at-a-time.
```
Plan (one call):  1. search flights  2. pick cheapest  3. book  4. email itinerary
Execute:          run steps 1→4, replanning only if a step fails
```
*Trade-off:* fewer expensive planning calls and more predictable cost, but less adaptive mid-task. *When:* well-structured, decomposable tasks.

### Tree-of-Thought (ToT) **[AWARE]**
Explore multiple reasoning branches, evaluate them, and search (with backtracking) for the best path — reasoning as tree search. Powerful for puzzle-like problems, very expensive (many calls per node). *Treat as a black box unless a project forces you deeper.*

### Self-consistency **[WORKING]**
Sample the answer *N* times at higher temperature and take the **majority vote**.
```
Ask the same hard question 5×:  [66, 66, 70, 66, 66]  → majority "66" → answer 66
```
*Why it works:* independent reasoning paths that converge on the same answer are more likely correct. *When:* problems with a single verifiable answer. *Cost:* N× calls.

### Temperature & sampling **[WORKING]**
**Temperature** scales the randomness of token sampling.
```
temp ≈ 0    → near-deterministic, picks the most likely token
              → best for PLANNING, TOOL-ARG generation, EXTRACTION (want the single right output)
temp ≈ 0.7–1 → more diverse
              → best for BRAINSTORMING, and the N samples in self-consistency
```
Matching temperature to the sub-task is part of strategy design: high temperature on tool-arg generation → flaky arguments; low temperature on brainstorming → repetitive output.

## Comparison **[CORE]**

| Strategy | Extra cost | Adaptivity | Best for |
|---|---|---|---|
| CoT | small | n/a (single pass) | any non-trivial reasoning |
| ReAct | round-trip per step | high (reacts each step) | tasks needing tools/external info |
| Plan-and-Execute | one big plan | low–medium (replan on failure) | structured, decomposable tasks |
| Reflexion | ~2×+ calls | medium (revises) | quality-critical outputs |
| Self-consistency | N× calls | none | single-right-answer problems |
| ToT | many× calls | high (search) | puzzle/search problems |

## Failure modes & common mistakes (Part 2)

- **Over-reasoning cheap tasks** — CoT/ToT on a one-step classification burns tokens for nothing. *Fix:* match strategy to task.
- **Trusting the reasoning trace** — CoT can produce fluent wrong justifications (the failure mode above). *Fix:* verify outcomes.
- **Wrong temperature** — high temp on tool args → flaky calls; low temp on creative work → repetition.
- **Rigid Plan-and-Execute** — a plan that can't replan gets stuck on a surprising step. *Fix:* always allow a replan path.

## Interview & practice (Part 2)

1. Mechanically, *why* does Chain-of-Thought improve answers? Walk the shirt-price example.
2. When would you choose Plan-and-Execute over ReAct, and what do you sacrifice?
3. Why does self-consistency need higher temperature, and when is it worth N× cost?
4. Why low temperature for tool arguments and high for brainstorming?

*Practice — Easy:* which strategy is "sample N, majority vote"? • *Medium:* pick a strategy *and temperature* for: (a) extracting a date from text, (b) a hard math word problem, (c) drafting marketing copy. • *Hard:* an agent is accurate but 5× too expensive — which strategies would you swap out and why?

---

# PART 3 — THE BRIDGE: Idempotency Keys Make an Agent's Retries Safe

*References only Part 1 and Part 2 — no new concepts.*

## The core realization

Every reasoning strategy in Part 2 shares one behavior: **it retries.** ReAct retries failed tool calls; Reflexion re-attempts; Plan-and-Execute replans and re-runs steps; self-consistency runs the whole thing N times. And from [Week 3](phase1-week3.md), a tool result can *explicitly* tell the model to retry (`429`/`5xx`-style). Therefore:

> **Agents retry constantly — far more than a human-driven client — so any side-effecting tool an agent can call MUST be idempotent, or the agent will duplicate real-world actions.**

The idempotency-key discipline (Part 1) is the exact fix.

## The dependency map

```
      AGENTIC (Part 2)                         BACKEND (Part 1)
 ┌────────────────────────────┐      ┌──────────────────────────────────┐
 │ ReAct retries failed tools ┼─────►│ idempotency key (dedupe retries)  │
 │ Reflexion re-attempts      ┼─────►│ idempotency key                   │
 │ Plan-and-Execute re-runs   ┼─────►│ idempotency key                   │
 │ self-consistency: N runs   ┼─────►│ N× API load (your own rate limits)│
 │ strategy = calls × cost    ┼─────►│ latency / throughput budget       │
 └────────────────────────────┘      └──────────────────────────────────┘
```

## Interlink 1 — the double-charge, agent edition **[CORE]**

Picture a travel agent with a `book_flight` tool. **Worked scenario:**

```
1. ReAct loop: model emits book_flight(BA123, 2026-08-01)
2. the tool succeeds, but the RESULT is slow to come back (or the loop times out)
3. the model, unsure it worked, emits book_flight(BA123, 2026-08-01) AGAIN  (Week 3 retry)
4. WITHOUT idempotency: two flights booked, customer charged twice.
```

This is *precisely* the double-charge from Part 1 ([Week 3](phase1-week3.md)), now triggered by an autonomous, retry-happy caller instead of a flaky mobile network. **The fix (Part 1):**
```
derive an idempotency key from the LOGICAL action:  key = hash(user, "BA123", "2026-08-01")
1st call: key NEW → book, store key → booking #55
2nd call: same args → same key → store HIT → return booking #55  (ONE flight)
```
Agents make this failure *more* likely than humans do, which is why the discipline is non-negotiable for agentic tools.

## Interlink 2 — reasoning strategy is a backend load decision **[CORE]**

Every strategy in Part 2 multiplies **model and tool calls**, and each call is a network round-trip on the event loop ([Week 2](phase1-week2.md)) hitting real APIs with rate limits (`429`, [Week 3](phase1-week3.md)).

### Worked example — self-consistency vs. rate limits
```
Self-consistency N=5 on a task that makes 4 tool calls each:
  5 runs × 4 tool calls = 20 tool calls for ONE user's ONE task
  100 concurrent users → 2,000 tool calls in a burst → you trip your OWN 429 rate limit
```
Choosing self-consistency is therefore *also* choosing to 5× your API load. **Plan-and-Execute**, conversely, reduces expensive planning calls → cheaper, more predictable backend load. Picking a strategy is picking a **latency/throughput/cost budget** measured in the same backend terms as any API workload.

## Interlink 3 — idempotency *unlocks* aggressive retry strategies

Because Part 1's idempotency layer makes a side-effecting tool safe to repeat, you can adopt retry-heavy strategies (Reflexion, retry-on-`5xx`) **without** fearing duplicate effects. The backend discipline doesn't merely *defend* — it **enables** the agentic capability: safe retries are the precondition for robust reasoning loops. Without idempotency, you'd have to forbid retries on any tool that changes the world, crippling the agent.

## Under the hood — a retried side-effecting tool call **[CORE]**

```
1. [MODEL] (ReAct) emits book_flight(flight=BA123, date=2026-08-01)         (Part 2)
2. [YOU]   derive idempotency key = hash(user, BA123, date)                 (Part 1)
3. [YOU]   POST /book  Idempotency-Key: <key>  → book, store {key→#55}       (Part 1)
   ... result slow / lost; model retries book_flight with the SAME args ...
4. [YOU]   same derived key → store HIT → return the SAME booking #55        (Part 1)
5. [MODEL] sees success once; no duplicate flight                           (Part 2)
```

## Coupled failure modes

- **Non-idempotent tool + retry-happy strategy → duplicate real-world actions** (Part 1 idempotency × Part 2 retries) — the flagship agent bug.
- **Self-consistency without rate-limit budgeting → `429` storms** (Part 2 N× × [Week 3](phase1-week3.md) `429`) — the agent throttles itself.
- **Aggressive Reflexion on side-effecting tools without idempotency → compounding duplicates** (Part 2 × Part 1).

## Interview & practice (Part 3)

1. Why do agents make the double-charge failure *more* likely than a human-driven client? Walk the `book_flight` scenario.
2. Derive an idempotency key for a `send_email` tool — what goes in it and why?
3. With numbers, how is choosing self-consistency also a rate-limit decision?
4. Why does the idempotency-key discipline *enable* aggressive retry strategies, not merely defend against them?

*Practice — Medium:* add idempotency keys to every side-effecting tool in your agent. • *Hard:* an agent using Reflexion on a `create_ticket` tool files 3 duplicate tickets per task — trace the cause across both Parts and fix it.

---

# Cheat Sheet

**Part 1:** idempotency key = client-generated UUID; server stores {key→response}, replays on repeat, **atomic with the side effect**, TTL'd · cursor > offset pagination (offset skips/dupes under writes) · RFC 9457 Problem Details for errors · version up front (additive = safe; remove/rename/retype = breaking).

**Part 2:** **tokens *are* the computation** (why CoT works) · CoT (reason then answer) · ReAct (reason+act loop) · Reflexion (self-critique) · Plan-and-Execute (plan once) · ToT (tree search, expensive) · self-consistency (sample N, vote) · low temp = planning/args, high temp = brainstorm/self-consistency.

**Part 3:** agents retry constantly ⇒ side-effecting tools MUST be idempotent (the double-charge, agent edition); strategy choice = backend load/rate-limit decision; idempotency *enables* aggressive retry strategies.

**Mnemonics:** "retry ⇒ idempotency key" · "tokens are the scratchpad" · "N samples = N× the bill" · "cursor, not offset."

---

# Build This

**Definition of done:**
1. **(Part 1)** Add an idempotency-key layer to `POST /transfers` (store, replay, in-flight `409`, TTL, atomic ordering). Test: retry the same key → confirm exactly one transfer. Convert an offset-paginated list to cursor-based and prove it survives a concurrent insert with no skips/dupes.
2. **(Part 2)** Implement ReAct and Plan-and-Execute over the *same* task in your [Week 2](phase1-week2.md) agent. Log tokens/calls/latency for each and write a two-line comparison.
3. **(Part 2)** Add a self-consistency wrapper (N=5, majority vote) to a single-answer sub-task; measure accuracy vs. cost.
4. **(Bridge)** Give your agent a side-effecting `book_flight` tool. Force a retry (simulate a lost/slow result) and confirm a duplicate booking. Then add an idempotency key derived from `hash(user, flight, date)` and confirm exactly one booking — *the* aha of the week.

---

# Active Recall & Self-Test

**From memory:** (1) Walk all four cases of the idempotency-key contract and why storage must be atomic. (2) Show with a worked insert how offset paging skips/dupes rows. (3) Why does CoT work mechanically — walk the shirt example. (4) When Plan-and-Execute over ReAct? (5) Why low temp for tool args? (6) Why must every retryable agent tool be idempotent, and how does that *enable* Reflexion?

**Teach-back (60s, aloud):** explain why agents' retry-heavy reasoning makes idempotency keys mandatory, using the `book_flight` example. Stumble → re-read Part 3.

---

# Primary Sources

- **Idempotency keys** — Stripe API docs, "Idempotent requests" (canonical industry reference); IETF draft "The Idempotency-Key HTTP Header Field."
- **Pagination** — GraphQL/Relay cursor-connection spec (relay.dev) for the canonical cursor model; Stripe/GitHub API pagination docs.
- **Error format** — RFC 9457 "Problem Details for HTTP APIs" (obsoletes RFC 7807).
- **API versioning** — Microsoft REST API Guidelines / Google AIP versioning sections.
- **Reasoning strategies** — Chain-of-Thought (Wei et al., 2022), ReAct (Yao et al., 2022), Reflexion (Shinn et al., 2023), Tree-of-Thoughts (Yao et al., 2023), Self-Consistency (Wang et al., 2022). **Temperature/sampling — verify against your model provider's current API docs.**

---

# Key Takeaways & Summary

- **Part 1:** Production API design is about the unhappy path — idempotency keys (client-generated, atomic with the side effect) for safe retries, cursor pagination for correctness under writes, standard error envelopes, and a versioning plan chosen up front.
- **Part 2:** Reasoning strategies tune *how* the model thinks (tokens are the computation); each multiplies cost, so match strategy and temperature to the task.
- **Part 3:** Agents retry constantly, so side-effecting tools must be idempotent (the double-charge, agent edition); strategy choice is a backend load decision; and idempotency *enables* aggressive retry strategies.

**10-second:** Design APIs for the unhappy path (idempotency keys, cursors), learn the menu of reasoning strategies, and see that idempotency keys are what make an agent's constant retries safe.

**1-minute:** Production APIs handle retries (client-generated idempotency keys stored atomically and replayed on repeat), concurrent writes (cursor not offset pagination), uniform errors (RFC 9457), and change (versioning up front — additive safe, rename/remove/retype breaking). Reasoning strategies exploit that a model's generated tokens *are* its computation — CoT, ReAct, Reflexion, Plan-and-Execute, Tree-of-Thought, and self-consistency each trade tokens/latency for quality, with temperature matched to the sub-task. The bridge: agents retry far more than human-driven clients (failed tools, replanning, N-sample voting), so every side-effecting tool must be idempotent — the same key that prevents a double charge prevents a double booking — and choosing a strategy is simultaneously choosing a backend cost and rate-limit budget.

**5-minute:** read Part 1 §1 (idempotency-key contract + atomicity) and §2 (offset-vs-cursor worked example), Part 2's token-compute foundation + the strategy table, then Part 3's agent-double-charge trace and the self-consistency rate-limit example.

**Expert summary:** production API discipline and agent reasoning meet at retries. The backend teaches you to make operations safe to repeat and to pay deliberately for reliability; agents, being autonomous and retry-happy, need exactly that safety, and their reasoning strategies are a cost/quality dial priced in model and API calls. Master idempotency and you can let an agent retry as hard as it needs to.

---

# Next: [Week 5 — Auth mechanisms & serving an agent as a service](phase1-week5.md) · Prev: [Week 3](phase1-week3.md)
