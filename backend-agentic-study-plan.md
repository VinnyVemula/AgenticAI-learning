# Unified Study Plan: Backend Engineering + Agentic AI
### One interleaved path, sequenced so each backend concept feeds directly into an agentic one

The organizing principle: **an agent is a distributed backend system with a non-deterministic dependency.** So this plan pairs each backend topic with the agentic topic it directly enables. You learn idempotency the same week you learn agent tool-retry semantics — because they're the same problem. You learn Durable Functions' replay model the same week you learn agent checkpointing — same idea, different name.

Structure: **6 phases over ~24 weeks** (adjust to your pace — this assumes ~8–12 focused hours/week alongside a job). Each week has a **Backend track**, an **Agentic track**, the **Bridge** (why they're paired), an **Under-the-hood** focus, and a **Build** deliverable. Do the build every week — reading without building doesn't stick for this material.

A pacing note: the agentic tooling landscape shifts monthly (mid-2026). Treat every specific framework/version name as "correct at time of writing, verify before you commit architecture to it." The *concepts* below are stable; the *product names* are not.

---

# PHASE 0 — Environment & Mental Models (Week 1)

Before anything else, get the modern toolchain and the two foundational mental models in place.

## Week 1 — Setup + the two mental models
- **Backend track**: Install and adopt the modern Python toolchain — `uv` (package/env/version management), `Ruff` (lint+format), a type checker (mypy strict), `pytest` + `pytest-asyncio`. Set up a `pyproject.toml` (PEP 621), `src/` layout, committed lockfile, pinned Python version. This is your standard project skeleton for everything that follows.
- **Agentic track**: Get the two foundational mental models straight, on paper, before touching a framework:
  1. **An LLM call is a stateless input→output function.** It has no memory. "Memory" and "conversation" are illusions your backend constructs by re-sending prior messages every turn.
  2. **An agent is a `while` loop around that function**: perceive (read state/tool results) → reason (decide next action) → act (call a tool or answer) → repeat until a stop condition. Everything else in this plan is variations on how that loop is structured, who runs it, and how state persists between iterations.
- **Bridge**: Both tracks start from the same realization — the LLM is stateless, so *all* durability, state, and memory are backend responsibilities. This is why backend rigor is the prerequisite, not a nice-to-have.
- **Under the hood**: How `uv` resolves and locks dependencies (a resolver producing a locked graph, not sequential `pip install`s). How Ruff runs 800+ rules in milliseconds (Rust, single-pass AST). Why type checking catches real bugs (Pydantic + strict typing = validated boundaries).
- **Build**: A project skeleton repo you'll reuse every week — `uv`-managed, Ruff+mypy+pytest wired into a pre-commit hook and a CI stub. Add a trivial FastAPI "hello" endpoint and one passing async test.

---

# PHASE 1 — Foundations: Async, HTTP, and the Raw Agent Loop (Weeks 2–5)

## Week 2 — Concurrency model + the raw ReAct loop
- **Backend track**: Concurrency precisely — processes vs threads vs coroutines; the **GIL** (only one thread runs Python bytecode at a time; releases during I/O; fine for I/O-bound, useless for CPU-bound parallelism); the **asyncio event loop** (single thread, runs a coroutine until `await`, then switches).
- **Agentic track**: Hand-roll a **ReAct loop** in plain Python with the OpenAI/Azure OpenAI SDK — no framework. A `while` loop that sends messages, parses a tool-call from the response, executes it, appends the result, and loops until the model emits a final answer.
- **Bridge**: The agent loop *is* an async I/O-bound workload — every iteration is a network wait on the model or a tool. Understanding the event loop now is what lets you make agents concurrent later (parallel tool calls via `asyncio.gather`) without blocking.
- **Under the hood**: What "blocking the event loop" concretely means — a sync `time.sleep()` or blocking DB call inside `async def` freezes *every* concurrent request on that worker. In the agent loop: how the model never executes anything; it only emits *text describing an intended tool call*, and your code does the actual execution.
- **Build**: A hand-rolled single-tool ReAct agent (e.g., a calculator or a weather-lookup tool) with an explicit max-iteration stop condition. No framework.

## Week 3 — HTTP semantics + tool-call internals
- **Backend track**: HTTP semantics done properly — idempotent vs non-idempotent methods; the real status-code taxonomy (409 vs 422 vs 400 are *not* interchangeable); caching headers (`ETag`, `Cache-Control`, conditional requests); content negotiation.
- **Agentic track**: **Function calling / tool use internals** — you send JSON-schema tool descriptions with the prompt; the model is constrained to emit a structured tool-call object (name + args); your code parses, executes, and appends the result as a `tool`-role message. Constrained/structured decoding (JSON mode, grammar-constrained decoding) at the token level is the mechanism.
- **Bridge**: A tool *is* an API call. A well-designed tool is a well-designed API endpoint — clear schema, typed inputs/outputs, correct semantics. The HTTP status taxonomy you learn this week is exactly how a tool should signal success/failure back to the model so it retries sensibly instead of spiraling.
- **Under the hood**: How constrained decoding forces valid JSON (the sampler masks tokens that would break the grammar). Why tool outputs tokenize worse than prose (JSON/stack traces are token-dense) and eat context budget.
- **Build**: Give your Week 2 agent a *second* tool that's a real HTTP call, and make both tools return structured, LLM-readable errors (not raw stack traces) on failure.

## Week 4 — API design details + reasoning strategies
- **Backend track**: Senior-level API design — **idempotency keys** for side-effecting POSTs (client sends a unique key, server dedupes retries); **cursor-based pagination** (vs offset, which skips/dupes under concurrent writes); consistent error format (RFC 7807); versioning strategy chosen up front.
- **Agentic track**: Reasoning strategies as prompt-level patterns and *why each works mechanically* — Chain-of-Thought (more reasoning tokens = more forward-pass compute before answering), **ReAct** (the loop you built), Reflexion/self-critique, Plan-and-Execute, Tree-of-Thought, self-consistency (sample N, majority vote).
- **Bridge**: Idempotency keys are *the* pattern that makes retries safe — and agents retry constantly (failed tool calls, re-planning). The idempotency discipline you build this week is exactly what stops an agent's retry of a "create order" tool from double-charging a customer.
- **Under the hood**: Why CoT works — the intermediate tokens *are* the computation; the model has no scratchpad except the tokens it generates. Why low temperature suits planning steps and higher suits brainstorming (sampling determinism).
- **Build**: Add an idempotency-key layer to a side-effecting tool in your agent, and implement a Plan-and-Execute variant (plan the full task list once, then execute) alongside your ReAct loop. Compare token cost.

## Week 5 — Auth mechanisms + FastAPI internals
- **Backend track**: Auth at the mechanism level — session vs token; **OAuth 2.0 / OIDC** authorization-code-with-PKCE flow end to end (what's exchanged at each redirect, access vs refresh vs ID token); **JWT internals** (header.payload.signature; signature *verification* is the security boundary, not decoding; claim validation).
- **Agentic track**: **FastAPI internals** — ASGI vs WSGI (FastAPI on Starlette = async-native, which is *why* it supports SSE/WebSockets); Pydantic v2's Rust core (`pydantic-core`); dependency injection as a per-request dependency graph; the `lifespan` context manager (open connection pools once at startup); the **blocking-call trap** explicitly.
- **Bridge**: When you expose your agent behind an API (soon), it needs auth, and its tools need service-to-service auth. Learn the mechanisms now so the agent's tool layer isn't a security hole. FastAPI's DI is *also* how you'll cleanly scope per-request agent/tool instances with the right credentials and user context.
- **Under the hood**: Why an `async def` handler calling a sync blocking library freezes the whole worker (the coroutine never yields control back to the event loop). Why FastAPI's `def` handlers auto-run in a threadpool but `async def` ones don't.
- **Build**: Wrap your agent in a FastAPI endpoint with proper auth, using DI to inject a per-request agent instance. Deliberately introduce a blocking call inside an async handler and observe the concurrency collapse under load; then fix it.

---

# PHASE 2 — Data, Memory & Retrieval (Weeks 6–10)

## Week 6 — Relational model + memory taxonomy
- **Backend track**: The relational model properly — normalization (and deliberate denormalization for reads); the four **ACID** properties; **transaction isolation levels** (Read Committed / Repeatable Read / Serializable) and the exact anomaly each prevents (dirty / non-repeatable / phantom reads).
- **Agentic track**: **Memory taxonomy** — working memory (the context window itself), episodic (past sessions), procedural (learned skills/plans), semantic/world (your RAG knowledge base). Context-window management: sliding-window truncation, running summarization, structured scratchpads.
- **Bridge**: Agent "memory" is just data persistence with a retrieval strategy. Episodic memory is rows in a table/vector store; the context window is your *working set*. Isolation levels matter the moment two agent sessions touch shared state concurrently.
- **Under the hood**: Why the context window is a flat token sequence reprocessed every turn (via the **KV-cache** — every prior token gets re-attended to, which is *why* long agent conversations get slow and expensive). "Context rot" — quality degrading as the window fills.
- **Build**: Persist your agent's conversation history to a real relational DB (Postgres), keyed by session ID, and reload it on the next request. You now have episodic memory backed by a real transaction boundary.

## Week 7 — Indexing + embeddings & vector search
- **Backend track**: **Indexing internals** — B-tree indexes (fast for equality/range, useless for `LIKE '%x%'`); composite index column-order rules; why every index slows writes; reading `EXPLAIN ANALYZE` as a habit; spotting a sequential scan that should use an index.
- **Agentic track**: **Embeddings and vector search** — dense vectors from an embedding model; similarity as cosine/dot-product distance; ANN (approximate nearest neighbor) indexes (HNSW) and why exact search doesn't scale.
- **Bridge**: A vector index is *an index* — same fundamental job (avoid scanning everything), different distance metric. The `EXPLAIN ANALYZE` intuition (is this using the index or scanning?) transfers directly to reasoning about vector-search recall/latency tradeoffs.
- **Under the hood**: How HNSW builds a navigable multi-layer graph for sub-linear nearest-neighbor search, and the recall-vs-latency dial that comes with it. Why embedding model choice (general vs code vs multilingual) and context limits matter.
- **Build**: Stand up a vector store (start with `pgvector` on Postgres so vectors live next to your relational data — zero new infra) and embed a small document set. Query by similarity.

## Week 8 — Query optimization + RAG pipeline internals
- **Backend track**: The **N+1 query problem** (recognize the loop-that-queries-per-iteration pattern; fix with eager loading/joins); **connection pooling** (why pools exist; how pool size interacts with concurrency; the serverless scale-out problem where many worker instances each want a pool).
- **Agentic track**: **RAG pipeline internals** — chunking (semantic/structure-aware, not fixed-char — the #1 cause of bad RAG); **hybrid search** (dense + sparse/BM25, because vector search misses exact-match IDs/codes); **reranking** (a cross-encoder second pass — usually the highest-ROI addition to mediocre RAG); chunk-to-context assembly with citations.
- **Bridge**: Naive RAG that retrieves per-sub-query in a loop is literally an N+1 problem against your vector store. The connection-pooling lesson matters because your RAG store is one more pooled dependency your scaled-out Functions workers hammer.
- **Under the hood**: Why reranking helps — bi-encoder retrieval (fast, approximate) surfaces candidates; a cross-encoder (slow, accurate, sees query+doc together) reorders them. Why bad chunking poisons everything downstream (retrieval can only return what chunking preserved).
- **Build**: A full RAG pipeline — semantic chunking → hybrid search → rerank → assemble with citations. Measure retrieval quality against a small golden Q&A set (you'll formalize eval later).

## Week 9 — NoSQL & CAP + agentic RAG and vector store selection
- **Backend track**: NoSQL models and the **CAP theorem** — document (Cosmos/Mongo), key-value (Redis), wide-column, graph; under a partition you *must* choose consistency or availability; why Cosmos DB exposes consistency as an explicit dial rather than hiding the tradeoff. **PACELC** (even without a partition, latency vs consistency).
- **Agentic track**: **Agentic RAG** (retrieval becomes a *tool* the agent calls, possibly multiple times, reformulating queries — vs. fixed pre-retrieval); **GraphRAG** (traversal-based retrieval for relationship-heavy data); vector store selection tradeoffs (Azure AI Search's hybrid+semantic reranking, pgvector, Pinecone/Qdrant/Weaviate, Cosmos DB vector).
- **Bridge**: Choosing a vector store *is* a CAP/PACELC decision — consistency of your knowledge base vs. retrieval latency vs. operational simplicity. GraphRAG is why you learned graph data models. Agentic RAG turns retrieval from a data-access step into an agent action, closing the loop with Phase 1's tool-use work.
- **Under the hood**: How Cosmos DB's five consistency levels map onto real read/write latency and staleness. Why agentic RAG can outperform static RAG (the agent can notice a bad retrieval and re-query) but costs more tokens/latency.
- **Build**: Convert your Week 8 RAG into **agentic RAG** — make retrieval a tool the agent can call and reformulate. Migrate (or add) Azure AI Search as the store and compare hybrid+semantic reranking against your pgvector baseline.

## Week 10 — Caching + prompt caching & cost control
- **Backend track**: **Caching patterns** — cache-aside (default), write-through, write-behind, and their consistency tradeoffs; **Redis internals** (in-memory data-structure server: strings/hashes/sorted-sets/lists/pub-sub; eviction policies LRU/LFU/TTL; RDB vs AOF persistence); cache invalidation (the hard problem — TTL vs explicit-on-write).
- **Agentic track**: **LLM cost & latency control** — prompt caching (reusing unchanged prefix tokens across turns), model routing (cheap model for triage, expensive only for hard reasoning), context compression/summarization to avoid re-sending huge histories.
- **Bridge**: Prompt caching *is* caching — same cache-aside intuition, applied to token prefixes. Model routing *is* the "use the cheap path when you can" instinct behind cache-aside. Redis is where you'll actually store session/agent working state and rate-limit counters.
- **Under the hood**: How prompt caching works at the provider level (the KV-cache for a stable prompt prefix is retained/reused, so you pay less and wait less for the cached portion — connecting straight back to Week 6's KV-cache). Cache invalidation as the reason stale RAG context is dangerous.
- **Build**: Add Redis-backed session state and a rate limiter to your agent API. Add prompt-prefix caching and a cheap-model triage step; measure the cost/latency delta.

---

# PHASE 3 — Resilience, Durability & the Agent State Machine (Weeks 11–15)

This is the conceptual heart of the whole plan — where "durable execution" unifies both tracks.

## Week 11 — Resilience patterns + agent error handling
- **Backend track**: **Resilience patterns precisely** — retries with exponential backoff **+ jitter** (naive fixed retries synchronize into a thundering herd); **circuit breakers** (fail fast on a clearly-failing dependency, test recovery periodically); **bulkheads** (isolate resource pools per dependency); **timeouts everywhere** (every network call needs one).
- **Agentic track**: **Agent error handling as a first-class design problem** — what the model *sees* when a tool fails determines whether it retries sensibly or loops forever; guarding against infinite agent loops (max iterations, loop detection, cost ceilings).
- **Bridge**: An agent calling flaky tools is a distributed system calling flaky dependencies — the *same* patterns apply. A runaway agent retry loop is a thundering herd; a circuit breaker around a failing tool stops the agent from burning tokens hammering it. This is a near-perfect 1:1 mapping between the two tracks.
- **Under the hood**: Why jitter matters mathematically (decorrelates retry timing across many callers/agent steps). How to make a tool timeout surface to the model as a structured, retryable signal rather than a hang.
- **Build**: Wrap every tool in your agent with timeout + backoff-with-jitter + a circuit breaker, plus a global loop-guard (max iterations + cost ceiling). Chaos-test by making a tool fail on purpose.

## Week 12 — Messaging & delivery guarantees + async multi-agent communication
- **Backend track**: **Messaging** — queues vs pub/sub vs event streaming (Service Bus queue/topic, Event Grid, Event Hubs/Kafka — one-consumer vs fan-out vs replayable log); **delivery guarantees** ("exactly-once" = at-least-once + idempotent consumers); **dead-letter queues**.
- **Agentic track**: **Async multi-agent communication** — when agents (or agent tasks) communicate across service boundaries rather than in one in-process conversation; the blackboard pattern (shared state vs direct message passing); long-running agent tasks that shouldn't hold an HTTP connection open.
- **Bridge**: Multi-agent systems are distributed systems — agents talking via a queue have the *same* delivery-guarantee problems as any producer/consumer. "Exactly-once = at-least-once + idempotent consumer" is exactly why your agent's tools (from Week 4) needed idempotency keys.
- **Under the hood**: How a queue's at-least-once delivery + visibility timeout + poison-message handling works, and why your consumer must be idempotent. Why holding an HTTP connection for a multi-minute agent run is an anti-pattern (use a queue + background worker + status polling/webhook).
- **Build**: Move a long-running agent task off the HTTP request path — trigger it via a queue, process in a background worker, return a job ID, expose a status endpoint. Handle the poison-message case.

## Week 13 — Distributed systems patterns + agent architecture patterns
- **Backend track**: **Architecture patterns** — the **outbox pattern** (write event + business change in one transaction so no event is lost on crash), the **saga pattern** (multi-step cross-service transaction with compensating actions), CQRS, event sourcing.
- **Agentic track**: **Agent architecture patterns** — single-agent (ReAct loop as a state machine, plan-and-execute, reflection loop) and multi-agent (sequential/pipeline, hierarchical orchestrator-subagent, group-chat/debate, handoff, blackboard).
- **Bridge**: The **saga pattern and the multi-step agent are the same shape** — a sequence of steps where any step can fail and you need compensating actions. Event sourcing (state as an append-only event log) is *exactly* how an agent's message history works. Recognizing these as the same pattern is a genuine level-up moment.
- **Under the hood**: How a saga's compensating transactions unwind partial progress on failure — and why a well-designed agent needs the same (an agent that booked a flight but failed to book the hotel needs a "cancel flight" compensating action, not just an error message). Why the hierarchical/orchestrator-subagent pattern maps onto a manager coordinating worker services.
- **Build**: Build a multi-step agent workflow (e.g., a 3-service saga) with explicit compensating actions on each step's failure, structured as a hierarchical orchestrator + specialist subagents.

## Week 14 — Durable Functions internals + agent checkpointing (the keystone week)
- **Backend track**: **Durable Functions internals** — the orchestrator's **replay model**: the orchestrator code is *replayed from the start* each time it resumes after a `yield`/`await` on an activity, with already-completed steps returning cached results from history instantly rather than re-executing. This is *why* orchestrator code must be deterministic (no direct random/datetime/network calls in the orchestrator — push those into activities). Fan-out/fan-in, and `waitForExternalEvent` for human interaction.
- **Agentic track**: **Durable agent execution / checkpointing** — serializing agent/graph state after each step so a crashed run resumes from the last checkpoint instead of restarting an expensive multi-step run. This is what LangGraph's checkpointer, Microsoft Agent Framework's durable execution, and Temporal all solve.
- **Bridge**: **These are the same concept.** Once the Durable Functions replay model clicks, you already understand what every durable-agent-execution framework is doing under a different name. This is the single highest-leverage pairing in the entire plan — and it's your Azure home turf, so you get it "for free."
- **Under the hood**: How replay + history reconstructs orchestrator state deterministically without persisting the whole call stack; why non-determinism in the orchestrator breaks replay. How a checkpointed agent graph serializes state channels to a store between nodes.
- **Build**: Build a nontrivial Durable Functions orchestration (fan-out/fan-in *or* a human-approval-gate via `waitForExternalEvent`), then **deliberately kill the worker mid-run** and confirm it resumes correctly. This is the "aha" moment — do not skip the crash test.

## Week 15 — Human-in-the-loop + approval gates & durable agents
- **Backend track**: The **human-interaction durable pattern** in depth — `waitForExternalEvent` as a durable pause; timeout handling on human steps; combining with the saga pattern for approvals that can be rejected.
- **Agentic track**: **Human-in-the-loop approval gates** for agents — any irreversible/high-cost action (payment, deletion, external comms) pauses for explicit approval; this is a *design pattern* (a durable wait), not an afterthought.
- **Bridge**: The agent approval gate *is* the Durable Functions human-interaction pattern from Week 14. You already built the mechanism; now you apply it as a safety control for the agent. Backend and agentic tracks fully converge here.
- **Under the hood**: How a durable wait survives process restarts (the orchestration is checkpointed as "waiting for event X"; the event, when it arrives, resumes the specific waiting instance). Why this is far more robust than an in-memory `await` on a future.
- **Build**: Add a human-approval gate to your Week 13 multi-agent saga — the agent pauses before a destructive action, a human approves/rejects via an endpoint, and the durable workflow resumes or compensates accordingly.

---

# PHASE 4 — Protocols, Frameworks & Interop (Weeks 16–19)

## Week 16 — OpenAPI & contract testing + MCP server authoring
- **Backend track**: **Contract testing** (catch breaking API changes before a consumer does); OpenAPI as a first-class contract (FastAPI generates it automatically); how OpenAPI schemas map to client generation.
- **Agentic track**: **Model Context Protocol (MCP)** at the wire level — roles (Host / Client / Server, one client per server connection); transports (stdio for local, Streamable HTTP for remote); the three primitives (**Resources** = read-only context ≈ GET, **Tools** = side-effecting functions ≈ POST, **Prompts** = reusable templates); discovery + `listChanged` notifications.
- **Bridge**: Your existing FastAPI app is *nearly agent-ready* — its auto-generated OpenAPI spec maps almost directly onto tool definitions, and authoring a clean MCP server is the same skill as writing a good REST API (clear schemas, idempotent tools, good error messages). Contract testing is exactly how you keep MCP tools stable for the agents that depend on them.
- **Under the hood**: MCP messages are JSON-RPC 2.0 over the transport; how `listChanged` keeps a host in sync without polling; the N×M → N+M integration-problem framing MCP solves.
- **Build**: Author an MCP server exposing 2–3 of your existing FastAPI endpoints as tools (proper schemas, idempotent, structured errors). Connect it to a real MCP host (Claude Desktop or another) and drive the full loop from outside your code. Add a contract test for the tool schemas.

## Week 17 — Security deep-dive + prompt injection & tool poisoning
- **Backend track**: **OWASP API Security Top 10** as a working checklist — especially **BOLA** (broken object-level authorization: forgetting the "does this user own this object" check, the most common real API vuln); injection classes (SQL injection → parameterized queries always); secrets via Key Vault + Managed Identity; RBAC vs ABAC.
- **Agentic track**: **Agent security** — **prompt injection** (untrusted content — web pages, docs, tool outputs, MCP resources — carrying instructions the model may follow); **tool poisoning** (a compromised MCP server returning data crafted to manipulate the agent); least-privilege tool scoping; sandboxed code execution.
- **Bridge**: **Prompt injection is the LLM-era cousin of SQL injection** — untrusted input interpreted as instructions instead of data. The parameterization instinct (separate code from data) and the BOLA instinct (check authorization on every object access) both transfer: treat all tool/RAG/MCP output as untrusted data, and enforce tool permissions with *real backend RBAC*, not prompt instructions telling the model to "be careful."
- **Under the hood**: Why prompt injection is unsolved-in-general (the model has no reliable channel separation between trusted instructions and untrusted content in a single token stream) — hence defense-in-depth: input/output guardrail classifiers as a *separate, non-bypassable* step outside the agent's own reasoning. Why least-privilege must live in the backend the tool calls, since the model's judgment is exactly what you can't trust.
- **Build**: Red-team your Week 16 MCP agent — attempt a prompt injection via a tool's returned data, and a BOLA-style attack where the agent tries to access another user's resource via a tool. Add backend RBAC enforcement + an output guardrail classifier and confirm the attacks fail.

## Week 18 — Compute selection + framework #1 (Azure-native)
- **Backend track**: **Choosing Azure compute** — Functions (Consumption/Premium/**Flex Consumption**) for event-driven scale-to-zero; **Container Apps** for always-on containers with scale-to-zero; App Service for simple web apps; AKS only when you truly need Kubernetes control. Cold starts (why they happen; Premium/Flex pre-warming; smaller packages; lazy imports).
- **Agentic track**: **Framework #1 — Microsoft Agent Framework** (unifies Semantic Kernel's enterprise orchestration + AutoGen's multi-agent patterns; sequential/concurrent/handoff/group-chat/Magentic-One patterns) wired to **Azure AI Foundry Agent Service** (hosted agents, model catalog, Foundry IQ for managed retrieval).
- **Bridge**: This is your path of least resistance given your stack — the framework runs on the Azure compute you're now choosing deliberately, and Foundry Agent Service's hosted agents are just another compute+state option alongside Functions/Container Apps. Rebuild an earlier agent here so you're comparing frameworks against a baseline you already understand.
- **Under the hood**: How Foundry Agent Service manages agent state/orchestration server-side vs. running the framework yourself in a Function. How cold starts specifically hurt agent latency (a cold worker + a slow first model call compound).
- **Build**: Rebuild your Week 13 multi-agent saga in Microsoft Agent Framework, deployed on Azure (Functions/Container Apps), with Durable Functions handling the checkpointed orchestration underneath. Note what the framework automated vs. what you hand-built earlier.

## Week 19 — System design synthesis + framework #2 (industry-standard)
- **Backend track**: **System design fundamentals** — horizontal vs vertical scaling (stateless services + externalized state); sharding & replication (and replication-lag consistency); **rate-limiting algorithms** (token bucket allows bursts, leaky bucket smooths, sliding-window counter avoids the fixed-window boundary flaw); designing for failure as the default posture.
- **Agentic track**: **Framework #2 — LangGraph** (explicit directed graph with conditional edges + state channels + checkpointer; the largest ecosystem and the most auditable/controllable model). Plus know-the-name coverage of the rest: CrewAI (fast role-based prototyping), OpenAI Agents SDK (handoffs), Claude Agent SDK (subagents), Google ADK (agent tree), LlamaIndex Workflows (RAG-first), Pydantic AI (type-safe).
- **Bridge**: LangGraph's graph + checkpointer is a **stateless-compute-over-externalized-state** design — exactly the horizontal-scaling principle from the backend track, and exactly the Durable Functions replay model from Week 14. Rate limiting matters because agents can generate bursty, expensive load. You should now be able to look at *any* new framework and classify it as "which of these patterns (handoff/hierarchical/group-chat/checkpointed-graph) does it implement?"
- **Under the hood**: How LangGraph's state channels + checkpointer serialize graph state between nodes (compare directly to Durable Functions history/replay). Why the graph model maps cleanly to auditability/rollback (each node transition is a checkpoint).
- **Build**: Rebuild (or extend) your agent in LangGraph with a real checkpointer and a conditional-edge branch. Write a one-page comparison: LangGraph vs Microsoft Agent Framework vs your hand-rolled loop — what each abstracts, what each costs.

---

# PHASE 5 — Observability, Evaluation & Production Hardening (Weeks 20–24)

## Week 20 — The three pillars + agent tracing
- **Backend track**: **Observability's three pillars** — structured logs (correlated by request/trace ID; never `print`), metrics (the RED method: Rate/Errors/Duration + latency percentiles), distributed tracing (one request's path across every service with per-hop timing). **OpenTelemetry** as the vendor-neutral standard.
- **Agentic track**: **Agent observability** — capturing the full *trajectory* (every LLM call, tool call + args/result, retrieval step, handoff, per-step token cost + latency, full message history), because agent failures are multi-step causal chains, not single bad responses. Tooling: LangSmith (LangGraph-native), Langfuse (open-source, self-hostable), Arize Phoenix (OTel-native, eval-heavy), Foundry-native tracing.
- **Bridge**: **Agent observability is distributed tracing** with LLM-specific spans. The trace-ID correlation and OpenTelemetry instrumentation you learn on the backend side is *literally the same instrumentation* (OpenLLMetry/OpenInference are OTel semantic conventions for LLM spans) — instrument once, view agent traces and infra traces in one system.
- **Under the hood**: Why a distributed trace is a tree of spans with parent-child causal links — and why an agent run is exactly that tree (orchestrator → subagent → tool → model). Why single-call LLM monitoring misses agent failures (the failure is in the *chain*, not any one call).
- **Build**: Instrument your Week 19 agent end-to-end with OpenTelemetry so a single user request is one connected trace across FastAPI → orchestrator → subagents → tools → model calls. View it in Langfuse or Foundry tracing.

## Week 21 — Testing strategy + agent evaluation
- **Backend track**: **The test pyramid** (many unit, fewer integration via testcontainers, few e2e); pytest fluency (async fixtures, parametrization, `dependency_overrides` for test doubles); load/performance testing (Locust/k6 to find your real breaking point).
- **Agentic track**: **Agent evaluation methodology** — offline golden-dataset evals (fixed inputs+expected behavior, run in CI); **LLM-as-judge** (and its failure modes: verbosity bias, position bias — calibrate against human labels); **trajectory evaluation** (scoring *how* the agent got there — right tools, right order, no redundant calls — not just the final answer); online/production evals + drift detection.
- **Bridge**: A golden-dataset eval is a **regression test suite** for a non-deterministic system — same role in CI, adapted for "approximately correct" instead of "exactly equal." Trajectory eval is integration testing for the agent's decision path. The `dependency_overrides` skill lets you mock tools/models in agent tests exactly as you mock DB sessions in API tests.
- **Under the hood**: Why LLM-as-judge needs calibration (a model scoring model output inherits biases; you validate the judge against human-labeled samples before trusting it). Why trajectory matters — a right answer reached via an unsafe/wrong path is still a production bug.
- **Build**: Build a golden-dataset eval suite (task success rate + tool-call accuracy + a trajectory check) and gate it in CI so a score regression blocks a merge exactly like a failing unit test.

## Week 22 — Containers & CI/CD + deploy-gated agent evals
- **Backend track**: **Docker** at the concept level (layered image filesystems + build-cache ordering; containers = namespaced/cgroup-isolated processes, not VMs); **CI/CD** as a non-negotiable automated gate (lint → type-check → test → build → deploy).
- **Agentic track**: **Eval-gated deployment** for agents — wiring the Week 21 eval suite into CI/CD so agent quality regressions block deploys; canary/shadow deployment for prompt/model changes; adversarial/red-team tests as part of the suite.
- **Bridge**: The CI/CD pipeline you build for the backend is the *same* pipeline that gates agent quality — you just add eval and red-team stages alongside unit tests. Containers matter for agents specifically when you need sandboxed code-execution tools (isolated, resource-limited execution environments).
- **Under the hood**: Why Dockerfile layer ordering affects rebuild speed (unchanged layers are cached; put rarely-changing deps before frequently-changing code). How a sandboxed code-execution tool uses container/microVM isolation as a real security boundary, not a suggestion.
- **Build**: A full CI/CD pipeline for your agent service: lint/type/unit tests → integration tests (testcontainers) → agent eval suite → red-team tests → containerized build → deploy. A regression at any stage blocks the deploy.

## Week 23 — Production engineering + agent cost/latency/reliability at scale
- **Backend track**: **SLIs/SLOs/error budgets** (pick indicators that reflect real user experience; spend the budget on shipping riskier changes deliberately); scaling stateless workers over externalized state; production reliability (retries, circuit breakers, idempotency — now second nature).
- **Agentic track**: **Agent production concerns at scale** — cost control (prompt caching + model routing + context compression, revisited under real load), latency (streaming, parallel tool calls, short-circuit paths for common intents), reliability (idempotent tools, durable checkpointing, graceful degradation when the model/API is down).
- **Bridge**: SLOs for an agent are just SLOs — but you now define them on agent-specific SLIs (task success rate, cost per task, escalation rate) using the trajectory data from Week 20. The scaling model (stateless workers + externalized state) is the same one that made both Durable Functions and LangGraph checkpointers work.
- **Under the hood**: How streaming (SSE/WebSocket, from Week 5's FastAPI internals) cuts *perceived* latency on long agent runs even when total time is unchanged. Why cost per task (not per token) is the metric that actually predicts your bill under agentic workloads (an agent can make many calls per task).
- **Build**: Define real SLIs/SLOs for your agent (task success, p99 latency, cost per task, escalation rate), build a dashboard from your OTel data, load-test the whole system, and fix whatever breaks first (likely connection pool exhaustion or cost blowout — both things you now know how to diagnose).

## Week 24 — Capstone: full production agentic system
- **Backend track**: Everything converges — a stateless, observable, resilient, secured, tested, CI/CD-deployed service.
- **Agentic track**: Everything converges — a hierarchical multi-agent system with RAG, MCP tools, durable checkpointed orchestration, human-approval gates, guardrails, full tracing, and CI-gated evals.
- **Bridge**: The capstone *is* the thesis of this plan made concrete: a production backend system where the agent layer is the single non-deterministic component sitting on infrastructure you fully trust and can debug.
- **Under the hood**: You should now be able to trace any failure to its layer — is it the model's non-determinism, a tool/API bug, a state/durability issue, a retrieval-quality issue, or an infra/scaling issue? — because you built each layer deliberately.
- **Build**: A complete production-grade agentic system in your domain: multi-agent orchestrator + specialist subagents, agentic RAG over a real knowledge base (Azure AI Search), your own FastAPI endpoints exposed as MCP tools, durable checkpointed orchestration on Azure with a human-approval gate on one destructive action, guardrails, end-to-end OTel tracing, a CI-gated eval + red-team suite, and an SLO dashboard. Load-tested and documented.

---

# Optional Deep-Dive Modules (as-needed, not in the main sequence)

These are worth studying *when a project demands them*, not upfront:

- **Fine-tuning & model customization** — when to fine-tune (rigid format at high reliability; cost/latency reduction by moving a task to a smaller model; large volume of a narrow repeated task) vs. when better prompting/tools/RAG suffices (usually). LoRA/QLoRA (train small low-rank adapter matrices instead of full weights — understand the mechanism even if you use a managed service); distillation (large model's outputs as training data for a cheaper model); RLHF/DPO at a theory level (explains *why* models behave as they do — you won't run it).
- **Transformer internals in depth** — self-attention, causal masking, RoPE positional encoding, the KV-cache (worth a dedicated week if you want to reason precisely about latency/cost/context limits rather than just accept them).
- **A2A (Agent2Agent) protocol** — when you actually need cross-organization/cross-framework agent delegation (complementary to MCP: MCP is agent↔tool, A2A is agent↔agent). Study it when you have a real second agent to delegate to.
- **GraphRAG in depth** — when your domain data is genuinely relationship-heavy and vector similarity alone underperforms.
- **Kubernetes/AKS depth** — only if a project genuinely outgrows Functions/Container Apps.

---

# How the two tracks map, at a glance

| Backend concept | Agentic concept it enables | Same underlying idea |
|---|---|---|
| Idempotency keys | Safe tool-call retries | Making retries non-destructive |
| Transaction isolation | Concurrent agent session state | Consistency under concurrency |
| Indexing / EXPLAIN | Vector search / recall tuning | Avoid scanning everything |
| N+1 query problem | Naive per-query RAG loops | Batch, don't loop |
| Caching (cache-aside) | Prompt caching / model routing | Reuse the expensive result |
| Retries + backoff + jitter | Agent loop guards / tool retries | Fail gracefully, don't stampede |
| Circuit breaker | Stop hammering a failing tool | Fail fast, test recovery |
| Saga pattern | Multi-step agent + compensation | Undo partial progress on failure |
| Event sourcing | Agent message history | State as an append-only event log |
| Durable Functions replay | Agent checkpointing | Resume, don't restart |
| `waitForExternalEvent` | Human-in-the-loop approval gate | Durable pause for external input |
| OpenAPI contract | MCP tool schema | Machine-readable capability contract |
| SQL injection defense | Prompt injection defense | Separate instructions from data |
| BOLA authorization checks | Least-privilege tool scoping | Enforce authz on every access |
| Distributed tracing | Agent trajectory observability | One request as a span tree |
| Regression test suite | Golden-dataset evals | Block regressions in CI |
| SLIs / SLOs | Task success / cost-per-task | Measure real user outcomes |
| Stateless + externalized state | LangGraph checkpointer / hosted agents | Scale compute over shared state |

That last table is the real point of the plan: there are far fewer *distinct* ideas here than there are topics. Master the backend idea and the agentic version is mostly recognition, not new learning.

---

*Landscape caveat (mid-2026): framework and Azure product names/versions move monthly. The backend fundamentals and the conceptual bridges above are stable; re-verify specific agentic product names, versions, and pricing before committing architecture to them.*
