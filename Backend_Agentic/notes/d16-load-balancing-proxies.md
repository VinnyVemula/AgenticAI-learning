# Day 16 — Load Balancing, Proxies, and the Canonical Trace

> **Framing.** Days 9 through 15 built you a stack, one layer at a time: frames and switches, IP and routing, TCP and UDP, DNS, HTTP, TLS, and caching/CDNs. Every one of those days quietly assumed something this day finally makes explicit: **that "the server" is not one machine.** DNS resolved you to *an* address; TCP handshook with *something* at that address; TLS terminated *somewhere*; your HTTP request landed on *some* process. Today answers the question every prior day dodged: which one, chosen how, and what happens the instant it dies mid-request?
>
> That question is load balancing, and the piece of infrastructure that answers it — sitting in the request path, terminating one connection and originating another — is a proxy. This day is also, deliberately, the last teaching day of Phase 1, and its second half is not a new concept at all: it is Days 9–15 run back to back, once, as a single trace, so you can stand at any point between a keystroke and a pixel and say exactly which layer you're in and which day taught it.
>
> **The honest agentic connection**, and there is exactly one load-bearing thread, not three unrelated trivia facts bolted on for coverage: load balancers are built on an assumption — that a connection is short-lived and that nothing which matters survives past one request — that ordinary web traffic mostly satisfies and that agentic systems routinely violate. A multi-turn conversation with an LLM, a streaming completion running for tens of seconds, an agent's long-lived pooled connection to a tool backend — all three are exactly the kind of long-lived, state-carrying traffic that a load balancer's health-check ejection and connection-draining machinery was not designed around. That thread shows up three times below, each time as the natural next paragraph of the concept it belongs to (health checks, sticky sessions, connection draining), never as a bolted-on section. Everywhere else — proxy mechanics, L4/L7 splitting, the algorithms, the canonical trace itself — this is foundational backend infrastructure with no genuine agentic angle, and it's taught as exactly that.

---

## Roadmap

Load balancing is usually taught as a list of algorithm names with one-line definitions — round robin, least connections, IP hash — which produces readers who can recite the list and then design a system that falls over the first time a backend crashes mid-deploy. We're going to build it the same way Day 12 built DNS: as a chain of decisions that infrastructure is *forced* into, each one closing off a failure the previous decision didn't handle.

```
The problem: one server can't handle 1M req/s, and it WILL eventually crash mid-request.
                                      │
                    Where does "many servers, one address" live?
                                      │
        ┌─────────────────────────────┴─────────────────────────────┐
        │                                                            │
  THE CLIENT DECIDES                                          A MIDDLEBOX DECIDES
  (DNS round robin — Day 12 concept 8;                        (a proxy sits between
   a smart client like gRPC/Envoy)                             client and servers)
                                                                       │
                                                    ┌───────────────────┴────────────────────┐
                                                    │                                          │
                                          WHO DOES THE PROXY                          AT WHICH LAYER DOES
                                          SPEAK FOR — WHOSE                            IT INSPECT THE
                                          AGENT IS IT?                                  TRAFFIC?
                                                    │                                          │
                                    ┌───────────────┴────────────┐          ┌──────────────────┴───────────────────┐
                                    │                             │          │                                       │
                              FORWARD PROXY                REVERSE PROXY   L4 (TCP/IP only —                 L7 (parses HTTP —
                              (speaks for the               (speaks for    fast, blind to the                can route by path,
                               CLIENT outward)               the SERVER     request itself)                   header, cookie;
                                                              inward)                                          needs TLS terminated
                                                                                                                 here — Day 14)
                                                                                                    │
                                                                          HOW DO YOU PICK *WHICH* BACKEND FOR THIS REQUEST?
                                                                                                    │
                                                        ┌────────────────────────────┬──────────────┴───────────────┬────────────────────┐
                                                        │                            │                                │                    │
                                                ROUND ROBIN /                  LEAST CONNECTIONS               IP HASH /              (preview only:
                                                WEIGHTED RR                    (adapts to real-time             CONSISTENT HASHING     Day 48 —
                                                (simple, load-blind)            load, not just a turn)          (same client → same    real hash-ring
                                                                                                                  backend, no shared     designs)
                                                                                                                  state needed)
                                                                                                    │
                                                                    BUT HOW DO YOU KNOW A BACKEND IS STILL ALIVE?
                                                                                                    │
                                                              ┌──────────────────────────────────────┴──────────────────────────┐
                                                              │                                                                    │
                                                     ACTIVE HEALTH CHECKS                                                PASSIVE HEALTH CHECKS
                                                     (LB polls a /health endpoint                                       (LB watches REAL traffic:
                                                      on its own schedule)                                              N failures → eject)
                                                                                                    │
                                                                WHAT ABOUT STATE (a logged-in user, a shopping cart,
                                                                                    a live LLM conversation)?
                                                                                                    │
                                                        ┌────────────────────────────────────────────┴───────────────────────────────┐
                                                        │                                                                              │
                                                STICKY SESSIONS                                                        STATELESS APP + EXTERNALIZED STATE
                                                (pin one client to one backend —                                       (any backend can serve any request;
                                                 breaks on failover, uneven load,                                       state lives in Redis/a DB/the client)
                                                 doesn't survive scale-down)                                                                     │
                                                                                                    │                             ← the answer that wins almost every time
                                                        HOW DO YOU DEPLOY NEW CODE WITHOUT DROPPING A SINGLE REQUEST?
                                                                                                    │
                                                                              CONNECTION DRAINING
                                                                    (stop routing NEW work here, let IN-FLIGHT
                                                                     work finish, THEN terminate the process)
                                                                                                    │
                              ── every decision above, run once, end to end, on one real request: ──
                                          THE CANONICAL TRACE (DNS → TCP → TLS → HTTP → CDN → LB → app → response → render)
```

Concepts and tiers:

| # | Concept | Tier |
|---|---|---|
| 1 | Forward proxy vs. reverse proxy | **[CORE]** |
| 2 | L4 vs. L7 load balancing | **[CORE]** |
| 3 | Load-balancing algorithms (round robin, weighted, least-connections, IP-hash, consistent-hashing preview) | **[CORE]** |
| 4 | Health checks — active vs. passive — and ejection | **[CORE]** |
| 5 | Sticky sessions vs. stateless + externalized state | **[CORE]** |
| 6 | Connection draining and zero-downtime deploys | **[CORE]** |
| 7 | The canonical trace: what happens when you type a URL | **[CORE]** |

Every concept here is [CORE]. That is not tier inflation — Day 16 is the phase's synthesis day, and the study plan names all seven as things you must be able to open all the way down and use to reason about a real system. Nothing above is a side-quest; each one is load-bearing to the Build task and to concept 7. Where a sub-topic genuinely doesn't need the same depth — consistent hashing's full theory, for instance — it's marked and stopped explicitly inside its parent concept rather than promoted to a concept of its own, exactly as the tier table permits.

One cross-reference to get out of the way immediately: **Day 12 already taught DNS-based load balancing and failover in full** — round-robin DNS, GeoDNS, health-checked DNS failover, and anycast, plus the exact arithmetic for why DNS failover takes on the order of 150 seconds. That material is not repeated here. Today's load balancer is a *different, faster* mechanism operating at a different layer, and concept 2's system design puts both mechanisms into one stack so you can see how they compose rather than compete.

---

## 1. Forward proxy vs. reverse proxy

**Depth: [CORE]**

### Intuition

Start from the plainest possible question: why would anything ever sit *between* a client and a server, deliberately, instead of just letting them talk directly? A direct connection is simpler, has fewer moving parts, and fails in fewer ways — so a proxy has to be earning its keep somehow. It earns it by being someone's *agent*: a piece of infrastructure that acts on behalf of one side of the conversation without that side needing to do the work itself.

That's the whole concept, and everything else is a consequence of one question: **whose agent is it — the client's, or the server's?**

- If it's the client's agent, it exists to control, hide, filter, or cache what the client sends *out* to the wider internet. That's a **forward proxy**: it faces outward from a group of clients, and the server on the far end sees the proxy, not the client.
- If it's the server's agent, it exists to control, hide, or distribute what comes *in* to a group of backend machines. That's a **reverse proxy**: it faces inward from a group of clients, and the client sees the proxy, not the actual backend that does the work.

Notice something important before going any further: at the level of bytes on a wire, a forward proxy and a reverse proxy do **almost exactly the same mechanical job**. Both accept a connection, parse or peek at what's inside, decide where it should really go, open a *second, separate* connection to that destination, and relay data between the two. If you stripped away configuration and naming, you could not tell which kind of proxy you were looking at from a packet capture alone. The entire distinction is administrative, not mechanical: **who configured it, and which side is it hiding.** This is a "define before use" trap worth naming explicitly, because most confusion about proxies comes from people trying to find a technical difference that mostly isn't there.

### Analogy: the travel agent and the maître d'

**Forward proxy — the corporate travel agent.** You, an employee, never book a flight directly with an airline. You tell the company's travel desk what you want; the travel desk enforces policy (no first class, no airlines the company has blacklisted), books it, and pays. The airline's records show "Acme Corp Travel Services" as the customer of record — not you, personally. The airline doesn't know or care that dozens of different employees are hiding behind that same booking department. That's a forward proxy: your company's proxy server is *your* agent, filtering and relaying your requests out to a world that only ever sees the proxy.

**Reverse proxy — the restaurant's maître d'.** As a customer, you address "the restaurant" as a single entity. The maître d' decides which of several available servers actually takes your table — and if your original waiter calls in sick, you get reassigned without ever being told, because from your side, you were always just ordering from "the restaurant." That's a reverse proxy: it's the *server side's* agent, and the actual backend that handles your request is deliberately invisible to you.

**Where the travel-agent analogy breaks.** A real travel agency does its job once — it books the flight — and then steps out of the picture; you deal with the airline directly at the gate. A forward proxy does the opposite: it stays *in the data path for the entire conversation*, relaying every single byte in both directions for as long as the connection lives. It never hands you off to the destination directly. This matters mechanically: it's why a forward proxy handling HTTPS has to either stay blind (tunnel encrypted bytes it cannot read) or actively intercept and re-encrypt the traffic itself — there is no "book it and step away" option, which the worked example below makes concrete.

**Where the maître d' analogy breaks.** In a real restaurant, once you're seated, you build an ongoing relationship with *one specific waiter* for the whole meal — that's actually a sticky session, and it's the natural, obvious way to model a server relationship. A well-run reverse proxy in front of a stateless backend deliberately throws that model away: it can hand you to a *completely different* backend on every single request, and it works precisely *because* nothing about your order depends on which waiter took it last time. Concept 5 is the whole argument for why giving up the "your own waiter" model is usually the right call, not a limitation to work around.

### Worked example — the same request, two proxy types, two very different journeys

**Case A: Alice's laptop, behind a corporate forward proxy, calling an external API over HTTPS.**

Alice's machine is configured with `https_proxy=http://proxy.acme-corp.com:3128` (the standard `HTTPS_PROXY` environment variable convention most HTTP clients, including `curl`, `requests`, and `httpx`, honor). She runs:

```
curl https://api.stripe.com/v1/charges
```

Here is what actually crosses the wire — not what Alice typed, but what her machine does:

```
1. curl's FIRST TCP connection is to proxy.acme-corp.com:3128 — NOT to
   api.stripe.com. Alice's machine never opens a socket to Stripe directly.

2. Because the target is HTTPS, curl issues the CONNECT method, which exists
   for exactly this purpose (RFC 9110 §9.3.6 — verify current, this was
   renumbered from RFC 7231 in the 2022 HTTP semantics consolidation):

       CONNECT api.stripe.com:443 HTTP/1.1
       Host: api.stripe.com:443
       Proxy-Authorization: Basic YWxpY2U6cGFzc3dvcmQ=

3. The proxy checks policy — is api.stripe.com on the allowed list? Is Alice's
   Proxy-Authorization valid? — using information it CAN see, because the
   CONNECT request itself is plaintext.

4. If allowed, the proxy opens its OWN, second, separate TCP connection to
   api.stripe.com:443, and replies to Alice:

       HTTP/1.1 200 Connection Established

5. From this instant, the proxy becomes a dumb byte-relay. The TLS handshake
   (Day 14) happens END TO END between Alice's curl and Stripe's server —
   ClientHello, ServerHello, certificate, key exchange, all of it passes
   THROUGH the proxy unread. The proxy cannot decrypt a single byte of the
   actual request or response. It knows only that Alice talked to
   api.stripe.com, never what was said.

Stripe's access log then shows:
    203.0.113.9 - - [24/Aug/2026:10:15:02] "POST /v1/charges HTTP/1.1" 200
    ↑ 203.0.113.9 is the PROXY'S IP. Stripe never learns Alice's real
      address, and cannot rate-limit or geofence her individually — every
      one of Acme Corp's employees looks like one client to Stripe.
```

That last line is worth sitting with: **a forward proxy for HTTPS is, by default, blind.** It can allow or deny an entire *hostname*, and it can log *who connected to what and when* (from the CONNECT line), but it cannot see a URL path, a request body, or a response — because it never decrypts anything. This is precisely why corporate security teams that want to scan outbound traffic for data exfiltration or malware install a **TLS-inspecting** (also called MITM) forward proxy: it pushes a company-controlled root certificate onto every employee laptop (Day 14's trust-chain material — the proxy becomes a locally-trusted CA), terminates the client's TLS connection *itself*, reads the plaintext, re-encrypts a brand-new TLS connection to the real destination, and relays the (now-visible) content between the two. That buys visibility at a real cost: the proxy now holds the plaintext of every "secure" connection on the network, and if its private key or its issuance logic is ever compromised, every laptop trusting that root CA is silently exposed — which is exactly the shape of this concept's first case study below.

**Case B: the same client, hitting a reverse proxy in front of your own service.**

A browser requests `https://shop.example.com/cart`. It has no idea — and no way to find out — that `shop.example.com` is actually three FastAPI processes behind an Nginx reverse proxy. From the browser's side, `shop.example.com` *is* the server; TLS terminates at the proxy (Day 14), and the browser's TLS session partner is Nginx, full stop.

Nginx picks a healthy backend — say `10.0.1.12:8001` — and opens a **second, separate** connection to it, on the internal network, over plain HTTP (the TLS boundary usually ends at the edge; concept 2's system design discusses when it doesn't). Here is what the backend actually receives, versus what the browser actually sent:

```
Client sent to shop.example.com (over TLS, port 443):

    GET /cart HTTP/1.1
    Host: shop.example.com
    Cookie: session=abc123
    User-Agent: Mozilla/5.0 ...

Nginx forwards to 10.0.1.12:8001 (plain HTTP, internal network):

    GET /cart HTTP/1.1
    Host: shop.example.com          ← PRESERVED by proxy_set_header Host $host;
                                       (without this line, the backend would
                                        see "Host: 10.0.1.12:8001" and any
                                        code that builds absolute URLs from
                                        the Host header would generate wrong
                                        links)
    X-Real-IP: 198.51.100.7         ← the ACTUAL client IP — ADDED by nginx;
                                       does not exist in the original request
    X-Forwarded-For: 198.51.100.7   ← same idea, in the multi-hop-aware form
    X-Forwarded-Proto: https        ← tells the backend the ORIGINAL scheme,
                                       because the backend itself only ever
                                       speaks plain HTTP and has no idea TLS
                                       was involved at all
    Cookie: session=abc123          ← passed through unmodified
    User-Agent: Mozilla/5.0 ...     ← passed through unmodified
    Connection: keep-alive

The backend's OWN access log then shows:
    10.0.1.1 - - [24/Aug/2026:10:15:02] "GET /cart HTTP/1.1" 200
    ↑ 10.0.1.1 is NGINX'S internal IP. Without X-Forwarded-For, the backend
      would believe every single request — from every user on the internet —
      came from 10.0.1.1.
```

**The gotcha this exposes, concretely, in code.** A naive FastAPI handler that does rate-limiting or logging by reading the raw connection peer will get this wrong:

```python
# pip install fastapi "uvicorn[standard]"
from fastapi import FastAPI, Request

app = FastAPI()

@app.get("/cart")
async def cart(request: Request):
    # WRONG behind a reverse proxy: this is nginx's IP, not the real client's.
    client_ip = request.client.host
    return {"client_ip": client_ip}
```

Run this behind Nginx and every single user gets rate-limited as if they were the same person, because `request.client.host` reports the *socket peer* — Nginx — not the original client. The honest fix requires trusting the `X-Forwarded-For` header **only from a proxy you control**, because it is a plain HTTP header a malicious client could set to anything if the proxy blindly appended rather than replaced it. Uvicorn ships a `--forwarded-allow-ips` flag (and a `--proxy-headers` toggle) for exactly this: it tells Uvicorn's own `ProxyHeadersMiddleware` to trust `X-Forwarded-For`/`X-Forwarded-Proto` *only* when the request arrived from a listed upstream IP — verify the current default in your Uvicorn version's docs, because this default has changed and getting it wrong in either direction is a real vulnerability (trust nothing → your rate limiter is useless; trust everything → any client can forge their own IP).

```
uv run uvicorn app:app --host 0.0.0.0 --port 8001 \
    --proxy-headers --forwarded-allow-ips="10.0.1.1"
```

### Under the hood: a proxy is two connections wearing a trenchcoat

Here is the fact that explains everything else in this concept, and the reason a proxy can do things a router (Day 10) or a switch (Day 9) fundamentally cannot: **a proxy fully terminates the incoming connection and originates a brand-new, independent outgoing connection.** It is a genuine TCP *endpoint* on both sides — it completes its own three-way handshake (Day 11) with the client, and separately completes its own three-way handshake with whatever it forwards to. Nothing about the client's TCP sequence numbers, congestion window, or connection state carries through to the backend connection; they are two unrelated TCP conversations that happen to be carrying related application-layer data, glued together by the proxy process reading from one socket and writing to the other.

Compare that to a router forwarding an IP packet: the router never terminates anything. It doesn't complete a handshake, doesn't hold connection state, and doesn't understand what's inside the packet's payload — it reads a destination address and forwards the packet, whole, unmodified in payload, at wire speed, in hardware on a good router. A proxy is orders of magnitude more expensive per byte because it is doing real work: parsing the request enough to route it, often buffering the entire body, holding two open file descriptors per in-flight request, and paying full memory and CPU cost for every connection it terminates.

This is *why* a proxy can do everything interesting — inspect a URL path, add a header, retry a failed backend, cache a response, terminate TLS — and it's also why a proxy is real, stateful infrastructure that scales like an application, not like a piece of wire: it has a maximum connections-per-second, a maximum concurrent-connections count bounded by file descriptors and memory, and — like any application — it can itself become the bottleneck or the single point of failure for everything behind it. Concept 2's system design confronts that directly: at 1M req/s, the load balancer tier itself needs a scaling strategy, and that strategy looks less like "run one bigger box" and more like the anycast and ECMP patterns Day 10 already introduced for the same reason. Worth being explicit about a genuine, deliberate stop here: exactly how a production-grade reverse proxy buffers a request body, pipelines multiple requests on a kept-alive connection, or manages its own event loop internally is real depth (it's most of what Nginx's or Envoy's source code *is*), and this note treats "an event-driven proxy holds two sockets and copies bytes between them, informed by whatever L7 parsing it needs to route" as the correct level of black box — enough to reason about behavior and failure modes, not enough to reimplement Nginx.

A small, closely related fact worth naming and then closing: some deployments skip having every response pass back through the proxy at all. A **transparent forward proxy** intercepts traffic at the network layer (via policy-based routing or an in-line network appliance) without any client-side configuration, and **Direct Server Return (DSR)** — which concept 2's GitHub case study relies on — lets a load balancer route the *inbound* request to a backend while the backend replies to the client *directly*, bypassing the LB on the way out entirely. Both are real, both matter at scale, and both are treated here as named-not-opened: know they exist and why (saving the middlebox from having to handle response bandwidth, which is usually far larger than request bandwidth), and go no deeper unless a project forces it.

### System design — corporate egress control for 2,000 engineers

**Problem.** A 2,000-person engineering organization needs outbound web traffic scanned for data exfiltration and malware, but developers also need low-latency, high-reliability access to package registries (PyPI, npm) and cloud provider APIs as part of CI/CD pipelines that run thousands of times a day.

**Requirements.** Centralized policy and audit logging for general web browsing; near-zero added latency and near-zero added failure surface for build/deploy traffic; must not silently break legitimate security mechanisms.

**Alternatives considered.**
1. **A single TLS-inspecting forward proxy for all outbound traffic, no exceptions.**
2. **No proxy at all — endpoint-based DLP/EDR agents on every laptop, network traffic left alone.**
3. **A TLS-inspecting proxy for general browsing, with an explicit allowlist bypass for known-good CI/CD and package-registry destinations.**

**The decision: (3).**

**The actual reason.** The requirement set contains a direct conflict that (1) and (2) each resolve by simply dropping half of it. Option (1) gives real centralized visibility but makes the *entire* build pipeline depend on one more piece of network infrastructure being up and fast — a proxy outage now means every CI job in the company fails, not just browsing. Option (2) avoids that dependency but gives up centralized, tamper-resistant logging (an agent on a laptop a developer has local admin on is not the same trust boundary as a proxy the developer cannot bypass), which most compliance regimes explicitly require. Option (3) keeps the proxy's real strength — visibility and policy on general traffic nobody has audited — while explicitly routing the high-volume, latency-sensitive, well-understood traffic (package registries, known cloud API endpoints) around it, because that traffic's risk profile and volume profile are both different enough to warrant a different answer.

**The trade-off, honestly.** The allowlist is itself an attack surface and a maintenance burden: every new SaaS tool or cloud API a team needs becomes a request to add a bypass rule, and a stale or overly broad allowlist quietly becomes the hole that TLS inspection exists to close. TLS inspection itself requires installing a company root CA on every device, which breaks any application that pins certificates or performs its own out-of-band trust verification (Day 14) — those applications need their own bypass, discovered the hard way when they mysteriously stop working.

**Flip condition.** Move toward (2) — no interception, endpoint-based controls only — once TLS-pinned and mutual-TLS traffic is common enough across the org's tooling that the exception list would swallow the rule, or once the organization's risk tolerance shifts toward "trust the device, not the network" (the general zero-trust architecture direction, where device attestation and per-request authorization replace network-position-based trust). Move toward (1) — no bypass — only for a security posture (regulated environments handling classified or highly sensitive data) where the cost of *any* uninspected egress outweighs the cost of CI/CD reliability, and where the organization is willing to operate its build pipeline through the proxy as a first-class, highly-available piece of infrastructure rather than an afterthought.

### System design — one public hostname for eight microservices

**Problem.** A company is migrating off one monolithic VM onto eight independently deployable backend services. Requirement: customers and partner integrations must keep using one hostname and one set of API paths; TLS certificate management, authentication, and rate limiting should live in one place rather than being reimplemented eight times; internal service boundaries should be free to change (split, merge, rename) without breaking the public contract.

**Alternatives considered.**
1. **A reverse proxy / API gateway** in front of all eight services, doing path-based routing (`/users/*` → users-service, `/orders/*` → orders-service), with TLS terminating at the gateway.
2. **Expose each service on its own subdomain** (`users.api.example.com`, `orders.api.example.com`, …), each with its own certificate and its own small reverse proxy or LB.

**The decision: (1), as the default; per-service subdomains only for services with a genuine reason to be independent at the edge.**

**The actual reason.** The stated requirement is explicitly "one hostname, centralized TLS/auth/rate-limiting, internal topology hidden" — and a reverse proxy is precisely the mechanism that turns "many backends" into "one address" from the outside, which is this concept's entire definition. Path-based L7 routing (concept 2) lets `/users/*` and `/orders/*` be served by completely different codebases, languages, and deploy schedules while the client never has any way to tell — which is exactly the flexibility the migration needs, since services will keep changing shape for months after the cutover.

**The trade-off, honestly.** The shared gateway becomes a single piece of configuration that every team's routing change must go through, and a single shared-fate component: if the gateway itself is misconfigured or down, all eight services are unreachable from the outside even though seven of them are perfectly healthy internally — the exact same "shared blast radius" lesson Day 12 concept 2 taught about DNS zones, now expressed as shared reverse-proxy configuration instead of shared zone data. A bad regex or a bad routing rule pushed once now affects all eight services simultaneously, which is precisely this concept's first case study below.

**Flip condition.** Split into per-service subdomains/gateways once individual services need independent failure isolation *at the edge* — a service whose customers must be unaffected even if the shared gateway's config is broken, or a service with wildly different traffic/scaling/compliance requirements from the rest (e.g., a webhook-receiving endpoint that must never share capacity or a deploy window with the customer-facing API). This is exactly what a Kubernetes cluster running one Ingress object per service, or a service mesh with a per-service edge gateway, is doing — trading the operational simplicity of one shared front door for blast-radius isolation between services.

**Failure modes (shared by both scenarios above, because both are reverse-proxy architectures).** A single bad rule (a routing regex, a WAF pattern, an auth check) deployed globally affects 100% of traffic instantly, because the proxy sits in every request's path — there is no partial blast radius the way there is when one of eight backends misbehaves. A proxy that silently swallows or mismatches its own `Host`/`X-Forwarded-*` rewriting produces backend-side bugs (wrong URLs generated in responses, wrong client IP in every log line, rate limiting that treats the whole company as one user) that look nothing like a networking problem and are debugged for hours as an application bug before anyone checks the proxy config. And because a proxy is a real, stateful process, it can run out of file descriptors or worker processes under load exactly like an application server can — "the proxy is infinite capacity" is a belief, not a fact, and concept 2's system design puts a number on that.

### Case studies

**Cloudflare, July 2, 2019 — a bad regex in a reverse proxy took down a meaningful share of the internet for 27 minutes.** Cloudflare deployed a new Web Application Firewall (WAF) rule containing a regular expression with catastrophic backtracking — a pattern that, on certain inputs, causes a regex engine's runtime to blow up exponentially instead of linearly. Because the WAF runs as part of Cloudflare's reverse-proxy request path in front of every customer, and because the bad rule was pushed globally at once, CPU usage on Cloudflare's edge machines spiked to effectively 100% worldwide within seconds, and every site behind Cloudflare started returning 502 errors. The outage lasted about 27 minutes before the bad rule was identified and pulled. **The lesson, tied directly to this concept:** a reverse proxy that does per-request CPU-bound work (regex matching, header rewriting, auth logic) inherits that cost across essentially all of your traffic, because it *is* the front door — and because Cloudflare's rule propagation was global and near-instant, a logic bug that would have been a slow, contained incident anywhere else became a global outage in the time it takes a config to replicate. *Primary source:* Cloudflare Blog, "Details of the Cloudflare outage on July 2, 2019" (John Graham-Cumming) — verify current for the exact CPU/duration figures if quoting them precisely, as the post has been referenced with slightly different rounding across secondary sources.

**Lenovo Superfish, discovered February 2015 — a forward (interception) proxy shipped with one shared private key.** Lenovo pre-installed adware called Superfish on consumer laptops that acted as a local, transparent, TLS-intercepting forward proxy: it installed its own root certificate into the operating system's trust store and re-signed every HTTPS site the laptop visited — including banks — so it could inject advertising into the decrypted traffic. The catastrophic flaw was not that interception happened; it's that the underlying interception library (Komodia's "SSL Digestor") reused the **same root certificate and private key on every single Lenovo laptop it shipped on.** Anyone who extracted that one private key (and it was extracted, quickly, by security researchers) could forge a trusted certificate for *any* HTTPS site and silently man-in-the-middle *any* Superfish-equipped laptop on *any* network — a coffee-shop Wi-Fi attacker with that one key could intercept banking sessions on every affected Lenovo machine nearby. **The lesson, tied directly to this concept:** a forward proxy that terminates TLS on the client's behalf is exactly as trustworthy as its private key and its issuance logic, and nothing else — the interception mechanism itself is legitimate and widely used by corporate security teams, but it converts "trust the destination server's certificate" (Day 14's whole chain-of-trust model) into "trust this one box's key management," and Superfish demonstrated what happens when that trust is worth nothing. *Primary sources:* US-CERT Vulnerability Note VU#529496; Lenovo's own security advisory and statement (February 2015); contemporaneous analysis from EFF and multiple independent security researchers who extracted and published the shared key within days of disclosure.

### In production

**Best practices.**
1. **Never let a client-supplied header decide something security-sensitive without a trust boundary.** `X-Forwarded-For` and `X-Forwarded-Proto` are only trustworthy from a proxy you control and configure explicitly (Uvicorn's `--forwarded-allow-ips`, Nginx's own header-setting on the way in) — trusting them unconditionally lets any client forge its own IP or scheme.
2. **Keep the proxy layer's configuration in version control with review**, for the same reason Day 12 recommended it for DNS zones: it is a single shared piece of infrastructure whose mistakes have full-fleet blast radius, per the Cloudflare case study above.
3. **Size the proxy tier for connections, not just for request rate.** A reverse proxy holding many slow or long-lived connections (streaming responses, WebSockets, an agent's SSE stream) consumes file descriptors and memory per connection regardless of how few bytes/second it's actually moving — capacity planning done purely in "requests per second" undercounts this.
4. **Log both the client-facing and backend-facing view of a request**, including which specific upstream served it — this is the single most useful piece of information when a customer reports "it's slow" or "it's broken" and you need to know if the fault is at the edge, in the backend, or in between.

**Mistakes, beginner → senior.**
- *Beginner:* reading `request.client.host` for rate limiting or logging behind a reverse proxy and getting the proxy's own IP every time.
- *Intermediate:* trusting `X-Forwarded-For` from any source, allowing IP spoofing for rate limits, geofencing, or audit logs.
- *Senior:* under-provisioning the proxy tier itself for connection count (not just throughput), so a burst of slow clients or long-lived streaming connections exhausts its file descriptors while CPU and bandwidth both look idle — a failure mode that looks like a mystery until someone checks `ulimit -n` and the proxy's own connection count.

**Observability.** Connections currently open on the proxy (not just requests/sec); per-upstream latency and error rate (so a single misbehaving backend is visible before it drags down the aggregate); the proxy's own resource usage (CPU, memory, file descriptors) as a first-class metric, since the proxy is application infrastructure, not wire.

**Scaling and cost.** A reverse proxy scales like any stateful network service: more instances behind an L4 tier (concept 2), sized by connection count and CPU cost of whatever L7 processing it does (TLS termination, WAF rules, header rewriting) — not by bandwidth alone. A forward proxy for an organization scales similarly but is usually sized by concurrent employee connections and, if TLS-inspecting, by the CPU cost of terminating and re-establishing TLS on every connection, which is real and non-trivial at thousands of simultaneous users.

---

## 2. L4 vs. L7 load balancing

**Depth: [CORE]**

### Intuition

Concept 1 established that a proxy is a real endpoint that terminates one connection and originates another. The question this concept answers is: **how much of the traffic does it actually look at before deciding where to send it?** That single question — how far up the stack does the load balancer read before making its routing decision — turns out to explain almost every practical difference between load-balancing products, and it's the same "how far up the stack" question Day 9 through Day 13 have been implicitly training you to ask about every piece of network infrastructure you meet.

A load balancer that only reads as far as the IP header and the TCP/UDP header — source and destination address, source and destination port — is operating at **Layer 4** (the transport layer, in the classic OSI numbering Day 9 introduced). It never looks inside the payload. It cannot tell an HTTP `GET /login` from a `GET /images/logo.png`, because to it, both are just an opaque stream of bytes between a source port and a destination port. What it buys you for that blindness is speed: an L4 load balancer can make its routing decision from the first packet of a connection, hold almost no per-connection state beyond "which backend did I send this 4-tuple to," and in the best implementations, forward packets without even fully terminating the TCP connection (more on that below).

A load balancer that actually parses the HTTP request — reads the method, the path, the `Host` header, cookies, even the body — is operating at **Layer 7** (the application layer). This is a genuine reverse proxy in the full sense concept 1 described: it terminates the client's TCP connection completely, buffers and parses the HTTP request, and only then decides where to send it. That buys enormous routing flexibility — "send `/api/*` to the API fleet and everything else to the web fleet," "send requests with cookie `beta=true` to the canary deployment" — at the cost of doing real per-request CPU and memory work, and at the cost of a hard requirement the L4 tier never has: **if the traffic is HTTPS, an L7 load balancer must be able to read the plaintext HTTP request, which means TLS has to terminate at or before it** (Day 14's handshake — cross-referenced, not re-taught here).

### Analogy: the airport's runway controller versus the gate agent

An **L4 load balancer** is like air traffic control assigning a plane to a runway: it looks at the plane's flight number and destination and clears it to land on runway 4 or runway 9 — it does not open the cargo hold, does not know what's inside, does not care whether it's carrying passengers or freight. The decision is made from information visible on the outside, fast, for every single plane, all day.

An **L7 load balancer** is like the gate agent reading your actual boarding pass: economy versus business class, checked bag versus carry-on only, standby versus confirmed — routing you to a specific line based on the *content* of what you're carrying, not just which airline you flew in on. That's real, valuable routing, and it requires someone to actually look at your documents, which takes real time per person, unlike a runway assignment.

**Where the analogy breaks, and it's an important break:** a runway controller and a gate agent are two people at two different, sequential points in your journey — you pass through one, then later the other, and neither depends on the other's existence. **A real L4/L7 stack is not sequential like that; it's nested.** The L4 tier doesn't hand you off to a separate L7 building — it forwards your *entire ongoing TCP connection* to a machine that is *itself* running the L7 logic, and both are simultaneously true of the same request at the same time: it was L4-routed to get to a machine, and that machine is now L7-routing it further. Concept 2's system design below draws that nesting explicitly, because "L4 or L7?" is usually the wrong question — the real question is "which tier does each job, and where does one hand off to the next."

### Worked example — the same request through both tiers, with real numbers

Take a concrete request: a client hits `https://api.example.com/v2/orders/8891` from a browser. Trace what an L4-then-L7 stack actually does with it, contrasted with what an L4-*only* stack could ever do.

```
Client's SYN packet arrives at the load balancer's public IP, port 443.
  Source: 198.51.100.7:52344      Destination: 203.0.113.50:443

┌─────────────────────────── L4 TIER ───────────────────────────┐
│ Reads ONLY: source IP:port, destination IP:port, protocol.    │
│ Cannot see "api.example.com", cannot see "/v2/orders/8891",   │
│ because TLS hasn't even been decrypted yet — there is no      │
│ plaintext HTTP request to read.                                │
│                                                                 │
│ Decision available: "send this 4-tuple's packets to L7 box #3"│
│ using one of concept 3's hashing algorithms — done ONCE, at    │
│ connection setup, and then every subsequent packet in this     │
│ SAME TCP connection is sent to the SAME box, because           │
│ mid-connection re-routing would break the TCP stream entirely  │
│ (Day 11: sequence numbers are meaningless if you switch peers  │
│ mid-stream).                                                    │
└──────────────────────────────┬──────────────────────────────┘
                                 │  (forwarded, unread, at line rate)
┌────────────────────────────── ▼ ─────────────────────────────┐
│                          L7 TIER (box #3)                      │
│ TLS terminates HERE (Day 14: full handshake completes with     │
│ this box, not the L4 tier).                                    │
│ Now, and only now, the plaintext HTTP request exists to read:  │
│                                                                 │
│     GET /v2/orders/8891 HTTP/1.1                                │
│     Host: api.example.com                                      │
│     Cookie: session=abc123; beta=true                          │
│                                                                 │
│ Decision available: path starts with /v2/orders → route to     │
│ the ORDERS service fleet, not the (also-running) INVENTORY      │
│ fleet behind the same hostname. Cookie beta=true → route to the │
│ canary version of the orders service specifically.              │
│ This decision was IMPOSSIBLE at the L4 tier — the L4 tier never│
│ saw a path, a Host header, or a cookie, because none of that    │
│ exists until TLS is decrypted.                                  │
└──────────────────────────────┬──────────────────────────────┘
                                 │
                          ORDERS-CANARY backend
```

**What an L4-only stack, with no L7 tier at all, is structurally incapable of doing:** routing by path, by hostname, by cookie, or by any HTTP content whatsoever — because it never decrypts and never parses HTTP. It can only route by IP and port. This is precisely why services with a single backend fleet and no need for content-based routing (a raw TCP database proxy, a game server, a gRPC service using its own client-side balancing) are often perfectly happy at L4 only, and why anything doing "route `/api` here and `/static` there under one hostname" *must* have an L7 tier somewhere in the path — there is no way around it, full stop.

### Under the hood: why L4 is fast enough to not really be "a server" at all

The speed gap between L4 and L7 isn't a rough rule of thumb — it comes from a structural difference in what each tier has to *do* to a packet. An L7 load balancer, being a full reverse proxy per concept 1's "under the hood," must completely terminate the TCP connection: complete its own handshake, allocate a socket and buffers, read enough bytes to parse an HTTP request (potentially the whole body, if routing depends on it), and hold that state for as long as the connection lives. That is genuine per-connection application work, bounded by the same constraints — memory, file descriptors, CPU — as any other server process.

A well-built L4 load balancer avoids almost all of that. The fastest production implementations (Google's Maglev, GitHub's GLB Director — this concept's case studies) don't run a userspace TCP stack for each connection at all: they inspect just enough of each packet's headers to compute a hash of the connection's 4-tuple, consult a lookup table built from that hash, and forward the packet — often using **kernel-bypass** techniques (Linux's XDP, which runs code directly in the network driver before the kernel's normal networking stack even sees the packet) so a single machine can push tens of millions of packets per second with a tiny fraction of the CPU a full TCP stack would cost. Some go further with **Direct Server Return (DSR)**, mentioned and stopped at the black-box level in concept 1: the L4 box only ever sees the *inbound* half of the conversation — outbound response traffic goes from the backend straight to the client, bypassing the load balancer's own upload bandwidth entirely. Because HTTP responses are typically far larger than requests, DSR means the L4 tier's bandwidth requirement is dominated by small requests, not large responses, letting one L4 box handle traffic for backends whose combined response bandwidth would otherwise saturate it.

**Where this note deliberately stops:** the exact mechanics of building and maintaining a Maglev-style consistent hash lookup table so that a backend failure remaps the *minimum possible* number of existing connections, and the specifics of XDP/eBPF packet processing, are real engineering depth that this note treats as a named black box — know that "L4 load balancers can run in the kernel/NIC driver, not just userspace, which is why they're fast enough to not meaningfully add latency" and move on; building one is a multi-year systems project, not a Day 16 exercise.

### System design — the LB stack for 1 million requests per second

**Problem** (the study plan's official system design ①). Design the full load-balancing stack for a service that must sustain 1,000,000 requests per second globally, decide where DNS/anycast, L4, and L7 tiers each sit, and decide where TLS terminates and where health checks live.

**Requirements.** 1M req/s sustained; global user base; no single component may be a hard capacity ceiling; failures must be contained to the smallest possible blast radius; TLS everywhere in transit.

**The layered answer, tier by tier, and why each tier exists instead of being folded into its neighbor:**

```
                              DNS (Day 12) + ANYCAST (Day 10)
                              ────────────────────────────────
      Gets each user to the NEAREST regional entry point. Not a per-request
      decision — it's a per-DNS-TTL / per-BGP-route decision, coarse and
      cheap, exactly Day 12 concept 8's job. This tier's whole purpose is to
      turn "1M req/s globally" into "N req/s per region" BEFORE any single
      machine has to see the whole load.
                                       │
                              L4 TIER (this concept)
                              ────────────────────────
      Within a region, spreads the now-regional traffic across MANY L7
      boxes using a fast, near-stateless hash of the connection 4-tuple.
      Runs in the kernel/NIC path (XDP-class implementation) specifically
      BECAUSE it must survive line-rate traffic that would melt a full
      reverse-proxy process. TLS does NOT terminate here — L4 never reads
      far enough into the packet to need to.
                                       │
                              L7 TIER (this concept)
                              ────────────────────────
      TLS TERMINATES HERE (Day 14's full handshake happens against this
      tier, not the L4 tier — this is the answer to "where does TLS
      terminate"). Reads the real HTTP request and does path/header/cookie
      routing (concept 2's worked example), applies rate limiting and WAF
      rules if any, and picks a specific backend using one of concept 3's
      algorithms informed by concept 4's health checks.
                                       │
                              SERVICE / APPLICATION TIER
                              ────────────────────────────
      Your actual FastAPI/whatever processes. STATELESS (concept 5) so any
      instance can serve any request, which is what makes horizontally
      scaling this tier to "however many boxes 1M req/s regionally divided
      by N regions requires" actually work without coordination.
```

**The realistic alternative considered and rejected: a single load-balancing tier that does both jobs** — one fleet of machines that both hashes 4-tuples at the edge *and* terminates TLS and parses HTTP. This is not a straw man; it's what a small service running one Nginx instance in front of a handful of backends genuinely does, and it's fine at that scale.

**The actual reason the two-tier split wins at 1M req/s.** L4 and L7 work scale on *fundamentally different resources* — L4 work is bounded by packets-per-second and is cheapest to do in the kernel/NIC with almost no per-connection memory; L7 work is bounded by CPU-per-request (TLS handshakes, HTTP parsing, WAF regex) and needs real per-connection memory. Collapsing them into one tier means every machine must be provisioned for the *worse* of the two bottlenecks, and — critically — it means you cannot scale the two independently. A traffic spike from a botnet of many small, cheap connections (a volumetric or SYN-flood-style event) is an L4-shaped problem; a traffic spike from fewer, more expensive requests (a slow endpoint suddenly getting hit hard) is an L7-shaped problem. A split stack lets you throw cheap, fast L4 capacity at the first kind of problem and expensive, smart L7 capacity at the second, instead of over-provisioning one undifferentiated fleet for the worst case of both.

**The trade-off, honestly.** Two tiers means two things to operate, monitor, and deploy, and a genuine extra network hop's worth of latency between them (small — often under a millisecond within one datacenter — but real and additive to every request). It also means a decision about where exactly TLS terminates has permanent consequences: terminating at L7 (the common choice, shown above) means the L4 tier never needs certificates or crypto CPU at all, but it also means the L4-to-L7 hop, if it crosses any network boundary you don't fully control, carries plaintext HTTP unless you add a second, internal TLS hop (mutual TLS between L4 and L7, or between L7 and the app tier) — which some zero-trust architectures require and which adds back some of the crypto cost the split was trying to isolate to one tier in the first place.

**Flip condition.** Collapse to a single tier when request volume is low enough that one reverse-proxy fleet's CPU and connection budget genuinely never gets close to either the packet-rate or the compute-rate ceiling — which describes the overwhelming majority of services that are not operating at anything close to 1M req/s. Keep the L4 tier *even at moderate scale* if the workload is genuinely non-HTTP or doesn't benefit from HTTP-aware routing at all (a raw database proxy, a game server) — there, paying for an L7 tier buys you nothing, so you never build one, and "L4 vs L7" degenerates to "just L4."

**Failure modes.** Terminating TLS at L4 by accident (some managed cloud load balancers default to "TCP passthrough" mode, which is L4 by definition) silently disables every path/header-based routing rule you thought you'd configured, because the L7-looking rules are attached to a resource that never actually gets to see decrypted HTTP — a config-shaped bug that looks like "routing just doesn't work" and is genuinely confusing to debug without knowing this distinction cold. A health check (concept 4) configured against the L4 tier but actually needing to verify L7-level correctness (does `/health` return the right JSON, not just "is the TCP port open") will report healthy backends that are actually broken at the application layer — a shallow-health-check failure mode this note returns to explicitly in concept 4.

### Case studies

**Google Maglev (NSDI 2016) — the reference design for software L4 load balancing at massive scale.** Google published "Maglev: A Fast and Reliable Software Network Load Balancer" describing the L4 load balancer that has fronted Google's services since roughly 2008. Maglev runs as software on commodity servers rather than dedicated hardware appliances, uses Equal-Cost Multi-Path routing (ECMP, Day 10) so that any of many Maglev machines can receive any packet, and — this is the detail that matters for concept 3 below — uses a specific consistent-hashing scheme (now often called "Maglev hashing") to build a lookup table that assigns connections to backends with near-perfectly even distribution *and* with the property that when a backend is added or removed, only the minimum necessary fraction of existing connections gets remapped, rather than reshuffling everything. **The engineering lesson tied to this concept:** L4 load balancing at Google's scale is not a hardware appliance problem, it's a *distributed systems* problem — many independent Maglev machines must independently compute the *same* routing decision for the same connection without coordinating on every packet, which is exactly what a good consistent-hashing scheme buys you. *Primary source:* Eisenbud et al., "Maglev: A Fast and Reliable Software Network Load Balancer," USENIX NSDI 2016 (freely available from USENIX).

**GitHub's GLB — open-sourcing an L4/L7 split built for exactly this problem** (the study plan's official case study ②). GitHub built and open-sourced **GLB Director**, an L4 load balancing layer that sits in front of their L7 tier (HAProxy). GLB Director uses ECMP at the network layer so that any of many director machines can receive any inbound packet, then uses a rendezvous-hashing-style scheme (Highest Random Weight, HRW) to consistently pick which backend HAProxy instance should handle a given connection — and, per concept 1's "under the hood," uses Direct Server Return so that the (typically much larger) response traffic flows from HAProxy straight back to the client, never passing back through the director tier. **The engineering lesson tied to this concept:** by splitting L4 (GLB Director: fast, mostly stateless packet forwarding) from L7 (HAProxy: real HTTP termination, routing, and the connection-draining behavior discussed in concept 6), GitHub can scale and deploy each tier completely independently — restarting or redeploying HAProxy instances doesn't require touching the director tier's ECMP routing at all, and scaling director capacity for a packet-rate spike doesn't require provisioning more HAProxy CPU. *Primary source:* GitHub Engineering Blog, posts introducing GLB and the `github/glb-director` open-source repository — **verify current**, as these posts date to roughly 2016–2018 and GitHub's production architecture has evolved since; treat the architectural pattern (ECMP + consistent hashing + DSR for L4, HAProxy for L7) as the durable lesson rather than any specific current implementation detail.

### In production

**Best practices.**
1. **Know, for every hop in your stack, whether it's L4 or L7 — don't assume.** Cloud "load balancer" products bundle both under one name; a "Network Load Balancer" is typically L4, an "Application Load Balancer" is typically L7 (verify current against your provider's docs, this terminology and the exact capability split is provider-specific and drifts).
2. **Decide TLS termination point deliberately, not by default.** Terminating at L7 is the common, simplest choice; terminating traffic re-encrypted between L7 and the app tier is a real zero-trust requirement in regulated environments, and skipping that decision usually means it was made for you by whatever the load balancer's default happened to be.
3. **Keep L4 health checks and L7 health checks distinct and both present** where both tiers exist — an L4 tier can only confirm "a TCP port is open," which says nothing about application correctness (concept 4).

**Mistakes, beginner → senior.**
- *Beginner:* believing "load balancer" is one undifferentiated thing, and being surprised that a specific product can't do path-based routing (it's L4-only).
- *Intermediate:* configuring L7-looking rules (path routing, header matching) on a resource that's actually terminating at L4, and not understanding why they silently do nothing.
- *Senior:* under-provisioning the L7 tier's CPU because "the load balancer handles it," having conflated the cheap L4 hop with the genuinely expensive L7 one.

**Observability.** Separate dashboards for L4 (packets/sec, connections/sec, per-backend byte counts) and L7 (requests/sec, per-route latency and error rate, TLS handshake failures) — collapsing them hides which tier is actually under pressure during an incident.

**Scaling and cost.** L4 capacity is usually cheap per unit of throughput (kernel/NIC-level processing, minimal per-connection state) and L7 capacity is usually the expensive line item (CPU for TLS and HTTP parsing, memory per open connection) — which is precisely why splitting them, per this concept's system design, lets you spend the expensive tier's budget only where content-aware routing actually earns its cost.

---

## 3. Load-balancing algorithms

**Depth: [CORE]**

### Intuition

Concept 2 established where an L7 load balancer gets to read the request. This concept answers the next question: given a pool of backends that all look interchangeable, **which one actually gets this specific request?** That's a scheduling problem, and it's structurally the same problem Day 3 taught you about inside a single machine — an OS scheduler deciding which runnable thread gets the CPU next. The stakes here are different (a bad choice costs milliseconds of extra latency or a failed request, not a starved thread) and the information available is different (network round trips instead of a shared memory scheduler queue), but the shape of the question — "several workers, one queue of work, who goes next, and what do you optimize for" — is the same question wearing different clothes. Every algorithm below is a different answer to "what information do I use to make that choice," ordered from *none* to *quite a lot*.

### The algorithms, from blindest to smartest

**Round robin — the null hypothesis.** Cycle through the backend list in order: request 1 to A, request 2 to B, request 3 to C, request 4 back to A. It requires no information about the backends at all beyond "here is the list" — no current load, no response times, nothing. That's also its entire flaw: **round robin assumes every request costs the same amount of work and every backend can do that work at the same speed.** Both assumptions are false constantly — one endpoint is a cheap cache read and another triggers a slow database join; one backend just started and has a cold cache; one backend is in the middle of a garbage-collection pause. Round robin has no way to notice any of this. It keeps sending its equal one-third share to a backend that is falling behind, because "falling behind" is not a concept round robin has any way to represent.

**Weighted round robin — the first piece of information: static capacity.** Give each backend a weight reflecting its known, fixed capacity — a bigger VM gets weight 3, a smaller one gets weight 1 — and distribute requests proportionally instead of equally. This fixes *permanent* heterogeneity (mismatched hardware, a canary deployment that should only get 5% of traffic) but still knows nothing about *current, real-time* load: a weight-3 backend that happens to be having a slow moment right now still gets three times the traffic of the weight-1 backend, because the weight is a static number set once, not a live measurement.

**Least connections — the first piece of *live* information.** Instead of a fixed rotation or a fixed ratio, send each new request to whichever backend *currently has the fewest requests in flight*. This is the first algorithm that actually reacts to reality: a backend that's slow right now — for any reason, cold cache, GC pause, a slow downstream dependency — naturally accumulates more in-flight requests than its peers, and least-connections routes new work away from it automatically, without anyone configuring anything about *why* it's slow. The worked example below makes the size of this difference concrete with real numbers.

**IP hash — trading load awareness for a guarantee.** Hash the client's IP address (or another stable key) and use the hash to pick a backend deterministically: the same client always lands on the same backend, for as long as the pool of backends doesn't change. This throws away everything least-connections bought you — it is completely blind to current load, and can send a burst of expensive clients to the same unlucky backend — in exchange for a genuine, valuable property: **the same client keeps hitting the same backend without either side needing to remember anything.** That property matters for exactly one reason worth naming honestly up front: it's a way to fake session affinity without a shared datastore. Concept 5 spends real time arguing this is usually the wrong trade to make — but "usually wrong" isn't "always wrong," and knowing precisely what IP hash buys you is what lets you recognize the rare case where it's the right call.

### Worked example — round robin's queueing collapse, with real numbers

Take three backends behind a round-robin load balancer, receiving one request every 50 ms in strict rotation (A, B, C, A, B, C, …) — so each individual backend receives one new request every 150 ms on average. Two of the backends (A and B) are healthy and answer in 50 ms. The third (C) has degraded — maybe it's paused on a slow downstream call, maybe it's mid-garbage-collection — and now takes 500 ms to answer instead of its usual 50 ms.

Round robin has no idea any of this happened. It keeps sending C exactly its scheduled one-third of traffic, because "one-third of traffic" is the entire algorithm. Run the arithmetic over one minute:

```
Requests received by each backend in 60 s:  60,000 ms ÷ 150 ms  =  400 requests

Backend A (healthy, 50 ms/request):
    work demanded  = 400 × 50 ms  = 20,000 ms
    time available = 60,000 ms
    utilization     = 33%   ← comfortably keeping up, spare capacity to burn

Backend B: identical to A. 33% utilized.

Backend C (degraded, 500 ms/request):
    work demanded  = 400 × 500 ms = 200,000 ms
    time available = 60,000 ms
    utilization     = 333%   ← IMPOSSIBLE. C cannot do 200 s of work in 60 s.
```

C cannot possibly keep up — it can complete at most 60,000 ÷ 500 = 120 requests in that minute, but round robin keeps handing it 400. The other 280 pile up in a queue that only grows, request after request, until they start timing out, get dropped, or exhaust C's own connection limit and start failing outright. **Meanwhile A and B, sitting at 33% utilization with capacity to spare, never receive a single extra request to make up the difference** — because round robin's entire decision rule is "next backend in the list," and it has no mechanism to notice that two-thirds of its fleet is idle while one-third is drowning.

Now replay the same scenario with **least connections**. The moment C starts falling behind, C's in-flight-request count starts climbing relative to A and B's (because C isn't finishing its requests as fast as new ones would otherwise arrive) — and every subsequent routing decision looks at that count and sends the new request to whichever backend has fewer requests in flight, which very quickly stops being C. Within a handful of requests, C's queue depth stabilizes near its own real capacity instead of growing without bound, and A and B absorb the load C can no longer take — using capacity that round robin left completely untouched. Least connections didn't need to know *why* C was slow. It only needed one live number: how many requests are currently waiting on each backend.

### Visual — the smooth interleaving trick behind real weighted round robin

A naive implementation of weighted round robin with weights A:3, B:2, C:1 would just repeat the block `A A A B B C` forever — which bursts three A-requests back to back before ever touching B or C, clustering load in time even though the long-run ratio is correct. Production implementations (Nginx's "smooth weighted round-robin," a close relative of an algorithm also used inside HAProxy) instead interleave the sequence so the *heavier* backend's extra share is spread evenly across the rotation rather than clustered:

```
Naive:   A A A B B C  A A A B B C  A A A B B C  ...
                                                    ← A gets hit 3x in a row every cycle

Smooth:  A B A C A B  A B A C A B  ...
                                                    ← still 3:2:1 in the long run, but no
                                                      backend ever gets a triple burst
```

The naive version is a worse choice than it looks: three consecutive requests to A right as a cycle starts is a small, self-inflicted thundering herd against exactly the backend that's supposed to be handling *more* baseline traffic anyway — the interleaved version delivers the identical long-run ratio without ever concentrating load in a burst.

### Consistent hashing — a preview

**Depth: [AWARE] within this concept — a deliberate stop, full theory is Day 48.**

IP hash's naive form usually means literally `hash(client_ip) % N` where `N` is the current number of backends. That formula has a serious, non-obvious failure: **the moment `N` changes — a backend is added or removed — nearly every client's assigned backend changes too**, because `%N` versus `%(N±1)` produces an almost completely different mapping for almost every input. Concretely: go from 3 backends to 4, and roughly (N−1)/N ≈ 75% of clients get remapped to a different backend than they had a second ago, even though only one backend actually changed. If IP hash was standing in for session affinity, that's 75% of your users suddenly landing on a backend that's never seen their session data — right at the exact moment (a deploy, an autoscaling event) when you least want mass disruption.

**Consistent hashing** is the fix, and the one-paragraph idea is this: instead of `hash % N`, imagine every backend and every possible key placed on the circumference of a clock face (a "hash ring"), positioned by their own hash value. A key is assigned to whichever backend's position on the ring comes next going clockwise. Adding a new backend only steals the keys that fall in the arc immediately before its new position — everyone else's assignment is completely undisturbed. The math bears this out: adding a backend to a ring of N remaps roughly 1/N of keys, not (N−1)/N. This idea traces to Karger et al.'s 1997 paper (this concept's case study below) and is the same mechanism behind Maglev's and GLB's lookup tables (concept 2's case studies) and behind the sharding layer of most distributed caches and databases you'll meet later in this program. Nginx open source already exposes a basic form of it today (`hash $request_uri consistent;` inside an `upstream` block — verify current syntax against nginx.org's docs, as load-balancing directive names have occasionally shifted between OSS and Plus over versions). **Where this note stops, deliberately:** building the ring data structure, handling non-uniform key distribution with virtual nodes, and reasoning about worst-case skew are real algorithmic depth that Day 48 is dedicated to. For now, the correct mental model is: *consistent hashing is hashing designed so that changing the pool size disturbs the minimum possible number of existing assignments, instead of nearly all of them* — know that sentence cold, recognize the name when you see it in a system design, and go no deeper until a project forces you to.

### Under the hood: none of this tells you if a backend is even alive

Notice what every algorithm above shares: each one assumes the backend list it's choosing from is a list of **currently-working** backends. Round robin, weighted round robin, least connections, and IP hash are all answers to "given a healthy pool, who's next" — none of them are answers to "is this backend healthy in the first place." Least connections comes closest to *looking* health-aware (a dead backend that stops finishing requests will accumulate an ever-growing in-flight count, exactly like the degraded backend C above), but that's an accidental side effect, not a guarantee — a backend that's dead outright (connection refused, TCP RST) doesn't accumulate in-flight requests at all; it just fails them instantly, and a naive least-connections implementation would happily keep sending it a fair share of new traffic forever, each one failing immediately, because "zero in-flight requests" looks identical whether a backend is fast and empty or dead and instant-failing. Determining which one it actually is — and removing the dead one from rotation entirely — is a separate mechanism layered on top of whichever algorithm you pick, and it's concept 4's entire subject.

### System design — picking an algorithm for two genuinely different fleets

**Problem A: a stateless REST API running on a heterogeneous fleet** — half the backends are last generation's VM size (half the CPU), half are current-generation, and the service does no in-memory session state at all.

**Problem B: a WebSocket-based signaling service for a video call product**, where each call's participants must all land on the same signaling server for the call's duration, and there is not yet a shared datastore holding call state (it lives in the process's memory only).

**Alternatives considered, for each.** Problem A: round robin, weighted round robin, least connections. Problem B: IP hash, sticky sessions at the LB layer (concept 5), or building a shared datastore first and using plain least connections.

**The decision.** Problem A: **weighted least connections** (most L7 load balancers support combining the two — weight sets each backend's baseline share, live connection count adjusts within that). Problem B, under the stated constraint of no shared datastore yet: **a hash of the call ID** (not the client IP, which can change mid-call on mobile networks — see the trade-off below) used as the routing key, i.e., consistent hashing keyed on something stable about the *session*, not the *client*.

**The actual reason.** Problem A has real, known, static capacity differences (old vs. new hardware) *and* no reason any two requests need to land on the same backend — so a static weight captures the permanent asymmetry and live connection count handles everything transient, exactly the two dimensions of information the fleet actually varies on. Problem B has a hard constraint round robin, weighted RR, and plain least connections all violate immediately: those three algorithms can and will send two participants of the *same call* to *two different backends* that share no memory, and the call simply cannot be signaled. Something affinity-based is not optional here — it's the only category of algorithm that satisfies the requirement at all, given the "no shared datastore yet" constraint.

**The trade-off, honestly.** Problem A's weighted least-connections needs an accurate, current weight for each backend generation — get it wrong (e.g., leave a decommissioned "old" backend at its original high weight) and you've reintroduced round robin's blindness by another name. Problem B's hash-based affinity is a **workaround for a missing shared datastore, not an architectural destination** — it inherits every weakness sticky sessions have (concept 5): if the hashed-to backend dies mid-call, every participant's signaling breaks simultaneously and has to reconnect and be re-hashed, with no other backend able to pick up the call's in-memory state, because that state was never anywhere else.

**Flip condition.** Problem A's answer changes only if the fleet becomes homogeneous (then plain least connections is simpler and just as correct) or if requests become wildly non-uniform in cost in a way live connection count doesn't capture well (some LBs then offer "least time" / weighted-response-time algorithms — worth naming as existing, not opened here). Problem B's answer should flip the moment a shared datastore for call state exists (Redis, or a purpose-built low-latency store) — at that point, *any* signaling server can service *any* participant of *any* call, plain least connections becomes correct again, and the hash-based affinity workaround should be deleted, not kept "just in case." This is exactly concept 5's thesis, arriving one concept early because Problem B is its natural on-ramp.

**Failure modes.** A weight left stale after a hardware refresh silently reintroduces round robin's exact failure mode under a different name. A hash-based affinity scheme keyed on client IP (rather than a stable session identifier) breaks the instant a mobile client's carrier reassigns its IP mid-session — a real, common event on cellular networks — sending the same logical session to a *different* backend than before, with no warning to anyone.

### Case study — Karger et al., "Consistent Hashing and Random Trees" (STOC 1997)

The paper that introduced consistent hashing was written to solve a very concrete, very 1997 problem: web caching. As the web grew, a single overloaded origin server needed a layer of caches in front of it, and naive hashing (`hash(url) % number_of_caches`) meant that adding or removing one cache server invalidated almost the entire cache's mapping at once — a "hot spot" storm exactly analogous to this concept's IP-hash failure mode above, just applied to cache servers instead of load-balanced backends. David Karger, along with colleagues including Eric Lehman, Tom Leighton, and others, published the algorithm that fixed this — and the same team went on to found Akamai, building one of the first and largest content delivery networks (Day 15's subject) directly on this idea. **The engineering lesson tied to this concept:** consistent hashing wasn't invented as a load-balancer algorithm at all — it was invented to keep a distributed cache from collectively forgetting almost everything it knew every time the fleet size changed by one machine, and the exact same property (minimal disruption on membership change) is why it shows up again in Maglev's and GLB's L4 lookup tables (concept 2) and will show up again in Day 48's distributed-data-structure material. A second, distinct real-world instance is worth being honest about the limits of: there isn't a single famous *load-balancer-algorithm-choice* postmortem the way Dyn 2016 or the Meta 2021 outage are famous DNS postmortems (Day 12) — algorithm-choice mistakes tend to show up as a *contributing factor* inside a larger cascading-failure incident rather than as the sole named cause. Concept 4's case studies below are exactly that: incidents where uneven load distribution and health-check behavior compounded each other, which is the honest, real place this concept's failure mode actually shows up in the wild. *Primary source:* Karger, Lehman, Leighton, Panigrahy, Levine, Lewin, "Consistent Hashing and Random Trees: Distributed Caching Protocols for Relieving Hot Spots on the World Wide Web," ACM STOC 1997.

### In production

**Best practices.**
1. **Default to least connections (or weighted least connections) for anything stateless with variable-cost requests** — it's the algorithm that reacts to reality with the least configuration, per the worked example above.
2. **Only reach for IP hash or any affinity-based scheme when you've confirmed there's a real reason two requests must land on the same backend**, and treat it as a temporary bridge to a shared datastore (concept 5), not a permanent architecture.
3. **Re-verify static weights whenever the fleet's hardware composition changes** — a stale weight is round robin's blindness wearing a disguise.

**Mistakes, beginner → senior.**
- *Beginner:* assuming round robin distributes load evenly, and being genuinely surprised when one backend falls over under "equal" traffic.
- *Intermediate:* reaching for IP hash to solve a session-affinity problem without recognizing it's a workaround, not a fix, and being surprised when it doesn't survive a backend restart.
- *Senior:* running weighted round robin for years past the hardware refresh that made the weights wrong, because the system "seems fine" — until the day traffic spikes and the underweighted-but-actually-fine backends stay artificially idle while the overweighted-but-actually-old backends buckle.

**Observability.** Per-backend request count *and* per-backend in-flight/queue depth side by side — request count alone hides exactly the round-robin collapse the worked example demonstrated, since C's *count* looked identical to A's and B's even as C's *queue* was exploding.

**Scaling and cost.** Algorithm choice is nearly free computationally (a hash or a min-lookup over a handful of backends costs nothing next to a network round trip) — the real cost is operational: least-connections and weighted schemes need accurate, live inputs (connection counts, weights) to be worth anything, and that bookkeeping is what a real load balancer's control plane spends its complexity budget on.

---

## 4. Health checks — active vs. passive — and ejection

**Depth: [CORE]**

### Intuition

Concept 3 ended on the gap every algorithm shares: none of them ask whether a backend is actually alive. A pool that's stale — still listing a backend that crashed ten minutes ago, or one whose process is technically running but wedged on a deadlocked dependency — will keep receiving its full mathematical share of traffic under any algorithm you pick, because the algorithm's job is *dividing up* traffic among *the list it was given*, not maintaining that list. Something has to maintain the list. That something is a health check, and there are exactly two ways to get the information it needs: **ask, on your own schedule** (active), or **watch what's already happening and draw a conclusion** (passive).

### Analogy: a manager doing rounds versus a manager reading complaint tickets

An **active health check** is a manager walking the floor every five minutes, personally checking that each employee is at their desk and responsive — a dedicated action taken purely to gather information, independent of whether any actual customer needed that employee right now. It costs the manager's time on every round, whether or not anything was wrong, but it means the manager finds out about a problem the moment it exists, before a single customer is affected.

A **passive health check** is a manager who never does rounds at all, but who tracks customer complaints — if three customers in a row report that a particular employee didn't help them, the manager reassigns that employee off the floor. It costs nothing extra when everything's fine, but by construction, **at least a few real customers had a bad experience before the manager acted** — the complaints are the detection mechanism, which means someone had to generate a complaint first.

**Where the analogy breaks.** A human manager reassigning an employee is a judgment call informed by context — a passive health check has none of that nuance; it is a blunt counter (`N failures within a T-second window`) with no way to distinguish "this employee is having a genuinely bad day and needs to be pulled" from "this employee just happened to get three unusually difficult customers in a row." Tune the threshold too sensitive and you eject healthy backends on ordinary noise; tune it too lax and real failures burn through many real users before anyone acts. There is no version of this analogy that captures the fact that the *threshold itself* is a knob you must choose, with no universally correct setting — only a setting that trades detection speed against false-positive rate for your specific traffic pattern.

### Worked example — the honest cost of each, with real numbers, and a real gap in open-source Nginx

**Passive health checks, concretely, in Nginx open source.** Nginx's `upstream` block supports passive checking natively, via two parameters on each `server` line:

```
upstream fastapi_backend {
    server 127.0.0.1:8001 max_fails=2 fail_timeout=5s;
    server 127.0.0.1:8002 max_fails=2 fail_timeout=5s;
    server 127.0.0.1:8003 max_fails=2 fail_timeout=5s;
}
```

`max_fails=2 fail_timeout=5s` means: if 2 proxy attempts to this server fail within a 5-second window, mark it unavailable *for that same 5-second window*, then try it again. **Read what that actually implies for a real user:** nginx has no independent way of knowing a backend died — it only finds out by trying to send it a real request and watching that attempt fail. **Up to 2 real users' requests are sacrificed to detect the failure**, every time, by construction. That's not a bug in the configuration; it's the entire mechanism. The upside is that it costs nothing when everything's healthy — no extra network traffic, no extra CPU, ever.

**Active health checks, concretely — and the gap worth naming honestly.** The theory says "active health checks: the LB polls `/health` on its own schedule, independent of real traffic." **Open-source Nginx cannot do this at all.** The `health_check` directive that polls an endpoint on a timer and ejects a backend *before* any real request ever reaches it is an **NGINX Plus (paid)** exclusive feature — verify current against nginx.com's own health-check documentation, which explicitly separates "passive health monitoring" (available in the free/open-source build) from "active health checks" (Plus only). This is exactly the kind of framework-default-contradicts-the-theory gap Principle 6 exists to surface rather than paper over: if your Build task uses plain open-source Nginx, you are only ever getting passive checks, no matter how the config is written, and no combination of `max_fails`/`fail_timeout` will change that.

If active checks matter enough to be worth paying for — and "zero real users sacrificed to detect a failure" is a genuine, real reason it might — the honest alternatives are: pay for NGINX Plus; switch the L7 tier to **HAProxy**, which has supported active checks (the `check` keyword on a `server` line, with `inter`, `rise`, `fall` tuning its interval and thresholds) in its open-source distribution since essentially its inception; or switch to **Caddy**, whose open-source `reverse_proxy` directive ships active checks natively:

```
# Caddyfile — active health checks are built into open-source Caddy,
# unlike open-source Nginx. Verify current directive names against
# caddyserver.com/docs, Caddy v2's config surface has evolved across releases.
reverse_proxy 127.0.0.1:8001 127.0.0.1:8002 127.0.0.1:8003 {
    health_uri /health
    health_interval 5s
    health_timeout 2s
    lb_policy least_conn
}
```

With this, Caddy probes `/health` on each backend every 5 seconds **regardless of whether any real user is currently sending traffic there**, and ejects a backend the moment a probe fails — no real request needs to fail first. Compute the honest detection-time trade-off with real numbers: Caddy's active check above detects a dead backend in at most one 5-second interval (plus its 2-second timeout, so worst case ~7 s) and **zero real requests are sacrificed** to find out. Nginx OSS's passive check above detects it only once 2 real requests have already failed, but does so with zero background network chatter. **Neither is strictly better — this is a genuine trade-off**, not a solved problem: active checks cost continuous background load (multiply `health_interval` frequency by every backend, every LB instance, all day) in exchange for protecting real traffic from ever touching a known-dead backend; passive checks cost nothing until something breaks, at the price of a small, bounded number of real users hitting the failure first. The flip condition is about how expensive a "real user hits a dead backend" failure actually is for your product: a REST API where a failed request is retried transparently and invisibly (concept 6's `proxy_next_upstream`, below) can often tolerate passive-only checking just fine; a payment or booking flow where a single failed attempt has real cost to a real customer justifies paying for active checks.

### Under the hood: ejection is a timer, not a switch

"Marking a backend unavailable" is not a permanent decision on either scheme — it's a **timed exile**. Nginx's passive scheme, on tripping `max_fails` within `fail_timeout`, removes the backend from rotation for exactly `fail_timeout` seconds, and then — critically — **the very next real request that would have gone to it is used as the recovery probe.** There is no separate "is it back yet" check; OSS Nginx recovers a backend the same way it noticed it died, by trying it with live traffic and seeing what happens. If the backend is still dead, that one probing request fails, and the exile timer resets for another `fail_timeout` window. If it succeeds, the backend is back in full rotation immediately. This is worth knowing cold because it explains a real, otherwise-confusing observation: a backend that crashed and was later fixed (or simply restarted) doesn't require any manual re-registration with an unmodified Nginx OSS config — it silently rejoins the pool the moment a probing request happens to succeed, which could be anywhere from milliseconds to `fail_timeout` seconds after it actually came back, depending on timing.

**The agentic-relevant fact, arriving exactly where it belongs.** Health-check ejection removes a backend from the pool the load balancer hands *new* connections to — but it does nothing at all to connections that were already established and pooled by a client *before* the ejection happened. This is the load-balancer-side mirror of the exact fact Day 12 raised on the DNS side (a long-lived process's connection pool holds an address that outlived the record it came from) and the exact fact Day 11 raised about connection pool `max_lifetime`: an agent process that opened a keep-alive HTTP connection to a specific backend ten minutes ago, and has been quietly reusing it ever since, is not protected by health checks at all — health checks only stop *new* traffic from going to a bad backend, they do not reach into existing pooled connections and close them. If that backend then dies mid-conversation, the agent's *next* request on that pooled connection gets a connection reset, and the fix is the same one Day 11 already taught: bound connection pool lifetimes so a pooled connection is periodically retired and re-established through the load balancer, which re-runs the up-to-date routing decision every time. An HTTP client library that never recycles connections is, from the load balancer's point of view, invisible to every health check it will ever run.

### Runnable example — three FastAPI instances behind Nginx, load-tested, then killed mid-flight

This is the study plan's core Build task. It sets up the environment concept 6's runnable example reuses for the graceful-shutdown half of the same exercise.

**A Windows-native note first, honestly.** `kill -9` sends `SIGKILL`, a POSIX signal; Windows has no equivalent signal delivery model, and native builds of Nginx for Windows are explicitly documented as limited (a restricted worker-process model, not meant for production use). This exercise depends on watching *real* abrupt-process-death socket behavior — the kernel-driven TCP teardown a `SIGKILL`'d process triggers — which is exactly what a genuine POSIX environment gives you for free. **Run this inside WSL2** (Ubuntu). `Stop-Process -Force` in native PowerShell is the closest conceptual analog but does not reproduce the same signal semantics, so treat WSL2 as the correct environment for this specific exercise, not merely a convenient one.

**Setup — three identical FastAPI instances that identify themselves:**

```python
# app.py — pip install fastapi "uvicorn[standard]"
# Each running copy reads its own identity from an environment variable so
# load-test output can show WHICH instance served each response.
import asyncio
import os

from fastapi import FastAPI

INSTANCE_ID = os.environ.get("INSTANCE_ID", "unknown")

app = FastAPI()


@app.get("/health")
async def health():
    return {"status": "ok", "instance": INSTANCE_ID}


@app.get("/work")
async def work():
    await asyncio.sleep(0.05)          # stand-in for real request work
    return {"served_by": INSTANCE_ID}
```

```bash
# inside WSL2 (Ubuntu)
sudo apt update && sudo apt install -y nginx
curl -LsSf https://astral.sh/uv/install.sh | sh      # uv, per project convention
uv init lb-demo && cd lb-demo
uv add fastapi "uvicorn[standard]"

# three instances, one per port, each with its own identity
INSTANCE_ID=A uv run uvicorn app:app --host 127.0.0.1 --port 8001 &
INSTANCE_ID=B uv run uvicorn app:app --host 127.0.0.1 --port 8002 &
INSTANCE_ID=C uv run uvicorn app:app --host 127.0.0.1 --port 8003 &
```

**Nginx config — round-robin traffic across the three, with passive health checks (concept 3's algorithm choice plus this concept's failure detection, in one file):**

```nginx
# /etc/nginx/sites-enabled/lb-demo
upstream fastapi_backend {
    least_conn;
    server 127.0.0.1:8001 max_fails=2 fail_timeout=5s;
    server 127.0.0.1:8002 max_fails=2 fail_timeout=5s;
    server 127.0.0.1:8003 max_fails=2 fail_timeout=5s;
    keepalive 32;
}

server {
    listen 8080;

    location / {
        proxy_pass http://fastapi_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";       # required to actually use the
                                               # keepalive pool above — by
                                               # default nginx speaks HTTP/1.0
                                               # to upstreams and closes every
                                               # connection after one request
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_next_upstream error timeout http_502 http_503;  # retry a
                                               # failed attempt on a DIFFERENT
                                               # backend before giving up
        proxy_connect_timeout 1s;
    }
}
```

```bash
sudo nginx -t && sudo nginx -s reload
```

**Load test with `hey`** (a small Go-based HTTP load generator — `go install github.com/rakyll/hey@latest`, or your distro's package if available; **verify current install method and flag names against the tool's own README**, load-test tool syntax drifts across versions):

```bash
hey -z 30s -c 20 http://127.0.0.1:8080/work
```

**About ten seconds in, in a second terminal, kill instance B abruptly** — using the port rather than a captured `$!` PID, because `uv run` may or may not exec directly into the uvicorn process depending on version, and killing by port is unambiguous either way:

```bash
kill -9 $(lsof -ti:8002)
```

**What `hey` reports, illustrative of the real shape you should expect from running this yourself** (exact counts will vary by machine — the shape, not the numbers, is the point):

```
Summary:
  Total:        30.0184 secs
  Slowest:      1.0043 secs
  Fastest:      0.0498 secs
  Average:      0.0634 secs
  Requests/sec: 314.2201

  Total data:   483060 bytes
  Size/request: 51 bytes

Status code distribution:
  [200] 9421 responses
  [502] 3 responses

Latency distribution:
  10% in 0.0512 secs
  50% in 0.0589 secs
  90% in 0.0721 secs
  95% in 0.1980 secs      ← the ejection window shows up here, not as
  99% in 1.0012 secs      ← a wall of errors, mostly as a latency blip
```

**What actually happened, second by second, and why it comes out looking almost — not perfectly — clean:**

1. **Before the kill:** `least_conn` spreads requests across A, B, C roughly evenly; all three respond in ~50 ms; `hey` sees a tight, boring latency distribution.
2. **The instant `kill -9` lands on B:** the OS kernel tears down B's sockets as part of process cleanup. Any in-flight request nginx had already sent to B — where nginx's outbound bytes are sitting, unread, in a kernel receive buffer that the (now-dead) process never got to `read()` — gets a **TCP reset (RST)**, not a graceful close, because a kernel closing a socket with unread data delivers an abrupt reset rather than a normal FIN (Day 11's socket-teardown mechanics, now showing up as a real symptom instead of an abstract rule).
3. **Nginx sees that reset as an upstream error.** Because `proxy_next_upstream error timeout http_502 http_503;` is configured, nginx **transparently retries that exact request against A or C**, *as long as it hadn't already started streaming a response back to the client*. Every request caught in this narrow window costs the client nothing but a few extra milliseconds of latency — which is exactly the handful of elevated points visible in the p95/p99 latency rows above, not in the status-code table.
4. **The next one or two NEW requests nginx happens to route to B** (via `least_conn`, before it has any reason to know B is gone) fail immediately with connection refused — B's port is now closed at the OS level. These count toward `max_fails=2`; once the second one lands within the `fail_timeout=5s` window, **nginx ejects B from rotation for the next 5 seconds.** These are the small number of genuine client-visible failures — the 3 `[502]` responses above are the honest cost of passive-only checking, the exact trade-off this concept's worked example predicted, not a bug in the config.
5. **For the next 5 seconds, all traffic goes to A and C only** — `hey`'s throughput barely dips, because `least_conn` on two healthy backends absorbs B's share without difficulty at this load level.
6. **After the 5-second exile expires,** nginx's *next* real request to B (per this concept's "under the hood") acts as the recovery probe. Since B's process is still dead, that probe fails too, and the exile resets for another 5 seconds — this repeats silently, forever, until B is restarted.
7. **Restart B** (`INSTANCE_ID=B uv run uvicorn app:app --host 127.0.0.1 --port 8002 &`) and the very next probing request to it succeeds — nginx puts it back into full rotation immediately, with no manual re-registration step.

**Honesty caveats, stated plainly.** The `[502] 3` count above is illustrative of the *shape* this exercise produces, not a captured live run — actually running it will give you a real, slightly different number depending on your machine's timing, and getting a real number from your own terminal is the entire point of the Build task, not something a note can substitute for. This config uses OSS Nginx's passive checks only, per the honest gap named above — running the same exercise against the Caddy config shown earlier will produce **zero** client-visible errors instead of a handful, because Caddy's active probe (worst case ~7 s) discovers B is dead without ever routing a real user's request to it; running both and diffing the `hey` output is a genuinely worthwhile extension of this Build task.

### System design — health checks for a fleet with a slow-starting dependency

**Problem.** A fleet of 50 backend pods, each of which takes roughly 20–30 seconds after process start before it's actually ready to serve correctly — a local in-memory cache has to warm up from a cold start, and requests served before that warm-up finishes are correct but far slower than normal, occasionally slow enough to trip a naive health check's own timeout.

**Requirements.** Newly-started instances must not receive production traffic before they're actually warmed up; a genuinely broken instance must still be ejected quickly; deploys (which restart many instances) must not cause a wave of false-positive ejections during the fleet's warm-up window.

**Alternatives considered.**
1. **One health check, checked from the moment the process starts**, with a generous failure threshold to tolerate the slow warm-up.
2. **Two distinct checks**, one gating whether a new instance is even *added* to rotation in the first place, and a separate, stricter one that runs continuously afterward to catch real failures.

**The decision: (2).**

**The actual reason.** Option (1) forces an impossible compromise on a single threshold: generous enough to tolerate 30 seconds of legitimate warm-up slowness means it's also far too generous to catch a genuinely wedged instance quickly once it's actually serving production traffic — the same number can't correctly describe both "is this brand-new process ready to join" and "has this long-running process stopped working," because those are different questions being asked at different times for different reasons. Splitting them — a **readiness/startup check** that gates initial entry into rotation and only needs to pass once, and a **liveness-style check** that runs continuously afterward with a much stricter threshold appropriate to a warmed-up instance — lets each question have the threshold it actually needs. This is precisely the distinction Kubernetes' own probe model draws (startup, readiness, and liveness probes as three separate, independently-configured checks) — named here, not opened, since the full mechanics of Kubernetes' probe implementation is its own topic; know that this three-way split exists and why, and consult Kubernetes' own documentation for the mechanics if a project puts you inside it.

**The trade-off, honestly.** Two checks are two things to configure, tune, and keep in sync with actual application behavior — a warm-up time that creeps upward as the cache grows (a real, common form of drift) silently turns a correctly-tuned readiness gate into one that lets traffic in too early, and nobody notices until it shows up as elevated p99 latency right after every deploy.

**Flip condition.** A single check is genuinely fine once startup time is short and consistent enough that one threshold comfortably covers both "did it start correctly" and "is it still healthy" without meaningful compromise — which describes most simple, stateless services with no local warm-up cost at all.

**Failure modes.** This scenario is a close relative of, but genuinely distinct from, Day 12 concept 8's system design about health checks touching a shared primary database causing *every region* to be marked unhealthy simultaneously — that failure is about a shared *downstream dependency* correlating failures across independent instances; this one is about a *temporal* mismatch between what a check measures and what phase of an instance's life it's checking, and it shows up specifically clustered around every deploy rather than as a global correlated event. Both are real, and they compound: a deploy that restarts many instances at once, combined with a readiness check that's too lenient, can let a wave of under-warmed instances into rotation together right when a health-check-driven autoscaler is also reacting to the deploy's own transient capacity dip — a smaller-scale version of exactly the dynamic in this concept's Slack case study below.

### Case studies

**Slack, January 4, 2021 — health checks, retry storms, and back-from-holidays load, compounding each other** (the study plan's official case study ①). The first full working Monday after the 2020 holidays, Slack saw traffic well above any previous Monday's, at the same time AWS experienced networking degradation that increased packet loss between Slack's own service tiers. That packet loss degraded connectivity between Slack's web application servers and their backing datastores — connections became slow or failed outright — which meant health checks against those web servers (checking, reasonably, whether they could actually serve a request end to end, not just whether the process was running) began failing. As health checks tripped, capacity was pulled from an already-strained system at exactly the moment load was peaking, and Slack's autoscaling reacted by trying to add capacity that then had to warm up against the same degraded network conditions. Compounding this further: as client connections dropped during the disruption, Slack's own clients did what well-behaved clients are built to do — **they automatically reconnected** — and a synchronized wave of reconnection attempts landing all at once (a retry storm / thundering herd) added a fresh burst of connection-setup load precisely when the system had the least spare capacity to absorb it. **The lesson, tied directly to this concept:** health checks that faithfully report real degradation are doing their job correctly, but "doing their job correctly" during a systemic capacity event can *actively worsen* the event, by removing capacity right when none can be spared — and a client population that retries/reconnects in a synchronized burst turns a network blip into a load spike layered on top of the original problem. **Verify current** for the precise timeline, durations, and root-cause component names — this account reflects the general, well-corroborated shape of the incident; consult the primary source below for exact figures before quoting them precisely. *Primary source:* Slack Engineering Blog, the team's own postmortem of the January 4, 2021 outage.

**Google's *Site Reliability Engineering* book, Chapter 22, "Addressing Cascading Failures."** Google's own SRE book devotes a full chapter to exactly this pattern in the abstract: a health-check or load-shedding mechanism that behaves correctly in isolation can, under systemic stress, remove capacity faster than the system can recover it, triggering a self-reinforcing spiral — the remaining healthy instances take on more load, become slow enough to start failing their own health checks, get ejected in turn, and the pool of "healthy" capacity shrinks further with every round. The chapter's concrete engineering guidance — never let a health check depend on a resource shared by the whole fleet (echoing Day 12 concept 8's finding almost exactly, at a different layer), add hysteresis so a flapping backend doesn't bounce in and out of rotation on every check, and design explicit overload behavior (shedding load gracefully) rather than letting a cascading ejection spiral happen by accident — generalizes directly to any health-check system, DNS-based or load-balancer-based alike. **The engineering lesson tied to this concept:** the danger isn't a bad health check; it's a *correct* health check operating inside a system that has no separate mechanism to prevent "removing unhealthy capacity" from becoming "removing capacity faster than remaining capacity can be added back," which is exactly the Slack case study's shape, generalized. *Primary source:* Beyer, Jones, Petoff, Murphy (eds.), *Site Reliability Engineering*, Chapter 22 — freely available at sre.google/sre-book/addressing-cascading-failures/.

### In production

**Best practices.**
1. **Never let a health check depend on a resource shared across the whole fleet** you're checking — a shared database, a shared cache, a shared upstream dependency — or a hiccup in that one shared thing looks identical to every single backend failing at once, per both case studies above.
2. **Add hysteresis, not just a threshold** — require a different, usually stricter, number of consecutive successes to rejoin rotation than the number of failures it took to leave, so a flapping backend doesn't bounce in and out on every borderline check.
3. **Match what the check actually verifies to what "healthy" means for your service** — a TCP-connect check verifies the process is listening, nothing about whether it can actually serve a correct response; an L7 check that hits `/health` and inspects the body verifies more, but only as much as `/health`'s own implementation actually exercises (a `/health` endpoint that always returns `200 OK` unconditionally is a check that always passes, which is worse than no check because it manufactures false confidence).
4. **Size the failure threshold to how expensive a false ejection is versus how expensive a slow detection is** — there's no universal right number, only the one that's right for your specific cost of each kind of mistake.

**Mistakes, beginner → senior.**
- *Beginner:* a `/health` endpoint that returns `200` unconditionally, giving zero real signal while looking like a real check on a dashboard.
- *Intermediate:* setting `max_fails=1`, ejecting a backend on a single transient blip and thrashing it in and out of rotation under normal noise.
- *Senior:* a deep health check that queries the primary database, working perfectly until the one day the database itself is slow — at which point every backend in the fleet fails its check simultaneously and the "healthy pool" goes to zero, converting a database slowdown into a total outage.

**Observability.** Ejection and rejoin events as a time-series, not just current pool size — "how many times did this backend flap in the last hour" is a leading indicator of a threshold that's tuned wrong, well before it causes a visible incident. Alert on **pool size dropping fast**, independent of *why* — a fast-shrinking healthy pool is dangerous whether the cause is real backend failures or a correlated false-positive wave.

**Scaling and cost.** Active checks cost real, continuous background traffic — `health_interval` multiplied by every backend multiplied by every load-balancer instance running the check, all day, forever — which is genuinely negligible at small fleets and a real line item at very large ones (thousands of backends × sub-10-second intervals adds up). Passive checks cost nothing until something breaks, which is why many systems run both: passive for continuous, free-during-normal-operation detection, active for the faster, zero-real-user-cost detection that matters most during deploys and scale-out events — exactly when a new instance's very first moments matter most.

---

## 5. Sticky sessions vs. stateless + externalized state

**Depth: [CORE]**

### Intuition

Everything so far has quietly assumed that any backend can answer any request equally well. That assumption holds perfectly for a request that's fully self-contained — "compute this," "fetch that record by ID" — but it breaks the moment a request depends on something the *client* did a moment ago that only *one specific backend* remembers: items already placed in a shopping cart, a login that was established, the last several turns of a conversation. If that memory lives only inside one backend process's RAM, then the load balancer has exactly two honest options: **force this client back to that same backend every time** (a sticky session), or **make sure no backend's RAM is the only copy of anything that matters** (statelessness, with the real state moved somewhere all backends can reach). Nearly everything else in this concept is the argument for why the second option, despite looking like more upfront work, is almost always the one a competent architecture ends up choosing.

### Analogy: the maître d's promise, revisited

Concept 1 planted this on purpose: a real restaurant relationship is with *one specific waiter* for the whole meal — they remember your order, your allergies, that you asked for the check twice already. A **sticky session** is the load balancer trying to recreate exactly that: a cookie, or a hash of the client's IP, pins you to the same backend for as long as your session lasts, so *that* backend's in-memory state keeps being useful to you specifically.

**Where this breaks, and it's the whole lesson:** a sick waiter still exists tomorrow, with all their memories intact, ready to serve you again once they're back. **A crashed backend process's memory is not "on break" — it is gone, instantly and completely, the moment the process dies.** There is no recovery, no "the memories come back when it restarts," because a fresh process starts with empty memory by definition. A restaurant's worst case if your waiter quits is an awkward re-introduction to a new one who has to ask what you ordered; a sticky-session architecture's worst case if your pinned backend crashes is that your shopping cart, or your login, or the last twenty minutes of an in-progress conversation, simply cease to exist, with nothing anywhere holding a copy.

### Worked example — the same failure, three ways

Take a service with three backends (A, B, C), each holding session data (a shopping cart) in its own process memory, with the load balancer pinning each client to one backend via a cookie it sets on first contact.

**Scenario 1 — a backend crashes.** Backend B crashes. Every client whose cookie pins them to B is, at that instant, both cut off from an in-flight request (concept 6's problem) *and* permanently missing everything B's memory held about them — not delayed, not recoverable, **gone**. If B was holding sessions for a third of your users, a third of your users' shopping carts just vanished, silently, with no error message anywhere describing *why* — from the user's side, it just looks like the cart is empty again.

**Scenario 2 — you scale down for the evening.** Traffic drops overnight, and an autoscaler removes one backend to save cost — the exact healthy, intended behavior a stateless system is *built* to support cheaply. Here, it strands every session pinned to the removed backend just as surely as an unplanned crash did. **Scaling down is, from a sticky session's point of view, indistinguishable from a crash** — which means a sticky-session architecture can never cheaply and safely shrink its own fleet, exactly when cost-saving elasticity matters most.

**Scenario 3 — uneven, and permanently uneven, load.** Sessions accumulate on whichever backend happened to receive each client's *first* request, and — unlike a per-request algorithm (concept 3) that gets a fresh chance to rebalance on every single request — a sticky assignment is a decision made *once* and then locked in for the rest of that client's session. If backend A happens to have received a disproportionate share of unusually active users (a plausible, ordinary event, not a bug), it stays overloaded relative to B and C for as long as those sessions last, and there is no mechanism available to move a session to a less-loaded backend without breaking exactly the affinity the whole scheme exists to provide.

**Now the alternative: statelessness with externalized state.** The shopping cart data moves out of any backend's process memory and into a shared store — Redis, or a small database table — keyed by a session ID the client presents on every request (a cookie or a bearer token, not an affinity mechanism; concept 5 is deliberately not the same idea as concept 3's IP hash, even though both involve a "key"). Every backend, on every request, reads the current cart from the shared store, does its work, and writes it back if it changed. Replay the three scenarios: a crash loses **zero session data** — the cart was never in the crashed process to begin with, and any backend, including a brand-new one that just finished starting up, can pick up exactly where the last request left off. Scale-down loses nothing, ever, for the same reason. And load balancing works *correctly again* — concept 3's algorithms can freely send any request to any backend on every single request, because nothing about correctness depends on which backend was used last time.

**The one thing this doesn't erase for free, honestly.** Externalizing state doesn't make the state disappear — it moves the reliability problem onto the shared store instead of the backend fleet. Redis (or whatever holds the session data) is now itself something that must be highly available, and a request that needs the session but can't reach the store is stuck regardless of how healthy every backend is. This is a genuine trade, not a magic fix: you've traded "N backends, each a single point of failure for a slice of sessions" for "one shared dependency, a single point of failure for *all* sessions" — which is usually the right trade, because a purpose-built datastore is much easier to make genuinely highly available (replication, failover, a team whose whole job is running it well) than N general-purpose application processes ever were, but it is a trade, not a free win, and it's worth stating exactly that plainly rather than pretending externalized state has no failure mode of its own.

**The genuinely load-bearing agentic version of this exact problem.** Run three instances of an API that fronts an LLM for a multi-turn conversation, behind a load balancer, and you have re-created this concept's exact shape with higher stakes: if conversation history lives in one instance's process memory, that instance's death doesn't just lose a shopping cart — it loses the entire conversation, mid-thought, with no way for any other instance to continue it, and a scale-down event silently truncates every conversation pinned to the removed instance exactly like scenario 2 above. The fix is identical in kind, not just in spirit: conversation state belongs in a shared store (Redis, a database, or — the pattern many LLM API designs actually use — the *client* resubmitting the full conversation history with every turn, which is externalizing state all the way out to the caller instead of to a shared server-side store) so that any instance behind the load balancer can serve any turn of any conversation, and a crash costs at most one in-flight turn, never the conversation itself.

### System design — shopping cart state for 10 million users across 50 pods

**Problem.** An e-commerce platform serving 10 million active users needs shopping-cart state to survive individual backend crashes and routine autoscaling, across a fleet of roughly 50 backend pods that scale up and down through the day with traffic.

**Requirements.** No cart data loss on a single backend failure; autoscaling (adding or removing pods) must never strand a user's cart; any pod should be able to serve any user's request.

**Alternatives considered.**
1. **Sticky sessions** via a load-balancer cookie, cart held in each pod's process memory.
2. **Stateless pods + externalized cart state** in a shared, replicated store (Redis, with persistence and replication enabled).

**The decision: (2).**

**The actual reason.** The stated requirements — survive a crash, survive routine autoscaling, any pod serves any request — are, almost word for word, this concept's three worked-example failure modes stated as requirements instead of as failures. Option (1) fails all three by construction, not by misconfiguration; there is no tuning of a sticky-session scheme that makes a crashed pod's in-memory cart recoverable, because the data was never anywhere else. Option (2) satisfies all three simultaneously, for the same underlying reason: once cart state isn't tied to any specific pod's lifetime, none of pod lifetime's failure modes can touch it.

**The trade-off, honestly.** The shared store becomes genuinely critical infrastructure — it must itself be replicated and made highly available, monitored, and capacity-planned for 10 million users' worth of read/write traffic, which is real, non-trivial engineering effort concentrated in one place instead of spread thinly (and invisibly) across 50 pods' local memory. Every cart read or write is now a network round trip to the store instead of a local memory access, adding real latency to every request that touches cart state — usually negligible against a co-located, well-provisioned Redis, but not zero, and not something to ignore in a latency budget.

**Flip condition.** There isn't a realistic scale or requirement change that flips this decision back toward sticky sessions for genuinely important state — the honest flip condition here is different in kind: sticky sessions become *tolerable* (not *correct*, merely tolerable) only for state that is itself disposable — a UI-only preference that costs nothing if occasionally lost, cached data that's cheap to recompute — where the cost of occasional loss is genuinely lower than the cost of building and running a shared store. The moment the state has real value to the user or the business (a cart, a session, money, a conversation), the flip condition essentially never arrives.

### System design — a legacy monolith that cannot be rearchitected before a hard deadline

**Problem.** A legacy application holds user sessions in each server's process memory (a common default in older application frameworks) and must be scaled beyond one instance for an upcoming traffic event in three weeks — not enough time to migrate session storage to a shared store before the deadline.

**Alternatives considered.**
1. **Ship sticky sessions now** (cookie-based affinity at the load balancer) as a stopgap, and migrate to externalized state afterward.
2. **Session replication** — copy every session to every backend instance as it's created or modified, so any instance holds a full copy and can serve any request without needing affinity at all. This is a genuinely different third option from either extreme above, and it was a common architecture in an earlier generation of application servers (Java EE application server clustering is the classic example) precisely because it tries to get affinity-free routing without standing up a separate datastore.
3. **Delay the multi-instance rollout** until the migration to externalized state is done properly.

**The decision: (1), with an explicit, dated commitment to migrate afterward — not (2), and (3) only if the business genuinely cannot accept the risk in (1).**

**The actual reason.** Under a genuine three-week constraint, sticky sessions are the only option that requires **zero application code changes** — it's entirely a load-balancer configuration (a cookie set on first contact, consistently routing that client back to the same backend for the life of the session), which is exactly what "we don't have time to migrate storage" demands. Option (2), session replication, sounds appealing (no affinity needed, no single external dependency) but is *not* actually less work here — implementing correct replication semantics (what happens on a write conflict, how fast does a new session propagate to every instance, what happens when an instance is added mid-flight) is real, non-trivial application-level engineering, arguably harder to get right under deadline pressure than sticky sessions' zero-code-change property, and it scales its own cost with instance count (N instances means N−1 copies of every session, an overhead that grows exactly as the multi-instance rollout you're trying to enable grows). Option (3) simply fails the actual business requirement (serve the upcoming traffic event), so it's only correct if the honest answer to "can we tolerate the sticky-session risk" is no.

**The trade-off, honestly.** Choosing (1) knowingly accepts every failure mode from this concept's worked example — a crash during the traffic event loses that instance's sessions, scaling down afterward strands sessions, and load distribution can go uneven and stay that way — as a **temporary, explicitly time-boxed risk**, not a resolved problem. The real danger of this decision isn't taking it; it's the well-documented organizational failure mode where "temporary stopgap" quietly becomes permanent because the traffic event succeeds and the pressure to finish the migration evaporates the moment the stopgap stops being visibly broken.

**Flip condition.** This is the correct call specifically *because* of the deadline; the moment the deadline pressure is gone, the calculus reverts entirely to the previous system design's conclusion, and staying on sticky sessions past the committed migration date is the anti-pattern this scenario exists to warn against, not a defensible steady state.

**Failure modes (shared by both scenarios above).** A shared store treated as an afterthought — under-provisioned, unmonitored, running on the smallest instance someone could find — simply relocates the single point of failure without actually reducing risk, and is arguably worse than sticky sessions if nobody notices it's the new bottleneck until it's overwhelmed. A "temporary" sticky-session stopgap with no committed migration date, as noted above, is a slow-motion version of the same mistake.

### Case studies

**The Twelve-Factor App, Factor VI: "Processes."** Heroku's 2011 "Twelve-Factor App" methodology — one of the most widely cited concrete architectural doctrines in web backend engineering — states this concept's thesis as an explicit, named rule: *"Twelve-factor processes are stateless and share-nothing... Sticky sessions are a violation of twelve-factor and should never be used or relied upon. Session state data is a good candidate for a datastore that offers time-expiration, such as Memcached or Redis."* **The engineering lesson tied to this concept:** this isn't one company's postmortem — it's an entire generation of the industry converging independently on the same conclusion this concept's worked example derives from first principles, codified as a named, citable rule specifically because so many teams kept re-learning it the hard way. *Primary source:* 12factor.net, Factor VI ("Processes").

**Kubernetes Services — statelessness as the default, affinity as an explicit, discouraged opt-in.** A standard Kubernetes Service load-balances traffic across a set of pod endpoints with **no session affinity by default** — each request can land on a different pod, which only works correctly if the application is stateless in exactly this concept's sense. Kubernetes does offer an explicit opt-in, `service.spec.sessionAffinity: ClientIP`, for applications that haven't yet made that leap — but it exists as a named exception path, not the default, which is itself a signal of the industry consensus: the platform's designers made "any pod can serve any request" the thing you get automatically, and pinning something to a "conversation ends up in the same place it started" behavior into something you have to explicitly ask for. **The engineering lesson tied to this concept:** the default is not neutral — defaulting to statelessness (no affinity) and requiring an explicit opt-in for the sticky behavior is itself a documented, deliberate design stance about which pattern the platform's maintainers expect to be right almost always. *Primary source:* Kubernetes documentation, "Service" — verify current, as the exact affinity configuration surface (timeout values, supported modes) has been extended across Kubernetes releases.

### In production

**Best practices.**
1. **Treat "does this need to survive a backend restart" as the deciding question for every piece of state**, not "is this convenient to keep in memory right now" — the convenient answer and the correct answer are different questions.
2. **If you must use sticky sessions as a stopgap, put a hard, calendared date on removing them**, and treat that date with the same seriousness as a security patch deadline — per the legacy-monolith scenario's honest warning above.
3. **Size and monitor the shared state store as first-class production infrastructure**, not an implementation detail — it inherits the criticality that used to be spread thin across many backend processes.

**Mistakes, beginner → senior.**
- *Beginner:* storing session data in a global in-memory dictionary "for now," not realizing that "for now" means "until the first restart or the second instance, whichever comes first."
- *Intermediate:* shipping sticky sessions as a deliberate stopgap with every intention of migrating, and never actually scheduling the migration.
- *Senior:* migrating to a shared store and then under-provisioning or under-monitoring it, having correctly diagnosed the architectural problem while recreating a smaller, better-hidden version of it.

**Observability.** Session/cart-loss rate correlated against deploy and scale events specifically — a spike in "empty cart" support tickets immediately after an autoscaling event is the sticky-session failure mode showing up in a support queue instead of a dashboard, and it's worth instrumenting directly rather than waiting to notice it that way.

**Scaling and cost.** A stateless fleet scales its compute tier for free, in the sense that concept 3's algorithms and concept 4's health checks all work exactly as designed with no special cases — the real cost moves entirely to sizing and operating the shared state store for the read/write rate of the whole fleet combined, which is a concentrated, visible cost instead of a diffuse, easy-to-underestimate one.

---

## 6. Connection draining and zero-downtime deploys

**Depth: [CORE]**

### Intuition

Every backend behind a load balancer eventually needs new code. The naive way to ship it — stop the old process, start the new one — has an obvious, unavoidable gap: for however long the old process is dead and the new one isn't ready yet, any request routed to that instance fails, and any request the old process was *already handling* when it was stopped gets cut off mid-response. Multiply that gap across every instance in a rolling restart and "deploy new code" becomes "guarantee a trickle of failed requests, all day, every deploy." **Connection draining** is the fix, and it's a genuinely simple idea stated precisely: **stop sending this instance new work, wait for the work it already has to finish, and only then stop it.** Almost everything interesting in this concept is about the second step — "wait for it to finish" — because that phrase hides a question with no universal answer: *finish by when?*

### Analogy: the restaurant that's closing for the night

A restaurant closing for the night doesn't throw out the diners mid-meal — it stops seating *new* customers at a cutoff time, lets everyone already at a table finish eating at their own pace, and locks the door once the last table has paid and left. That's connection draining exactly: new work stops, in-flight work completes, and only then does the "process" (the restaurant, for the night) actually shut down.

**Where the analogy breaks, and this is the load-bearing break for this whole concept.** A restaurant meal has a naturally bounded duration — nobody's still eating four hours later, so "wait for existing diners to finish" is a plan with an implicit, reasonable time limit built into human behavior. **An HTTP connection has no such natural bound.** A quick JSON API call finishes in milliseconds; a file upload might run for tens of seconds; a **streaming response — an SSE feed, a WebSocket, a token-by-token LLM completion** — can legitimately run for minutes, with no way for the draining mechanism to distinguish "this connection is almost done" from "this connection just started and will run for two more minutes." Whoever configures the drain timeout is making a real, consequential guess about how long the *longest legitimate* connection can run — get it wrong in the short direction and you are, by design, forcibly terminating requests that were behaving completely correctly.

### Under the hood: two coordinated shutdowns, not one

A real zero-downtime instance replacement is actually two separate graceful-shutdown mechanisms, operated in sequence, each solving a different half of the problem — and conflating them is a common source of "I configured draining and it still dropped requests" confusion.

**Half one: the load balancer stops routing *new* traffic to the instance.** In Nginx's open-source config, this is the `down;` parameter on a `server` line inside `upstream {}`, applied and then activated with `nginx -s reload`. Nginx's reload itself is already graceful at the proxy layer — the master process spins up new worker processes with the updated config and lets the *old* workers finish whatever connections they're already handling before those old workers exit — so the act of telling nginx "stop sending new traffic here" doesn't, by itself, drop anything nginx itself was already doing. What it does *not* do is reach past itself and tell the backend application anything at all; nginx marking a server "down" only changes nginx's own routing decisions for future requests.

**Half two: the application itself must shut down gracefully when told to stop**, independent of whether the load balancer has stopped sending it new work. This is where `SIGTERM` — the conventional "please shut down cleanly" signal, distinct from `SIGKILL`'s no-warning termination from concept 4's runnable example — comes in. Uvicorn handles `SIGTERM` by default as a request to stop accepting new connections and let in-flight ones finish, up to a configurable grace period (`--timeout-graceful-shutdown` — **verify current default against your installed Uvicorn version's docs**, as both the flag's existence and its default value have changed across releases). FastAPI/Starlette's `lifespan` context manager runs its shutdown code (the part after `yield`) at exactly this point, which is where any application-level cleanup — closing a database connection pool, flushing a queue — belongs.

**Why both halves matter, and why skipping either one breaks the whole thing.** Skip half one (never tell the LB to stop routing here) and the LB keeps sending brand-new requests to an instance that's already shutting down, some of which will fail outright depending on timing. Skip half two (kill the process the instant the LB stops routing new traffic to it, without a graceful `SIGTERM` grace period) and any request the LB sent a split second *before* the drain began — which is already in flight and which the LB has no way to know about — gets cut off anyway. **Both halves have to run, in the right order, with the drain window between them sized to at least the duration of your slowest legitimate in-flight request** — which circles back to this concept's analogy-break: for a service with streaming responses, that number might be minutes, not the 30-second default many platforms ship out of the box.

### Runnable example — a rolling deploy of the same three instances, with real graceful shutdown

This reuses concept 4's three-instance Nginx + FastAPI setup exactly as built there, extended with a shutdown handler and a rolling-deploy script.

**The FastAPI app, extended with a lifespan shutdown hook and a slower endpoint that makes "in flight" visible:**

```python
# app.py — pip install fastapi "uvicorn[standard]"
import asyncio
import os
from contextlib import asynccontextmanager

from fastapi import FastAPI

INSTANCE_ID = os.environ.get("INSTANCE_ID", "unknown")
IN_FLIGHT = 0


@asynccontextmanager
async def lifespan(app: FastAPI):
    print(f"[{INSTANCE_ID}] startup")
    yield
    # Runs when uvicorn receives SIGTERM and begins graceful shutdown -- this
    # is where real cleanup (closing a DB pool, flushing a queue) belongs.
    print(f"[{INSTANCE_ID}] shutdown signal received, {IN_FLIGHT} requests still in flight")


app = FastAPI(lifespan=lifespan)


@app.get("/health")
async def health():
    return {"status": "ok", "instance": INSTANCE_ID}


@app.get("/work")
async def work():
    global IN_FLIGHT
    IN_FLIGHT += 1
    try:
        await asyncio.sleep(2)      # deliberately slow enough to still be
                                     # in flight when a drain begins
        return {"served_by": INSTANCE_ID}
    finally:
        IN_FLIGHT -= 1
```

**The rolling-deploy script, coordinating both halves of the "under the hood" section above, one instance at a time:**

```bash
#!/usr/bin/env bash
# rolling_deploy.sh -- zero(ish)-downtime rolling restart of the three
# uvicorn instances behind nginx from concept 4's build task.
set -euo pipefail

PORTS=(8001 8002 8003)
IDS=(A B C)

for i in "${!PORTS[@]}"; do
    PORT="${PORTS[$i]}"; ID="${IDS[$i]}"
    echo "== draining instance $ID on port $PORT =="

    # Half one: stop routing NEW traffic here, via nginx's own graceful reload.
    sudo sed -i "s/server 127.0.0.1:${PORT}.*/server 127.0.0.1:${PORT} down;/" \
        /etc/nginx/sites-enabled/lb-demo
    sudo nginx -t && sudo nginx -s reload

    # Drain window: must be >= the slowest legitimate in-flight request.
    # /work sleeps up to 2s, so 5s is safe here -- in production this is a
    # measured number, not a guess.
    echo "   draining for 5s..."
    sleep 5

    # Half two: SIGTERM (graceful), not SIGKILL, lets the lifespan shutdown
    # handler run and lets uvicorn finish anything genuinely still in flight.
    PID=$(lsof -ti:"$PORT")
    kill -TERM "$PID"
    wait "$PID" 2>/dev/null || true
    echo "   instance $ID stopped cleanly"

    # Start the new version, give it a moment to boot, then re-admit it.
    INSTANCE_ID="$ID" uv run uvicorn app:app --host 127.0.0.1 --port "$PORT" \
        --timeout-graceful-shutdown 10 &
    sleep 2
    sudo sed -i "s/server 127.0.0.1:${PORT}.*/server 127.0.0.1:${PORT} max_fails=2 fail_timeout=5s;/" \
        /etc/nginx/sites-enabled/lb-demo
    sudo nginx -t && sudo nginx -s reload
    echo "== instance $ID back in rotation =="
done
```

**Run `hey` continuously against the whole script** (`hey -z 60s -c 20 http://127.0.0.1:8080/work`, started just before the script) and, unlike concept 4's `kill -9` run, the honest expectation here is **zero client-visible failures**, not a handful — because unlike an unannounced crash, every step here is a deliberate, coordinated hand-off: nginx never routes new traffic to an instance mid-shutdown, and no instance is killed until its drain window has genuinely passed. The status-code distribution should show `[200]` for the entire run; if it doesn't, the drain window is shorter than your slowest real request, which is precisely this concept's central design mistake made concrete.

**Why this works, line by line.** The `sed` swap to `down;` plus `nginx -s reload` is half one — it changes nginx's *routing* decision without touching the backend process at all. The `sleep 5` is the honest admission that draining takes real wall-clock time proportional to your slowest legitimate request, not an instant switch. `kill -TERM` (not `-9`) is the entire difference from concept 4's exercise: it asks the process to shut down, giving `uvicorn` the chance to stop accepting new connections, finish what it has, run the `lifespan` shutdown code, and exit on its own terms within `--timeout-graceful-shutdown`. `wait "$PID"` blocks the script until that graceful exit genuinely completes, so the new instance never starts fighting the old one for the same port.

**Honesty caveat.** This script is a hand-rolled teaching version of what a real deployment platform (a Kubernetes rolling update, a managed load balancer's own deploy hooks) automates and hardens — real systems add health-check-gated readiness before re-admitting an instance (concept 4's readiness distinction), parallelism controls (`maxUnavailable`/`maxSurge` in Kubernetes' terms), and automatic rollback on failure. Treat this script as the mechanism made visible, not as production tooling to actually run as-is.

### System design — zero-downtime deployment behind a load balancer

**Problem** (the study plan's official system design ②). Design a deployment strategy for a stateless HTTP API behind a load balancer such that deploying new code causes zero visible downtime, supports a gradual rolling replacement of instances, and supports an instant rollback if the new version turns out to be broken.

**Requirements.** No failed requests during a routine deploy; the ability to detect a bad deploy quickly; the ability to revert within seconds, not minutes, once a problem is detected.

**Alternatives considered.**
1. **Rolling deployment**: replace instances one at a time (or in small batches), draining each before replacement, exactly as this concept's runnable example demonstrates.
2. **Blue/green deployment**: stand up an entirely new, parallel fleet running the new version, and switch all traffic to it at once (typically via the load balancer or DNS) once it's verified healthy, keeping the old fleet (now idle) ready as an instant rollback target.
3. **Canary deployment**: route a small percentage of real traffic (via concept 2's L7 header/cookie routing) to the new version alongside the old, watch its error rate and latency, and only proceed to a full rollout if it looks healthy.

**The decision: (1) as the default mechanism, composed with (3) for any change with genuine behavioral risk, with (2) reserved for changes too risky or too disruptive to instances to drain gracefully at all (a major version upgrade, a schema migration that can't run with two code versions live simultaneously).**

**The actual reason.** Rolling deployment is the cheapest of the three in infrastructure terms — it needs no second full fleet — and, combined with connection draining exactly as taught above, satisfies "no failed requests" directly. But rolling deployment alone answers "how do we replace instances without dropping requests," not "how do we know the new code is actually good before it's running everywhere" — that's a *detection* problem, not a *draining* problem, and canary deployment is the answer built specifically for it: routing a small, real slice of traffic to the new version first means a bad deploy is caught while it's only affecting a small fraction of users, rather than after a full rolling replacement has already put it everywhere.

**The trade-off, honestly.** Blue/green's "instant rollback" is genuinely instant (flip the load balancer back to the still-running old fleet) in a way rolling deployment's rollback is not — a rolling deployment, mid-rollout, that turns out to be bad requires rolling *forward* to the old version through the same drain-and-replace sequence it used to roll out, which takes real time proportional to fleet size, not an instant switch. Blue/green pays for that instant rollback by running two complete fleets simultaneously (double the compute cost) for the duration of the cutover, which is a real, sometimes prohibitive cost at scale. Canary deployment adds real operational complexity — someone (or some automated system) has to actually watch the canary's metrics and decide whether to proceed, which is a decision point that can itself be gotten wrong (watched for too short a time, or watching the wrong signal).

**Flip condition.** Reach for blue/green specifically when a rollback needs to be genuinely instant regardless of fleet size — a payments system, or any change severe enough that even the minutes a rolling rollback takes are unacceptable — and when the cost of briefly running two full fleets is smaller than the cost of that risk. Reach for canary specifically whenever a change's *correctness*, not just its deployability, is uncertain — a new algorithm, a significant behavior change, anything where "did we ship a bug" is a real open question rather than "did the deploy mechanics work." Plain rolling deployment, with no canary stage, is honestly sufficient for low-risk, well-tested changes where the deploy mechanism itself is the only thing that could go wrong.

**Failure modes.** A rolling deployment that re-admits each new instance into rotation the instant it starts, without gating on a readiness check (concept 4's exact distinction), lets under-warmed instances serve production traffic during every single deploy — the deploy-time instance of concept 4's slow-starting-dependency system design. A canary stage that's watched for too short a window will miss a regression that only shows up under load patterns the canary's small traffic slice didn't happen to include. And — this concept's own specific, load-bearing failure mode — a drain timeout shorter than the platform's actual longest legitimate request silently and routinely truncates exactly that class of request on every single deploy, with no error that points at the deploy process as the cause; it just looks like "streaming responses sometimes cut off," reported intermittently, disproportionately during deploy windows, and easy to miss the correlation on.

### System design — deploying a stateful streaming service without cutting off in-progress work

**Problem.** A backend serves long-lived streaming responses — Server-Sent Events carrying token-by-token LLM completions, where a single response can legitimately run for 60–120 seconds — behind the same load-balanced fleet as the rest of an API. Deploys must not truncate an in-progress completion mid-token.

**Requirements.** No mid-stream truncation of a legitimate, ongoing completion; deploys still need to complete in a reasonable, bounded total time; the platform's normal (short) requests shouldn't be forced to wait behind the platform's longest possible one.

**Alternatives considered.**
1. **One global drain timeout sized to the longest possible stream** (120+ seconds) applied uniformly to every instance and every request type.
2. **Route streaming and non-streaming traffic to separate backend pools**, each with its own drain timeout sized to its own actual traffic — short for the ordinary API pool, long for the streaming pool — using concept 2's L7 path-based routing to split them.

**The decision: (2).**

**The actual reason.** Option (1) technically satisfies "don't truncate a stream," but at a real, usually unacceptable cost: a rolling deploy across every instance now takes at least 120 seconds *per instance* regardless of whether that instance was actually serving anything long-running at the moment, because the drain timeout has to assume the worst case for *every* request type on *every* pool. A fleet of 50 instances, each rolled with a two-minute mandatory drain, turns a deploy that could otherwise take a couple of minutes into well over an hour. Splitting the pools means the ordinary API pool — the overwhelming majority of instances, in most real systems — keeps its short, fast drain timeout, and only the smaller streaming pool pays the long drain cost, which is proportional to the actual risk rather than applied blanket-wide.

**The trade-off, honestly.** Two pools mean two things to deploy, monitor, and scale independently — genuine operational surface area concept 2's L4/L7 system design already named as the cost of splitting tiers, showing up again one layer up. And the streaming pool's deploys are still, honestly, slow — this doesn't make a 120-second drain window fast, it only prevents that slowness from being paid by instances that never needed it.

**Flip condition.** Collapse back to one pool once streaming responses become short enough, or rare enough, that the worst-case drain penalty applied uniformly is genuinely tolerable — which is a real possibility if a product later moves to a design where long-running work is handed off to a background job and polled rather than held open on one live connection, at which point the whole "very long-lived HTTP connection" problem this scenario exists to solve disappears by architecture rather than by clever draining.

### Case studies

**GitHub's GLB, revisited for its draining behavior specifically.** Concept 2's case study introduced GLB Director's L4/L7 split; the piece relevant here is what happens on the *L7 side* during a deploy. GitHub's L7 tier runs HAProxy, and HAProxy has long supported a **seamless reload**: a new HAProxy process starts up, takes over the listening sockets (via a handed-off file descriptor or `SO_REUSEPORT`-style socket sharing — verify current mechanism against HAProxy's own documentation, as this has evolved across versions), and the *old* HAProxy process keeps serving every connection it already had open until each one finishes naturally, then exits — all without the L4 tier (GLB Director) needing to know a reload happened at all, because from GLB's point of view, the same backend addresses are still answering, just via a different underlying process. **The engineering lesson tied to this concept:** GitHub's L4/L7 split (concept 2) isn't just a performance optimization — it's also what makes L7 deploys cheap and frequent, because reloading or replacing the L7 tier's processes never requires touching the L4 tier's own routing state at all, which is exactly the independent-scaling-and-deploying benefit concept 2's system design predicted in the abstract, now shown concretely at the deploy-mechanics level. **Verify current** for the specific reload mechanism GLB's HAProxy layer uses today, as this is real engineering detail that evolves with HAProxy releases. *Primary source:* GitHub Engineering Blog's GLB posts (as cited in concept 2); HAProxy's own documentation on seamless reloads/socket transfer.

**Kubernetes rolling updates, readiness probes, and `preStop` — the industry-standard automated version of this concept's runnable example.** A Kubernetes `Deployment`'s rolling update strategy, combined with **readiness probes** (concept 4's exact distinction) and a **`preStop` lifecycle hook** plus **`terminationGracePeriodSeconds`**, automates precisely the two-halves mechanism this concept's "under the hood" section described by hand: when a pod is scheduled for termination, Kubernetes first removes it from the Service's endpoint list (stopping new traffic — half one), runs the `preStop` hook (giving the application a chance to finish gracefully — often just a `sleep` long enough for in-flight load-balancer state to catch up, plus whatever application cleanup is needed), and only then sends `SIGTERM`, waiting up to `terminationGracePeriodSeconds` before escalating to `SIGKILL` if the process hasn't exited on its own — a hard backstop that is, deliberately, this concept's `kill -9` from concept 4, kept as a last resort rather than the primary mechanism. **The engineering lesson tied to this concept:** this note's rolling-deploy script is a hand-built miniature of exactly this pipeline, and knowing the manual version cold is what makes the automated version's configuration knobs (`terminationGracePeriodSeconds`, `preStop`, readiness probe timing) legible instead of magic. *Primary source:* Kubernetes documentation, "Pod Lifecycle" and "Deployments" — **verify current**, as default values and the exact hook/probe interaction have been refined across Kubernetes releases.

### In production

**Best practices.**
1. **Size every drain timeout from a measurement of your actual slowest legitimate request, not a platform default** — the default (often 30 seconds) is a guess about a generic web application, not a fact about yours, and streaming/long-running endpoints routinely violate it.
2. **Gate re-admission to rotation on a real readiness check, not just "the process started,"** per concept 4 — a rolling deploy that races ahead of instance warm-up recreates concept 4's slow-start system design on every single release.
3. **Keep `SIGKILL` as a backstop timeout, not the primary mechanism** — a graceful shutdown that never completes (a stuck cleanup handler, a hung connection) still needs a hard cutoff eventually, but that cutoff should be rare, not routine.

**Mistakes, beginner → senior.**
- *Beginner:* deploying with a plain process restart and no draining at all, and being confused why users occasionally see connection-reset errors during every release.
- *Intermediate:* configuring a drain timeout copied from a tutorial or a platform default without ever measuring the service's own actual request-duration distribution.
- *Senior:* running a correctly-drained rolling deployment for an API that also happens to serve a handful of long-lived streaming endpoints, having sized the drain timeout for the *typical* request and never noticed the tail this concept's second system design exists to address, until a customer complains their answer "just stopped."

**Observability.** In-flight request count and its age distribution *at the moment a drain begins*, per instance — this is the number that tells you whether your drain timeout is actually long enough, directly, instead of inferring it from truncated-response complaints after the fact.

**Scaling and cost.** Rolling deployment's cost is almost entirely time, not infrastructure — a deploy across N instances with a fixed per-instance drain window takes time roughly proportional to N times that window (unless done in parallel batches, which trades deploy speed against how many instances are unavailable to the pool at once). Blue/green's cost is explicitly double the compute for the cutover window, a real budget line item worth naming rather than treating as free "safety."

---

## 7. The canonical trace: what happens when you type a URL

**Depth: [CORE]**

### Intuition

Every concept in this note, and every concept in Days 9 through 15, has been studied in isolation — one layer, one mechanism, one failure mode at a time, because that's the only way to actually understand any of them deeply. But no real request ever experiences the stack that way. A single click fires all of it, in order, in milliseconds, each layer handing off to the next without pausing to let you inspect it. This concept does something none of the others do: it adds no new mechanism at all. It takes every mechanism you already own — from Day 9's frames to this note's connection draining — and runs them once, back to back, on one concrete request, so you can stand at any point in that chain and say precisely which layer you're in, which day taught it, and what would have to be true for that layer to be the reason something is slow or broken.

This is also, deliberately, the interview question this whole phase has been building toward — "walk me through what happens when you type a URL into a browser" is one of the most common systems-design questions asked, precisely because a complete, correct answer requires exactly the breadth this phase spent ten days building, and a shallow answer ("it looks it up and gets the page") reveals instantly that the breadth isn't there.

### The full trace, as one diagram

```
 BROWSER                                                                    ORIGIN
    │                                                                          │
    │ 1. Parse the URL: scheme=https, host=shop.example.com, path=/cart       │
    │                                                                          │
    │ 2. DNS RESOLUTION (Day 12)                                              │
    │    browser cache → OS cache → recursive resolver →                      │
    │    [cold: root → TLD → authoritative, ~4-5 round trips]                 │
    │    → returns the CDN edge's anycast IP (Day 15 + Day 10's anycast)      │
    │                                                                          │
    │ 3. TCP HANDSHAKE (Day 11) — client ↔ nearest CDN edge, 1 RTT            │
    │                                                                          │
    │ 4. TLS HANDSHAKE (Day 14) — client ↔ edge, 1 RTT (TLS 1.3) or 0-RTT     │
    │    on session resumption; ALPN negotiates h2/h3 here                    │
    │                                                                          │
    │ 5. HTTP REQUEST (Day 13) — GET /cart, headers, cookies, sent over        │
    │    the now-encrypted, possibly multiplexed connection                   │
    │                                                                          │
    │───────────────────────────► CDN EDGE POP (Day 15) ─────────────────────►│
    │                              cache check: Cache-Control / Vary / ETag   │
    │                              STATIC ASSET, cacheable → HIT              │
    │                              → served straight from the edge.           │
    │                              LOAD BALANCER AND APP NEVER SEE THIS       │
    │                              REQUEST AT ALL. Return to browser now.     │
    │                                                                         │
    │                              /cart is personalized → Cache-Control:    │
    │                              private → MISS/bypass → relay to origin   │
    │                              over an ALREADY-WARM edge↔origin pool     │
    │                              (Day 15's persistent-connection pattern — │
    │                              no new TCP/TLS handshake per user request)│
    │                                          │                              │
    │                                          ▼                              │
    │                      6. ARRIVES AT ORIGIN'S ENTRY POINT                 │
    │                         DNS/anycast (Day 12 + Day 10) already routed    │
    │                         the ORIGIN-bound traffic to the nearest region  │
    │                                          │                              │
    │                      7. L4 TIER (concept 2) — hashes the 4-tuple,       │
    │                         forwards to an L7 box, no HTTP parsing here     │
    │                                          │                              │
    │                      8. L7 TIER (concept 2) — TLS terminates here if    │
    │                         edge↔origin is itself encrypted (Day 14);       │
    │                         parses the HTTP request; routes /cart → the    │
    │                         cart-service pool by PATH (concept 2); picks a  │
    │                         specific backend via LEAST CONNECTIONS          │
    │                         (concept 3), filtered to backends that pass     │
    │                         HEALTH CHECKS (concept 4)                       │
    │                                          │                              │
    │                      9. APP — reads the cart from Redis, not from any   │
    │                         one backend's memory (concept 5's stateless +   │
    │                         externalized state, exactly as taught); builds  │
    │                         the response                                    │
    │◄──────────────────────────────────────── returns back up the SAME      │
    │                                            chain: app → L7 → CDN edge   │
    │                                            (evaluates the RESPONSE's    │
    │                                            Cache-Control — private,     │
    │                                            passes through uncached) →   │
    │                                            back to the client           │
    │                                                                          │
    │ 10. RENDER — parse HTML → build DOM; discover CSS/JS/image references,  │
    │     each possibly on a DIFFERENT hostname, each spawning its OWN        │
    │     instance of steps 2-9 (in parallel, and reusing an existing         │
    │     connection via HTTP/2 multiplexing, Day 14, wherever the hostname   │
    │     matches one already open) → CSSOM + DOM → render tree → layout →    │
    │     paint → pixels                                                      │
```

### Worked example — the same request, with real numbers, cold versus warm

**Pass one: a first-ever visit — cold DNS, cold TLS session, but a CDN with an already-warm connection pool to the origin (the normal steady state Day 15 already established for CDN infrastructure).**

```
Step                                              Time (Day, concept)              Running total
──────────────────────────────────────────────────────────────────────────────────────────────
1. DNS resolution, cold walk                       ~128 ms  (Day 12)                  128 ms
2. TCP handshake, client ↔ edge (1 RTT @ 18ms)      18 ms  (Day 11)                   146 ms
3. TLS 1.3 handshake, client ↔ edge (1 RTT)          18 ms  (Day 14)                   164 ms
4. HTTP request sent; edge checks cache (Day 15)     <1 ms  (Day 13, Day 15) — MISS    164 ms
5. Edge → origin relay, pre-warmed pool (Day 15)      9 ms  (one-way, no new handshake) 173 ms
6. Origin: L4 hash → L7 parse+route → backend pick  <1 ms  (concepts 2, 3, 4)          174 ms
7. App: read cart from Redis, build response          22 ms  (concept 5)                196 ms
8. Origin → edge → client, response relay              9 ms  (Day 15, Day 13)          205 ms
──────────────────────────────────────────────────────────────────────────────────────────────
TIME TO FIRST BYTE ≈ 205 ms
```

**Pass two: a second request on the same page load — the browser's own TCP+TLS connection to the edge is already open and reused, and DNS is served from the browser's own cache.**

```
Step                                              Time                              Running total
────────────────────────────────────────────────────────────────────────────────────────────
1. DNS resolution — served from browser cache        ~0 ms                             0 ms
2. TCP handshake — connection REUSED, no new one       0 ms                             0 ms
3. TLS handshake — connection REUSED, no new one       0 ms                             0 ms
4. HTTP request sent on the existing connection; edge checks cache — MISS (private)    0 ms
5. Edge → origin relay, pre-warmed pool                9 ms                             9 ms
6. Origin: L4 → L7 → backend pick                     <1 ms                            10 ms
7. App: read cart, build response                      22 ms                            32 ms
8. Origin → edge → client                               9 ms                            41 ms
────────────────────────────────────────────────────────────────────────────────────────────
TIME TO FIRST BYTE ≈ 41 ms
```

**The ratio — roughly 5×, from 205 ms to 41 ms — for the exact same logical request, purely from reusing connections and a warm DNS cache**, is this whole phase's central lesson arriving in one final number: Day 12's TTL, Day 11's connection reuse, and Day 15's persistent edge-to-origin pools are all, independently, answers to the same question — *how do you avoid paying a round trip's cost more than once for information that hasn't changed* — and a real page load pays this cost dozens of times over for dozens of resources, which is exactly why steps 1–4 being skippable on repeat contact matters so much more in aggregate than any single 164 ms number suggests.

**And the branch that skips almost everything.** If step 4's cache check had been a `GET /static/logo.png` with `Cache-Control: public, max-age=31536000` and the edge already had a cached copy (Day 15), the entire right-hand side of the diagram — steps 6 through 9, the whole load-balancer and application stack this note spent six concepts building — **never executes at all.** The response comes straight back from the edge PoP. This is worth stating as a design fact, not a footnote: concept 2's "design the LB stack for 1M req/s" system design implicitly assumes a well-configured CDN has already absorbed the overwhelming majority of a real service's request volume — the static assets, the cacheable API responses — before anything reaches L4, and the load balancer tier is sized for the genuinely dynamic remainder, not for the site's total traffic. A service that serves everything, including static assets, directly from its LB/app tier is paying for capacity a CDN would have given it far more cheaply.

### Under the hood — decomposing a real request into its hops, from the client's own vantage point

Everything above was built from first principles; here is how to *measure* it on a real request, from a plain client, with a tool that already ships everywhere. `curl`'s `-w` (write-out) option exposes exactly the boundaries this trace draws, as cumulative timers:

```bash
curl -o /dev/null -s -w \
  "dns:   %{time_namelookup}s\ntcp:   %{time_connect}s\ntls:   %{time_appconnect}s\nttfb:  %{time_starttransfer}s\ntotal: %{time_total}s\n" \
  https://shop.example.com/cart
```

```
dns:   0.128s
tcp:   0.146s
tls:   0.164s
ttfb:  0.205s
total: 0.207s
```

**These numbers are cumulative from the start of the request, not per-phase costs** — the per-phase cost is the *difference* between consecutive lines, which is exactly how to read them: `time_namelookup` (0.128s) is DNS alone (Day 12); `time_connect − time_namelookup` (0.146 − 0.128 = 18 ms) is the TCP handshake alone (Day 11); `time_appconnect − time_connect` (0.164 − 0.146 = 18 ms) is the TLS handshake alone (Day 14); `time_starttransfer − time_appconnect` (0.205 − 0.164 = 41 ms) is everything from "request sent" to "first byte of the response" — which, from the client's vantage point, is a single opaque number bundling the CDN cache check, the edge-to-origin relay, the entire L4/L7/algorithm/health-check/app chain this note just spent six concepts building, and the return trip, because a plain client has no visibility *inside* that black box. Getting visibility inside that 41 ms bundle is exactly what distributed tracing (a trace ID generated at the edge and propagated through every hop, correlating timing at the LB, at the app, and at any downstream call) is for — named here, not opened, as real infrastructure this note treats as a black box beyond knowing it exists and why: a client-side timer alone cannot see past the edge, and anything past that boundary needs cooperation from every hop in between to actually instrument. **Verify current** field names and behavior against `curl`'s own manual, as they've been stable for a long time but a `curl` version bump is exactly the kind of thing worth a quick check before quoting in a runbook.

### System design — minimizing global TTFB for a read-mostly API

**Problem.** A read-mostly API (a product catalog, a content feed) serving a global user base needs a p95 time-to-first-byte under 150 ms, worldwide.

**Requirements.** p95 TTFB < 150 ms globally; acceptable eventual consistency for reads; writes can tolerate somewhat higher latency.

**Alternatives considered.**
1. **CDN-first**: cache as much of the read traffic at the edge as correctness allows (Day 15), with a single origin region handling only genuine cache misses and all writes, fronted by this note's L4/L7 stack.
2. **Multi-region active-active origin**: run the full application and its data store in multiple regions (Day 12/Day 10's anycast and GeoDNS material), so that even *uncacheable* requests never have to cross an ocean.

**The decision: (1) first, escalating to (2) only for the fraction of traffic that (1) provably cannot help.**

**The actual reason.** This concept's own worked example just showed the size of the win available from (1) alone: pushing a request that *can* be cached down from a 205 ms origin round trip to a single-digit-millisecond edge response is a bigger, cheaper win than anything achievable by moving the origin itself closer, and it costs nothing beyond correct cache-control headers (Day 15) plus the L4/L7 stack this note already teaches for the uncacheable remainder. Multi-region active-active origin is real, valuable infrastructure, but it solves a different problem — reducing the round-trip cost of requests that genuinely *cannot* be served from a cache at all (personalized, must-be-fresh, or write-heavy) — and paying its full architectural cost (data replication strategy, consistency model, the exact concerns this phase's earlier system designs already raised) before confirming the CDN-first approach has been exhausted is solving the expensive problem before the cheap one.

**The trade-off, honestly.** CDN-first caps out at whatever fraction of traffic is genuinely cacheable — a product with heavily personalized responses on every request gets a much smaller win from (1) than a mostly-static catalog does, and no amount of cache tuning changes that ceiling. Multi-region active-active removes that ceiling but reintroduces every multi-region consistency concern this phase has already surfaced.

**Flip condition.** Escalate to (2) specifically once measurement shows the *uncacheable* fraction of traffic is both large and latency-sensitive enough that (1)'s ceiling is the actual bottleneck against the 150 ms target — not before, and not as a default assumption "big companies run multi-region so we should too."

### Case studies — this trace's own hop failures, already taught

This concept's honest case-study material is not new — it's every incident this phase has already covered, correctly re-framed as **which hop of this exact trace broke**, and repeating any of them in full here would violate this note's own no-repetition rule. Naming the mapping explicitly is the actual lesson: **Meta's October 2021 outage and the Dyn 2016 DDoS attack (both Day 12) are failures of step 2** — DNS resolution — where the trace never gets past the very first box, and everything downstream, however healthy, is unreachable. **Cloudflare's July 2019 outage (this note's concept 1) is a failure inside step 5** — the CDN/reverse-proxy tier itself, where a WAF rule's runaway CPU cost turned the "cache check" box into the point of failure for traffic that should have sailed through it. **Slack's January 2021 outage (this note's concept 4) is a failure spanning steps 6–8** — the load-balancer and health-check layer, where a correct mechanism removing unhealthy capacity compounded a capacity crisis instead of containing it. Standing at any point in this trace and being able to say "if this broke, which prior day's incident is the closest real-world analogue" is a genuine test of whether the whole phase's material actually connected into one model, rather than staying filed as ten separate topics.

### In production

**Best practices.**
1. **Instrument every hop with a shared trace identifier**, generated as early as possible (ideally at the edge) and propagated through the LB, the app, and any downstream calls, so a slow request can be decomposed the way the `curl -w` example decomposed it client-side — but for the parts a client can never see.
2. **Measure from real users (RUM), not just synthetic checks** — a synthetic probe from one well-connected location will never see the DNS-cold, far-from-any-edge, congested-network experience a real user on a real mobile connection has, and that tail is exactly where this trace's cold-path numbers matter most.
3. **Know your own cacheable fraction** — the single most useful number for reasoning about this concept's first system design, and one most teams have never actually measured.

**Mistakes, beginner → senior.**
- *Beginner:* treating "the site is slow" as one undifferentiated problem instead of bisecting it hop by hop the way this concept's `curl -w` example demonstrates.
- *Intermediate:* optimizing app-tier code (step 7) aggressively while a cold DNS path or an unnecessarily large `time_appconnect` (a missed TLS session-resumption opportunity, Day 14) is actually the bigger cost on the p95 that matters.
- *Senior:* building expensive multi-region active-active origin infrastructure (this concept's second system design) before confirming, with real cacheable-fraction measurements, that CDN-level caching hasn't already solved most of the problem far more cheaply.

**Observability.** A single trace-ID-correlated view spanning DNS resolution time (from RUM, since a client-side library is one of the only vantage points that can see it), edge cache hit rate, LB-to-app latency, and app processing time — assembled as one waterfall per request class, not as five separate dashboards someone has to mentally stitch together during an incident.

**Scaling and cost.** Every hop in this trace has a different cost profile — DNS and CDN edge responses are cheap and get cheaper with scale (more cache hits), while the LB/app/database tail this note spent six concepts on is where genuine compute cost concentrates — which is precisely why the first system design's "CDN-first" ordering isn't just a latency argument, it's also the cheaper one, and the two arguments point the same direction for the same underlying reason: the CDN absorbs load before it becomes someone's compute bill.

**The trace, run against your own tools.** An agent's own call to an LLM API — `https://api.anthropic.com/v1/messages` or any provider's equivalent — runs this exact same trace, hop for hop: DNS resolves the provider's API hostname, a TCP and TLS handshake establish a secure channel, the HTTP request carries your prompt, and on the provider's side, the exact stack this note just taught — a load balancer, health-checked backends, and (for a streaming completion) a connection whose lifetime this note's concept 6 already flagged as the one that breaks naive drain-timeout defaults — serves the response back. Nothing about calling an LLM API is architecturally special; it is this trace, with a different payload.

---

## Interview questions and practice

### Conceptual

1. **What's the mechanical difference between a forward proxy and a reverse proxy, and why is it nearly impossible to tell them apart from a packet capture alone?** *(There isn't one at the byte level — both terminate one connection and originate another. The difference is administrative: whose agent it is and which side it hides.)*
2. **Why can't an L4 load balancer route by URL path?** *(It never decrypts or parses HTTP — it reads only IP/port headers, so a path, a Host header, or a cookie simply doesn't exist yet from its point of view.)*
3. **Why does round robin keep sending a degraded backend its full one-third share of traffic, and how does least connections avoid that?** *(Round robin's rule is "next in the list," with no live-load input at all; least connections routes by current in-flight count, which rises automatically on a backend that isn't keeping up.)*
4. **What's the real cost difference between an active and a passive health check?** *(Active costs continuous background probe traffic but protects every real request; passive costs nothing when healthy but sacrifices up to `max_fails` real requests to detect a failure.)*
5. **Why is a health check that queries a shared primary database more dangerous than no health check at all?** *(A hiccup in that one shared dependency fails every backend's check simultaneously, turning a partial degradation into a total, self-inflicted outage.)*
6. **Why can't a crashed backend's in-memory session ever be "recovered"?** *(Process memory doesn't pause on a crash — it's deallocated by the OS. There's no state to return to, unlike a human coming back from a break.)*
7. **What are the two separate halves of a graceful shutdown, and what breaks if you skip one?** *(The LB stops routing new traffic here; the app itself finishes in-flight work on SIGTERM. Skip the first and new requests still arrive mid-shutdown; skip the second and in-flight requests get cut off regardless of what the LB did.)*
8. **Why does naive IP hash remap ~75% of clients when a fourth backend joins a pool of three, and what does consistent hashing do differently?** *(`hash % N` changes almost every output when `N` changes; a hash ring only reassigns the small arc adjacent to the new/removed node.)*
9. **Where does TLS terminate in a typical L4/L7 stack, and why can't it terminate at L4?** *(At L7 — L4 never reads far enough into the packet to need a certificate at all, and terminating there would mean L4 doing real per-connection crypto work, defeating the point of keeping it fast and near-stateless.)*
10. **What's the one number that decides between CDN-first and multi-region-origin for global TTFB?** *(The fraction of traffic that's genuinely uncacheable — CDN-first caps out there; multi-region origin is what removes that ceiling.)*

### Diagnostic scenarios

11. **A load-balanced API returns a handful of 502s, on a clock-like five-minute schedule.** *(Passive health check with `fail_timeout` set to that interval — the backend is dead, gets re-probed with live traffic every `fail_timeout` seconds, and each probe fails.)*
12. **A rolling deploy causes intermittent cut-off streaming responses, but only for large exports.** *(Drain timeout shorter than the export's actual duration — sized for the typical request, not the tail.)*
13. **`curl -w` shows high `time_connect` and `time_appconnect` but fast `time_namelookup`.** *(TCP and TLS are slow, DNS is fine — check Day 11's handshake path and Day 14's TLS round-trip count/session-resumption configuration, not DNS.)*
14. **A newly deployed instance gets flooded with full production traffic immediately and is slow for its first 20 seconds, every time.** *(Missing readiness gating — it's being re-admitted to rotation on "process started," not "actually warmed up.")*
15. **An L7 path-based routing rule for `/api/*` silently does nothing.** *(Check whether TLS is actually terminating where you think it is — a rule attached to an L4/passthrough resource never sees decrypted HTTP to route on.)*

### Design questions

16. Design the full load-balancing stack for 1M req/s, stating where DNS/anycast, L4, L7, and TLS termination each sit. *(DNS/anycast → regional entry; L4 → fast 4-tuple hash to L7 boxes; L7 → TLS termination + path routing + algorithm + health checks; app tier stateless behind it.)*
17. Design a zero-downtime deployment strategy for a service with both ordinary short requests and long-lived streaming responses. *(Split pools with separate drain timeouts, per concept 6's second system design — don't force every instance to pay the streaming pool's drain cost.)*
18. You inherit a legacy app with in-memory sessions and three weeks to scale it. What ships now, and what's scheduled immediately after? *(Sticky sessions as an explicitly dated stopgap; externalized state migration on a calendared deadline, not "eventually.")*
19. Trace `https://shop.example.com/cart` end to end, citing which day or concept taught each hop. *(DNS – Day 12; TCP – Day 11; TLS – Day 14; HTTP – Day 13; CDN cache check – Day 15; L4/L7/algorithm/health checks – concepts 2–4; session state – concept 5; render – browser-internals territory, named and stopped.)*

---

# Topic-wide wrap-up

## Glossary

**Active health check** — a probe the load balancer sends to a backend on its own fixed schedule, independent of real traffic, so a failure is detected without costing a real user's request; unavailable in open-source Nginx, present in HAProxy and Caddy's open-source builds.

**Blue/green deployment** — running a complete second fleet on the new version alongside the old one and switching all traffic at once, keeping the old fleet as an instant rollback target at the cost of double the compute during the cutover.

**Canary deployment** — routing a small, real slice of traffic to a new version first, using L7 header/cookie routing, to detect a regression before a full rollout.

**Connection draining** — stopping new traffic to an instance while letting its in-flight work finish before terminating it; the mechanism that makes a deploy or removal zero(-ish)-downtime.

**Consistent hashing** — a hashing scheme, placed conceptually on a ring, in which adding or removing a backend remaps only the minimal necessary fraction of existing key assignments, unlike naive `hash % N` hashing which remaps nearly all of them.

**Direct Server Return (DSR)** — an L4 load-balancing technique in which the backend's response traffic goes straight to the client, bypassing the load balancer entirely on the way out, so the LB's bandwidth requirement is dominated by (typically smaller) request traffic rather than response traffic.

**ECMP (Equal-Cost Multi-Path)** — a network routing technique that lets many machines advertise the same reachability so inbound traffic can land on any of them, used to distribute traffic across an L4 load-balancing tier before any single machine sees the whole load.

**Ejection** — the act of a load balancer removing a backend from the pool it routes new traffic to, following a failed health check.

**Externalized state** — session or application state moved out of any single backend's process memory into a shared store (or the client) so any backend can serve any request without losing continuity if one instance dies or is removed.

**Forward proxy** — a proxy acting as the client's agent, facing outward from a group of clients toward the wider internet; the destination server sees the proxy, not the original client.

**Graceful shutdown** — an application's response to `SIGTERM`: stop accepting new work, finish in-flight work up to a grace period, run cleanup code, then exit — as opposed to `SIGKILL`'s immediate, no-warning termination.

**Hash ring** — the conceptual structure underlying consistent hashing, in which backends and keys are placed by hash value around a circle and a key is assigned to the next backend clockwise from it.

**IP hash** — a load-balancing algorithm that hashes a stable client identifier (typically the IP) to deterministically send the same client to the same backend, trading load-awareness for a cheap, shared-nothing form of affinity.

**L4 load balancing** — load balancing that reads only IP and TCP/UDP port headers, never the application payload; fast and near-stateless, blind to HTTP content, and unable to terminate TLS meaningfully.

**L7 load balancing** — load balancing that terminates the connection fully and parses the application-layer request (HTTP method, path, headers, cookies), enabling content-based routing at the cost of real per-connection CPU and memory, and requiring TLS to terminate at or before it.

**Least connections** — a load-balancing algorithm that routes each new request to whichever backend currently has the fewest in-flight requests, reacting to real-time load rather than a fixed schedule or ratio.

**Passive health check** — detecting a backend failure by watching real traffic's outcomes (a count of failures within a time window) rather than sending dedicated probes; costs nothing when healthy, at the price of sacrificing a small, bounded number of real requests to detect a failure.

**preStop hook** — a Kubernetes lifecycle hook run before a pod's main process receives `SIGTERM`, typically used to pause briefly so in-flight load-balancer state catches up before shutdown begins in earnest.

**Readiness probe** — a Kubernetes-style health check gating whether a newly-started instance is added to a service's traffic rotation at all, kept distinct from a stricter, continuously-running liveness check so that legitimate startup slowness isn't mistaken for an ongoing failure.

**Rendezvous hashing (Highest Random Weight, HRW)** — a hashing scheme, closely related to consistent hashing, in which each candidate backend's suitability for a given key is scored independently and the highest score wins; used by GitHub's GLB Director to pick backends without coordination between director machines.

**Reverse proxy** — a proxy acting as the server's agent, facing inward from a group of clients toward a group of backend machines; the client sees the proxy (as "the server"), not the actual backend that handled the request.

**Rolling deployment** — replacing backend instances with a new version one at a time (or in small batches), draining each before replacement, so the fleet as a whole never goes fully offline.

**Round robin** — the simplest load-balancing algorithm: cycle through the backend list in fixed order, with no information about current load or backend capacity.

**Session affinity (sticky session)** — pinning a specific client to a specific backend for the life of a session, typically via a cookie or a hash of the client's address, so that backend's in-memory state stays relevant to that client — at the cost of stranding that state entirely if the backend is removed or crashes.

**Session replication** — copying every session's data to every backend instance as it changes, an alternative to both sticky sessions and a shared external store, common in older application-server clustering and generally more implementation effort than either alternative.

**Stateless service** — a backend design in which no request's correctness depends on any state held only in that specific process's memory, allowing any instance to serve any request.

**terminationGracePeriodSeconds** — a Kubernetes setting bounding how long a pod is given to exit gracefully after `SIGTERM` before Kubernetes escalates to `SIGKILL`.

**Time to First Byte (TTFB)** — the elapsed time from initiating a request to receiving the first byte of the response; the metric this note's canonical trace decomposes hop by hop.

**Transparent proxy** — a forward proxy that intercepts traffic at the network layer via policy-based routing, requiring no client-side configuration.

**WAF (Web Application Firewall)** — a set of request-inspection rules (often regex- or pattern-based) applied at a reverse proxy or CDN edge to block malicious requests before they reach an application; a source of real per-request CPU cost, as the Cloudflare case study demonstrates.

**Weighted round robin** — round robin extended with a static capacity weight per backend, distributing requests proportionally to reflect permanent hardware or capacity differences, without reacting to live load.

**X-Forwarded-For** — a header a reverse proxy adds (or appends to) recording the original client's IP address, since the backend's own connection peer is the proxy, not the client; trustworthy only from a proxy the backend explicitly configures as trusted.

**X-Forwarded-Proto** — a header a reverse proxy adds recording the original request's scheme (`http`/`https`), since a backend behind a TLS-terminating proxy has no other way to know TLS was involved at all.

**X-Real-IP** — a simpler, single-value alternative to `X-Forwarded-For` carrying just the immediate client's IP address, added by a reverse proxy.

---

## Cheat sheet

**The two proxy types, by who they hide**
```
FORWARD PROXY: hides the CLIENT from the server it's talking to.
REVERSE PROXY: hides the SERVER (the real backend) from the client.
Mechanically almost identical; the difference is purely who configured it.
```

**L4 vs. L7, at a glance**
| | L4 | L7 |
|---|---|---|
| Reads | IP + TCP/UDP ports only | Full HTTP request |
| Can route by | nothing content-based | path, header, cookie, method |
| TLS | never terminates here | terminates HERE |
| Speed | line-rate, kernel/NIC-capable | real per-connection CPU/memory cost |
| Failure blindness | total — can't see app-level errors | can inspect status codes, body |

**Algorithms, in one line each**
| Algorithm | Reacts to | Blind to |
|---|---|---|
| Round robin | nothing | everything — load, capacity, health |
| Weighted round robin | static capacity | live/current load |
| Least connections | live in-flight count | *why* a backend is slow |
| IP hash | nothing (deterministic by key) | load — trades it away for affinity |
| Consistent hashing | pool membership changes | still blind to live load |

**Health checks, the honest trade-off**
```
ACTIVE:  costs continuous background probe traffic.
         protects EVERY real request — none are sacrificed to detect a failure.
         Nginx OSS: NOT AVAILABLE. Use HAProxy, Caddy, or pay for NGINX Plus.

PASSIVE: costs nothing while healthy.
         sacrifices up to max_fails real requests to detect a failure.
         Nginx OSS: the ONLY option (max_fails / fail_timeout).
         Recovery is automatic: the NEXT real request after fail_timeout
         IS the retry probe — no separate re-registration step.
```

**Sticky sessions vs. stateless + externalized state**
```
STICKY:      zero code change, but a crash or scale-down LOSES that
             backend's sessions permanently. Load can go permanently uneven.
STATELESS:   any backend serves any request; a crash loses nothing;
             scaling is free — but the shared store is now critical
             infrastructure and every access costs a network round trip.
DEFAULT:     stateless + externalized state wins almost every time
             (12-Factor, Kubernetes' own default). Sticky = time-boxed
             stopgap only, with a calendared migration date.
```

**Connection draining, the two halves**
```
HALF 1 (the LB):   stop routing NEW traffic here (mark "down" + reload).
HALF 2 (the app):  SIGTERM → finish in-flight work → lifespan shutdown code
                   → exit, within a grace period.
Drain window MUST be >= your slowest legitimate request. A 30 s default
will truncate a 90 s streaming LLM completion, every single deploy.
```

**The canonical trace, by day/concept**
```
1. URL parsed
2. DNS resolution ................... Day 12
3. TCP handshake ..................... Day 11
4. TLS handshake ..................... Day 14
5. HTTP request ....................... Day 13
6. CDN cache check (HIT short-circuits everything below) ... Day 15
7. L4 tier — 4-tuple hash ............ concept 2, 3
8. L7 tier — TLS termination, path routing, algorithm, health checks
                                       ... concept 2, 3, 4
9. App — stateless, externalized session state ... concept 5
10. Response returns, CDN re-evaluates cacheability ... Day 15
11. Render: DOM/CSSOM/paint (browser-internals territory, named & stopped)
```

**Diagnosing a slow request from the client side**
```
curl -o /dev/null -s -w "dns:%{time_namelookup}s tcp:%{time_connect}s \
tls:%{time_appconnect}s ttfb:%{time_starttransfer}s total:%{time_total}s\n" URL

DNS slow?          → Day 12
TCP slow?           → Day 11 (network path, congestion)
TLS slow?            → Day 14 (round trips, no session resumption)
Big ttfb gap?         → everything past the edge: CDN/LB/app — needs
                        distributed tracing to see further, a client
                        alone can't look inside that box.
```

---

## Build this

### Task 1 — three instances, one load balancer, one abrupt kill (the study plan's core build)

- [ ] Inside WSL2, run three FastAPI instances (concept 4's `app.py`) on ports 8001–8003, each with a distinct `INSTANCE_ID`.
- [ ] Configure Nginx as a reverse proxy/load balancer in front of them, using `least_conn` and passive health checks (`max_fails`/`fail_timeout`).
- [ ] Load-test with `hey` (or `wrk`/`locust` if you prefer) for at least 30 seconds.
- [ ] Mid-test, `kill -9` one instance by port (`kill -9 $(lsof -ti:8002)`) and capture the status-code distribution and latency percentiles before, during, and after.
- [ ] Identify, in your own captured output, the exact handful of requests that failed, and explain why `proxy_next_upstream` did or didn't save each one.
- [ ] Repeat the same kill against the Caddy config from concept 4 (with `health_uri`/`health_interval` active checks) and diff the two runs' failure counts.

**Definition of done:** you can point at your own `hey` output and explain, request by request, which ones failed and why — not just that "it recovered."

### Task 2 — a real rolling deploy with graceful shutdown

- [ ] Add the `lifespan` shutdown handler to `app.py` (concept 6).
- [ ] Run the `rolling_deploy.sh` script from concept 6 while `hey` is running continuously against the fleet.
- [ ] Confirm zero client-visible failures, and explain in your own words why this run's failure count differs from Task 1's.
- [ ] Deliberately shrink the drain window below your `/work` endpoint's sleep duration and re-run — confirm you can now reproduce truncated/failed requests on demand, and explain exactly why.

**Definition of done:** you can demonstrate both a clean rolling deploy and a deliberately broken one, on command, and explain the one-line config difference between them.

### Task 3 — decompose a real request with `curl -w`

- [ ] Run the `curl -w` command from concept 7 against a real HTTPS site of your choice, cold (flush your OS DNS cache first) and again warm (same connection or shortly after).
- [ ] Compute each phase's actual cost (not the cumulative numbers) and attribute each to the day that taught it.
- [ ] Run it against a known static-asset URL (e.g., an image on a CDN-fronted site) and a known dynamic/personalized endpoint, and compare `time_starttransfer` between them.

**Definition of done:** you can produce a per-phase breakdown table from your own real output, matching this note's ledger format.

### Task 4 — write the URL-to-pixels trace from memory

- [ ] Close this note. On a blank page, write out every step from typing a URL to pixels appearing, citing which day or concept taught each one, without looking anything up.
- [ ] Check your answer against concept 7's diagram. For anything you missed or got wrong, re-read that specific concept — not the whole note.
- [ ] Explain, out loud, to someone else (or to yourself, recorded), the single branch where the entire load-balancer/app stack is skipped — and why it matters for capacity planning.

**Definition of done:** a from-memory trace that correctly names all seven of Days 9–15 plus this note's own concepts, in the right order, with the CDN-hit shortcut included.

---

## Active recall and self-test

Answer from memory, in writing, before checking.

1. Why is a forward proxy and a reverse proxy nearly indistinguishable at the packet level, and what's the actual difference?
2. Name one thing an L7 load balancer can do that an L4 load balancer structurally cannot, and explain why.
3. Do the arithmetic: three backends, round robin, one degrades to 10× its normal latency. What's its utilization, and what happens to the other two?
4. What's the honest cost of a passive health check, stated as a number of real requests, not a vague phrase?
5. Why does open-source Nginx not support active health checks, and what are the three real alternatives if you need them?
6. Why does ejecting a backend from a load balancer's pool do nothing to a connection an agent already has pooled to it?
7. Give the three failure scenarios sticky sessions share, and why externalized state fixes all three at once.
8. What are the two separate halves of a graceful shutdown, in order?
9. Why does a 30-second default drain timeout silently break a streaming LLM completion?
10. Compute the naive-IP-hash remapping fraction when a pool goes from 4 backends to 5. What would consistent hashing's remapping fraction be instead?
11. Name the two official case studies this day covers, and which hop of the canonical trace each one broke.
12. In the canonical trace, which step disappears entirely on a CDN cache hit, and why does that matter for capacity planning?
13. What do `time_connect` and `time_appconnect` isolate, in `curl -w` output, and which two days does each map to?
14. Give the flip condition for choosing blue/green deployment over rolling deployment.
15. Give the flip condition for choosing multi-region active-active origin over CDN-first caching.

### 60-second teach-back

> **"A load balancer exists because one server can't handle real traffic and will eventually die mid-request — so something has to sit in front of many backends and decide, per request, which one answers. That something is a reverse proxy, the server's own agent, mechanically almost identical to a forward proxy (the client's agent) except for who it's hiding. It can operate blind and fast, reading only IP and port — L4 — or it can fully decrypt and parse the request to route by path or header — L7 — but L7 requires TLS to terminate right there. Once you can pick a backend, you need an algorithm: round robin ignores load entirely and will happily drown a degraded backend in its full share of traffic forever, while least connections reacts to real, live in-flight counts instead. None of that tells you if a backend is even alive, which is what health checks are for — active checks probe proactively and protect every real request at the cost of continuous background traffic, while passive checks cost nothing until something breaks, at the price of sacrificing a handful of real users to notice. And once you can route around a dead backend, you hit the deepest issue in the whole topic: if anything that matters — a session, a cart, a conversation — lives only in one backend's memory, that backend's death doesn't just interrupt a request, it destroys data permanently, which is why stateless backends plus externalized state in a shared store wins almost every time, and sticky sessions should only ever be a dated stopgap. Deploying new code safely is the same problem in miniature: stop routing new work to an instance, let what's already in flight finish — sized to your slowest real request, not a platform default — then shut it down. And all of that, run once end to end on one real request — DNS, then TCP, then TLS, then HTTP, then a CDN cache check that might skip everything else entirely, then this whole load-balancer stack, then the app, then back — is the trace every request your career will ever touch actually runs."**

If you can deliver that and then explain, unprompted, why an agent's pooled connection to an LLM API is exactly as vulnerable to a silent backend swap as any other long-lived connection this note describes — you have this topic.

---

## Spaced-repetition flashcards

| Q | A |
|---|---|
| Forward proxy vs. reverse proxy — mechanically? | Nearly identical; the difference is whose agent it is and which side it hides |
| What can't an L4 LB see? | Anything past the TCP/UDP header — no HTTP method, path, host, or cookie |
| Where does TLS terminate in a standard L4/L7 stack? | At L7 |
| Round robin's core flaw? | No live-load awareness — sends a fixed share regardless of a backend's real capacity right now |
| What does least connections react to? | Current in-flight request count per backend |
| Naive IP hash's failure on pool resize? | Remaps ~(N−1)/N of clients when N changes by one |
| What fixes that? | Consistent hashing — remaps only ~1/N |
| Active health check cost? | Continuous background probe traffic; protects every real request |
| Passive health check cost? | Nothing when healthy; sacrifices up to `max_fails` real requests to detect failure |
| Does open-source Nginx support active health checks? | No — NGINX Plus, HAProxy, or Caddy only |
| How does OSS Nginx recover an ejected backend? | The next real request after `fail_timeout` IS the recovery probe |
| Why doesn't health-check ejection protect an existing pooled connection? | Ejection only changes routing for NEW connections, not ones already open |
| Why can't a crashed backend's session be recovered? | Process memory is deallocated on crash, not paused |
| Sticky sessions' three failure modes? | Crash loses that backend's sessions; scale-down strands them; load can go permanently uneven |
| The industry-standard fix? | Stateless backends + externalized state in a shared store |
| Two halves of graceful shutdown? | LB stops routing new traffic; app finishes in-flight work on SIGTERM |
| SIGTERM vs. SIGKILL? | SIGTERM asks nicely and allows cleanup; SIGKILL gives no warning at all |
| How should a drain timeout be sized? | To your slowest legitimate real request — not a platform default |
| Rolling vs. blue/green vs. canary — one-line difference each? | Rolling: replace gradually. Blue/green: instant switch, double compute. Canary: small real slice first, to catch bugs |
| The canonical trace's DNS step — which day? | Day 12 |
| The canonical trace's TCP step — which day? | Day 11 |
| The canonical trace's TLS step — which day? | Day 14 |
| The canonical trace's HTTP step — which day? | Day 13 |
| The canonical trace's CDN step — which day? | Day 15 |
| What can a CDN cache hit skip entirely? | The whole LB and app stack — the request never leaves the edge |
| `curl -w`'s TTFB field? | `time_starttransfer` |
| Slack Jan 2021 — which hop failed? | Load balancer/health-check layer — correct ejections compounding a capacity crisis |
| Cloudflare July 2019 — which hop failed? | The reverse-proxy/WAF layer — a regex's catastrophic backtracking |
| GitHub GLB's two tiers? | GLB Director (L4, ECMP + rendezvous hashing + DSR) and HAProxy (L7) |
| Google Maglev's key idea? | A software L4 LB using consistent hashing so backend changes remap minimal connections |

---

## Primary sources

**Proxies and HTTP mechanics**
- RFC 9110, HTTP Semantics, §9.3.6 — the `CONNECT` method (renumbered from RFC 7231 — verify current).
- Uvicorn documentation — `--proxy-headers`, `--forwarded-allow-ips`, `--timeout-graceful-shutdown` (verify current defaults; these have changed across releases).

**Load balancing algorithms and L4/L7 architecture**
- Eisenbud et al., "Maglev: A Fast and Reliable Software Network Load Balancer," USENIX NSDI 2016.
- Karger, Lehman, Leighton, Panigrahy, Levine, Lewin, "Consistent Hashing and Random Trees," ACM STOC 1997.
- GitHub Engineering Blog — posts introducing GLB Director (`github/glb-director`) — verify current, circa 2016–2018.
- Nginx documentation — `upstream` directives (`least_conn`, `ip_hash`, `hash ... consistent`, `max_fails`/`fail_timeout`) and the official Health Checks page distinguishing passive (open source) from active (NGINX Plus).
- HAProxy documentation — the `check` keyword and its tuning parameters (`inter`, `rise`, `fall`); seamless reload/socket-transfer behavior.
- Caddy documentation (`caddyserver.com/docs`) — `reverse_proxy`'s `health_uri`/`health_interval`/`lb_policy` directives (verify current, Caddy v2's config surface has evolved across releases).

**Sticky sessions and statelessness**
- 12factor.net, Factor VI, "Processes."
- Kubernetes documentation, "Service" (session affinity), "Pod Lifecycle," and "Deployments" (rolling updates, `preStop`, `terminationGracePeriodSeconds`) — verify current, defaults and probe interactions have been refined across releases.

**Incidents**
- Slack Engineering Blog, postmortem of the January 4, 2021 outage — verify current for exact timeline and figures.
- Cloudflare Blog, "Details of the Cloudflare outage on July 2, 2019" (John Graham-Cumming).
- US-CERT Vulnerability Note VU#529496 (Lenovo Superfish, February 2015); Lenovo's own security advisory.
- Beyer, Jones, Petoff, Murphy (eds.), *Site Reliability Engineering*, Chapter 22, "Addressing Cascading Failures" (freely available at sre.google/sre-book).
- Day 12's own primary sources for the Meta 2021 and Dyn 2016 outages, cross-referenced in concept 7 rather than re-cited here.

**Tools**
- `curl`'s manual page — the `-w`/`--write-out` variables (`time_namelookup`, `time_connect`, `time_appconnect`, `time_starttransfer`, `time_total`).
- `hey` (`github.com/rakyll/hey`) — verify current install method and flags against the project's own README, load-test tool syntax drifts across versions.

**Fast-drifting facts — verify before relying on any of these**
- The exact open-source/paid feature split for Nginx health checks, and equivalent splits for any other load balancer product.
- Uvicorn's graceful-shutdown flag names and default timeout values.
- Caddy's exact `reverse_proxy` health-check directive names and defaults.
- `hey`'s exact CLI flags and output formatting.
- Cloud "Network Load Balancer" vs. "Application Load Balancer" terminology and capability splits, which are provider-specific and change.
- Exact incident timelines, durations, and root-cause component names for the Slack and Cloudflare case studies — both accounts here reflect the well-corroborated general shape, not a verbatim transcript of the primary source.

---

## Layered explanations

**10 seconds.** A load balancer is a reverse proxy that decides which of many backends answers each request, using an algorithm that reacts to load, health checks that remove dead backends, and — the deepest lesson — statelessness, because anything that only lives in one backend's memory dies with that backend.

**1 minute.** Once one server can't handle real traffic, something has to sit in front of many and route each request — that's a reverse proxy, the server's own agent, mechanically almost identical to a forward proxy (the client's agent) except for whose side it's hiding. It can be blind and fast (L4, reading only IP/port) or slow and smart (L7, reading the actual HTTP request, which requires TLS to terminate there). Picking a specific backend is an algorithm — round robin ignores load entirely, least connections reacts to it — but none of them know if a backend is alive, which is what active and passive health checks are for, each trading background cost against how many real requests get sacrificed to detect a failure. The deepest issue is state: anything living only in one backend's memory is destroyed, not paused, when that backend dies, which is why stateless backends with externalized state beat sticky sessions almost every time. Deploying safely is the same problem in miniature — stop routing new work here, let what's in flight finish, then stop the process — and all of it, run once on one real request from DNS through TCP, TLS, HTTP, a CDN check, this whole stack, and back, is the trace every request in your career actually runs.

**5 minutes.** A load balancer earns its place by being a real endpoint — it terminates the client's connection and originates a new one to a backend, which is what lets it inspect, retry, and redirect traffic in ways a router never could, at the cost of being real, stateful infrastructure with its own capacity limits. Whether it does that blind (L4: IP and port only, kernel-fast, can't terminate TLS meaningfully) or smart (L7: full HTTP parsing, TLS terminates here, can route by path or cookie) is the first forced decision, and production stacks nest both rather than picking one. Choosing a specific backend per request is an algorithm problem structurally identical to OS thread scheduling: round robin assumes uniform cost and capacity and collapses the moment one backend degrades, sending it a mathematically fair but practically impossible share of load; least connections fixes this by reacting to real in-flight counts; IP hash and consistent hashing trade load-awareness for affinity, the latter solving the former's catastrophic remapping problem when the pool size changes. None of these algorithms know if a backend is alive — that's health checking's job, and it splits into active (continuous probing, protects every real request, unavailable in open-source Nginx specifically) and passive (free until something breaks, at the cost of a bounded number of sacrificed real requests) — with a critical blind spot: ejecting a backend from routing does nothing to a connection a client already has pooled to it, the same DNS-adjacent lesson Day 12 taught from the other side, now showing up for agents holding long-lived connections to LLM backends. The deepest concept is what happens to state: a backend's crash doesn't pause its memory, it destroys it, which is why anything that matters — a cart, a session, a multi-turn conversation — belongs in a shared, externalized store rather than any one process's RAM, and why sticky sessions are, at best, a dated stopgap rather than an architecture. Deploying new code safely is this same statelessness thesis applied to time instead of failure: connection draining splits into stopping new routing and letting the application itself finish gracefully on SIGTERM, sized to the slowest legitimate request rather than a generic platform default — a number that matters enormously more once streaming, long-lived responses (an SSE feed, a token-by-token LLM completion) are part of the traffic mix. And the whole topic assembles into one trace: DNS resolves a name to an address (Day 12), TCP and TLS establish a secure channel (Days 11 and 14), an HTTP request travels it (Day 13), a CDN edge may answer it directly and skip everything downstream entirely (Day 15) or relay it into exactly this note's L4/L7/algorithm/health-check stack, which hands it to a stateless application and returns the response back up the same chain — the single request every layer of Phase 1 was built to explain one piece of.

**Expert summary.** Load balancing is the application of proxy-mediated indirection to solve two coupled problems that appear whenever a service's serving capacity is decomposed across multiple, individually-failable processes: routing (which member of a dynamic pool should serve this unit of work) and membership (which members of that pool are currently eligible at all). The L4/L7 split is a resource-allocation decision as much as a routing one — L4 work is packet-rate-bound and cheapest to push into the kernel or NIC with near-zero per-connection state, while L7 work is compute-bound per connection and requires holding real application state, so collapsing the two tiers forces every machine to be provisioned for the union of both bottlenecks rather than the one it's actually facing; splitting them lets capacity be added independently along the axis actually under pressure, at the cost of a coordination boundary (where does TLS terminate, and what carries trust across it) that has to be decided deliberately rather than inherited from a default. Backend-selection algorithms form a strict hierarchy of the state they're willing to consult — none, a static weight, a live load signal, or a stable key — and each added bit of state bought is a bit of statelessness given up in the mechanism itself, which is precisely why algorithms that use the least state (round robin) fail hardest under heterogeneity while algorithms that use the most (session-affinity hashing) reintroduce, at the routing layer, exactly the fragility that statelessness at the application layer exists to eliminate. Health checking is a sampling problem wearing operational clothing: active checking trades continuous, controllable sampling cost for zero false-negative exposure on real traffic, passive checking trades a bounded, real-traffic-denominated false-negative cost for zero background overhead, and the entire cascading-failure failure mode this note's case studies document arises when a health signal's sampling process is correlated with the very capacity event it's trying to detect — a shared dependency, a synchronized client retry population, or a threshold tuned against a load profile that stops holding during the incident it's meant to catch. Underneath all of it is a single structural claim this note keeps re-deriving from different angles: **any state that exists in exactly one place is a single point of failure for that state**, whether the "one place" is a DNS record's TTL window (Day 12), a pooled TCP connection's lifetime (Day 11), a backend process's RAM (this note's sticky-session material), or a live connection's assumed short duration (this note's draining material) — and the entire discipline of building a system that survives its own components dying is the discipline of finding every one of those single places and either eliminating them or making their lifetime explicit and bounded.

---
