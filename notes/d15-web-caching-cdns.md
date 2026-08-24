# Day 15 — Web Caching and CDNs: Not Recomputing, Not Re-Fetching

> **Framing.** Day 1 built one idea that this whole note is an application of: the closer the data, the cheaper it is to read, and the pyramid you built there — register, L1, L2/L3, RAM, SSD, network — didn't stop at your machine's boundary. It kept going: a nearby network service is roughly 1–10 ms away, a cross-ocean one is tens to hundreds of ms away, and *recomputing an answer from scratch on a busy origin server* can cost more than either. Day 9 through 12 then spent four days getting one HTTP request from your NIC to a DNS answer to a TCP connection. This note picks up exactly where that leaves off: **you have an address and a connection — now what makes the actual response fast, and cheap, on the tenth request as well as the first?**
>
> The answer is caching, but at a layer most engineers have weaker intuition for than DNS, because HTTP caching gives you something DNS never did: a **validator** (Day 12 had none — a DNS record is either trusted or re-fetched wholesale) and, at the CDN layer, an actual **purge** mechanism (DNS has no invalidation at all, full stop). That extra machinery is not free complexity for its own sake — it exists because an HTTP response is usually far more expensive to regenerate than a DNS record is to re-fetch, so the protocol gives you finer control over exactly when to pay that cost. You will see Day 12's core lesson — *a cache with no way to reach into the past is characterized entirely by promises made in advance* — reappear almost unchanged when we get to `Cache-Control: max-age`, and you'll see where HTTP genuinely differs from DNS when we get to purge and the cache-stampede problem, which DNS's TTL model doesn't have to face in the same way because TTLs are so much longer relative to query cost.
>
> There is one genuinely load-bearing, mechanical connection to agentic systems here, and it shows up twice. First: a cache-stampede — a hot key expiring while a thousand callers are mid-flight — is not unique to serving HTML. An agent framework that caches LLM completions or embeddings by a hash of the prompt hits the *exact same* failure the moment that key expires under concurrent load, and the fix (a single-flight lock so only one caller actually pays for the recomputation) is identical code solving an identical problem — we'll build that fix once, in concept 7, and it transfers without translation. Second, smaller but genuinely useful: an agent's web-fetching tool that re-reads the same URL across turns can use conditional requests (concept 3) to avoid re-spending context-window tokens on content that hasn't changed — a `304 Not Modified` costs your prompt nothing extra to process, a `200` with a full body costs real tokens. Both connections are woven in where they belong, not bolted on at the end.

---

## Roadmap

Caching is usually taught as a list of header names to memorize. That produces engineers who can recite `max-age` and `no-cache` and still ship a CDN that leaks one user's account page to another, or a homepage that falls over the instant its cache entry expires. We'll build it instead as a sequence of forced decisions, the same way Day 12 built DNS — because every header and every CDN feature in this note is somebody's answer to a specific, nameable problem.

```
The problem: regenerating a response is expensive; the network hop to fetch
it again is expensive; doing either on EVERY request doesn't scale.
                              │
              Store a copy somewhere closer/cheaper. WHERE?
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
  PRIVATE cache          SHARED cache           CDN EDGE cache
  (one browser,          (a corporate proxy,    (a PoP near the
   one user)              one ISP)               USER, not the origin)
        │                     │                     │
        └─────────────────────┴─────────────────────┘
                              │
              A cached copy needs an EXPIRY POLICY,
              or every cache is a liability, not an asset
                              │
                      Cache-Control:
              max-age / s-maxage / no-cache / no-store /
              private / public / stale-while-revalidate
                              │
              (Same shape as Day 12's DNS TTL: a promise made
               in advance, honoured with no way to reach back
               and revoke it once it's cached — for max-age.)
                              │
              But unlike a DNS record, an HTTP response often
              has a WAY TO PROVE "it didn't change" cheaply —
                              │
                        VALIDATORS
              ETag / Last-Modified + conditional requests
              (If-None-Match → 304 Not Modified, no body)
                              │
              The cache key isn't always just the URL —
                              │
                     Vary: what ELSE distinguishes
                     two cached copies of "the same" URL
                              │
              Now put the cache PHYSICALLY near the user,
              at scale, for millions of users at once
                              │
                        CDN MECHANICS
              edge PoPs (anycast, Day 12 concept 1) + cache
              keys + origin shielding (fan-in protection)
                              │
              Unlike DNS, you CAN reach into a CDN cache and
              tell it "forget this" — but that reach is neither
              instant nor free across a globally distributed system
                              │
                ┌─────────────┴──────────────┐
                │                            │
             PURGE                    VERSIONED / HASHED URLS
       (push an invalidation      (never invalidate — mint a
        to every edge; has         brand-new name instead;
        propagation lag)           make the OLD name immutable
                │                   forever)
                └─────────────┬──────────────┘
                              │
              And when a hot key's TTL expires under heavy
              concurrent load, ALL of that load arrives
              at the origin in the same instant —
                              │
                       CACHE STAMPEDE
              (single-flight lock / request coalescing /
               probabilistic early expiration)
```

Concepts and tiers:

| # | Concept | Tier |
|---|---|---|
| 1 | Why cache HTTP responses: the layers, and Day 1's pyramid extended over the network | **[CORE]** |
| 2 | `Cache-Control` anatomy: freshness, scope, and staleness policy | **[CORE]** |
| 3 | Validators and conditional requests: ETag, Last-Modified, 304 | **[CORE]** |
| 4 | `Vary`: when the cache key needs more than the URL | **[WORKING]** |
| 5 | CDN mechanics: edge PoPs, cache keys, origin shielding | **[CORE]** |
| 6 | Cache invalidation: purge vs. versioned/hashed URLs | **[CORE]** |
| 7 | The cache stampede problem and single-flight | **[CORE]** |

A note on scope, stated plainly rather than left implicit: Day 13 owns HTTP's request/response anatomy, methods, and status codes — including `304` itself as a status code and the general shape of statelessness — so this note uses `304` without re-deriving what a status code is. Day 14 owns HTTP/1.1 → 2 → 3 and TLS; this note doesn't re-teach multiplexing, but concept 5 will note in passing why HTTP/2's multiplexing changed how many parallel connections a CDN edge needs to hold open per client. Day 12 owns TTL-as-a-promise and no-invalidation caching; this note leans on that reasoning directly for `max-age` and marks precisely where HTTP's story diverges from DNS's (validators, and a real if imperfect purge mechanism).

---

## 1. Why cache HTTP responses: the layers, and Day 1's pyramid extended over the network

**Depth: [CORE]**

### Intuition

Every HTTP response is the result of *some* amount of work: at minimum, a network round trip; often, a database query, a template render, an authorization check, and a serialization step layered on top of that round trip. None of that work is free, and — this is the part that's easy to under-rate — **most of it is being redone for an answer that hasn't changed since the last time someone asked.** A product page's price didn't change between two requests eleven seconds apart. A CSS file compiled at build time is bit-for-bit identical every time it's requested until the next deploy. A news article published at 9:00 a.m. is the same article at 9:00:03 as it was at 9:00:00. Serving each of those requests by re-running the full pipeline — hit the database, re-render, re-serialize, ship the bytes across an ocean — is exactly the mistake Day 1 taught you to avoid inside one machine, now happening across a network instead of across a memory bus.

That reframe is the entire idea of this note, so it's worth being precise about the analogy rather than waving at it. Day 1's pyramid existed because of a **physical** fact: distance and medium determine latency, and a value sitting in L1 cache is reachable in about a nanosecond while the same value sitting in RAM costs roughly a hundred times that, not because RAM is a worse technology but because electrons have to travel farther through more circuitry. Extend that same physical fact across a wide-area network and the multiplier gets far larger, not smaller: Day 9 and Day 10 established that a nearby request costs single-digit milliseconds — dominated by the speed of light and the number of hops — while a cross-ocean one costs tens to low hundreds of milliseconds. Recomputing an expensive page server-side can add tens more milliseconds on top of that network cost. **A cache, at any layer, is just a way of answering the question "must I pay that cost again?" with "no" as often as correctness allows.**

### The four layers, and why there isn't just one

An HTTP response can be cached in at least four physically and administratively distinct places, and confusing them is the single most common source of "I changed it and nothing happened" bug reports — the direct HTTP-layer cousin of Day 12 concept 3's "seven places a stale DNS answer can live."

```
┌──────────────────────────────────────────────────────────────────┐
│ CLIENT (browser / mobile app / an agent's HTTP tool)              │
│                                                                    │
│   ①  PRIVATE CACHE — belongs to exactly one user.                 │
│      Lives in the browser's disk cache, or an app's local store.  │
│      Can hold responses to authenticated, personalized requests   │
│      because only the one user it belongs to will ever read it.   │
└──────────────────────────────────────────────────────────────────┘
                              │  (cache miss, or revalidation needed)
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ SHARED / INTERMEDIARY CACHE                                       │
│                                                                    │
│   ②  A corporate forward proxy, an ISP's transparent cache, or    │
│      a reverse proxy you run in front of your own app servers     │
│      (nginx, Varnish, Squid). Serves MANY users. Must never hold  │
│      a response meant for only one of them — this is precisely   │
│      what `private` vs `public` (concept 2) exists to prevent.    │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ CDN EDGE CACHE                                                     │
│                                                                    │
│   ③  A shared cache too, but purpose-built and placed             │
│      geographically close to users rather than close to your      │
│      origin (concept 5). Handles millions of users at once,       │
│      across hundreds of physical locations.                       │
└──────────────────────────────────────────────────────────────────┘
                              │  (edge miss → origin request)
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ ORIGIN-SIDE CACHE                                                  │
│                                                                    │
│   ④  Your own application's cache in front of its own database    │
│      or expensive computation — Redis, Memcached, an in-process   │
│      dict. Not an HTTP cache at all in the protocol sense (no      │
│      Cache-Control involved), but it exists for the identical     │
│      reason and suffers the identical failure mode (concept 7).   │
└──────────────────────────────────────────────────────────────────┘
```

Layers ①–③ are governed by the HTTP caching protocol itself — `Cache-Control`, validators, `Vary` — and are what RFC 9111 (the current HTTP Caching specification, obsoleting the earlier RFC 7234, which itself obsoleted the original RFC 2616 caching text) actually defines rules for. Layer ④ is not part of the HTTP spec at all; it's an application-level optimization that happens to face the exact same stampede problem as layers ①–③, which is why concept 7 treats it as one mechanism rather than a separate topic.

**The vocabulary worth fixing now, because RFC 9111 uses it precisely and casual writing doesn't:** a **private cache** serves one user (layer ①); a **shared cache** serves more than one user behind it (layers ② and ③ are both shared caches, just at different points in the topology). The spec's rules for what a shared cache is *allowed* to store are meaningfully stricter than a private cache's, for a reason that will become concrete and uncomfortable in concept 2's case studies: **a shared cache that gets this wrong doesn't slow one person down — it leaks one person's response to everyone else who asks for the same URL.**

### Analogy: a chain of warehouses, and where it breaks

Day 1 used a workbench-and-warehouse analogy for the CPU-to-network pyramid. Extend it: your origin server is a factory that manufactures a product (the response) to order. A private cache is a box in your own house — nobody else's order gets fulfilled from it, and you don't need the factory's permission to keep something there as long as you like. A shared cache is a warehouse serving a whole neighbourhood — efficient exactly because many people's orders can be filled from the same stock, but *dangerous* the moment it holds something that was custom-made for one customer and then hands it to the next person who asks. A CDN edge cache is that same neighbourhood warehouse model, deliberately built at every neighbourhood in the world instead of one central warehouse, so that "close to the customer" is true everywhere at once.

**Where this analogy breaks, and it matters:** a physical warehouse either has the item or it doesn't, and if it has it, it's simply correct — a warehouse doesn't need to ask the factory "is this still the right version of the product?" A cached HTTP response can be *wrong* in a way a warehouse's stock can't: it can be present, intact, and stale, silently describing a world that no longer exists (yesterday's price, last hour's inventory count). That's exactly the gap validators (concept 3) exist to close, and it's a gap the warehouse analogy has no room for — which is your first concrete signal that HTTP caching needed more machinery than "keep a copy and throw it away after a while."

### Worked example — the same page, three ways, with real numbers

Take a concrete, ordinary case: a product page whose HTML is 42 KB, generated by a template that queries a database, and requested by a user in Sydney from an origin server in Virginia (`us-east-1`, to make it concrete).

**No caching at all.** Every request pays: the network round trip Sydney↔Virginia (roughly 200 ms round-trip time at that distance — Day 9/10's speed-of-light-plus-hops arithmetic, not a guess but the physical floor for ~15,000 km of fiber), plus server-side work — say a 15 ms database query and an 8 ms template render, which is generous for a page that isn't pathologically slow. Total: **≈223 ms**, paid by *every single request*, including the ten thousand people who requested the identical page in the same minute.

**A CDN edge cache, warm.** The nearest edge PoP to Sydney is very likely also *in* Sydney or nearby (concept 5) — call the client-to-edge round trip 5 ms. If the edge already holds a fresh copy, the response comes back in essentially that 5 ms plus a few hundred microseconds of the edge server's own lookup and TLS overhead (TLS session resumption, covered in Day 14, keeps this cheap on a warm connection). Call it **≈8 ms**. That's roughly a **28× reduction** in latency, and — this is the part raw latency numbers hide — **zero additional load reaches the database or the origin server at all.** The origin didn't do 15 ms of database work it didn't do 8 ms of rendering; it did *nothing*, for this request.

**A CDN edge cache, cold (first request after the entry expired or was purged).** The edge must fetch from the origin once, pays the full 223 ms for that one request, caches the result, and then every subsequent request from anyone near that edge gets the 8 ms answer until the entry expires again. The cost of staying correct is paid *once per cache lifetime, per edge location* — not once per request.

The arithmetic that actually matters for capacity planning is the **hit ratio**: if 99% of requests for this page hit a warm edge cache and 1% are cold misses that reach the origin, the origin's actual load for this endpoint is 1% of naive traffic, and the *average* latency across all requests is `0.99 × 8ms + 0.01 × 223ms ≈ 10.2 ms` — barely worse than an always-warm cache, and radically better than no caching at all. **This is the entire economic case for every technique in this note**, in exactly the same way Day 12's four-round-trip DNS cost was the entire economic case for DNS caching: the gap between cold and warm is enormous, so pushing traffic toward "warm" is worth real engineering effort.

### Under the hood: what makes a response cacheable at all, mechanically

A cache doesn't get to decide "this looks safe to store" by inspecting the content — it follows explicit rules keyed on the **method** and the **status code**, because storing the wrong thing silently is worse than not caching at all.

- **Method.** `GET` responses are cacheable by default (subject to the headers in concept 2). `HEAD` responses are cacheable insofar as they update stored metadata for the same resource. `POST`, `PUT`, `DELETE`, and `PATCH` responses are *not* cached by default — the entire point of those methods is usually to cause a side effect, and serving a stale copy of "you successfully charged the card" would be a correctness disaster, not a performance win. RFC 9111 §3 does allow caching `POST` responses if the response explicitly carries freshness information, but this is rare in practice and worth knowing exists rather than reaching for.
- **Status code.** RFC 9111 §3 lists status codes that are cacheable *by default* absent explicit directives — `200`, `203`, `204`, `206`, `300`, `301`, `404`, `405`, `410`, `414`, and `501` among them — and status codes that are *not* cacheable by default, most importantly `500`-class errors and `503`. This default matters more than most engineers realize: **an uncached `404` for a URL that will never exist is a small mercy to your origin, and a cached `500` served to everyone for the next hour because nobody thought to mark it `no-store` is a real, repeatable production incident.** We'll return to exactly this failure mode in concept 2.
- **Explicit permission always wins.** Whatever the defaults say, a `Cache-Control: no-store` on the response overrides everything — no cache anywhere is allowed to retain so much as a byte of it, regardless of method or status code.

**Where this note deliberately stops for now:** the full state machine for constructing a stored response from a `206 Partial Content` (range requests, used for video seeking and resumable downloads) is real and used constantly by CDNs serving media, but it's a black box we're naming and not opening — it doesn't change any of the reasoning in this note, and opening it fully belongs with a dedicated media-delivery topic.

### Judgment: an HTTP cache versus an application-level cache versus no cache

**The choice, stated concretely:** you're building a product-detail endpoint that's read far more often than it's written. Where does caching belong — in front of the whole HTTP response (a CDN or reverse-proxy cache), inside your application in front of the database (Redis/Memcached), or nowhere, relying on your database's own query cache and connection pooling?

**The realistic alternatives, and they genuinely compete:**
1. **HTTP-layer caching** (CDN or reverse proxy in front of the app) — caches the *entire rendered response*, keyed by URL (and `Vary`, concept 4).
2. **Application-layer caching** (Redis in front of the database) — caches *data*, and your application still runs on every request to assemble a response from that data.
3. **No dedicated cache** — rely on the database's own buffer pool and query plan cache, and on connection pooling (Day 11) to keep per-request overhead low.

**The actual reason HTTP-layer caching wins for this specific shape of endpoint:** the response is identical for every anonymous user asking for the same product, and the *entire cost* of the request — not just the database query, but the template render, the serialization, and the network round trip to the origin at all — disappears on a hit. An application-level cache still pays for a full request/response cycle to your app server on every single request; it just makes the *database* part of that cycle cheap. If the bottleneck is genuinely the database, application-level caching is the right layer to attack it at. If the bottleneck is the *whole round trip*, especially over long geographic distances, only a cache that sits physically closer to the user — an HTTP cache, ideally a CDN — removes the part of the cost that an application-level cache can't touch: the network hop itself.

**The trade-off, honestly:** HTTP-layer caching can only cache what's safe to serve identically to whoever asks — it's fundamentally awkward for anything personalized (a logged-in user's dashboard, a shopping cart), because the moment the response depends on *who's asking*, you either can't cache it at the HTTP layer at all, or you push that complexity into `Vary` and cache-key design (concept 4) in ways that can silently blow up your effective cache size or, worse, leak one user's data to another if done carelessly (concept 2's case studies). Application-level caching has no such restriction — you can cache a `user_id`-keyed value with total confidence about who it belongs to, because your application code, not a shared cache, controls exactly what's returned to whom.

**The flip condition:** move toward application-level (or no) caching as personalization increases — a feed ranked per-user, a dashboard showing account-specific numbers, anything behind meaningful authorization logic per request. Move toward HTTP/CDN caching as the response becomes more uniform across users and more expensive to generate relative to how often it changes — public marketing pages, product catalogs, published articles, and (concept 6) versioned static assets, which are the limiting case where the response *never* changes for a given URL at all. In practice, most real systems run all three simultaneously, layered: CDN caching for the genuinely public, uniform surface; Redis in front of the database for the personalized-but-expensive-to-compute surface; and the database's own internal caching underneath both, doing its job regardless of what's happening above it.

### In production — choosing what to cache at the HTTP layer at all

**Best practices:**
1. **Default to not caching, and add caching deliberately per endpoint**, rather than caching by default and discovering later that a personalized response got cached. The cost of under-caching is slower responses; the cost of over-caching a personalized response is a data leak (concept 2).
2. **Measure hit ratio before and after any change.** A cache with a 40% hit ratio might not be worth its complexity; the same cache at 95% almost certainly is. This number is the single most informative metric for this entire topic.
3. **Treat cached error responses as a deliberate, audited decision, never an accident.** `no-store` on error paths by default; opt specific, safe-to-cache errors (a genuinely permanent `404`) back in with intent.

**Mistakes, beginner → senior:**
- *Beginner:* assuming "caching" is one setting to turn on, rather than a per-endpoint decision with real correctness constraints.
- *Intermediate:* caching a response that includes a per-user element (a "Hello, `{name}`" banner baked into otherwise-public HTML) without noticing it's no longer safe for a shared cache.
- *Senior:* under-provisioning for the *cold* path — sizing your origin for the traffic you expect *after* caching kicks in, and then discovering during a cache-clearing incident or a stampede (concept 7) that the origin was never built to survive uncached load.

**Monitoring:** hit/miss ratio per endpoint (not just globally — a 95% site-wide hit ratio can hide a 10% hit ratio on your most expensive endpoint); origin request volume and latency, specifically watched for spikes that correlate with cache-clearing events; the actual latency distribution *including* cold misses, not just the median (which a high hit ratio will make look great while your p99 quietly reflects every miss).

**Cost:** every layer in this stack is cheaper per byte served than the layer below it — an edge cache hit is cheaper than an origin request, which is cheaper than a database query — so the honest cost conversation is almost always "what hit ratio justifies this engineering effort," not "should we cache at all." The counter-cost, easy to forget until it happens: a cache adds a place where **bugs hide as staleness** rather than as visible errors, which is more expensive to debug precisely because nothing looks broken.

---

## 2. `Cache-Control` anatomy: freshness, scope, and staleness policy

**Depth: [CORE]**

### Intuition

Concept 1 established *that* a response can be stored. It said nothing about *how long* it may be trusted before someone has to check with the origin again — and without an answer to that question, every cache in the chain is stuck between two bad defaults: never trust a stored copy (which throws away everything concept 1 just bought you) or trust it forever (which means the moment your database changes, some fraction of the internet keeps serving the old answer with no idea it's wrong). `Cache-Control` is the origin server's way of settling that question explicitly, per response, instead of leaving it to whatever guess each cache happens to make.

This is close enough to Day 12's DNS TTL that the reasoning transfers directly, so it's worth stating the transfer precisely rather than re-deriving it: **a DNS record's TTL is a promise — "trust this for N seconds, and I have no way to reach into your cache and take it back early" — and `Cache-Control: max-age` is the identical promise, made about an HTTP response instead of a name-to-address mapping.** Every consequence Day 12 drew from that shape carries over unchanged: the TTL is chosen in advance, without knowledge of what will go wrong; a short TTL buys fast correction at the cost of load on the origin; a long TTL buys cheap serving at the cost of slow correction; and lowering a TTL takes effect only after the *previous, longer* TTL has already expired everywhere it was cached. If you haven't internalized that logic, it's worth rereading Day 12 concept 5 before continuing, because this section builds on it rather than repeating it.

Where HTTP genuinely adds machinery DNS doesn't have — and this is the reason `Cache-Control` has eleven-plus directives instead of one number — is that an HTTP response caches in *both* private, single-user locations and shared, multi-user locations simultaneously, with different correctness requirements at each, and an HTTP cache (unlike a DNS resolver) can often *check* whether a stale copy is actually still correct without re-downloading it (concept 3). `Cache-Control` has to express freshness, scope, and staleness policy all at once, and every directive below is answering exactly one of those three questions.

### Before `Cache-Control`: `Expires`, and why a header full of dates was the wrong shape

HTTP/1.0 answered "how long is this good for" with an `Expires` header carrying an **absolute timestamp**: `Expires: Thu, 01 Dec 1994 16:00:00 GMT`. It looks equivalent to `max-age`, and for a single cache talking to a single, correctly-clocked origin it behaves equivalently — but it inherits a problem Day 12 already taught you to watch for in a different guise: **an absolute time is only as correct as the clocks agreeing about what time it is.** A cache with a clock five minutes fast treats a response as expired five minutes early; a cache with a clock five minutes slow serves it five minutes past when the origin meant to stop trusting it. `max-age`, introduced with `Cache-Control` in HTTP/1.1, sidesteps the entire class of bug by being a **relative** duration — "good for 300 seconds from *your* receipt of this response" — which needs no clock agreement between two machines at all, only a machine's ability to measure its own elapsed time correctly, which is a far easier property to guarantee.

`Expires` still exists and caches still honor it as a fallback, but per RFC 9111 §5.3, **a `max-age` or `s-maxage` directive present in the same response always overrides `Expires`.** The practical rule: set `max-age`, and either omit `Expires` or leave it as a legacy fallback for the rare HTTP/1.0-only client — never rely on it as your primary mechanism.

### The directives, and the question each one answers

**Freshness — "how long may this be trusted without asking anyone?"**

`max-age=N` is the core directive: this response is fresh for `N` seconds after it was generated, in *any* cache — private or shared. `s-maxage=N` does the identical job but applies **only to shared caches**, and when present it overrides `max-age` for those caches specifically. This split exists because a private browser cache and a shared CDN edge often want different lifetimes for the same response: you might want your own browser to hold a page for 5 minutes (`max-age=300`) but want a shared CDN in front of your API to hold it for a full hour because you trust your own purge mechanism (concept 6) to correct it sooner if needed (`s-maxage=3600`). A response with `s-maxage=3600, max-age=300` tells a browser "5 minutes" and a CDN "1 hour," simultaneously and correctly, from one header.

**Scope — "who is allowed to store this at all?"**

`private` means: this response is personalized to the requester, and only a private (single-user) cache may store it — a shared cache must not, under any circumstances, because doing so risks handing one person's response to another. `public` means the opposite extreme: this response may be cached even in situations a shared cache would normally treat with more suspicion by default — most importantly, per RFC 9111 §3, a response to a request that carried an `Authorization` header, which a shared cache otherwise must not store *unless* the response explicitly permits it via `public`, `must-revalidate`, or `s-maxage`. Absent either directive, a shared cache applies its own heuristics about what's safe, which is precisely the ambiguity that leads to the case studies below — **an engineer who never explicitly sets `private` on a response containing per-user data is trusting every cache between their origin and the user to guess correctly, and guessing correctly is not what shared caches are optimized to do.**

**Revalidation policy — "once this goes stale, what happens?"**

`no-cache` is the single most misread directive in this entire topic, because its name suggests "don't cache this" and it means something close to the opposite: **a cache MAY store this response, but MUST revalidate it with the origin — using the validators from concept 3 — before serving it to anyone, every single time**, regardless of how fresh it would otherwise be. It is a way of saying "cache the bytes to save bandwidth on a hit, but never trust them without asking first." Contrast that precisely with `no-store`, which means **no cache anywhere may retain any part of this request or response at all** — not the headers, not the body, not even long enough to revalidate later. `no-cache` is "always check, but you can keep a copy to check against"; `no-store` is "there is nothing here to keep." Conflating the two is the most common `Cache-Control` mistake in production code, and it's an easy one to make because English usage of "don't cache that" maps onto `no-store`, not the directive literally named `no-cache`.

`must-revalidate` is `no-cache`'s cousin for a response that *did* have an explicit freshness lifetime: it says "once this passes its `max-age`, you may not serve it stale for *any* reason — including your own unreachability to the origin — you must either revalidate successfully or fail the request." This matters because a cache's *default* behavior when it can't reach the origin to revalidate a stale entry is often to serve the stale copy anyway, on the theory that stale-but-available beats an outright failure. `must-revalidate` switches that default off for responses where serving stale data would be actively wrong — a financial balance, an inventory count you're about to sell against.

**Deliberately serving stale, on purpose (RFC 5861):** `stale-while-revalidate=N` inverts the usual trade-off between "fast" and "correct" by letting a cache serve an already-stale response *immediately*, while it quietly fetches a fresh one in the background for the *next* request — for up to `N` seconds past the original freshness lifetime. The user who happens to hit the cache in that window gets a stale-but-instant answer instead of waiting on a slow revalidation; the next user after that gets the freshly revalidated one. `stale-if-error=N` is the companion directive for the failure case specifically: if revalidation fails because the origin is erroring or unreachable, serve the stale copy for up to `N` more seconds rather than propagating the failure to the user. Both exist because "fresh" and "available" are different properties, and sometimes serving something 40 seconds old is a far better user experience than serving a `503` — we'll use exactly this pair in the system design scenario below.

**Skipping revalidation entirely (RFC 8246):** `immutable` tells a cache — specifically a *browser*, on a user-initiated reload — that the origin will never change this representation during its freshness lifetime, so there's no reason to even *ask*. Without `immutable`, a browser's reload button will often send a conditional revalidation request (concept 3) even for a response that's still fresh by `max-age`, just because the user asked to reload; `immutable` says that request is pointless and skippable. It only makes sense paired with a URL that structurally cannot describe two different pieces of content — which is exactly what concept 6's versioned/hashed filenames guarantee, and it's why you'll see `immutable` and content-hashed URLs appear together everywhere.

**When nothing is specified at all: heuristic freshness.** A response with no `Cache-Control` and no `Expires` is not automatically treated as uncacheable — RFC 9111 §4.2.2 explicitly permits a cache to *guess* a freshness lifetime using a heuristic, and the heuristic almost every real cache implements is: **10% of the time elapsed since the response's `Last-Modified` date.** A response last modified 10 days ago, served with no explicit caching headers, gets an implied freshness lifetime of roughly one day. This is a real, specified behavior and not a bug — but relying on it is exactly the "provider default you never looked at" mistake from concept 1's In Production section, because it means your caching policy is an accident of how old your content happens to be rather than a decision you made.

**The `Age` header, and why "how long has this been cached" matters even when you never asked for it.** Every shared cache that serves a stored response is required to compute and add an `Age` header stating how many seconds have passed since the response was generated (or last validated) at the origin — accounting for time spent in *every* cache along the path, not just the one answering you. A response with `Cache-Control: max-age=300` and `Age: 280` is telling you it has 20 seconds of freshness left, and if another cache is asking your cache for it, *that* cache's `Age` calculation must add your cache's own resident time on top. This is the direct HTTP-layer analogue of Day 12's "the TTL you see is the *remaining* TTL, not the configured one" — same shape, same reason: **you are usually looking at cached data that's already been sitting somewhere else, and the header tells you how long.**

### Analogy: a "best-before" stamp plus a manufacturer's hotline — and where it breaks

A packaged food item has a best-before date stamped on it (freshness), and it's either intended for one household's fridge or a shared cafeteria's walk-in cooler (scope), and some perishables carry a note — "check with the supplier before serving past this date" — rather than an outright discard order (revalidation policy). `max-age` is the stamped date; `private`/`public` is which kitchen it's allowed in; `no-cache` is "call to check before serving, every time"; `no-store` is "this isn't the kind of thing you keep in a fridge at all — eat it now or don't."

**Where the analogy breaks, and this is the load-bearing part:** a food item's best-before date is set once, by the manufacturer, and doesn't change based on who's asking. An HTTP response's freshness policy can — and often should — differ by *which kitchen is asking*, which is exactly what `s-maxage` versus `max-age` expresses and food packaging has no equivalent for. There's no real-world analogue to telling your own fridge "trust this for a week" while telling a restaurant's shared cooler "trust this for only a day" on the same stamp — but that's a completely ordinary, everyday `Cache-Control` header in HTTP.

### Worked example — one response header, read directive by directive, against a concrete timeline

Take a real response header you might see from a news site's article-listing API:

```
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: public, max-age=30, s-maxage=300, stale-while-revalidate=60
Age: 0
Date: Mon, 24 Aug 2026 09:00:00 GMT
```

Read what this actually commits the origin to, directive by directive:

- `public` — safe for any shared cache to store, including ones sitting in front of an `Authorization`-bearing request path, if this endpoint has one.
- `max-age=30` — a **browser** holding this response treats it as fresh for 30 seconds from `Date`. At `09:00:30` a browser's own cache would revalidate before reuse.
- `s-maxage=300` — a **CDN edge** holding the same response treats it as fresh for a full 5 minutes, ten times longer than the browser does, because the origin trusts its CDN's purge mechanism (concept 6) to push corrections faster than 300 seconds if something genuinely urgent needs fixing, and doesn't want to pay the origin-request cost of every browser's 30-second cycle at CDN scale.
- `stale-while-revalidate=60` — applies at *either* layer: once the applicable freshness lifetime (30s for the browser, 300s for the CDN) has passed, that cache may keep serving the now-stale copy *immediately* for up to another 60 seconds while it fetches a fresh one in the background, rather than making the very next requester wait through a full revalidation round trip.

Trace one CDN edge's timeline concretely:

```
t=0     Origin generates the response. Age: 0.
t=0..300   CDN edge serves this response to every requester, entirely from
           cache. Zero origin requests. Age header increments per request.
t=300   Freshness lifetime (s-maxage) expires. The NEXT request arrives.
t=300   Because stale-while-revalidate=60 is set: that request gets the
           STALE copy back immediately (no waiting), and the edge kicks off
           a background revalidation to the origin.
t=300+ε Origin responds with a fresh copy (or a 304 — concept 3). Edge
           cache updated. Age resets to reflect the new generation time.
t=301   The request AFTER that gets the fresh copy, warm, no wait.
t=360   If for some reason the background revalidation never completed
           (origin slow or down), the 60-second stale-while-revalidate
           window has now ALSO expired — the next request must wait on a
           real, synchronous revalidation, with no more stale copy to fall
           back on.
```

**The property worth naming explicitly:** with `stale-while-revalidate`, *nobody* except the one background revalidation request ever pays the full origin round-trip cost after the very first request — every user-facing request gets an immediate answer, either fresh or briefly stale, and never blocks on the origin. That's a materially better user experience than the naive alternative (block the unlucky request that arrives right after expiry on a full origin round trip) for a cost of at most 60 seconds of staleness on a small fraction of requests. This exact mechanism is one of the two legitimate defenses against a cache stampede (concept 7) — worth flagging now and returning to properly there.

### Visual — can a shared cache serve this without contacting the origin?

```
                    Incoming request for URL X
                              │
                 Is there a stored response for X
                 (matching the cache key, concept 4)?
                              │
                 ┌────────────┴────────────┐
                NO                         YES
                 │                          │
          MISS: ask origin.          Does it carry "no-store"?
          Store the answer            (Should never have been
          per its own                  stored — but check anyway)
          Cache-Control.                    │
                                   ┌─────────┴─────────┐
                                  YES                  NO
                                   │                    │
                            Serve nothing        Is it within its
                            from cache;          freshness lifetime
                            treat as MISS.        (max-age / s-maxage,
                                                   whichever applies
                                                   to THIS cache)?
                                                        │
                                          ┌─────────────┴─────────────┐
                                         YES                          NO
                                          │                            │
                                   Does it carry             Is stale-while-revalidate
                                   "no-cache"?                still within its window?
                                          │                            │
                                  ┌───────┴───────┐          ┌─────────┴─────────┐
                                 YES              NO         YES                 NO
                                  │                │          │                   │
                            MUST revalidate   HIT: serve   Serve STALE      MUST revalidate
                            first (conc. 3)   immediately,  immediately,    (or fail, if
                            before serving.   no origin     background      must-revalidate
                                              contact.       revalidate.     is set and
                                                                             origin is down)
```

### Under the hood: computing freshness, precisely

RFC 9111 §4.2 spells out the actual arithmetic a compliant cache runs, and it's worth seeing once in full because "is this fresh" is not a vibe, it's a computation with two clearly defined numbers being compared:

```
freshness_lifetime =
    s-maxage           if present AND this is a shared cache
    max-age            elif present
    Expires − Date     elif Expires is present
    heuristic (~10% of age since Last-Modified)   otherwise
    0                  if none of the above can be determined

current_age =
    (the age the response ALREADY had when this cache received it,
     taken from its own Age header if any, corrected for clock skew
     using the response's Date header)
    + (time this cache has held it since receiving it)

is_fresh  =  freshness_lifetime > current_age
```

Everything in concept 2's directives resolves to feeding this one comparison the right number. `max-age` and `s-maxage` set the left side directly; `no-cache` and `must-revalidate` change what happens on the `False` branch (must ask vs. may serve anyway if unreachable); `stale-while-revalidate` adds a *second*, more permissive threshold that only kicks in once the first one has already failed. Once you can read a `Cache-Control` header as "which of these named quantities is it setting, and which branch of this comparison is it changing," the eleven-plus directives stop being a list to memorize and become a small, closed vocabulary for one arithmetic decision.

### System design — `Cache-Control` for a multi-tenant SaaS: public marketing site vs. private account data

**Problem:** you run a SaaS product with two genuinely different surfaces on one domain: a public marketing site and blog (`example.com/blog/*`, `example.com/pricing`) and an authenticated customer dashboard (`example.com/app/*`) sitting behind a shared CDN that also does TLS termination and basic reverse-proxying for both. Design the `Cache-Control` policy for each, with the explicit requirement that **a bug on one surface must not be able to leak one customer's dashboard data to another customer or to an anonymous visitor.**

**Requirements:** the marketing site should be served almost entirely from CDN edge cache, since it's identical for every visitor and changes rarely; the dashboard must never have one user's account data served to a different user under any circumstances, including a misconfiguration; the team should be able to reason about the blast radius of a mistake on one surface without auditing the other.

**The realistic alternatives:**
1. **One blanket `Cache-Control` policy for the whole domain** (e.g., a reverse-proxy default of `public, max-age=60` applied everywhere unless a route opts out).
2. **Explicit, opposite-default policies per route class**: marketing routes opt *in* to aggressive shared caching; dashboard routes are `private, no-store` by default, with narrow, audited exceptions.
3. **Split the two surfaces onto genuinely separate hostnames or even separate CDN configurations** (`www.example.com` vs. `app.example.com`), each with its own default policy.

**The decision: (2) at minimum, (3) once the team and traffic justify the operational overhead of two configurations.**

**The actual reason:** a single blanket default is a bet that every route in the codebase will remember to override it correctly, forever, including routes written by someone who joined the team after the default was set and never read the CDN config. That bet is exactly the setup behind both case studies below — a route that *should* have been marked `private` or `no-store` simply wasn't, because there was nothing forcing anyone to think about it. **Making the dashboard's default `no-store` and requiring an explicit, reviewed opt-in for any caching there is a blast-radius decision identical in shape to Day 12's zone-per-environment DNS design (Day 12 concept 2): make the dangerous case require a deliberate action, not the safe case.** The marketing site gets the opposite default — cache aggressively — because there the dangerous case (a stale price on the pricing page for an extra minute) is cheap and the safe case (fast, cheap-to-serve pages) is what you want by default.

**The trade-off, honestly:** defaulting the dashboard to `no-store` means every dashboard request round-trips fully to the origin, all the time — you're deliberately giving up the CDN benefit that concept 1 spent an entire worked example establishing, for the entire authenticated surface of your product. For a SaaS dashboard that's usually the right call, because the *content itself* is rarely identical between two requests anyway (it's account-specific), so you weren't going to get much of a shared-cache hit ratio there regardless — you're giving up little and buying a hard safety guarantee. Splitting hostnames (option 3) buys the strongest isolation — a misconfigured CDN rule on `app.example.com` structurally cannot apply to `www.example.com` — at the cost of running and monitoring two configurations instead of one, and potentially two TLS certificates and two DNS zones to keep straight (Day 12's zone-as-blast-radius-boundary reasoning, again, one level up the stack).

**The flip condition:** stay with one hostname and per-route policy as long as the team is small enough that a change to caching rules always goes through a review step that specifically checks the scope directive, and as long as the dashboard genuinely has little cacheable content anyway. Move to separate hostnames when the team scales past the point where you can trust "someone will remember to check" as a control, or when the dashboard *does* start having genuinely public-shaped sub-resources (a public status page, a public API a customer's own integrations hit) that legitimately want CDN caching and you don't want that caching policy anywhere near the same configuration surface as account data.

**Failure modes:** a route added under `/app/` that happens to return a mostly-static asset (a company logo upload, a generated PDF) gets the blanket dashboard `no-store` policy by inheritance and is slower than it needs to be — the safe mistake, cheap to notice and fix. The dangerous mistake, the one this design exists to prevent, is the reverse: a route that should have inherited `no-store` instead inherits a permissive default because it was added to the wrong router, a wrong base path, or a copy-pasted decorator — and a shared cache in front of it starts serving one authenticated user's response to the next request for the same URL path. That failure mode is not hypothetical; it's documented, repeatedly, in the case studies below.

### Case study — Web Cache Deception: caching what was never supposed to be cached

**What it is.** In February 2017, security researcher Omer Gil published and later presented at Black Hat USA a technique he named **Web Cache Deception**: an attacker crafts a URL that *looks* like it points at a static, obviously cacheable resource — appending a fake extension, e.g. `https://example.com/myaccount/home/nonexistent.css` — to a path that actually serves a dynamic, personalized page. A misconfigured server or shared cache, judging cacheability by the URL's apparent file extension rather than the actual response, stores the personalized page under that crafted URL. The attacker then requests the same crafted URL **from a private or unauthenticated session**, and if the shared cache serves the stored copy, the attacker now holds another user's cached personal data — Gil's own proof-of-concept, demonstrated against PayPal, retrieved account balances, transaction data, and partial card numbers this way. Two other named companies were affected in his original research; per his own writeup, he agreed not to name them at their request. All three fixed their issues before he published.

**Why this belongs in this section specifically:** the underlying failure isn't a CDN bug or an exotic protocol flaw — it's exactly the scope question this concept opened with, answered wrong by a system component instead of by a person. The page being cached should have carried `Cache-Control: private, no-store` (it's a personalized account page) and instead was treated as cacheable because *something* in the caching layer inferred cacheability from the URL's shape rather than from an explicit, authoritative directive. **The lesson isn't "watch out for a clever attack" — it's "cacheability decided implicitly, by inference, is a security control, whether you intended it to be one or not."** An explicit `private, no-store` on every response that can contain another user's data removes the entire attack class regardless of what any intermediate cache infers from the URL.

*Primary source:* Omer Gil, "Web Cache Deception Attack" (blog post, February 2017, and Black Hat USA 2017 whitepaper/presentation, `omergil.blogspot.com` and the Black Hat USA 2017 archive). **Verify current**: the specific vulnerable configurations he describes (older Apache/PHP path-handling combined with certain CDN cache-rule defaults) are largely fixed in modern default configurations, but the general pattern — cacheability inferred from URL shape rather than explicit headers — remains exploitable wherever it recurs, which the second case study below confirms is not rare.

### Case study — "Cached and Confused": Web Cache Deception in the wild, at scale

**What it is.** In 2020, researchers Seyed Ali Mirheidari, Sajjad Arshad, Kaan Onarlioglu, Bruno Crispo, Engin Kirda, and William Robertson (University of Trento, Northeastern University, and Akamai Technologies) published "Cached and Confused: Web Cache Deception in the Wild" at the 29th USENIX Security Symposium. Rather than one company's disclosure, this is a systematic study: the authors built tooling to test real, popular websites for exactly Gil's attack class and its variants, across real production CDN and cache configurations, and found the pattern to be widespread rather than a one-off misconfiguration at a handful of companies — a meaningfully larger and more rigorous sample than any single postmortem could provide.

**The engineering lesson this adds on top of Gil's original finding:** a vulnerability class that depends on "cache decides what's cacheable by inference" doesn't get fixed once — it gets *reintroduced* independently, by different teams, at different companies, using different frameworks and different CDNs, because the underlying design mistake (letting a URL's shape stand in for an explicit cacheability decision) is easy to fall into by default and easy to miss in review, exactly as concept 1's In Production section warned. This is the strongest evidence in this note for the general principle stated in the system design above: **an explicit, audited default (`private`/`no-store` for anything personalized) beats a policy that relies on every future engineer noticing an edge case that a URL-parsing heuristic somewhere in the stack might get wrong.**

*Primary source:* Mirheidari et al., "Cached and Confused: Web Cache Deception in the Wild," USENIX Security 2020 (paper and presentation available via usenix.org). **Verify current** before citing exact figures from the paper (number of vulnerable sites tested, percentage found affected) — cite the paper directly for those numbers rather than a secondhand figure, since this note is summarizing the finding's shape and significance rather than reproducing its dataset.

### In production — operating `Cache-Control`

**Best practices, in order of how much pain they save:**
1. **Set `Cache-Control` explicitly on every response that matters, rather than relying on heuristic freshness or a framework default.** The 10% heuristic is a real, specified fallback for when nobody set anything — it should never be your actual caching strategy.
2. **Default anything touching authenticated or per-user data to `private, no-store`, and require an explicit, reviewed decision to loosen it** — the direct operational takeaway from both case studies above.
3. **Use `s-maxage` deliberately whenever a CDN sits in front of you**, rather than letting the CDN's own default heuristics decide how long to hold your responses — you almost always know your own data's actual change frequency better than a generic default does.
4. **Reach for `stale-while-revalidate` before reaching for a shorter `max-age`** on read-heavy, tolerant-of-a-few-seconds-staleness endpoints — it gets you both fast responses and bounded staleness, where a short `max-age` alone only gets you the latter, at the cost of more origin traffic.
5. **Never mark an error response cacheable by accident.** Explicitly set `Cache-Control: no-store` (or at minimum a very short `max-age`) on `5xx` paths in your framework's default error handler, because RFC 9111's default cacheable-status-code list includes several that will surprise you if you haven't checked it.

**Mistakes, beginner → senior:**
- *Beginner:* believing `no-cache` means "do not cache" and using it where `no-store` was meant, resulting in a personalized response sitting in a shared cache's storage (though not served without revalidation) when it should never have been stored at all.
- *Intermediate:* setting `max-age` without `s-maxage` and being surprised that a CDN in front of the service holds responses far longer than a browser would — or the reverse, setting only `s-maxage` and being surprised browsers revalidate on every load.
- *Senior:* trusting a shared cache's default cacheability inference for anything (Web Cache Deception's exact root cause) instead of setting explicit directives everywhere personalization is possible, including on error paths and redirects that "surely don't matter."

**Monitoring:** track the distribution of `Cache-Control` directives actually being sent across your endpoints (a quick audit tool, not a manual review) to catch endpoints with no explicit policy at all; alert on any authenticated-route response observed without `private` or `no-store`; monitor shared-cache hit ratios *segmented by route class* so a dashboard route accidentally getting cached shows up as an anomaly (unexpectedly high hit ratio where you expect near-zero) rather than going unnoticed.

**Cost:** `s-maxage` set too long trades correctness lag for CDN cost savings — same shape as Day 12's TTL cost trade-off, just priced in CDN egress and request-count billing instead of DNS query volume. `stale-while-revalidate` is nearly free in cost terms (it saves origin requests compared to synchronous revalidation) and should be treated as a default good idea wherever staleness of a few tens of seconds is tolerable, rather than something reached for only under load pressure.

---

## 3. Validators and conditional requests: ETag, Last-Modified, 304

**Depth: [CORE]**

### Intuition

Concept 2's entire mechanism has a hole in it, and it's worth stating precisely before patching it. `max-age` answers "how long may I trust this without asking," but it can't answer a different, equally important question: **once that time is up, is the content actually different, or did it just happen to reach its expiry?** Most of the time, for most cacheable content, the honest answer is "nothing changed" — a product's price is still the price, a CSS file compiled yesterday is still that file, an article published this morning is still that article. If the only tool you have is "expire and re-fetch the whole thing," you throw away the entire response and pay full price for a byte-for-byte identical answer, purely because a clock ran out.

A **validator** is the fix: a compact piece of evidence, attached to a response, that lets the *client* ask the origin "has this actually changed since the version I'm holding?" without asking the origin to send the whole thing again to find out. If the answer is no, the origin says so in a tiny reply with **no body at all** — that's the `304 Not Modified` status code Day 13 already introduced as one of the status codes in HTTP's vocabulary; this section is where you learn the mechanism that produces it and the header machinery that drives that mechanism.

### Before ETag: `Last-Modified`, and the two problems that motivated something better

HTTP/1.0 shipped exactly one validator: `Last-Modified`, a timestamp the origin attaches to a response, and `If-Modified-Since`, the header a client echoes back on its next request to ask "has this changed since the timestamp you gave me?" It works, and it's still supported and still useful — but RFC 7232, the specification that formalized conditional requests for HTTP/1.1, exists partly because two real limitations of a timestamp-only approach kept causing production bugs:

1. **One-second granularity.** `Last-Modified` timestamps are specified to whole-second resolution. A resource that's regenerated and changed *twice within the same second* — entirely plausible for a build pipeline or a high-frequency data feed — produces two genuinely different versions of the content carrying the *identical* `Last-Modified` value, and a client asking "has this changed since this timestamp" gets told "no" about content that, in fact, did change.
2. **Clock dependence, again.** Exactly the problem concept 2 flagged for `Expires`: a timestamp-based comparison across machines is only as trustworthy as those machines' clocks agreeing, and a distributed system with dozens of servers behind a load balancer is exactly the setting where clock skew, however small, becomes a real source of intermittent, maddening bugs — "it works when I test it, and fails one time in twenty in production" is the signature of a race against clock skew.

An **ETag** ("entity tag") sidesteps both problems by not being a timestamp at all — it's an **opaque identifier for a specific version of a representation**, and the specification puts no requirement on how it's generated beyond "identical content ⇒ identical tag is expected, and different content ⇒ a client should be able to tell the difference." In practice this is almost always a hash of the response body (this note will use one directly, in the runnable example) or, in simpler cases, a version number or a database row's revision counter. Because it's a value the origin computes and controls completely, it dodges the granularity problem (a hash changes the instant the underlying bytes change, at any resolution) and the clock problem (no clock is involved anywhere in the comparison) simultaneously.

### Strong versus weak validation, and why the distinction exists

RFC 7232 §2.1 defines two ways two ETags can be compared. A **strong comparison** requires the tags to match *and* neither to be marked weak — it certifies the two representations are byte-for-byte identical, which is what you need before, say, resuming a partial download or satisfying a `Range` request safely. A **weak comparison**, signaled by prefixing the tag with `W/` (e.g. `W/"abc123"`), only certifies the two representations are *semantically equivalent* — good enough to skip re-sending a page whose content is unchanged even if, say, a footer timestamp or whitespace differs byte-for-byte. Weak validators exist because generating a strong one — a real hash over the exact bytes about to be sent — sometimes costs more than the origin wants to pay for content where "close enough" is genuinely fine; they trade a small amount of caching precision for a cheaper validator to compute. This note's runnable example generates a strong, content-hash-based ETag, because for the case it's teaching — bandwidth savings for programmatic API consumers — byte-exact certainty is the more useful property and the cost of a hash is trivial next to a database round trip.

**A related but genuinely different mechanism, named and not opened here:** RFC 7232 also defines `If-Match` and `If-Unmodified-Since`, which use the same validators to solve a completely different problem — preventing a *write* from silently clobbering someone else's concurrent write (the "lost update" problem: you fetch a record, someone else updates it, you save your own stale copy over their change without realizing). That's an optimistic-concurrency-control mechanism, not a caching mechanism, and while it shares RFC 7232's plumbing, it belongs conceptually with database and API write-safety topics rather than with caching — this note stops here and names it rather than opening it, because opening it well would mean teaching optimistic concurrency control properly, which isn't this note's job.

### Worked example — a full conditional-request exchange, with real values

Trace a client that already holds a cached copy of a JSON document, checking whether it's still current, using the exact hash values this note will reuse in the runnable example below (computed with real Python, not invented for illustration):

**Request 1 — the client's first-ever fetch of this resource:**
```
GET /document HTTP/1.1
Host: api.example.com
```

**Response 1 — the origin serves the full body and attaches a validator:**
```
HTTP/1.1 200 OK
Content-Type: application/json
ETag: "929e0bb0dc656bb73b150d29e28da361c0b4b996d75d966d6232c1bcb005fad3"
Cache-Control: public, max-age=60
Content-Length: 211

{"body": "...", "id": 42, "title": "Day 15 notes", "version": 1}
```

The client stores the body *and* the `ETag`. That pairing — content plus the validator that identifies it — is what makes the next step possible.

**Request 2 — arrives after `max-age=60` has elapsed, so the client can no longer simply reuse the cached copy without checking. Instead of an unconditional `GET`, it sends a conditional one:**
```
GET /document HTTP/1.1
Host: api.example.com
If-None-Match: "929e0bb0dc656bb73b150d29e28da361c0b4b996d75d966d6232c1bcb005fad3"
```

`If-None-Match` carries the client's stored ETag and asks, in effect: "send me the full representation *unless* it still matches this tag — in which case, just tell me so." The name reads backwards from how people usually phrase the intent ("give me this if it's changed"), which is worth internalizing now so it doesn't trip you up later: the header is phrased as the *negative* condition on the tag matching.

**Response 2a — nothing changed:**
```
HTTP/1.1 304 Not Modified
ETag: "929e0bb0dc656bb73b150d29e28da361c0b4b996d75d966d6232c1bcb005fad3"
Cache-Control: public, max-age=60
```

No `Content-Length`, no body — RFC 7232 §4.1 is explicit that a `304` response must not include a message body, and must repeat the small set of headers (`Cache-Control`, `Content-Location`, `Date`, `ETag`, `Expires`, `Vary`) that a cache needs to *refresh the freshness metadata of its stored copy* without needing new content to go with it. The client's existing 211-byte body is still correct; only its "how long may I trust this" clock got reset.

**Response 2b — the underlying document actually changed (e.g. `version` incremented from 1 to 2):**
```
HTTP/1.1 200 OK
Content-Type: application/json
ETag: "b9f3695e09f69e37af7ebbc2ef903031c8cc10c7a695557a42e61370f0708c07"
Cache-Control: public, max-age=60
Content-Length: 211

{"body": "...", "id": 42, "title": "Day 15 notes", "version": 2}
```

A completely different ETag, computed from the new content, and a full body — because the tag didn't match, the condition on `If-None-Match` was true (the stored representation does *not* still match), so the server serves the current representation in full, exactly as it would to an unconditional `GET`.

**The property to take away, stated as sharply as possible:** conditional requests don't reduce how *often* a client has to talk to the origin — the client still sends a request every time it wants to be sure. What they reduce is **how many bytes the origin has to send back when the honest answer is "nothing changed."** That's a categorically different saving than `max-age`'s "skip the round trip entirely," and the two compose: `max-age` skips the conversation for a while, and once that expires, a conditional request keeps the conversation cheap for as long as the content stays the same.

### The agentic dimension: conditional requests save tokens, not just bytes

Here is where this concept's mechanism transfers directly to a cost that matters more in agentic systems than raw bandwidth usually does: **context-window tokens.** An agent with a web-fetching tool that re-reads the same URL across multiple turns of a long-running task — checking a status page, monitoring a document for edits, polling an API a tool wraps — faces the identical "did anything change" question this section just answered for browsers and CDNs, except the cost of a wasted `200` isn't merely bytes on a wire, it's tokens the model has to read on every single re-fetch, whether or not the content actually changed. A fetch tool that stores the `ETag` from its last read of a URL and sends `If-None-Match` on the next one gets a `304` with an empty body on the common case — costing the model's context window nothing beyond a short "unchanged" note the tool can substitute for the real body — instead of re-feeding an unchanged multi-kilobyte page into the prompt every single turn. This is a genuine, mechanical design choice for anyone building a "fetch" or "browse" tool for an agent, not a stretch: implement it, and a long-running agent's cost curve on repeated reads of the same slowly-changing resource looks completely different.

### Runnable example — real ETag generation and a real `304` in FastAPI, measured with `curl`

This is the study plan's core build task, run for real rather than described. The endpoint below computes a strong, content-hash ETag over its own response body and honors `If-None-Match` by returning an empty `304` when the hash matches.

```python
# etag_demo.py — real ETag / conditional-GET handling in FastAPI.
# pip install fastapi uvicorn
import hashlib
import json

from fastapi import FastAPI, Request, Response

app = FastAPI()

# Demo only: a fake "document" this endpoint serves. In a real service this
# would be a row fetched from your database on every request.
_document = {"id": 42, "title": "Day 15 notes", "body": "..." * 50, "version": 1}


def render_body(doc: dict) -> bytes:
    # sort_keys=True matters more than it looks: an UNSTABLE serialization
    # (dict key order varying between calls, or a freshly-generated
    # timestamp embedded in the body) would give every request a NEW hash
    # even when nothing meaningful changed, defeating the entire point of
    # a content-hash validator.
    return json.dumps(doc, sort_keys=True).encode("utf-8")


def compute_etag(body: bytes) -> str:
    # A STRONG validator: byte-for-byte identity of the representation.
    # sha256 avoids the load-balancer inconsistency that filesystem-based
    # ETags (inode + mtime + size) suffer under horizontal scaling — see
    # the Judgment section below for exactly why that matters.
    digest = hashlib.sha256(body).hexdigest()
    return f'"{digest}"'


@app.get("/document")
async def get_document(request: Request):
    body = render_body(_document)
    etag = compute_etag(body)

    # RFC 7232 §3.2: If-None-Match may list several ETags, comma separated,
    # or "*". A real intermediary in front of you may also apply weak vs.
    # strong comparison differently. This demo handles the common case —
    # exact match against the client-supplied tag(s).
    inm = request.headers.get("if-none-match")
    if inm is not None and etag in [t.strip() for t in inm.split(",")]:
        # RFC 7232 §4.1: a 304 carries NO BODY, but MUST repeat the
        # cache-relevant headers so the client can refresh its stored
        # copy's metadata without re-fetching the representation.
        return Response(
            status_code=304,
            headers={"ETag": etag, "Cache-Control": "public, max-age=60"},
        )

    return Response(
        content=body,
        media_type="application/json",
        headers={"ETag": etag, "Cache-Control": "public, max-age=60"},
    )


@app.post("/document/bump")
async def bump_version():
    _document["version"] += 1
    return {"version": _document["version"]}
```

**Honesty caveat, stated up front rather than discovered the hard way:** FastAPI and Starlette give you *nothing* for this automatically on a normal `async def` path function — no framework-computed ETag, no automatic `304` handling. (Starlette's `StaticFiles`, used for serving static assets directly off disk, *does* compute `Last-Modified` and honor conditional requests out of the box — but the moment your response is dynamically generated, as this one is, you're on your own, and that gap is exactly what the code above fills in by hand.)

Run it:
```
$ pip install fastapi uvicorn
$ uvicorn etag_demo:app --port 8000
```

And the actual transcript — every value below is real output from running this exact code, not invented for illustration:

```
$ curl -i http://127.0.0.1:8000/document
HTTP/1.1 200 OK
etag: "929e0bb0dc656bb73b150d29e28da361c0b4b996d75d966d6232c1bcb005fad3"
cache-control: public, max-age=60
content-length: 211
content-type: application/json

{"body": "......................................................................................................................................................", "id": 42, "title": "Day 15 notes", "version": 1}

$ curl -i http://127.0.0.1:8000/document \
    -H 'If-None-Match: "929e0bb0dc656bb73b150d29e28da361c0b4b996d75d966d6232c1bcb005fad3"'
HTTP/1.1 304 Not Modified
etag: "929e0bb0dc656bb73b150d29e28da361c0b4b996d75d966d6232c1bcb005fad3"
cache-control: public, max-age=60

$ curl -s -o /dev/null -w 'status=%{http_code} bytes=%{size_download}\n' http://127.0.0.1:8000/document
status=200 bytes=211
$ curl -s -o /dev/null -w 'status=%{http_code} bytes=%{size_download}\n' http://127.0.0.1:8000/document \
    -H 'If-None-Match: "929e0bb0dc656bb73b150d29e28da361c0b4b996d75d966d6232c1bcb005fad3"'
status=304 bytes=0

$ curl -s -X POST http://127.0.0.1:8000/document/bump
{"version":2}
$ curl -i http://127.0.0.1:8000/document \
    -H 'If-None-Match: "929e0bb0dc656bb73b150d29e28da361c0b4b996d75d966d6232c1bcb005fad3"'
HTTP/1.1 200 OK
etag: "b9f3695e09f69e37af7ebbc2ef903031c8cc10c7a695557a42e61370f0708c07"
cache-control: public, max-age=60
content-length: 211
content-type: application/json

{"body": "......................................................................................................................................................", "id": 42, "title": "Day 15 notes", "version": 2}
```

**Why this works, line by line, and what the numbers prove:**

- The first `GET` costs **211 bytes**. The conditional `GET` after it costs **0 bytes**, a **100% reduction on that poll**, because the underlying document hadn't changed. This is a deliberately small document so the hash and header logic are easy to read — on a real API response of, say, 50 KB polled every few seconds by a monitoring client, the exact same mechanism turns a 50 KB transfer into a sub-200-byte one (headers only) on every poll where nothing changed, which is the entire economic argument behind the GitHub API case study below.
- `json.dumps(doc, sort_keys=True)` is the least glamorous line in the file and the one most likely to be skipped by someone writing this quickly — and skipping it is exactly the kind of bug that makes ETags *appear* broken (every request gets a `200`, never a `304`) for a reason that has nothing to do with HTTP and everything to do with non-deterministic serialization.
- `compute_etag` hashes the exact bytes about to be sent, so the ETag is defined as "identical to this representation" by construction — there's no separate bookkeeping to keep in sync with the body, which is the property that makes this validator trustworthy.
- The `if inm is not None and etag in [...]` check is doing the actual conditional-request logic RFC 7232 describes: compare the client's presented tag(s) against the current tag, and only take the "nothing changed" branch on an exact match.
- After `POST /document/bump` changes `version` from 1 to 2, the *same* `If-None-Match` value that produced a `304` a moment earlier now produces a fresh `200` with a **different** ETag — because the underlying bytes genuinely changed, and the hash reflects that automatically, with zero additional code to detect "did the content change."

### Judgment: a content-hash ETag versus a filesystem-based one, for a horizontally scaled origin

**The choice:** you're deciding how to generate ETags for an API served by many identical application server replicas behind a load balancer — hash the actual response body on every request (as the runnable example does), or use the cheaper, built-in behavior most web servers ship by default, which derives an ETag from the served file's inode number, modification time, and size.

**The realistic alternative, and it's genuinely the *default*, not a strawman:** Apache and nginx both historically generate ETags for static files this way out of the box, because it's essentially free — no hashing, just metadata the filesystem already tracks.

**The actual reason a content hash wins the moment you scale horizontally:** an inode number is a property of *one specific machine's filesystem* — it is not preserved when the identical file is deployed to a second server, even from the exact same build artifact. A client bouncing between two replicas behind a load balancer (ordinary, expected behavior, not a misconfiguration) receives **two different ETags for byte-identical content**, purely because the two servers assigned the file different inode numbers. The client's `If-None-Match` then never matches whichever replica answers next, every conditional request comes back a full `200`, and the entire mechanism this section just built silently stops paying off — not because anything is broken in an alarming way, just quietly, permanently defeated. A content hash has no such dependency: the same bytes, hashed the same way, produce the same tag on every replica, forever, with no coordination between servers required at all.

**The trade-off, honestly:** hashing costs CPU on every request, proportional to the response size — for a large response served extremely frequently, that's real, measurable work that filesystem metadata costs nothing to produce. It's also strictly more code to write and maintain than "let the web server's static file handler do it," which is why the inode-based default persists in so many stacks: it's genuinely the right choice for a single-server setup serving static files, where the inconsistency problem literally cannot occur.

**The flip condition:** stay with cheap, framework/webserver-default ETags for genuinely static files served from a single origin or from storage with stable, content-addressed identifiers (this is effectively what CDN-served, versioned URLs in concept 6 already guarantee by construction — the filename itself is a hash, so the ETag question becomes almost moot). Move to explicit content hashing the moment either of two things becomes true: the response is dynamically generated (there's no file and no inode to begin with, as in the runnable example above), or the response is served from more than one machine that can independently compute different metadata for logically identical content.

### System design — conditional requests for a bandwidth-metered mobile API client

**Problem:** a mobile app polls `GET /feed` every 30 seconds while in the foreground to check for new content, over connections where users are frequently on metered mobile data. The feed changes on the order of a few times per hour, far less often than the poll interval. Design the conditional-request behavior for this endpoint.

**Requirements:** minimize bytes transferred on the overwhelmingly common case (nothing new since the last poll); never miss genuinely new content; work correctly even though the client's local clock cannot be trusted (mobile devices routinely have clock skew, a fact you should already expect after Day 12 and concept 2's `Expires` discussion).

**The alternatives:** (1) no conditional logic — always return the full feed on every 30-second poll; (2) `Last-Modified` / `If-Modified-Since`, keyed on the feed's most recent item's timestamp; (3) an `ETag` computed over a cheap fingerprint of the feed's current state (e.g., a hash of the most recent item's ID plus a monotonic version counter maintained server-side), returned with `If-None-Match` handling identical in shape to the runnable example above.

**The decision: (3).**

**The actual reason:** option (2)'s clock dependency is exactly the historical problem this concept opened with, and it's worse on mobile than on a data-center server, because you have far less control over and far less confidence in a random phone's clock than you do over your own infrastructure's. Option (3) sidesteps that: the server maintains its own monotonic "current feed version" (a counter incremented on every new post, or a hash of the latest item's ID), completely independent of any client's notion of time, and the ETag reflects that server-side truth directly. The overwhelming majority of the app's 30-second polls — the ones where nothing new has posted — cost the user a few hundred bytes of headers and no feed payload at all, which matters concretely on a metered connection.

**The trade-off:** computing and comparing a fingerprint on every request is more server-side logic than "just always return the feed," and if the fingerprint computation itself is expensive (imagine hashing the *entire* feed body rather than a cheap monotonic counter), you've spent real CPU to save network bytes — a trade that's obviously worth it when the fingerprint is cheap (as here — a counter comparison, not a hash of kilobytes of content) and needs re-examining if it isn't.

**The flip condition:** if the feed changed on nearly every poll anyway (a live sports score ticker updating every few seconds), conditional requests would save almost nothing — you'd be paying the fingerprint-comparison cost on every request and getting a `304` on almost none of them — and a plain, unconditional response (or a push-based mechanism entirely outside HTTP polling, like a WebSocket or Server-Sent Events connection) becomes the better design. Conditional GET earns its keep specifically in the gap this problem describes: polling interval materially shorter than actual change frequency.

### System design — ETag generation strategy for a horizontally scaled origin

**Problem:** the same product-detail API from concept 1's worked example is now served by twelve application server replicas behind a load balancer, has no shared filesystem, and needs ETags that remain valid regardless of which replica answers any given request. Design the ETag generation strategy.

**Requirements:** a client's cached ETag from replica #3 must still match when replica #9 answers the next request for the identical, unchanged resource; ETag computation must not become a new bottleneck under load; the design must not require the twelve replicas to coordinate with each other on every request.

**The alternatives:** (1) each replica generates ETags from its own process memory or local state (a per-process counter, a local cache timestamp); (2) a content hash computed independently by each replica over the exact response bytes, with no cross-replica coordination required at all; (3) a centrally assigned version identifier, written once when the underlying data changes (e.g., a `version` or `updated_at` column bumped on write) and read by every replica from the shared database.

**The decision: (2) for the common case, converging with (3) in practice because the input to the hash usually comes from data that already carries a version marker like the one in (3).**

**The actual reason:** this is the Judgment section's lesson applied at system scale rather than single-endpoint scale — option (1) reproduces the exact inode-based failure mode by a different mechanism: any per-replica state that isn't derived purely from the shared underlying data will disagree between replicas by construction, and disagreement is silent (no error, just permanently-defeated caching). Option (2) requires *zero* coordination between replicas precisely because a hash function is deterministic: feed it the same bytes on any of the twelve machines and it produces the same tag, with no network call, no shared cache, no consensus protocol needed. In practice this is rarely "hash the whole rendered response from scratch" (option 2 read narrowly) — it's usually hashing (or directly reusing) whatever version marker the database already stores per option (3), because that marker is itself already guaranteed consistent across replicas by the database being the single source of truth, and hashing it (or a small set of fields) is far cheaper than hashing an entire rendered page.

**The trade-off:** relying on a database-supplied version marker means the ETag is only as fine-grained as that marker — if the marker only updates on a full row change but the rendered response also depends on a joined table or a feature flag, the ETag can go stale relative to what's actually served (a false `304` for content that did, in a sense, change) unless every input to the response is folded into the fingerprint. Hashing the fully-rendered body sidesteps that risk entirely at the cost of the CPU work the Judgment section already flagged.

**Flip condition:** favor the cheap version-marker approach when the response's content is governed by one clear, already-versioned source of truth; fall back to hashing the full rendered body when a response is assembled from several independently-changing sources (a product row, an inventory count, a promotional-pricing rule) and no single marker can be trusted to reflect all of them changing.

**Failure modes:** a load balancer with session affinity ("sticky sessions") can mask this entire class of bug in testing — if your test client always happens to hit the same replica, per-replica ETag inconsistency never surfaces until a real, unpinned client bounces between replicas in production. Test conditional requests specifically *without* sticky sessions before trusting the result.

### Case study — GitHub's REST API: conditional requests as a first-class rate-limiting exemption

**What it is.** GitHub's REST API documentation explicitly instructs API consumers to use `ETag` (and, for some endpoints, `Last-Modified`) headers for conditional requests, and states plainly that a `304 Not Modified` response **does not count against the primary rate limit** that otherwise governs how many requests an authenticated client may make per hour. A client polling an endpoint that hasn't changed pays a small, separate "secondary" cost for the conditional check itself, but preserves essentially all of its regular request budget for calls that actually return new data.

**Why this is the positive case, worth sitting alongside the two failure-shaped case studies elsewhere in this note:** it's a rare example of a company making the *economic* incentive for conditional requests completely explicit and directly financial to the API consumer, rather than leaving bandwidth savings as a diffuse, easy-to-ignore benefit. A team polling GitHub's API for changes on a schedule has a concrete, immediate reason to implement `If-None-Match` correctly — doing so multiplies their effective rate limit by however often the resource is actually unchanged, which for most polling workloads is the overwhelming majority of requests.

*Primary source:* GitHub Docs, "Best practices for using the REST API" (docs.github.com/rest/guides/best-practices-for-using-the-rest-api), the section on conditional requests. **Verify current**: the exact rate-limit numbers (requests per hour for authenticated vs. unauthenticated clients, and the specific cost of a `304` against the secondary limit) are the kind of provider-specific figures explicitly flagged as fast-drifting by this note's own conventions — check the current docs before relying on a specific number.

### Case study — ETag inconsistency across replicas: a widely documented pattern, not one company's postmortem

**What it is, and why it's presented differently from the other case studies in this note:** unlike the Fastly and Web Cache Deception incidents elsewhere in this note, there is no single named company's public postmortem for "our load-balanced servers issued inconsistent ETags and broke our own caching" to cite here — this is, honestly, a pattern documented across web-server manuals and performance-engineering guidance rather than one dramatic incident, and Principle 6/7 of how this note is written means saying so plainly rather than manufacturing a fictional company to hang it on. What *is* real and citable: Apache httpd's own documentation for the `FileETag` directive explicitly exists because of this exact problem — it lets an administrator configure ETag generation to *exclude* the inode component specifically for deployments across multiple servers, which is the vendor's own acknowledgment that the default behavior breaks under horizontal scaling. Web performance guidance (including Google's web.dev caching documentation) independently recommends against relying on default filesystem-derived ETags in any multi-server deployment for the identical reason developed in this concept's Judgment section.

**The lesson, stated once rather than twice:** this is the same failure this concept's Judgment section already walked through in full — it's included here as a case study specifically to make the point that it is not a hypothetical concern invented for this note, but a documented-enough problem that a major web server shipped a configuration directive purely to work around it.

*Primary source:* Apache HTTP Server documentation, `mod_headers`/core `FileETag` directive reference (httpd.apache.org); Google, "Prevent unnecessary network requests with the HTTP Cache" (web.dev/articles/http-cache). **Verify current**: default ETag composition has changed across Apache and nginx versions over the years — check your specific server version's current default rather than assuming inode inclusion is still the out-of-the-box behavior everywhere.

### In production — operating validators and conditional requests

**Best practices:**
1. **Hash the response body directly for anything dynamically generated**, and reuse an existing, already-consistent version marker (a database row version, a build hash) for anything backed by one — never derive an ETag from per-process or per-machine state.
2. **Always return the small set of RFC 7232 §4.1 headers on a `304`**, not just the status code — a `304` missing `Cache-Control` or `ETag` leaves the requesting cache with no way to refresh its own freshness bookkeeping, defeating half the point.
3. **Test conditional-request correctness with sticky sessions disabled**, per the failure mode above — it's the single most common way this silently ships broken.
4. **Pair validators with a freshness lifetime (`max-age`), don't use one instead of the other.** A validator without any `max-age` means every single request, even ones a millisecond apart, triggers a full conditional round trip to the origin — you've built the "always ask, cheaply" behavior of `no-cache` by accident, when a short `max-age` plus a validator would have skipped the round trip entirely for genuinely back-to-back requests.

**Mistakes, beginner → senior:**
- *Beginner:* generating an ETag from something that changes on every request even when the content doesn't (a fresh timestamp embedded in a template, non-deterministic dict ordering in a serializer) and being confused why every request returns `200`.
- *Intermediate:* forgetting `If-Match`/`If-Unmodified-Since` exist for write-safety and instead building an ad hoc "check-then-update" pattern that has exactly the lost-update race those headers were designed to close — a real gap, correctly left unopened by this note per its stated scope, but worth knowing where to look when the need arises.
- *Senior:* an ETag strategy that's individually correct per-service but inconsistent across a fleet of internal services with different teams and different frameworks, so some endpoints support conditional requests properly and others silently don't, and nobody has an inventory of which is which.

**Monitoring:** the `304`-to-`200` ratio per endpoint is the direct measure of how much this mechanism is actually saving you — track it the same way you'd track cache hit ratio, because conceptually it's the identical kind of number; a validator-enabled endpoint sitting at a near-zero `304` rate under a polling workload is a strong signal something's generating inconsistent ETags.

**Cost:** hashing is CPU, paid on the origin, in exchange for bandwidth saved on the wire and (concept 1) load kept off the origin on a cache hit — for content behind a CDN or reverse-proxy cache that already re-validates on the CDN's behalf, the origin pays this cost far less often than raw client traffic would suggest, because the CDN itself absorbs most of the conditional-request traffic once its own copy is confirmed still fresh.

---

## 4. `Vary`: when the cache key needs more than the URL

**Depth: [WORKING]**

### Intuition

Every cache discussed so far has quietly assumed the cache key is just the method and the URL — ask for `GET /document` twice, get the same stored answer both times. That assumption breaks the moment a single URL can legitimately produce **more than one correct response**, depending on something about the request other than the URL itself. The clearest everyday case: your server can send a response either compressed (`gzip`) or uncompressed, depending on whether the client says `Accept-Encoding: gzip` — same URL, two legitimately different byte sequences on the wire, and a cache that doesn't know to tell them apart will happily serve a gzip-compressed body to a client that never said it could decompress one, or vice versa, waste bandwidth serving uncompressed bytes to a client that could have taken the smaller version.

`Vary` is the origin's way of telling every cache downstream **which additional request headers must match** before a stored response can be reused — it extends the cache key from "method + URL" to "method + URL + the values of these named headers," and it exists because without it, a shared cache has no principled way to know that `GET /document` sent by a French-speaking browser and `GET /document` sent by an English-speaking one are asking for genuinely different content, not the same content twice.

### Analogy: a librarian who sometimes needs to ask a follow-up question — and where it breaks

Normally a librarian fetches a book by title alone. For a book published in several language editions, they need one more piece of information before handing over the right copy: "which language?" `Vary: Accept-Language` is the origin telling the librarian "for this title, always ask that follow-up question before you decide which copy on the shelf is the right one to hand over."

**Where this breaks, and it's the load-bearing part:** a library has a small, fixed, known set of language editions — asking "which language" partitions the shelf into a handful of clearly bounded groups. An HTTP header's possible values are often unbounded. `Vary: User-Agent` doesn't ask "which of five editions" — it asks a question with, in practice, thousands of distinct answers, because browser and device user-agent strings vary combinatorially across browser version, OS version, and device model. The librarian analogy has no equivalent for a "which edition" question with five thousand possible answers, almost all of which describe requesters who would have been perfectly happy with the *same* edition — and that gap is exactly the production failure mode below.

### Worked example — one URL, `Vary: Accept-Encoding`, two stored variants

```
Request A:
  GET /app.js HTTP/1.1
  Accept-Encoding: gzip, br

Response A:
  HTTP/1.1 200 OK
  Content-Encoding: gzip
  Vary: Accept-Encoding
  Cache-Control: public, max-age=3600
  Content-Length: 41200
  <41,200 gzip-compressed bytes>

Request B (a client/proxy that doesn't advertise gzip support):
  GET /app.js HTTP/1.1
  Accept-Encoding: identity

Response B:
  HTTP/1.1 200 OK
  Content-Encoding: identity
  Vary: Accept-Encoding
  Cache-Control: public, max-age=3600
  Content-Length: 118000
  <118,000 uncompressed bytes>
```

A cache honoring `Vary: Accept-Encoding` stores **both** as distinct entries under the same URL, keyed additionally on the `Accept-Encoding` value each request presented, and serves each future request the variant matching its own `Accept-Encoding` — Request A's requester always gets the 41.2 KB gzip variant, Request B's always gets the 118 KB uncompressed one, and neither is ever served the other's copy. The cache key, made explicit, is effectively `("GET", "/app.js", accept_encoding_value)` rather than just `("GET", "/app.js")`.

### In production — `Vary` in practice

**The one thing to internalize:** every additional header named in `Vary`, if it has more than a handful of realistic distinct values, multiplies your effective number of cache entries for that URL — and a cache miss on a variant that's never been seen before behaves exactly like a miss on a brand-new URL, paying the full origin round trip. `Vary: Accept-Encoding` is safe because `Accept-Encoding` has a small, bounded set of realistic values (`gzip`, `br`, `identity`, and a few combinations). `Vary: Accept-Language` is usually safe for the same reason — a handful of supported locales. `Vary: User-Agent` and `Vary: Cookie` are the two directives that most reliably cause quiet, hard-to-diagnose cache-hit-ratio collapse, because both have effectively unbounded cardinality in practice.

**Mistakes, beginner → senior:** *beginner* — forgetting `Vary` entirely on a response that legitimately differs by a request header, and shipping a bug where the wrong language or the wrong compression gets stuck in cache for the next requester; *senior* — adding `Vary: User-Agent` (often to work around a device-specific rendering difference) without realizing it silently fragments the cache into thousands of near-empty entries, each essentially a permanent cache miss, and then spending a debugging session on "why did our CDN hit ratio collapse after this one header change" before finding it.

### Case study — `Vary: User-Agent` and CDN cache fragmentation

**What it is.** This is a widely-documented operational pattern rather than a single named incident: teams add `Vary: User-Agent` to work around a genuine device-specific difference in a response, and CDN cache-hit ratios for that route collapse, because — as this concept's analogy predicted — a real User-Agent string space has on the order of thousands of practically distinct values, so a cache keyed on it stores what is effectively a separate entry per visitor rather than per meaningfully-different-response. Cloudflare's own documentation states plainly that a response carrying `Vary: *` is never cacheable at all, and recommends routing genuinely high-cardinality, per-user, or "don't want this cached" signals through a cache-bypass rule rather than through `Vary` — an explicit acknowledgment, from a CDN operator, that not every header belongs in this directive. The general engineering guidance across CDN vendors converges on the same fix: where a response genuinely needs to differ by client capability rather than by literal device identity, vary on a narrower, low-cardinality signal instead (a normalized "is this a mobile client" flag, or a small, explicit set of device classes) rather than the raw User-Agent string.

*Primary source:* Cloudflare, "Vary" (Cloudflare Cache/CDN docs, developers.cloudflare.com/cache/concepts/vary) for the `Vary: *` bypass behavior; general CDN vendor guidance (Google Cloud CDN caching documentation; community-documented incidents such as the Adobe AEM project archetype's own tracked issue recommending removal of `User-Agent` from `Vary` to restore CDN hit ratio) for the fragmentation pattern itself. **Verify current**: exact cardinality figures for User-Agent strings and specific CDN vendor default behaviors around `Vary` drift as browsers change UA string policies (several major browsers have been actively *reducing* User-Agent granularity in recent years specifically to shrink this kind of fingerprinting surface) — check current guidance before quoting a specific number.

---

## 5. CDN mechanics: edge PoPs, cache keys, origin shielding

**Depth: [CORE]**

### Intuition

Everything in concepts 1 through 4 works for a single reverse-proxy cache sitting in front of one origin, and that's a completely legitimate, real thing to build (nginx or Varnish, one machine or a small cluster, in front of your application). A **CDN** (content delivery network) is the same set of rules — `Cache-Control`, validators, `Vary` — operated at a different scale of geography entirely: instead of one cache near your origin, you rent space in **hundreds of physical locations around the world**, each holding its own copy of your cacheable content, positioned near *users* rather than near your servers. The motivating fact is the one concept 1's worked example already quantified: round-trip latency is dominated by physical distance, so the only way to make a cross-ocean request feel local is to put a cache at the ocean's edge instead of trying to make the ocean crossing faster — you cannot beat the speed of light, so you stop needing to.

### The vocabulary: PoP, edge, anycast, origin

A **PoP** (point of presence) is one physical location in a CDN's network — a data center or a colocation facility holding some number of the CDN's edge servers, with its own connectivity to nearby ISPs. A large CDN operates hundreds of these globally; a request from a user in Mumbai should reach a PoP in or near Mumbai, not one in Virginia. The **edge server** is the actual cache serving traffic from that PoP; **origin** is this note's (and the industry's) term for *your* server — the one that generates the response the first time, before any edge has cached it.

**How does a request "reach the PoP in or near Mumbai" without you configuring per-user routing?** The same mechanism Day 12 concept 1 already taught you for DNS's root servers: **anycast** — the same IP address is advertised from every PoP simultaneously via BGP, and normal internet routing (Day 10) delivers each user's packets to whichever advertisement is topologically closest, with no application-level geolocation logic involved at all. A CDN's edge network and the DNS root server system solve an identical problem — "get a client to the nearest of many identical, geographically distributed servers, using the network's own routing rather than a lookup" — with the identical mechanism, which is worth noticing explicitly rather than re-deriving: **this is the same anycast you already understand**, applied to serving HTTP responses instead of DNS answers.

### Cache keys at CDN scale, and why they need more care than "the URL"

Concepts 1–4 already established that a cache key is `method + URL (+ Vary'd headers)`. At CDN scale, getting this exactly right matters more, because a CDN operator's default cache-key behavior is a genuine design choice with real consequences in both directions:

- **Too broad a key** (ignoring something that actually changes the response) causes the failure this whole note has been building toward avoiding: two different users' different content collapsing into one cache entry. A cache key that ignores an `Accept-Language` header on a response that genuinely varies by language will serve the wrong language to someone.
- **Too narrow a key is the opposite, quieter failure**: including something in the key that *doesn't* actually change the response fragments your cache into needless duplicates and tanks your hit ratio — the identical failure mode `Vary: User-Agent` produces (concept 4), and CDNs face it constantly via **query strings**. A tracking parameter appended to a URL for analytics purposes (`?utm_source=newsletter`) doesn't change the page's content at all, but if a CDN's cache key includes the full query string by default, `/article/42` and `/article/42?utm_source=newsletter` are treated as two entirely separate cache entries requiring two separate origin fetches for identical content — which is precisely why every major CDN offers a configuration to **normalize or strip specific query parameters from the cache key** while still passing them through to analytics.

The judgment underneath both failure modes is the same one concept 4 already taught for `Vary`, one level up the stack: a cache key should include *exactly* what changes the response, no more and no less — and getting that wrong in either direction is silent, not an error, which is exactly why it's worth naming explicitly rather than trusting a default to get it right for your specific traffic.

### Origin shielding: the fan-in problem a single reverse proxy never has

Here is where CDN-scale caching introduces a genuinely new failure mode that a single reverse-proxy cache, by construction, cannot have. A single cache in front of your origin has exactly one thing that can ever ask the origin for a fresh copy: itself. A CDN with, say, 300 PoPs worldwide has **300 independent caches**, each capable of independently deciding "I don't have this, or my copy expired — let me ask the origin." If a popular piece of content's cache entry expires at roughly the same time everywhere (which `max-age` naturally causes, since every PoP started its countdown from roughly the same original fetch), or if you purge it everywhere at once (concept 6), you can get **up to 300 near-simultaneous origin requests for the identical content** — a stampede multiplied by the number of PoPs in your network, layered on top of the single-process stampede problem concept 7 covers separately.

**Origin shielding** is the fix, and the idea is a direct application of hierarchy — the same idea Day 12 concept 2 used to explain why DNS delegates through a small number of TLD servers rather than every resolver asking the root for everything: **designate one shield location (or a small number, one per region) that every edge PoP must go through on a miss, rather than letting every edge PoP talk to the origin directly.** An edge miss becomes a request to the shield, not to the origin; only the *shield's* miss — a single request, not up to 300 — ever reaches the origin. The shield absorbs the fan-in exactly the way a TLD server absorbs millions of resolvers' queries into one delegation lookup instead of each resolver separately walking the whole tree.

```
        (before shielding)                    (after shielding)

  Edge PoP 1 ─┐                          Edge PoP 1 ─┐
  Edge PoP 2 ─┼──► all miss ──► ORIGIN    Edge PoP 2 ─┼──► SHIELD ──► ORIGIN
  Edge PoP 3 ─┤    simultaneously          Edge PoP 3 ─┤   (one miss,
     ...      ┘    (N requests)               ...      ┘    not N)
  Edge PoP N ─┘                          Edge PoP N ─┘
```

**Naming across vendors — verify current, these are exactly the fast-drifting feature names this note's conventions warn about:** Amazon CloudFront calls this feature **Origin Shield**; Fastly calls the underlying mechanism **Shielding** (you designate one Fastly PoP as the shield for a given origin); Cloudflare calls its version **Tiered Cache**, with a **Smart Tiered Cache** variant that automatically selects the best upper-tier data center per origin rather than requiring manual configuration. The concept is universal across CDN vendors even where the product name and exact configuration differ, and the product names in particular should be checked against current vendor docs before being quoted in a design document.

### Worked example — reading real CDN cache headers from three different vendors

These are live `curl -I` captures against real, production, CDN-fronted sites — not constructed for illustration — made from this note's own drafting session, so the values are genuine snapshots of real infrastructure behavior rather than invented numbers.

**Fastly, fronting MDN (`developer.mozilla.org`):**
```
$ curl -sI https://developer.mozilla.org/en-US/
cache-control: public, max-age=3600
etag: "c8561c548a959eb2811190e3b0443a00"
age: 1461
via: 1.1 google, 1.1 varnish, 1.1 varnish, 1.1 varnish
x-served-by: cache-sin-wsss1830041-SIN, cache-sin-wsss1830041-SIN, cache-sin-wsss1830041-SIN, cache-hyd1500025-HYD
x-cache: MISS, HIT, MISS
x-cache-hits: 0, 924, 0
```

Read this left to right in `x-served-by`, `x-cache`, and `x-cache-hits` together, and origin shielding stops being an abstract diagram and becomes a trace you can point at: **three cache identifiers are listed, meaning this response passed through a chain of caching tiers, not just one.** The tier with `924` recorded hits is a shield — a mid-tier cache that has answered this exact request 924 times already and is extremely warm — while the edge tier nearest this request shows `MISS` twice (`0` prior hits at that specific node), meaning *this particular edge machine* hadn't seen this URL recently, but rather than falling through to Fastly's actual origin, it fell through to the warm shield tier and got an instant answer from there. `via: 1.1 varnish` appearing three times confirms Fastly's edge software (Varnish-derived) sits at each of those hops. **This is origin shielding, observed rather than described:** the origin behind this response was very likely never contacted for this request at all, despite two `MISS` results appearing in the trace.

**CloudFront, fronting an Amazon S3 asset:**
```
$ curl -sI https://d1.awsstatic.com/
x-cache: Hit from cloudfront
via: 1.1 7ecb0a6d4447a88ba1a9b1102878b61e.cloudfront.net (CloudFront)
x-amz-cf-pop: HYD57-P6
age: 12214
```

CloudFront's vocabulary differs from Fastly's (`X-Cache: Hit from cloudfront` rather than a numeric hit counter, `X-Amz-Cf-Pop` naming the specific edge location — `HYD57` decodes to a Hyderabad-area PoP) but the underlying fact is identical: `Age: 12214` means this exact object has been sitting in this edge cache, untouched by any origin request, for **just over three hours (12,214 seconds)** — a very warm cache, and 12,214 more seconds during which the origin did zero work for anyone requesting this object from this PoP.

**A live example of `stale-if-error` in actual production use, on AWS's own documentation site:**
```
$ curl -sI https://docs.aws.amazon.com/
cache-control: max-age=300, stale-if-error=600
x-cache: Hit from cloudfront
age: 149
```

This is concept 2's `stale-if-error` directive, not as a textbook example but as a header genuinely being served, right now, by AWS's own documentation site: fresh for 300 seconds, and if a revalidation attempt after that fails (origin error, origin unreachable), CloudFront may keep serving this exact stale copy for up to 600 *additional* seconds rather than showing the user an error — a real, low-stakes, well-chosen use of the directive on content (documentation pages) where a few extra minutes of staleness during an origin incident is a far better outcome than an outage page.

**Cloudflare, fronting its own marketing site:**
```
$ curl -sI https://www.cloudflare.com/
cache-control: public, max-age=10, s-maxage=10
vary: accept-encoding
cf-placement: local-BOM
cf-ray: a30208f5ef2f1eed-BOM
```

`Cf-Placement: local-BOM` and the `-BOM` suffix on `CF-RAY` both name the specific PoP that answered this request — `BOM` is the IATA airport code for Mumbai, meaning anycast routing (this section's opening point) delivered this request to Cloudflare's Mumbai-area infrastructure, consistent with wherever this particular request actually originated from. This is the single most concrete way to *see* anycast-based edge routing working: the CDN's own diagnostic header is telling you, by airport code, which physical location on Earth just answered your request.

### Judgment: a commercial CDN versus operating your own multi-region reverse-proxy fleet

**The choice:** you need edge caching close to users in multiple continents — do you use a commercial CDN (Cloudflare, Fastly, CloudFront, Akamai, and others), or run your own reverse-proxy caches (nginx or Varnish) in colocation facilities or cloud regions you rent yourself?

**The realistic alternative, and companies genuinely do this:** self-operated regional caching, especially at companies with existing multi-region infrastructure and strong platform teams — you already have machines in several AWS or GCP regions, and adding a Varnish layer in front of each looks like a small incremental step rather than a new capability to buy.

**The actual reason a commercial CDN usually wins:** the value of a CDN isn't the caching software itself — nginx and Varnish are genuinely excellent and free — it's the **anycast network and the hundreds of PoPs already peered with residential and mobile ISPs worldwide**, which took the CDN vendor years and enormous capital to build and which you cannot replicate by renting a handful of cloud regions. Three or four cloud regions gets you three or four points of presence; a commercial CDN gets you hundreds, in places (a rural mobile network's own facility, an internet exchange point you'd never get direct access to) that aren't for sale to you at any price as an individual customer. This is precisely the Netflix Open Connect case study below in miniature: physical placement close to the actual last mile is the scarce resource, not caching logic.

**The trade-off, honestly:** a commercial CDN is a shared, multi-tenant piece of infrastructure you don't control end-to-end — you configure it through whatever API and rule language it exposes, you're bound by its purge propagation characteristics (concept 6) rather than anything you can change, and (the Fastly case study below, stated plainly) **its outages are your outages**, correlated with every other customer on the same platform, at a scale and suddenness a self-operated fleet's failures are unlikely to match. Running your own regional caches gives you complete control — your own deploy process, your own incident response, no shared blast radius with unrelated companies — at the cost of building and permanently operating the PoP footprint yourself, which very few companies' actual traffic volumes justify.

**The flip condition:** self-operated regional caching becomes the right call when your actual user base is concentrated enough geographically that a handful of self-run PoPs cover it adequately (a product serving primarily one country or region), or when you're already at a scale and have specific enough requirements — Netflix's video-delivery bandwidth economics are the extreme end of this, discussed next — that operating your own delivery infrastructure, including the physical placement inside ISPs that Open Connect represents, becomes cheaper than paying commercial CDN rates at your volume. For the overwhelming majority of companies below that scale, a commercial CDN's existing global footprint is not something you can cost-effectively rebuild yourself, and the trade-off resolves in the CDN's favor by a wide margin.

### System design — a CDN cache-key strategy for a localized, multi-tenant product catalog

**Problem:** an e-commerce platform serves product pages at URLs like `/products/12345` to visitors worldwide. The rendered page differs by currency (a query parameter, `?currency=EUR`) and by whether the visitor is logged into a specific tenant's storefront (a subdomain per tenant, `acme.platform.com`, `widgets.platform.com`, …), but the *underlying product data* for a given product ID is otherwise identical for every anonymous visitor in the same currency. Design the CDN cache-key strategy.

**Requirements:** a French visitor in EUR and a US visitor in USD must never see each other's price; two different tenants' storefronts must never cross-contaminate cached content even though they share the platform's CDN configuration; the cache should not fragment needlessly across dimensions that don't actually change the page.

**The alternatives:** (1) cache key = full URL including hostname and every query parameter, unmodified — the CDN's most naive default; (2) cache key = hostname + path + an explicitly normalized subset of query parameters (just `currency`, with every other parameter — session IDs, tracking parameters, `utm_*` — stripped from the key entirely, though still passed through to the origin on a miss); (3) move currency out of the URL entirely and into a `Vary`-style content-negotiation header, keeping the cache key to hostname + path alone.

**The decision: (2).**

**The actual reason:** the hostname is *correctly* included in the key regardless of chosen strategy, because it's structurally the tenant boundary — `acme.platform.com/products/12345` and `widgets.platform.com/products/12345` are different tenants' pages even if the path is identical, and a cache key that dropped the hostname would recreate exactly the cross-tenant leakage this concept's earlier system design (concept 2) was built to prevent, just at the CDN layer instead of the `Cache-Control` scope layer. `currency` genuinely changes the rendered content (different prices, different formatting) and belongs in the key. Everything else attached to the query string — a referral tracking parameter, a session or affiliate ID appended by a marketing link — doesn't change what's rendered and would, if left in the naive default key (option 1), fragment the cache into a near-infinite number of effectively-single-use entries, reproducing `Vary: User-Agent`'s failure mode (concept 4) through the query string instead of a header.

**The trade-off:** option (2) requires an explicit, maintained allowlist of "which query parameters actually matter" — someone has to notice when a new, content-affecting query parameter is added to the product page and update the cache-key configuration, or it'll be silently ignored (served identically to everyone regardless of its value) rather than causing an obvious error. Option (3), pushing currency into a header instead of the URL, avoids that maintenance burden by using the same mechanism `Vary` already provides — but currency is a case where a low-cardinality, real value (a small, known set of supported currencies) makes `Vary` genuinely safe, unlike the `Vary: User-Agent` cautionary tale, so either (2) or (3) is defensible here; the deciding factor is usually whether currency needs to be bookmarkable/shareable in the URL (in which case it has to stay in the query string, favoring option 2) or is better derived from a cookie or `Accept-Language`-style negotiation (favoring option 3).

**Flip condition:** if the platform later adds a query parameter that changes content on a much larger fraction of pages, or if the list of content-affecting parameters starts changing frequently enough that keeping the allowlist current becomes its own maintenance burden, that's the signal to move the varying dimension into a header-based `Vary` scheme instead of continuing to hand-maintain a growing query-parameter allowlist.

### System design — origin shielding for a flash-sale traffic spike

**Problem:** an e-commerce site runs a flash sale that goes live at a precise, announced time. Traffic to the sale's landing page, cached with `max-age=60`, is expected to jump from a baseline the origin handles comfortably to roughly 50× that baseline in the first ten seconds after launch, spread across a CDN with 200 PoPs worldwide. Design the caching architecture so the origin survives the spike.

**Requirements:** the origin must not receive anywhere close to 200× simultaneous cold-cache requests at launch; the page must still refresh within a reasonable window if the sale's content needs a last-minute correction; the design must survive the specific failure mode where all 200 PoPs' cache entries expire at approximately the same wall-clock moment, since they were all warmed from roughly the same original fetch.

**The alternatives:** (1) no origin shield — rely on `max-age` alone and accept that every PoP independently misses and refetches on roughly the same schedule; (2) origin shielding (this concept's mechanism) — a single shield tier absorbs every edge PoP's miss and makes at most one request to the true origin per shield location; (3) pre-warm the cache by making a request through every PoP shortly before launch, so no PoP is ever cold at the actual launch moment.

**The decision: (2) and (3) together, not either alone.**

**The actual reason:** shielding (2) solves the *fan-in* problem — it guarantees the origin sees at most "number of shield locations," not "number of edge PoPs," simultaneous requests — but it doesn't prevent the shield tier *itself* from experiencing exactly the single-process stampede problem concept 7 covers in full: if 200 edge PoPs all miss at once and all forward to the same one or two shield locations at once, the shield now faces its own concentrated burst, which is a smaller, more tractable problem than 200-way fan-in, but not a solved one on its own. Pre-warming (3) sidesteps the timing problem at its root: if every PoP already holds a fresh copy *before* launch, there's no "everyone misses simultaneously" moment to defend against at all. Doing only (3) without (2) leaves you exposed the moment the pre-warmed entries eventually expire on their `max-age` clock, potentially re-triggering the exact synchronized-expiry problem you pre-warmed to avoid — the two techniques cover each other's gap.

**The trade-off:** pre-warming requires knowing the exact launch time in advance and an operational step (a script hitting every PoP, or a CDN feature that does this natively) that has to be run correctly and on schedule — it's an extra deploy-time ritual with its own failure mode (someone forgets to run it, or runs it against the wrong URL). Shielding costs one additional network hop's latency for every genuine cache miss, since a miss now goes edge → shield → origin, one more hop than edge → origin directly — negligible against the origin-protection benefit at this scale, but worth naming, per this note's discipline, as an actual cost rather than a free lunch.

**Flip condition:** for launches whose exact timing isn't fixed in advance (organic virality rather than a scheduled event — a link going unexpectedly viral on social media), pre-warming isn't available as an option at all, because there's no "shortly before launch" moment to warm against — shielding plus `stale-while-revalidate` (concept 2) becomes the primary defense instead, accepting that the first wave of traffic will experience the fan-in-to-shield burst that pre-warming would otherwise have avoided.

### Case study — the Fastly outage of June 8, 2021: when the cache layer is the front door

**What happened.** On June 8, 2021, a large fraction of the internet — including Amazon, Reddit, the UK government's GOV.UK, The New York Times, Twitch, and Fastly's own status page — returned `503` errors or failed to load entirely, for roughly an hour, simultaneously. Fastly's own summary of the incident states the mechanism precisely: a software deployment on May 12 had introduced a latent bug that could be triggered by a *specific, valid* customer configuration change under specific circumstances. On June 8, one customer made exactly such a valid configuration change, and the bug caused **85% of Fastly's network to return errors.** Fastly's monitoring detected the disruption within one minute; engineers identified the specific customer configuration and disabled it; within 49 minutes, 95% of the network was reported operating normally.

**Why this belongs in this concept specifically, not just as a generic "outages happen" cautionary tale:** every one of the affected companies did *nothing wrong*. Their own applications, databases, and deploy pipelines were healthy the entire time. What failed was a piece of shared infrastructure that concept 1 established sits *in front of* the origin — and this incident is the sharpest possible illustration of what that positioning means when it goes wrong: **a CDN isn't a performance optimization bolted onto a working system; once it's in place, it *is* the front door, and if the front door won't open, it doesn't matter that everything inside the building is fine.** A single customer's legitimate, valid configuration change — not an attack, not a misuse of the platform — was sufficient to trigger a bug that took down a meaningful fraction of the visible internet, because Fastly's edge network is exactly the kind of shared, multi-tenant infrastructure this concept's Judgment section flagged: the blast radius of a bug in it is every customer on the platform, simultaneously, regardless of which customer's action triggered it.

**The engineering lesson to take away, concretely:** Fastly's own acknowledgment is worth quoting because it's an unusually direct admission for a public postmortem — paraphrased from their summary, they stated that even though specific conditions triggered the outage, they should have anticipated that class of failure, and committed to process changes including greater reliance on isolating customer configurations from each other (their post-incident commitments reference expanding use of WebAssembly-based isolation specifically to contain this class of bug to one customer rather than the whole network). For anyone *depending* on a CDN rather than operating one, the actionable lesson is not "don't use a CDN" — concept 1 through this section have made the performance and cost case for one too strong to abandon — it's: **know what happens to your own system when your CDN is unreachable**, the same "test the dependency failing" discipline Day 12's Meta 2021 case study argued for DNS. A `stale-if-error` policy (concept 2) served from a *second*, independent origin path, or a documented, tested static fallback page, is the direct mitigation; having neither means your outage duration is identical to your CDN vendor's, with zero ability on your part to shorten it.

*Primary source:* Fastly, "Summary of June 8 outage" (fastly.com/blog/summary-of-june-8-outage, published June 8/9, 2021) — Fastly's own account, including the timeline and the 85%/95%/49-minute figures used above. **Verify current**: this is a historical incident with a fixed, published record, so the facts themselves don't drift — but Fastly's specific architectural mitigations referenced in their own post-incident commitments are worth checking against their current engineering blog if you want to know what, if anything, has changed since 2021.

### Case study — Netflix Open Connect: the logical endpoint of "move the cache closer"

**What it is.** Every technique in this concept so far pushes a cache closer to users by renting space in more data centers. Netflix's Open Connect program pushes further: Netflix builds and provides, **free of charge**, purpose-built caching appliances (Open Connect Appliances, OCAs) that it embeds physically **inside individual ISPs' own networks** — not in a neutral data center or internet exchange point Netflix rents from a third party, but inside the last-mile provider's own facility, as close to the end user as infrastructure gets without being inside their home. According to Netflix's own account, this program spans **thousands of locations across more than 100 countries**, and — the figure worth taking directly from Netflix's own words rather than a secondhand summary — Netflix states that **"globally, close to 90% of our traffic is delivered via direct connections between Open Connect and the residential Internet Service Providers (ISPs)"** their members use to access the internet.

**Why this is the right case study to pair with everything else in this concept, rather than just a bigger number:** it's the same PoP-placement logic this concept has been teaching, taken to its structural limit. A commercial CDN's PoP sits *near* an ISP, peered with it at an exchange point; Open Connect's appliance sits **inside** the ISP, one hop from the subscriber, eliminating not just the cross-ocean trip but the peering hop itself. It's also a case study in the economics behind this concept's earlier Judgment section: Netflix's traffic volume and the extreme cost-sensitivity of video bandwidth at that scale made building and giving away purpose-built hardware, and negotiating placement inside hundreds of individual ISPs' own facilities, the economically rational choice — a level of infrastructure investment that makes sense only because of Netflix's specific traffic profile (enormous volume, highly cacheable content — the same video files requested repeatedly by huge numbers of subscribers), not something that generalizes to most companies' actual caching needs.

*Primary source:* Netflix, "How Netflix Works With ISPs Around the Globe to Deliver a Great Viewing Experience" (about.netflix.com/en/news, Netflix's own account, direct source for the ~90% figure quoted above); Netflix Open Connect technical overview (openconnect.netflix.com). **Verify current — flagged explicitly because this is exactly the kind of fast-drifting figure this note's conventions call out**: the specific traffic percentage, the country/location counts, and the OCA deployment model are all numbers Netflix updates over time as the program grows; check Netflix's current published figures rather than treating the number above as fixed indefinitely.

### In production — operating a CDN

**Best practices:**
1. **Configure your cache key deliberately, per route class, rather than accepting a CDN's default.** The naive "full URL, every query parameter" default is safe from a correctness standpoint but frequently terrible for hit ratio; a hand-tuned allowlist of parameters that actually affect content (this concept's first system design) is almost always worth the one-time configuration effort.
2. **Enable origin shielding for anything with meaningful traffic**, even before you think you need it — the cost (one extra hop on a miss) is small and the protection against synchronized fan-in is exactly the kind of failure that's invisible until the day it isn't.
3. **Test your application's behavior with the CDN in front of it disabled or in bypass mode**, not just with it healthy — the Fastly case study's actionable lesson, made concrete as a pre-launch checklist item rather than a paragraph you read once.
4. **Treat CDN configuration as code, under version control and review**, exactly the discipline Day 12 recommended for DNS zone files — a CDN's rule engine is powerful enough that a bad rule change is operationally equivalent to a bad deploy, and deserves the same rigor.

**Mistakes, beginner → senior:**
- *Beginner:* assuming "we're behind a CDN" means the origin can be sized for cached-traffic volumes without separately verifying what happens on a genuine cold-cache event across every PoP at once.
- *Intermediate:* leaving a CDN's default cache-key behavior untouched on a route with several irrelevant query parameters and not noticing the resulting hit-ratio collapse until someone happens to check the dashboard.
- *Senior:* architecting robust origin shielding and cache-key strategy while having zero tested fallback for the CDN itself being unreachable — protecting against the failure modes *inside* the CDN relationship while leaving the CDN itself as an unexamined single point of failure, precisely the Fastly lesson.

**Monitoring:** hit ratio broken out by PoP/region, not just globally (a global 98% hit ratio can hide a specific region's cache misconfiguration); shield-tier hit ratio specifically, as a leading indicator of whether shielding is actually doing its job; origin request volume during and immediately after any purge or deploy, watched for the fan-in spike this concept describes.

**Cost:** CDN pricing is typically metered on both bandwidth served from the edge and, separately, on requests that reach the origin — which means hit-ratio and cache-key tuning aren't purely a performance exercise, they show up directly on the bill. Origin shielding, beyond its protective value, is also frequently a direct cost optimization, since shield-tier hits are typically far cheaper than genuine origin fetches in both infrastructure load and, for many providers, metered pricing.

---

## 6. Cache invalidation: purge vs. versioned/hashed URLs

**Depth: [CORE]**

### Intuition

Day 12 built one uncomfortable fact into your model of caching and it's worth stating exactly how far it does and doesn't carry over here: **DNS has no invalidation mechanism at all** — once a resolver has cached a record, there is no API, no button, no protocol message that reaches into that resolver and makes it forget. The *only* control an operator has is the TTL, chosen in advance. HTTP and CDN caching are built on that same TTL-as-a-promise logic for `max-age` (concept 2), but they diverge from DNS in one genuinely important way: **a CDN typically does offer a purge API — a real, callable mechanism that tells edge caches "forget this specific thing, now."** That's machinery DNS simply doesn't have.

The honest, careful version of that fact, though, is the one this concept exists to teach: **a purge is not instant, not free, and not perfectly reliable across a system distributed over hundreds of physical locations** — it's an operation with its own propagation delay, its own failure modes, and its own cost, not a magic undo button. So while purge is real, in a way DNS invalidation simply isn't, treating it as your *primary* defense against staleness reproduces a weaker version of Day 12's exact problem: you're still trusting a distributed system to converge on a correction, just faster and with more control than DNS gives you, not instantaneously and with total control. The strategy that sidesteps this problem entirely, rather than mitigating it, is the second half of this concept: **make invalidation unnecessary by construction**, by ensuring a given URL's content literally never changes — so there's nothing to invalidate, ever.

### Strategy one: purge — actively telling every cache to forget

A **purge** (sometimes called "invalidation" in a provider's own terminology, e.g. AWS CloudFront's API literally calls it an "invalidation") is an explicit, API-driven request: "the content at this URL (or matching this tag, or this path prefix) has changed — discard any cached copy, everywhere, and fetch fresh on the next request." This is the right and often only tool available when a URL's content genuinely needs to change while the URL itself must stay the same — a CMS article published at a stable, bookmarked, SEO-indexed slug (`/blog/how-caching-works`) that gets a typo fix after publication; a product page whose price changes; an API response that must reflect new data immediately after a write.

**What a purge actually does, mechanically, and why "everywhere, instantly" isn't quite true:** a purge request goes to the CDN's control plane, which then has to propagate an invalidation instruction out to every edge location that might be holding a copy — and "every edge location" can be hundreds of physically separate machines across the globe, each running its own cache process, not a single database you can update atomically. Real, published figures from CDN vendors' own documentation illustrate the range honestly: Cloudflare's general-purpose cache-clear operation states it may take up to roughly 30 seconds to take effect; a single-file purge by exact URL is typically sub-second; Cloudflare's own "Instant Purge" feature, built specifically to tighten this, advertises a global propagation latency under 150 milliseconds at the 50th percentile for purge-by-tag or purge-by-prefix operations. **Every one of those numbers is a real vendor claim, not a protocol guarantee, and every one of them is explicitly the kind of fast-drifting, provider-specific figure this note's conventions require flagging: verify current numbers against the vendor's own documentation before relying on any of them in a design decision, including the ones just cited.** The durable lesson underneath the specific milliseconds is what matters and won't drift: **purge propagation takes measurable, non-zero time across a distributed edge network, and different vendors, and different purge granularities from the same vendor, can have meaningfully different propagation characteristics.**

### Strategy two: versioned/hashed URLs — making invalidation unnecessary

The alternative strategy doesn't try to correct a cached copy faster — it removes the need to correct anything, by guaranteeing that **a given URL's content can never change after it's first published.** The mechanism: compute a hash of a file's contents (or use an incrementing build/version number) at build time, and bake that hash directly into the filename — `app.js` becomes `app.3f2a91c9.js`, where `3f2a91c9` is (typically a short prefix of) a hash of the file's actual bytes. Every build tool in common use for frontend assets — Webpack, Vite, and others — does exactly this by default, under names like "content hashing" or "fingerprinting."

Once the filename *is* a function of the content, a genuinely new guarantee holds: **if the content ever changes, the hash changes, which means the filename changes, which means it's a completely different URL — the old URL's content is now permanently frozen, because nothing will ever again produce that exact hash for different content** (barring an astronomically improbable hash collision, the same reasoning that makes concept 3's content-hash ETags trustworthy). A URL that is *guaranteed*, by construction, to never point at different content on two different occasions can be cached with the longest freshness lifetime the protocol allows and marked `immutable` (concept 2) — invalidation was never a risk to manage, because there is nothing to invalidate. The real-world convention, seen directly in the verified header captures below, is `Cache-Control: public, max-age=31536000, immutable` — 31,536,000 seconds is exactly one year, effectively "cache this forever" in practical terms, on a filename that will never be reused for different content.

### Worked example — a real build's output, and the header policy that makes invalidation moot

A typical modern frontend build produces output like:

```
dist/
  index.html
  assets/
    app.3f2a91c9.js       ← hash of THIS BUILD's JS bundle
    app.3f2a91c9.css
    logo.8b6e77aa.svg
```

Deploy a change to the JavaScript — anything, even a single character — and the *next* build produces:

```
dist/
  index.html
  assets/
    app.9e01ffab.js       ← DIFFERENT hash: the content changed
    app.9e01ffab.css
    logo.8b6e77aa.svg     ← UNCHANGED: same hash as before, because
                             the SVG genuinely didn't change this build
```

Serve every file under `assets/` with:
```
Cache-Control: public, max-age=31536000, immutable
```

and the entire class of problem this concept opened with — "how do I get every cache in the world to notice this file changed" — never comes up for these files, because **the old hash's content genuinely never changes, so there is nothing to invalidate; you simply stop referring to the old filename and start referring to the new one.** The only thing that must actually change on deploy is `index.html`, because it's the file that *names* which hashed bundle to load — which is exactly the next problem.

### The HTML shell problem: the one file that must never be cached long

Notice the asymmetry the worked example just created: every hashed asset can be cached for a year with total safety, but **`index.html` — the file that tells the browser which hashed filenames to actually request — is the one artifact in this entire pipeline whose content changes on every single deploy, referencing a new hash each time, and which must therefore never be treated as long-lived.** If `index.html` itself were cached for a year, users would keep loading an old shell that requests `app.3f2a91c9.js` forever, even though that old bundle might eventually be deleted from the origin entirely, or — more subtly and more commonly — even though a *newer*, bug-fixed bundle exists under a different hash that the stale shell will never ask for. Every real "I deployed a fix and users still see the old broken bundle" incident on a modern SPA or JAMstack site traces back to exactly this: the immutable-asset strategy was applied correctly to the assets, and forgotten for the one file that points at them.

The standard, correct policy pairs the two extremes deliberately:
```
index.html:              Cache-Control: no-cache            ← ALWAYS revalidate (concept 3);
                          (small file, cheap to revalidate     validator-backed, so a fast 304
                           on every load)                       is the common case, not a full refetch
assets/*.<hash>.{js,css}: Cache-Control: public, max-age=31536000, immutable   ← never revalidate,
                                                                                   never expires in
                                                                                   any practical sense
```

`index.html` costs a cheap revalidation round trip on essentially every load — usually a `304` (concept 3), since the shell itself doesn't change between deploys — in exchange for guaranteeing every user always references the current build's actual bundles. The bundles cost nothing at all to keep correct, because they were built to never need correcting.

### Under the hood: purge and versioning are not actually competitors, they're complements

It's worth being precise that this concept has presented purge and versioning as "the two strategies" because they answer the *same* underlying question from opposite directions, not because a real system picks exactly one. A real deployment pipeline uses versioned URLs for everything that *can* be versioned (build artifacts, images, anything reachable only through a reference your own code controls, like `index.html`'s `<script src>` tag) and reserves purge for the residual category that genuinely can't be versioned this way — content whose URL is fixed by something outside your build process: a CMS article's SEO-indexed slug, a user-bookmarked page, an API endpoint whose URL is part of a public, stable contract. **Versioning eliminates the need for invalidation wherever the URL is entirely under your own control; purge remains necessary wherever it isn't.**

### System design — global static-asset delivery: why cache invalidation becomes a non-problem

**Problem:** design the caching architecture for a company's static assets — JavaScript bundles, CSS, images, fonts — served globally to millions of users, where the underlying build pipeline produces a new set of these files on every deploy, several times a day.

**Requirements:** users must never receive a mix of assets from two different, incompatible deploys (an old JS bundle paired with a new CSS file that assumes different class names, for instance); the CDN's cache hit ratio for these assets should approach 100%, since they genuinely never change once built; a rollback to a previous deploy must be possible without waiting on any cache to expire or purging anything.

**The realistic alternatives:** (1) serve every asset under a stable filename (`app.js`, `style.css`) with a moderate `max-age`, and purge the CDN on every deploy; (2) content-hashed filenames with `immutable, max-age=31536000`, and a short-lived `index.html` shell referencing the current build's hashes, as developed above; (3) a hybrid — stable filenames, but a `?v=<build-number>` query parameter appended, relying on the CDN's cache key (concept 5) to include the query string.

**The decision: (2).**

**The actual reason:** option (1) makes every single deploy a purge event across your entire global CDN footprint, at the propagation latencies this concept already established are real and non-zero — meaning there's a window, after every deploy, during which different users worldwide are being served a mix of old and new assets depending purely on which PoP's purge has landed and which hasn't, which is precisely the "old JS with new CSS" failure this problem's requirements rule out. Option (2) has no such window: the *assets themselves* never need a purge at all, ever, because they never change once built — the only thing that changes on deploy is `index.html`, one small, cheap-to-revalidate file, and every user who loads a fresh `index.html` (whether one second or one month after deploy) gets a fully self-consistent set of hashes for exactly one build, with no possibility of a mismatched mix, because the shell and the assets it names are generated together, atomically, at build time. Option (3) is a well-intentioned middle ground that reintroduces a subtler version of the same problem: many CDNs and browsers historically apply different (and sometimes surprising) caching heuristics to URLs with query strings than to path-based names, and relying on query-string cache-busting means your correctness now depends on every cache in the path treating the query string as part of the key — which concept 5 already flagged as a real, provider-specific configuration detail rather than a universal guarantee.

**The trade-off, honestly:** option (2) requires a build pipeline capable of content-hashing and rewriting every internal reference to point at the hashed name — genuinely more build tooling complexity than "just serve the files as named," though this is now the overwhelming default behavior of essentially every modern frontend build tool, so in practice the cost is closer to "use the tool's default" than "build this yourself." It also means old, orphaned hashed assets accumulate in storage indefinitely unless you separately clean them up — a storage cost, not a correctness cost, but a real one at scale.

**The flip condition:** option (1)'s purge-on-deploy approach becomes more attractive, or even necessary, for content whose URL genuinely cannot be versioned — which is exactly why the "In production" section below and the case studies pair this pattern with an honest acknowledgment that purge doesn't disappear as a tool, it just stops being needed for the specific category of content (build artifacts under your own naming control) that this system design is scoped to.

**Failure modes:** forgetting to also mark `index.html` (or any other unhashed entry point) with a short freshness lifetime, and having it cached long by a CDN default — this reintroduces exactly the "users stuck on an old shell forever" bug this system design exists to prevent, just moved from "every deploy" (option 1's failure window) to "invisible until the CDN's default TTL finally expires" (a longer, more confusing version of the identical bug). Orphaned old hashed assets being deleted from storage too aggressively — a rollback, or a user with an old `index.html` still cached somewhere unexpectedly (a corporate proxy ignoring your `no-cache`, echoing concept 1's "seven places a stale answer can live"), requesting a hash that no longer exists, producing a hard `404` for a legitimate, if late, request.

### System design — caching a news-site homepage: seconds-fresh content at a million requests per second

**Problem:** a news site's homepage must reflect newly published articles within seconds of publication, while serving on the order of a million requests per second globally during a major breaking-news event. Design the caching strategy.

**Requirements:** a newly published article must appear on the homepage within a few seconds, not minutes; the origin must never see anywhere close to a million requests per second, regardless of how often the homepage's content actually changes; a brief origin hiccup during a traffic spike must not turn into a visible outage for readers.

**The alternatives, and this is precisely where concepts 2, 6, and 7 have to work together rather than in isolation:** (1) a short `max-age` alone (e.g. `max-age=5`), relying purely on natural expiry to bound staleness; (2) purge-on-publish — serve the homepage with a long `max-age`, and have the publishing workflow itself trigger an explicit CDN purge the instant an article goes live; (3) `stale-while-revalidate` layered on top of a short `max-age`, so the homepage refreshes in the background rather than blocking any single reader on a synchronous origin fetch.

**The decision: (2) plus (3) together — purge-on-publish for the "must update within seconds" requirement, `stale-while-revalidate` as the safety net for everything in between.**

**The actual reason:** option (1) alone forces an uncomfortable trade purely through the freshness-lifetime knob — a `max-age` short enough to guarantee "a few seconds" staleness (concept 5's system design already established what "everyone's cache expires around the same moment" produces at CDN scale: a fan-in stampede) means the homepage is *constantly* re-fetched across every PoP, near-continuously, at a million-requests-per-second scale, which is exactly the load the requirements say the origin must never see. Purge-on-publish (2) decouples freshness from `max-age` entirely: you can set a comfortably long `max-age` — tens of minutes — because you no longer need short-`max-age` expiry to make new content visible; the publishing workflow's explicit purge does that job, deterministically, the moment an editor hits publish, rather than waiting on a clock. `stale-while-revalidate` (3) is what keeps the *rare* remaining cases — a purge landing at a PoP right as a request arrives, or an ordinary `max-age` expiry between publishes — from ever making a reader wait on a synchronous origin round trip: they get the instant, briefly-stale answer, and the background refresh catches the next reader up.

**The trade-off, honestly:** purge-on-publish adds a real dependency from your publishing workflow to your CDN's purge API — if that API call fails silently, or a bug in the publishing pipeline forgets to fire it for one code path (a scheduled post, an editorial correction made through a different tool than the main publish button), you're back to relying on `max-age` alone for that specific path, with all of option (1)'s downsides, and the bug is easy to miss because *most* publishes go through the primary path and work fine. This is a real operational dependency to monitor, not a "set it and forget it" integration.

**The flip condition:** for a site whose publish cadence is genuinely infrequent and staleness tolerance is measured in minutes rather than seconds, a short-`max-age`-plus-`stale-while-revalidate` approach without purge-on-publish is simpler to operate and entirely sufficient — purge-on-publish earns its added complexity specifically when "a few seconds" is a hard requirement that a TTL alone cannot satisfy without also re-triggering the stampede/fan-in problem concept 5 and concept 7 cover. The remaining piece of this exact problem — what happens at the *origin* itself when a purge (or a natural `max-age` expiry at this traffic volume) causes many simultaneous requests to arrive needing the same freshly-published content recomputed — is concept 7's subject in full, and this system design's protection against it is precisely the `stale-while-revalidate` half of the decision above: nobody blocks on that recomputation, because everyone gets an immediate answer while at most one background request does the actual work.

### Case study — jsDelivr and unpkg: immutable, versioned URLs as the entire caching strategy

**What it is.** jsDelivr and unpkg are both public CDNs that serve npm packages (and, for jsDelivr, GitHub-hosted files) directly to browsers by URL, at URLs that encode an exact package version — `unpkg.com/react@18.2.0/umd/react.production.min.js`. Because the version number is part of the URL and a published npm package version is immutable by the registry's own rules (you cannot silently overwrite `react@18.2.0` with different content later — a new release gets a new version number), these URLs have the identical property this concept's versioned-asset pattern relies on: **the content at a version-pinned URL literally cannot change**, so it can be cached as aggressively as the protocol allows. Verified directly from these services' own served headers: unpkg serves version-pinned files with `Cache-Control: public, max-age=31536000` (proposals exist, and some configurations already add `immutable`, to make that explicit); jsDelivr serves a shorter `max-age` specifically on its `@latest` alias URLs (which, unlike a pinned version, genuinely can point at different content over time as new versions are published) while pinned-version URLs are treated with the long-lived, effectively-permanent policy.

**The lesson this case study sharpens, beyond restating the pattern:** the *entire caching problem disappears exactly where the URL is genuinely immutable, and reappears exactly where it isn't* — jsDelivr's own two different policies for `@latest` versus a pinned version are a real, production illustration of that boundary, drawn by the same company, on the same infrastructure, for two URLs that look almost identical but have fundamentally different cacheability properties for the identical structural reason this concept has been building toward.

*Primary source:* jsDelivr and unpkg's own served response headers (directly observable via `curl -I` against either service, as with this note's CDN header captures in concept 5) and their public documentation/GitHub issue discussions on caching behavior for pinned versus `@latest` URLs. **Verify current**: exact `max-age` values for both services are configuration details that can and do change; the structural point (pinned-version URLs are immutable by construction, alias URLs are not) is the durable lesson, not the specific number of seconds.

### In production — operating invalidation

**Best practices:**
1. **Default to versioned/hashed URLs for anything your own build process controls**, and reserve purge for content whose URL is fixed by something outside that process. This is the single biggest lever available, per the system design above, and it's now the default behavior of essentially every modern build tool — the discipline is making sure it's actually configured correctly (the HTML-shell problem) rather than building it from scratch.
2. **Treat "did the purge actually take effect everywhere" as a thing you monitor, not assume.** Given real, vendor-documented propagation delays, a deploy runbook that fires a purge and immediately declares success without any verification step is trusting an assumption this concept has just shown isn't quite true.
3. **Keep a tested, documented fallback for when purge silently fails on one path** — the exact gap flagged in the news-homepage system design's honest trade-off.

**Mistakes, beginner → senior:**
- *Beginner:* applying a long, immutable `max-age` to the HTML shell along with the hashed assets it references, "for consistency," and shipping the exact stale-shell bug this concept spent a dedicated section on.
- *Intermediate:* relying on purge as the *sole* mechanism for keeping a high-traffic page fresh, and discovering under real load that purge-triggered fan-in (concept 5's origin-shielding system design) recreates the very stampede problem the design was trying to avoid.
- *Senior:* an otherwise-correct versioned-asset pipeline with no process for garbage-collecting old hashed files, quietly growing storage costs indefinitely, or conversely, garbage-collecting too aggressively and breaking a legitimate late request or an in-progress rollback.

**Monitoring:** purge success/failure rate and propagation time as an explicit metric, not an assumption; the delta between "we believe this is purged everywhere" and observed cache-hit content at a sample of PoPs after a purge, if your CDN exposes that visibility; growth of your versioned-asset storage over time, as a leading indicator of a missing cleanup process.

**Cost:** purge operations are frequently metered or rate-limited by CDN vendors specifically because propagating an invalidation to every PoP is real infrastructure work, unlike a cache simply expiring on its own schedule — a deploy process that fires an excessive number of narrow purges (one per file, on every deploy, for every affected file individually) can be a meaningfully different cost and rate-limit profile than one broad purge-by-tag or purge-by-prefix operation, which is part of why this concept's "why invalidation becomes a non-problem" framing is also, directly, a cost argument and not only a correctness one.

---

## 7. The cache stampede problem and single-flight

**Depth: [CORE]**

### Intuition

Every mechanism in this note so far has quietly assumed that a cache miss is a rare, isolated event — one request, at one moment, finds nothing cached, pays the origin cost, and moves on. That assumption fails in one specific, common, and completely predictable circumstance: **a popular cache entry's freshness lifetime expires, and a large number of requests for that exact same key arrive while it's expired, before any of them have finished recomputing it.** Every one of those requests independently sees "no fresh entry," independently decides to pay the full recomputation cost, and all of them hit the origin — the database, the expensive computation, the slow API call — at once. This is a **cache stampede** (also called a "dogpile" or, in the classic phrasing, "thundering herd"), and its defining, damaging property is a multiplication: a cache that was protecting your origin from, say, ten thousand requests per second is suddenly providing *zero* protection for the fraction of a second during which every one of those requests is a miss, because they all arrived in the gap between "the old answer expired" and "a new answer exists."

This is worth distinguishing precisely from concept 5's origin-shielding fan-in problem, because the two look similar and are solved differently. Origin shielding addresses **many separate cache locations** (hundreds of CDN PoPs) each independently missing and each independently reaching the origin — the fix coalesces *locations*. A cache stampede, as taught in this section, addresses **many concurrent requests hitting the *same* cache location** at the moment its one entry for a given key goes stale — the fix coalesces *requests*, within a single cache (whether that's one CDN edge node, one reverse proxy, or, most commonly in practice, one application-level cache like Redis sitting in front of a database). Both problems share a root cause — many parties acting on the same stale information at once — and both are variants of the identical underlying failure this note keeps returning to: **a cache with an expiry-driven correction model synchronizes everyone's "time to check again" around the same moment, unless something actively de-synchronizes them.**

### Worked example — a naive cache, measured, and exactly how it fails

The clearest way to see a stampede is to build the smallest possible thing that has one, and then measure it — not describe it, watch it happen. Take a cache in front of a slow "origin" operation (standing in for a database query or an expensive computation) with a short time-to-live, and fire 100 concurrent requests for the same key at the moment it expires.

The naive implementation looks completely reasonable on a read:
```python
hit = _cache.get(key)
if hit and hit[0] > now:
    return hit[1]                 # fresh — serve from cache

value = await slow_origin(key)    # MISS: recompute
_cache[key] = (now + TTL, value)
return value
```

The bug is not in any single line — it's in what happens when **100 requests execute this exact sequence concurrently, for the same key, while the cache is empty or expired.** Every one of them evaluates `hit and hit[0] > now` as false (there's nothing fresh to find), so every one of them proceeds to the `await slow_origin(key)` line independently, and every one of them pays the full cost of recomputation — the check-then-compute sequence has a **race condition** in exactly the sense Day 3 taught you to look for: multiple concurrent actors reading a shared piece of state, finding it in the same condition, and independently acting on that same finding without any coordination between them.

### Runnable example — measuring the stampede, then fixing it with a single-flight lock

This is the study plan's second build task, run for real. The full setup: a FastAPI service exposes two endpoints backed by the identical simulated slow origin (a 1-second `asyncio.sleep`, standing in for a slow database query or an expensive external call) and an in-memory cache with a 2-second TTL — `/naive/{key}` has the race condition above; `/single-flight/{key}` fixes it. A separate client script fires 100 concurrent requests at each endpoint for the same key and reports how many times the origin was actually invoked.

```python
# stampede_demo.py — demo only: an in-memory dict standing in for Redis/
# Memcached, and an asyncio.Lock standing in for a distributed lock. Both
# are per-PROCESS state — see "Honesty caveats" below for exactly what that
# does and doesn't protect against in a real, multi-worker deployment.
# pip install fastapi uvicorn
import asyncio
import time

from fastapi import FastAPI

app = FastAPI()

CACHE_TTL = 2.0          # seconds — deliberately short so it expires during the demo
ORIGIN_LATENCY = 1.0     # seconds — simulates a slow DB query / expensive computation

_cache: dict[str, tuple[float, str]] = {}     # key -> (expires_at, value)
_locks: dict[str, asyncio.Lock] = {}
_origin_calls = 0


async def slow_origin(key: str) -> str:
    """Stands in for an expensive DB query or recomputation."""
    global _origin_calls
    _origin_calls += 1
    await asyncio.sleep(ORIGIN_LATENCY)      # I/O-bound wait — NOT time.sleep()
    return f"value-for-{key}-computed-at-{time.time():.3f}"


@app.get("/reset")
async def reset():
    global _origin_calls
    _cache.clear()
    _locks.clear()
    _origin_calls = 0
    return {"reset": True}


@app.get("/stats")
async def stats():
    return {"origin_calls": _origin_calls}


@app.get("/naive/{key}")
async def naive(key: str):
    """THE BUG: check-then-compute with no coordination between requests.
    Every request that arrives while the entry is missing/expired takes
    the same 'miss' branch and calls the origin independently."""
    hit = _cache.get(key)
    now = time.monotonic()
    if hit and hit[0] > now:
        return {"value": hit[1], "origin_calls_so_far": _origin_calls}

    value = await slow_origin(key)                # <-- the stampede happens HERE
    _cache[key] = (now + CACHE_TTL, value)
    return {"value": value, "origin_calls_so_far": _origin_calls}


@app.get("/single-flight/{key}")
async def single_flight(key: str):
    """THE FIX: the first request to see a miss creates a Lock for that key
    and holds it while it recomputes. Every other concurrent request for
    the SAME key blocks on the same Lock instead of recomputing, then reads
    whatever the winner just wrote."""
    hit = _cache.get(key)
    now = time.monotonic()
    if hit and hit[0] > now:
        return {"value": hit[1], "origin_calls_so_far": _origin_calls}

    lock = _locks.setdefault(key, asyncio.Lock())
    async with lock:
        # Re-check AFTER acquiring the lock: a waiter that queued up behind
        # the winner will find the cache already warm and skip the origin
        # call entirely. Skipping this re-check is the single most common
        # way to implement single-flight WRONG.
        hit = _cache.get(key)
        now = time.monotonic()
        if hit and hit[0] > now:
            return {"value": hit[1], "origin_calls_so_far": _origin_calls}

        value = await slow_origin(key)
        _cache[key] = (now + CACHE_TTL, value)
        return {"value": value, "origin_calls_so_far": _origin_calls}
```

```python
# stampede_client.py — fires 100 concurrent requests at each endpoint.
# pip install httpx
import asyncio
import time

import httpx

BASE = "http://127.0.0.1:8000"
N = 100


async def hammer(client: httpx.AsyncClient, path: str, key: str) -> None:
    await client.get(f"{BASE}/{path}/{key}")


async def run(path: str, key: str) -> None:
    async with httpx.AsyncClient(timeout=30.0) as client:
        await client.get(f"{BASE}/reset")
        t0 = time.perf_counter()
        await asyncio.gather(*(hammer(client, path, key) for _ in range(N)))
        elapsed = time.perf_counter() - t0
        stats = (await client.get(f"{BASE}/stats")).json()
    print(f"{path:14s} elapsed={elapsed:6.2f}s  origin_calls={stats['origin_calls']:3d}  (of {N} requests)")


async def main() -> None:
    await run("naive", "hot-key")
    await run("single-flight", "hot-key")


if __name__ == "__main__":
    asyncio.run(main())
```

Run it (server in one terminal, client in another):
```
$ uvicorn stampede_demo:app --port 8000
$ python stampede_client.py
```

And the real, measured output from running exactly this code:
```
naive          elapsed=  1.16s  origin_calls=100  (of 100 requests)
single-flight  elapsed=  1.05s  origin_calls=  1  (of 100 requests)
```

**Why this works, and what the numbers prove:** the naive endpoint calls the simulated origin **100 times for 100 concurrent requests to the identical key** — zero protection, exactly the failure this section predicted, and the 1.16-second wall-clock time reflects a hundred 1-second `slow_origin` calls running concurrently (not serially — they're all genuinely in flight at once, just all needlessly). The single-flight endpoint calls the origin **exactly once**, and the other 99 requests get the winner's result by waiting on the same `asyncio.Lock` and then reading the cache the winner just populated, in the re-check immediately after acquiring the lock. The wall-clock time barely changes (1.05s versus 1.16s) — nearly every request still takes about a second, because 99 of them are genuinely waiting on the one real computation to finish — but the **origin load dropped from 100 to 1**, a 99% reduction, which is the number that actually matters: user-facing latency for this hot key was barely affected, while database or API load for that key dropped by two orders of magnitude.

**Honesty caveats — read these before treating this as production-ready, because it deliberately isn't:**
1. **`_cache` and `_locks` are plain Python dicts, and the lock is `asyncio.Lock()` — both are per-process, in-memory state.** They coordinate every request handled by *this one Python process*, and nothing else. The moment you run more than one worker process (Gunicorn/Uvicorn with multiple workers) or more than one replica of this service, each process has its *own* cache and its *own* lock, and a stampede across N processes still produces up to N origin calls — one per process, instead of one per request. Real production single-flight at that scale needs a **distributed** lock (Redis `SET key value NX` with an expiry is the common primitive, or the "lease" mechanism in the Facebook Memcache case study below) that every process, everywhere, actually shares.
2. **The blocking-call trap, demonstrated rather than just asserted:** `slow_origin` uses `await asyncio.sleep(...)`, not `time.sleep(...)`, and that choice is load-bearing, not stylistic. Swapping in `time.sleep(1.0)` inside an `async def` route blocks the *entire* event loop thread, not just the one request — measured directly: five concurrent requests to five *different* keys, with the origin function changed to `time.sleep(1.0)`, took **5.02 seconds total**, serialized one after another, instead of roughly 1 second running concurrently. That's not a stampede-specific bug; it's the general async foot-gun this note's stack conventions require calling out — a single blocking call inside an `async def` handler stalls every other concurrent request on that worker, for keys that have nothing to do with each other, which is strictly worse than the stampede this section is trying to fix.
3. This demo has no eviction policy, no memory bound on `_cache` or `_locks`, and no handling for `slow_origin` raising an exception while a lock is held (a real implementation needs to release the lock and let a subsequent request retry, not leave every waiter hanging on a poisoned lock forever). **Demo only — use a real cache (Redis) and a battle-tested distributed-lock or single-flight library in production**, exactly the caveat this note's conventions require for an in-memory store standing in for infrastructure it's deliberately not building.

### The agentic dimension: the identical stampede, on an LLM completion cache

This is the note's headline agentic connection, promised in the framing, and it transfers with essentially no translation required. An agent framework that caches LLM completions or embeddings — keyed by a hash of the prompt, to avoid paying for (and waiting on) an identical call twice — is running exactly the cache this section has been building: a key, a value, a time-to-live or an invalidation trigger, and a recomputation path that's expensive (an LLM API call costs real money and real seconds, arguably a more expensive "origin" than most database queries this note has used as examples). The moment that cache is used by **multiple concurrent workers** — parallel agent instances, or a single agent fanning out several tool calls that happen to share a cacheable sub-prompt — a popular cache key expiring under concurrent load produces the *identical* race condition demonstrated above: every worker sees a miss, every worker independently calls the LLM API for what is, semantically, the exact same request, and you pay for N identical completions instead of one. The fix is the same code, not just the same idea: a single-flight lock (backed by a real distributed primitive at scale, per the honesty caveat above) around the "check cache, call the model, populate cache" sequence, so that of N concurrent callers asking for the same prompt's completion, exactly one actually calls the model and the other N−1 await and reuse its result. This is worth building deliberately into any shared LLM-response or embedding cache from the start, precisely because the failure is invisible in low-traffic testing and appears exactly when it's most expensive: under real concurrent load.

### Judgment: an in-process lock versus a distributed lock versus probabilistic early expiration

**The choice:** you've identified a real stampede risk on a hot cache key. Do you use a simple in-process lock (as the runnable example does), a distributed lock shared across every process and replica (Redis-backed, typically), or a fundamentally different approach that avoids locking altogether — probabilistic early expiration?

**The realistic alternatives, and all three are genuinely used in production systems:**
1. **In-process lock** (`asyncio.Lock`, a `threading.Lock`, or a language-appropriate equivalent) — this note's runnable example.
2. **Distributed lock** — a shared external store (commonly Redis, using an atomic `SET ... NX EX` as the lock primitive, or a dedicated "lease" mechanism) coordinating across every process and every machine.
3. **Probabilistic early expiration** — no lock at all; instead, each process independently and randomly decides to refresh a cache entry *before* it actually expires, with a probability that increases the closer the entry gets to its true expiry, so that under heavy concurrent read load, one of the many readers is statistically very likely to trigger an early, solo refresh well before the hard deadline arrives and the herd would otherwise stampede.

**The actual reason to pick between them is entirely about topology, and it's worth stating precisely:** an in-process lock only ever coordinates requests handled by the *same* process, which is a real, useful, and cheap guarantee exactly when your deployment is a single process (or when a stampede confined to "one process's share of the load" is an acceptable, already-much-smaller problem) — this is why the runnable example's numbers (100→1) are genuinely representative for a single-worker deployment, and genuinely *not* representative the moment you scale to multiple workers. A distributed lock closes that gap by coordinating across every process everywhere, at the cost of a network round trip to the lock store on every cache miss and a new category of failure to design around: what happens if the process holding the lock crashes before releasing it (the reason real distributed locks are almost always implemented with an expiry/lease, not a lock that can be held forever) is a problem an in-process lock, scoped to one process's lifetime, doesn't have to solve. Probabilistic early expiration sidesteps *both* costs — no lock, no coordination, no network round trip on the common path — by changing the question from "how do we coordinate the recomputation" to "how do we make the recomputation happen just early enough, by exactly one reader, that a hard, synchronized expiry moment never actually arrives" — which is a fundamentally different strategy, not a variant of locking at all.

**The trade-off, honestly:** in-process locking is nearly free and trivial to add, but its protection is bounded by your process topology — it does nothing for a stampede spread across replicas, which is the common case for anything running behind a load balancer with more than one instance. Distributed locking closes that gap completely but adds real latency (a network round trip to the lock store, on the miss path specifically, which is already your slowest path) and a genuinely harder failure-mode surface: lock expiry tuned too short can release the lock while the original holder is still legitimately working, letting a second caller start a redundant recomputation anyway; tuned too long, a crashed holder blocks every other caller until the lease expires. Probabilistic early expiration avoids locking's failure modes entirely, but it's probabilistic by name and by nature — it reduces the *likelihood* of a synchronized stampede sharply without providing an absolute guarantee of exactly one recomputation, which matters if a recomputation has a side effect that's expensive or unsafe to run more than once concurrently (charging a payment provider's API to "refresh" a cached price, for instance, is a case where "almost certainly once" is a materially weaker guarantee than a lock's "provably at most one holder at a time").

**The flip condition:** stay with an in-process lock for a single-process deployment, or as a defense against the intra-process portion of the problem even when you also need something stronger for the cross-process portion. Move to a distributed lock when the recomputation has a side effect that must not run concurrently more than strictly necessary (an external API call you're billed per-call for, a write that isn't naturally idempotent) and your deployment spans more than one process or machine — the correctness guarantee is worth the added latency and failure-mode complexity there. Move to probabilistic early expiration when the recomputation is cheap enough, or idempotent enough, that "almost certainly exactly one, occasionally two" is a perfectly acceptable outcome, and when the cost of a network round trip to a distributed lock on every miss is itself a meaningful tax you'd rather avoid on a very hot, very latency-sensitive path — which is exactly the profile of the large-scale read caches both case studies below were built to protect.

### System design — single-flight for a product-detail cache shared across multiple worker processes

**Problem:** the product-detail endpoint from concept 1's worked example is now served by a Redis cache in front of the database, read by twelve independent application server processes (concept 3's horizontally-scaled-origin setting, revisited). A popular product's cache entry expiring under load currently produces up to twelve simultaneous database queries — one per process racing the identical check-then-compute sequence this concept opened with, each unaware of the other eleven. Design the fix.

**Requirements:** at most one process should recompute a given expired key at a time, regardless of which of the twelve processes happens to receive the request that discovers the miss; a crashed process mid-recomputation must not permanently block every other process from ever refreshing that key; the fix must not meaningfully slow down the overwhelming majority of requests, which are ordinary cache hits.

**The alternatives:** (1) this concept's in-process `asyncio.Lock`, applied independently in each of the twelve processes; (2) a distributed lock in Redis itself, using an atomic `SET key holder-id NX EX <lease>` as the lock primitive, checked before any process recomputes; (3) Facebook's "lease" pattern (detailed in the case study below) — the cache itself, not an external lock service, hands out a time-limited token to exactly one requester per key on a miss, and tells every other concurrent requester to briefly retry instead of proceeding to the database.

**The decision: (2) or (3), not (1) — the multi-process topology rules (1) out on its own, per the Judgment section above.**

**The actual reason:** with twelve independent processes, an `asyncio.Lock` in process #7 has no way to know process #3 is also mid-recomputation for the same key — they don't share memory, so they can't share a lock object. The stampede this design is trying to fix is *specifically* the cross-process case, so the fix has to live somewhere all twelve processes can see: either an external, shared lock (option 2) or logic built into the shared cache itself (option 3). Between those two, option (3)'s advantage is that it requires no separate lock-management code in the application at all — the cache's own `get`/`set` API is extended to also express "I don't have this, and nobody else appears to be computing it either — here's a lease, go compute it and set it before the lease expires" versus "I don't have this, but someone else has a live lease on it — wait and retry," which folds the coordination into the exact place the check-then-compute race already lives, rather than bolting a second, separate locking system on next to it.

**The trade-off:** option (2)'s hand-rolled distributed lock is more code an application team owns and must get exactly right (lease duration tuning, safe release, handling a process that dies holding the lock) — real distributed locking is a genuinely hard problem to get fully correct, which is part of why option (3) exists as a pattern: it's the cache infrastructure team's problem to solve once, correctly, rather than every application team's problem to re-solve. Option (3) requires cache infrastructure (or a client library) that actually implements leases, which is not a feature every caching layer provides out of the box — Redis alone doesn't have this built in; you build it on top using Redis's atomic operations, or you adopt a library/pattern that already has.

**Flip condition:** for a team without the infrastructure investment to build or adopt a leasing cache client, a straightforward Redis-backed distributed lock (option 2) using a well-tested library is the pragmatic default — it's more application-level code, but it's a well-understood pattern with mature libraries in most languages, versus building lease semantics into a caching layer from scratch.

### System design — probabilistic early expiration for a cache with very high fan-in

**Problem:** a single cache key backs a piece of content read by an extremely large number of concurrent requesters relative to how expensive recomputing it is to acquire cheaply — for instance, a computed ranking or aggregate that many thousands of requests per second read, where the recomputation itself is moderately expensive (a few hundred milliseconds) but not so expensive or side-effecting that occasionally running it twice concurrently would be a problem. Design the stampede protection.

**Requirements:** at this request volume, a distributed lock's added round-trip latency on every miss is itself a meaningful cost, paid by whichever unlucky requester's request happens to coincide with the expiry; the design should avoid creating a queue of thousands of waiters blocked on one lock at the exact moment of expiry, since even successfully coordinated waiting at that scale can itself become a bottleneck; occasional double-computation is an acceptable, low-cost outcome, unlike the payment-API example flagged in this concept's Judgment section.

**The decision: probabilistic early expiration (the XFetch approach, named in the case study below), not a lock.**

**The actual reason:** a lock-based approach — even a well-implemented distributed one — has every requester that arrives during the expired window queue up waiting on the same lock or lease, and at "thousands of requests per second all wanting this one key," that queue is itself a real structure with its own memory and scheduling cost, even though it correctly prevents redundant recomputation. Probabilistic early expiration eliminates the queue entirely by eliminating the synchronized expiry moment that creates it: each read, as it approaches the entry's true expiry, computes a small, randomized probability of treating the entry as "already expired" *early* and triggering a refresh right then, with the probability rising the closer the actual expiry gets. Because there are thousands of reads happening in that final approach window, one of them will very likely trigger the early refresh well before the hard deadline — spreading what would have been one synchronized stampede into a much smaller number of individually-triggered, staggered refreshes, with no lock and no queue required at all.

**The trade-off:** "very likely" is doing real work in that sentence — this is a probabilistic guarantee, not an absolute one, and under an unlucky distribution of random draws, more than one early refresh can still fire concurrently (a much smaller-scale, much less damaging echo of the original stampede, not a full recurrence of it). It also requires every reader to run a small amount of extra computation on every read (the early-expiration probability check), a cost a lock-free design pays on the *common* path in exchange for a lock-based design's cost on the *miss* path only.

**Flip condition:** use a lock (in-process or distributed, per the Judgment section) whenever the recomputation is expensive or side-effecting enough that even a small, non-zero probability of concurrent double-execution is unacceptable; reach for probabilistic early expiration specifically at the high-fan-in end of the spectrum, where the coordination cost of locking at scale rivals or exceeds the cost of occasionally recomputing something twice.

### Case study — Facebook's Memcache leases: solving thundering herds at social-network scale

**What it is.** Nishtala et al.'s "Scaling Memcache at Facebook" (NSDI 2013) documents the architecture behind Facebook's memcached deployment — a system handling, in the paper's own description, billions of requests per second across trillions of cached items. Among the specific production problems the paper addresses head-on is exactly this concept's subject, under the name **thundering herds**: a specific key undergoing very heavy concurrent read traffic, such that when it's invalidated or expires, a large number of clients simultaneously miss and simultaneously query the backing database for the same data. Facebook's fix, called a **lease**, is architecturally close to this section's third alternative (option 3 in the system design above): on a miss, memcached hands the *first* requester a lease token — permission to go compute the value and write it back — and tells every other concurrent requester for that same key to retry after a short delay instead of proceeding to the database itself. The paper reports this mechanism, alongside a related fix for a second problem (stale sets, where a slow write could overwrite a newer value with an older one), **reduced peak database query rates attributable to this failure mode from roughly 17,000 queries per second to roughly 1,300 queries per second** — better than a 90% reduction, at the scale of one of the largest cache deployments that has ever published details of its own internals.

**The engineering lesson, tied directly to this concept's mechanism:** this is a real, at-scale, independent confirmation that the stampede problem this section built from a 100-request toy example is not a toy problem — it's a documented, named, and specifically engineered-around failure mode at one of the largest cache deployments in the industry, and the fix Facebook shipped is structurally the same idea as the single-flight lock in this note's runnable example (coordinate so exactly one caller recomputes, make the rest wait), generalized into the cache's own protocol rather than bolted onto the application layer, precisely the trade-off this concept's first system design just walked through.

*Primary source:* Nishtala et al., "Scaling Memcache at Facebook," 10th USENIX Symposium on Networked Systems Design and Implementation (NSDI 2013) — available via usenix.org. **Verify current**: the 17K/1.3K figures are historical, describing Facebook's system and traffic as of the 2013 paper; they are not current production numbers and should be cited as the paper's own reported result, not as an ongoing or present-day figure.

### Case study — XFetch: probabilistic early expiration as an alternative to locking

**What it is.** Vattani, Chierichetti, and Lowenstein's "Optimal Probabilistic Cache Stampede Prevention" (Proceedings of the VLDB Endowment, 2015) formalizes and names the probabilistic-early-expiration approach used in this concept's second system design, under the algorithm name **XFetch**. The paper frames the cache-stampede problem in the identical terms this section has used — frequently-accessed items whose expiration triggers simultaneous concurrent regeneration attempts — and proves that a specific randomized early-refresh strategy, using an exponential-distribution-based decision at read time (the probability of treating an entry as expired early rises as the entry approaches its true expiry), is optimal in a precise, formal sense among lock-free strategies for this problem, and demonstrably outperforms simpler heuristics used in real-world systems at the time of writing.

**A deliberate correction, in the spirit of this note's honesty requirements:** it would be easy to assume, given how this algorithm is described in some secondary sources, that XFetch originated inside a large company like Google — it did not. The paper's listed affiliations at time of publication were Goodreads/Amazon (Vattani), Sapienza University of Rome (Chierichetti), and Bugsnag (Lowenstein): an industry-and-academia collaboration, not a single big-technology-company internal system. **Verify current** before citing any specific company as having adopted XFetch by name in production — this note is citing the algorithm and its formal treatment, not asserting a specific deployment, because asserting one without direct confirmation would be exactly the fabrication this note's conventions forbid.

**The engineering lesson, placed deliberately next to the Facebook case study rather than instead of it:** these two case studies are not "one right answer and one alternative" — they're evidence for the Judgment section's actual claim, that **the right stampede-prevention mechanism depends on topology and cost, not on which is more sophisticated.** Facebook's leases solve the problem with a small amount of added latency (a retry delay for non-winning requesters) and an absolute guarantee (exactly one computer per miss, modulo lease expiry edge cases); XFetch solves a related but distinct shape of the problem with zero added coordination and a probabilistic, not absolute, guarantee. Both are real, published, and in active use in different parts of the industry — knowing which shape of problem you actually have is the judgment this concept has been building toward, not memorizing that one of the two is "the modern one."

*Primary source:* Vattani, Chierichetti, and Lowenstein, "Optimal Probabilistic Cache Stampede Prevention," Proceedings of the VLDB Endowment, Vol. 8, No. 8 (2015), pp. 886–897. **Verify current**: as with the Facebook paper, treat the specific affiliations and performance comparisons as accurate to the paper's 2015 publication, and check current literature if citing state-of-the-art comparisons rather than the original algorithm.

### In production — operating stampede protection

**Best practices:**
1. **Assume any cache key that's both hot (high read concurrency) and expensive to recompute will eventually stampede, and design for it before the first incident, not after.** The failure is invisible in low-traffic testing by construction — it only appears under exactly the concurrent load that makes it expensive.
2. **Match the mechanism to the topology honestly**, per the Judgment section — an in-process lock in a single-process demo proves the *concept*, not the production fix, the moment you scale past one process.
3. **Pair whatever locking mechanism you choose with `stale-while-revalidate` (concept 2) wherever the cached value can tolerate a few seconds of staleness** — letting the *other* 99 requests get an instantly-served stale answer while the one winner recomputes is strictly better for user-facing latency than making them wait on the lock at all, and the two techniques compose cleanly: the lock controls who recomputes, `stale-while-revalidate` controls what everyone else sees while that happens.
4. **Give recomputation-side failures an explicit path**, not an implicit one: if the process holding a lock or lease crashes or the recomputation throws, every other waiter needs a way to eventually proceed (a lease timeout, a retry-with-backoff) rather than blocking forever on a coordination primitive whose owner is gone.

**Mistakes, beginner → senior:**
- *Beginner:* not noticing the check-then-compute race exists at all, because it never shows up in local development or low-concurrency testing — precisely this concept's warning.
- *Intermediate:* implementing an in-process lock (this note's runnable example, taken as final rather than as a teaching device) and discovering, only once deployed behind a load balancer with multiple workers, that the stampede is reduced but not eliminated.
- *Senior:* choosing a lock-based approach for an extremely high-fan-in key where the lock/lease queue itself becomes a bottleneck, when a probabilistic approach was the better fit for that specific traffic shape — or the mirror mistake, choosing a probabilistic approach for a recomputation with a real side effect that genuinely cannot tolerate occasional double-execution.

**Monitoring:** origin/database query rate for a specific hot key around its expiry moments, watched as a time series rather than an aggregate — a stampede shows up as a sharp, brief spike exactly at expiry, easy to miss in an average but unmistakable in a per-second graph; lock/lease wait time and timeout rate, as a direct signal of whether the coordination mechanism itself is becoming a bottleneck at your actual traffic volume.

**Cost:** an unprotected stampede's cost is paid by whatever's behind the cache — a database that receives a hundred-fold spike in queries for a fraction of a second, an LLM API billed per call receiving N duplicate, wasted charges for what should have been one call. The cost of *fixing* it is comparatively small: an in-process lock is nearly free; a distributed lock costs a network round trip on the miss path only; probabilistic early expiration costs a small amount of extra computation on every read in exchange for zero coordination overhead — in every case, cheaper than the failure it prevents, which is why this is one of the rare corners of caching where the honest cost conversation favors doing the work early rather than waiting to see if it's needed.

---

## Interview questions and practice

### Conceptual

1. **Walk through what "caching an HTTP response" actually means, and name the four layers a response can be cached at.** *(Private/browser, shared/proxy, CDN edge, origin-side application cache — concept 1. The first three are governed by `Cache-Control`/validators/`Vary`; the fourth is an application-level optimization outside the HTTP spec.)*
2. **What's the actual difference between `no-cache` and `no-store`?** *(`no-cache`: may store, must revalidate before every use. `no-store`: must not retain any part of it, anywhere.)*
3. **Why does `max-age` exist when `Expires` already did the same job?** *(`Expires` is an absolute timestamp, dependent on clock agreement between machines; `max-age` is a relative duration, immune to clock skew — the identical lesson as Day 12's DNS TTL being relative rather than absolute.)*
4. **What problem does an ETag solve that `Last-Modified` doesn't?** *(One-second timestamp granularity and cross-machine clock dependence; an ETag is an opaque, clock-free content identifier, typically a hash.)*
5. **What must a `304 Not Modified` response never contain, and what must it still include?** *(No body, ever. Must repeat the cache-relevant headers — `Cache-Control`, `ETag`, etc. — per RFC 7232 §4.1, so the client can refresh its freshness bookkeeping.)*
6. **Why can inode-based ETags break under load balancing?** *(An inode number is per-machine filesystem metadata, not derived from content — two replicas serving byte-identical files can assign different inodes, producing different ETags for identical content and permanently defeating conditional requests for that resource.)*
7. **What does `Vary: Accept-Encoding` actually change about how a cache stores a response?** *(Extends the cache key from method+URL to method+URL+the named header's value, so gzip and uncompressed variants of the same URL are stored and served as separate entries.)*
8. **Why is `Vary: User-Agent` usually a mistake?** *(User-Agent has extremely high cardinality — thousands of practically distinct values — so varying on it fragments the cache into near-single-use entries, collapsing hit ratio.)*
9. **What is a PoP, and what mechanism gets a user's request to the nearest one?** *(A physical CDN location; anycast — the same IP advertised from every PoP, with ordinary BGP routing delivering each user to the topologically closest one. Identical mechanism to Day 12's root-server anycast.)*
10. **What problem does origin shielding solve that a single reverse-proxy cache never has?** *(Fan-in: hundreds of independent edge PoPs each missing and hitting the origin simultaneously. A shield tier absorbs that fan-in so only the shield's own miss reaches the true origin.)*
11. **Why doesn't a CDN purge fully solve the "DNS has no invalidation" problem HTTP inherited from Day 12?** *(A purge is real, unlike DNS — but it has measurable, non-zero, vendor-specific propagation delay across a distributed edge network, so it mitigates staleness rather than eliminating the underlying risk.)*
12. **Why does a versioned/hashed filename make cache invalidation unnecessary rather than just easier?** *(The filename is a function of the content; if the content changes, the hash changes, which means it's a different URL — the old URL's content, by construction, can never change again, so there's nothing to invalidate.)*
13. **What's the "HTML shell problem," and why is it the opposite policy from the assets it references?** *(The entry HTML file names which hashed bundle to load and must update on every deploy — long-cached HTML means users stuck loading old, possibly-deleted bundles forever. Assets get `immutable, max-age=31536000`; the shell gets `no-cache`.)*
14. **What exactly is a cache stampede, and how is it different from origin-shielding's fan-in problem?** *(Many concurrent requests to the SAME cache location racing a check-then-compute sequence when a hot key expires, versus MANY DIFFERENT cache locations each independently missing. Different topology, different fix.)*
15. **What does a single-flight lock actually do, mechanically?** *(The first requester to see a miss acquires a lock and recomputes; every other concurrent requester for the same key blocks on the same lock, then re-checks the cache after acquiring it and finds the winner's fresh value instead of recomputing.)*
16. **Why does an in-process lock stop working the moment you scale to multiple worker processes?** *(Each process has its own memory — a lock in process A is invisible to process B, so N processes racing the same key can still each independently stampede once, one per process.)*

### Diagnostic scenarios

17. **A team changed a CDN-cached page's content, but users report seeing the old version for up to a minute afterward, inconsistently by region.** *(Purge propagation delay — concept 6. Expected behavior, not a bug; the fix is either tolerating the window or switching that content to versioned URLs if it's under your own build control.)*
18. **An API endpoint behind a shared CDN started returning one authenticated user's data to a different, unauthenticated user.** *(Missing or wrong `Cache-Control` scope directive — the endpoint needed `private, no-store` and either had no explicit directive or relied on a shared cache's default inference, precisely the Web Cache Deception failure class.)*
19. **A team added conditional-request support to an endpoint and the `304` rate stayed at zero.** *(Almost certainly a non-deterministic serialization producing a new hash on every call — check for unstable dict ordering or an embedded timestamp in the response body before the hash is computed.)*
20. **After switching a route from `s-maxage` to `Vary: Accept-Language`, CDN hit ratio for that route collapsed.** *(Check the cardinality of the varying header — if it's a proxy for something with far more distinct values than intended, e.g., a raw locale string with regional variants nobody actually serves differently, it's fragmenting the cache exactly like `Vary: User-Agent`.)*
21. **A hot product page's database load spikes to 100x baseline for about one second, exactly every 60 seconds.** *(Cache stampede on a `max-age=60` key with no single-flight protection — the periodicity is the giveaway; fix with a lock, `stale-while-revalidate`, or both.)*
22. **The same fix (a single-flight lock) was applied, and the spike dropped from 100x to roughly (number of app server replicas)x instead of 1x.** *(In-process lock in a multi-process deployment — the fix reduced the stampede per-process but didn't coordinate across processes; needs a distributed lock or lease.)*

### Design questions

23. **Design the caching strategy for a public marketing site and an authenticated dashboard sharing one CDN.** *(Opposite defaults per route class — aggressive public caching for marketing, `private, no-store` by default for the dashboard — or separate hostnames once the team/traffic justifies it. Concept 2's system design.)*
24. **Design global delivery for a frontend build's static assets such that a deploy never requires a CDN purge.** *(Content-hashed filenames with `immutable, max-age=31536000`; a short-lived, always-revalidated HTML shell referencing the current build's hashes. Concept 6's system design.)*
25. **Design caching for a news homepage that must reflect new articles within seconds at a million requests per second.** *(Purge-on-publish decoupled from `max-age`, layered with `stale-while-revalidate` so no reader ever blocks on a synchronous origin fetch. Concept 6's second system design.)*
26. **Design stampede protection for a cache read by thousands of concurrent requests per second, where the recomputation is cheap and idempotent.** *(Probabilistic early expiration (XFetch) rather than a lock — avoids building a large waiter queue at the exact moment of high fan-in. Concept 7's second system design.)*

---

# Topic-wide wrap-up

## Glossary

**Age** — a response header, added by shared caches, stating how many seconds have passed since the response was generated or last validated at the origin, accumulated across every cache the response passed through.

**Anycast** — advertising the same IP address from many physical locations via BGP, so ordinary internet routing delivers each client to the topologically nearest instance; the mechanism behind both DNS root-server placement (Day 12) and CDN edge-PoP routing.

**Cache-Control** — the response (and request) header defining freshness, scope, and staleness/revalidation policy for a cached representation; the central mechanism of HTTP caching, specified by RFC 9111.

**Cache key** — the set of values a cache uses to decide whether two requests may share a stored response; at minimum method + URL, extended by any headers named in `Vary` and, at CDN scale, by cache-key configuration over the query string.

**Cache stampede** (also thundering herd, dogpile) — many concurrent requests independently missing on the same expired or invalidated cache key and independently recomputing/re-fetching at once, temporarily removing the cache's protective effect on whatever sits behind it.

**CDN (content delivery network)** — a network of geographically distributed cache servers (PoPs) that store cacheable content close to users rather than close to the origin, using the same HTTP caching rules as any other shared cache.

**Conditional request** — a request carrying a validator-based precondition (`If-None-Match`, `If-Modified-Since`, etc.) asking the origin to act only if that precondition holds; the mechanism behind `304 Not Modified`, specified by RFC 7232.

**Distributed lock** — a lock coordinated through a shared external store (commonly Redis) rather than in one process's memory, so it can prevent a stampede across multiple processes or machines, at the cost of a network round trip and its own failure modes (lease expiry, crashed holders).

**ETag (entity tag)** — an opaque, origin-assigned identifier for a specific version of a representation, used as a validator; typically a hash of the response body (strong) or a cheaper approximate identifier (weak, prefixed `W/`).

**Heuristic freshness** — a cache's own estimate of freshness lifetime when no explicit `Cache-Control` or `Expires` is present, commonly ~10% of the time elapsed since `Last-Modified`, per RFC 9111 §4.2.2.

**HTML shell problem** — the pattern where a build's entry HTML file must be cached very briefly (it names which hashed asset filenames to load) even though the assets it references can be cached immutably forever.

**Immutable** — a `Cache-Control` extension (RFC 8246) telling a browser that a response will not change during its freshness lifetime, so a user-initiated reload need not even attempt revalidation; meaningful only paired with a URL that structurally cannot describe two different contents, such as a content-hashed filename.

**Lease** — Facebook Memcache's stampede-prevention mechanism: on a miss, the cache hands the first requester a time-limited token to recompute, and tells concurrent requesters for the same key to retry shortly rather than recompute themselves.

**Last-Modified** — the original HTTP/1.0 validator: a timestamp attached to a response, checked via `If-Modified-Since` on a subsequent request; limited by one-second granularity and cross-machine clock dependence.

**`max-age`** — a `Cache-Control` directive setting freshness lifetime in seconds from response generation, honored by both private and shared caches.

**`must-revalidate`** — a `Cache-Control` directive forbidding a cache from serving a stale copy for any reason, including origin unreachability, once its freshness lifetime has passed.

**`no-cache`** — a `Cache-Control` directive permitting storage but requiring revalidation with the origin before every use, regardless of apparent freshness; not the same as `no-store`.

**`no-store`** — a `Cache-Control` directive forbidding any cache anywhere from retaining any part of the request or response.

**Origin** — the server that generates a response the first time, before any cache has a copy of it.

**Origin shielding** — designating one cache tier (a "shield") that every edge PoP's miss must go through, so only the shield's own miss reaches the true origin, coalescing fan-in from many PoPs down to one.

**PoP (point of presence)** — one physical location in a CDN's network, holding some number of edge servers with their own connectivity to nearby ISPs.

**Private cache** — a cache serving exactly one user (a browser's own cache), permitted to store personalized responses that a shared cache must not.

**Probabilistic early expiration** — a lock-free stampede-prevention strategy (the XFetch algorithm) in which each read independently and randomly decides, with rising probability as true expiry approaches, to treat an entry as expired early and trigger a solo refresh before a synchronized hard-expiry stampede can occur.

**Public** — a `Cache-Control` directive permitting a shared cache to store a response even in cases (such as a response to an `Authorization`-bearing request) it would otherwise treat with more caution by default.

**Purge** (invalidation) — an explicit, API-driven request telling caches to discard a stored copy immediately, rather than waiting for it to expire naturally; has real, non-zero, vendor-specific propagation delay across a distributed cache network.

**`s-maxage`** — a `Cache-Control` directive setting freshness lifetime specifically for shared caches, overriding `max-age` for them while leaving `max-age` in effect for private caches.

**Shared cache** — a cache serving more than one user (a corporate proxy, a CDN edge node), subject to stricter default storage rules than a private cache because a mistake leaks one user's response to another.

**Single-flight** — the general pattern of coordinating concurrent callers for the same cache key so that exactly one of them performs the expensive recomputation while the rest wait for and reuse its result.

**`stale-if-error`** — a `Cache-Control` extension (RFC 5861) permitting a cache to serve a stale response, for a bounded extra time, when revalidation fails due to an origin error, rather than propagating the failure to the user.

**`stale-while-revalidate`** — a `Cache-Control` extension (RFC 5861) permitting a cache to serve an already-stale response immediately while revalidating it in the background, for a bounded extra time past the original freshness lifetime.

**Strong validator / weak validator** — RFC 7232's two comparison modes for ETags: strong certifies byte-for-byte identity; weak (prefixed `W/`) certifies only semantic equivalence, at lower generation cost.

**Vary** — a response header (RFC 9110 §12.5.5) naming additional request headers that must match before a cache may reuse a stored response, extending the cache key beyond method + URL.

**Versioned/hashed URL** — a URL whose filename encodes a hash or version of its content, guaranteeing the content at that exact URL can never change, which makes long, `immutable` caching safe by construction and removes the need to ever invalidate it.

**Web Cache Deception** — an attack (Omer Gil, 2017) in which a crafted URL causes a personalized page to be cached by a shared cache under a URL an unauthenticated or different user can then request, exploiting cacheability decided by URL shape rather than explicit `Cache-Control`.

## Cheat sheet

**The four cache layers**
```
private (browser)  →  shared/proxy  →  CDN edge  →  origin-side app cache (Redis, etc.)
Governed by Cache-Control/ETag/Vary ─┘              Not part of the HTTP spec at all.
```

**`Cache-Control` directives, what question each answers**
| Directive | Answers | Applies to |
|---|---|---|
| `max-age=N` | how long is this fresh, in seconds | private + shared |
| `s-maxage=N` | how long is this fresh, for SHARED caches only (overrides max-age there) | shared only |
| `private` | only a single-user cache may store this | scope |
| `public` | a shared cache may store this even for an `Authorization`-bearing request | scope |
| `no-cache` | may store, but MUST revalidate before every use | revalidation |
| `no-store` | may not store any part of this, anywhere | revalidation |
| `must-revalidate` | once stale, never serve stale — not even if origin is unreachable | revalidation |
| `stale-while-revalidate=N` | serve stale immediately for up to N more seconds while refreshing in the background | staleness |
| `stale-if-error=N` | serve stale for up to N more seconds if revalidation itself errors | staleness |
| `immutable` | never revalidate during freshness lifetime, even on a user reload | freshness + validators |

**Freshness arithmetic (RFC 9111 §4.2)**
```
freshness_lifetime = s-maxage (shared cache) > max-age > Expires−Date > ~10% of Last-Modified age > 0
current_age        = age already reported by upstream + time held by THIS cache
is_fresh            = freshness_lifetime > current_age
```

**Conditional requests**
```
ETag: "<hash>"                              ← origin's validator, opaque, ideally a content hash
If-None-Match: "<hash>"                     ← client's "only send me a body if this changed"
304 Not Modified                            ← NO BODY, but repeats Cache-Control/ETag/etc.
Strong match: byte-identical.  Weak (W/"..."): semantically equivalent only.
```

**Vary — extends the cache key**
```
key = (method, URL) + { header: value  for header in Vary }
Safe: Accept-Encoding, Accept-Language (small, bounded value sets)
Dangerous: User-Agent, Cookie (huge/unbounded value sets → cache fragmentation)
```

**CDN vocabulary and header cues (verify current per vendor)**
| Vendor | Shield/tiered-cache feature name | Cache status header |
|---|---|---|
| Cloudflare | Tiered Cache / Smart Tiered Cache | `cf-cache-status`, `cf-ray` (PoP suffix) |
| Fastly | Shielding | `x-served-by`, `x-cache`, `x-cache-hits` (per hop) |
| CloudFront | Origin Shield | `x-cache: Hit/Miss from cloudfront`, `x-amz-cf-pop` |

**Invalidation: two strategies, not competitors**
```
PURGE            — for URLs you don't control the naming of (CMS slugs, API paths).
                   Real, but has non-zero, vendor-specific propagation delay.
VERSIONED URL    — for anything your own build controls. Content hash IN the filename
                   → old URL's content literally can't change → cache it forever,
                   immutable, max-age=31536000. Nothing to invalidate, ever.
HTML SHELL RULE  — the file that NAMES the hashed assets must be no-cache/short-lived;
                   the assets it names can be cached forever.
```

**Cache stampede: pick the fix by topology**
| Topology / cost profile | Fix |
|---|---|
| Single process | in-process lock (`asyncio.Lock` / threading lock) |
| Multiple processes/replicas, side-effecting recompute | distributed lock or lease (Redis `SET NX EX`, Facebook-style lease) |
| Very high fan-in, cheap/idempotent recompute | probabilistic early expiration (XFetch) |
| Any topology, staleness-tolerant | pair with `stale-while-revalidate` so waiters aren't blocked |

## Build this

### Task 1 — real ETag/`304` handling, measured

- [ ] Build a FastAPI endpoint that computes a SHA-256 ETag over its own serialized response body and returns it in an `ETag` header alongside `Cache-Control`.
- [ ] Add `If-None-Match` handling: return an empty-bodied `304` with the same `ETag`/`Cache-Control` headers when the client's tag matches.
- [ ] Prove it with `curl -i`: capture one `200` (full body) and one `304` (no body) for the same unmodified resource.
- [ ] Measure bytes saved with `curl -s -o /dev/null -w '%{http_code} %{size_download}\n'` on both requests and state the percentage reduction.
- [ ] Mutate the underlying data, repeat the conditional request with the *old* ETag, and confirm you get a fresh `200` with a *new* ETag, not a stale `304`.
- [ ] Explain, in your own words, why `json.dumps(..., sort_keys=True)` (or an equivalent stable-serialization choice) matters for this to work at all.

**Definition of done:** you have a real transcript showing both status codes, real byte counts, and you can explain why the hash must be computed over the exact bytes being served.

### Task 2 — reproduce a cache stampede, then fix it

- [ ] Build a cache-backed endpoint with a short TTL (1–2 seconds) in front of a simulated slow operation (`asyncio.sleep`, not `time.sleep` — see below).
- [ ] Fire 100 concurrent requests for the same key at the endpoint right as its cache entry is expired or empty, and record how many times the slow operation actually ran.
- [ ] Add a single-flight lock (`asyncio.Lock`, keyed per cache key) with a re-check of the cache immediately after acquiring the lock, and repeat the same 100-concurrent-request test.
- [ ] Report both numbers side by side (naive origin-call count vs. protected origin-call count) and the wall-clock time for each run.
- [ ] Deliberately swap the simulated slow operation to `time.sleep` instead of `await asyncio.sleep`, fire concurrent requests to *different* keys, and measure how much slower they become — this is the blocking-call trap, made visible rather than asserted.
- [ ] Write down, honestly, what would break if this exact code were deployed behind two worker processes instead of one — and what you'd need to change to fix it (this is the multi-process honesty caveat from concept 7).

**Definition of done:** you have real, measured before/after origin-call counts proving the lock works, a measured demonstration of the blocking-call trap, and a written, correct answer for why this specific fix doesn't survive scaling to multiple processes.

### Task 3 — read real CDN headers in the wild

- [ ] Pick three CDN-fronted sites you use regularly (check with `curl -sI <url>` for headers like `cf-ray`, `x-served-by`, `x-amz-cf-pop`, `via`).
- [ ] For each, identify which CDN vendor is in use from the header vocabulary alone, and identify the specific PoP or region that answered you.
- [ ] Find at least one response with an `Age` header greater than 60 and explain what it tells you about how long that object has been cached without an origin hit.
- [ ] Find at least one response using `stale-while-revalidate` or `stale-if-error` in the wild and explain, from the site's likely traffic pattern, why that directive makes sense there.

**Definition of done:** three real, annotated header captures, with the vendor, the PoP, and at least one caching directive correctly explained from the trace alone.

### Stretch — the two case studies as drills

- [ ] **Fastly drill:** name one dependency your own project (or a project you use daily) has on a single CDN or caching layer, and write down what "the CDN is unreachable for an hour" would actually do to it — and whether you have a tested fallback.
- [ ] **Stampede drill:** identify one real cache in a system you've built or used (even a simple `functools.lru_cache` or a Redis-backed one) that has no stampede protection, and write the specific single-flight or `stale-while-revalidate` fix you'd apply, and why that mechanism over the alternatives.

---

## Active recall and self-test

Answer from memory, in writing, before checking.

1. Name the four layers an HTTP response can be cached at, and which one isn't governed by the HTTP spec at all.
2. State the exact difference between `no-cache` and `no-store`, using the word "revalidate" correctly in your answer.
3. Why is `max-age` immune to a problem `Expires` has? Name the problem.
4. What two limitations of `Last-Modified` motivated the ETag?
5. What must a `304` response never include, and what must it still include?
6. Explain, precisely, why an inode-based ETag breaks under horizontal scaling.
7. What does `Vary` do to a cache key, mechanically?
8. Why is `Vary: User-Agent` usually a mistake, in one sentence about cardinality?
9. What is anycast, and where else in this course have you seen the identical mechanism?
10. What specific new problem does a CDN have that a single reverse-proxy cache never faces, and what solves it?
11. Is a CDN purge instant? What did real vendor documentation say about propagation time?
12. Why does a content-hashed filename make invalidation unnecessary rather than merely faster?
13. What is the "HTML shell problem," and what's the opposite-extremes policy that fixes it?
14. Define a cache stampede in one sentence, distinct from origin-shielding's fan-in problem.
15. Walk through, step by step, why an in-process lock reduces a 100-concurrent-request stampede to one origin call — and why it stops working with more than one process.
16. Name the three stampede-prevention strategies from this note's Judgment section, and the one property of the recomputation (side-effecting vs. idempotent, low vs. very high fan-in) that decides between them.
17. What did Facebook's "lease" mechanism actually do, and by what factor did it reduce peak database query rate in the published paper?
18. What is XFetch, who actually published it, and what guarantee does it give up compared to a lock?
19. Two case studies: what made the Fastly outage total rather than partial for affected customers, and what single fact about Netflix Open Connect makes it the "logical endpoint" of everything else in concept 5?
20. Explain the agentic connection between an LLM-completion cache and this note's stampede mechanism, in your own words, without looking back at the text.

### 60-second teach-back

> **"An HTTP response is expensive to generate and expensive to send across a real distance, so at every layer — your browser, a shared proxy, a CDN edge, your own application — something keeps a copy close by instead of asking again. `Cache-Control` is the origin's explicit, in-advance promise about how long that copy may be trusted, and it's the exact same TTL-is-a-promise logic as a DNS record's TTL, just applied to a response instead of a name. What HTTP has that DNS doesn't is a validator — an ETag, usually a hash of the body — that lets a client ask 'did this actually change?' and get back an empty `304 Not Modified` instead of paying for the whole body again on the common case where nothing did. A CDN takes the exact same rules and runs them at hundreds of physical locations near users instead of one cache near the origin, using anycast — the same trick DNS uses for its root servers — to route each user to the nearest one; and because hundreds of independent caches can all miss at once, CDNs need an extra trick, origin shielding, that a single cache never needs. Unlike DNS, a CDN does let you actively purge a stale entry — but that purge takes real, non-zero, vendor-specific time to reach every location, so the strategy that actually removes the problem is to never need invalidation at all: bake a content hash into the filename, so the URL's content literally cannot change, and cache it forever. And the last failure mode is the one that doesn't care whether you're caching HTML or an LLM completion: when a hot key expires under heavy concurrent load, every waiting request can independently decide to recompute at once — a stampede — and the fix, a single-flight lock so exactly one caller pays the cost while everyone else waits for its answer, is the same code whether the expensive thing behind the cache is a database query or a call to a language model."**

If you can deliver that, and then explain why an `asyncio.Lock` alone doesn't fix a stampede once you run two worker processes, you have this topic.

---

## Spaced-repetition flashcards

| Q | A |
|---|---|
| Four HTTP cache layers | Private (browser), shared/proxy, CDN edge, origin-side app cache — only the last isn't part of the HTTP spec |
| `no-cache` vs `no-store` | `no-cache`: store, but revalidate every time. `no-store`: never store any part, anywhere |
| Why `max-age` beats `Expires` | Relative duration, immune to clock skew between machines — `Expires` is an absolute timestamp |
| Two limits of `Last-Modified` that motivated ETag | One-second granularity; cross-machine clock dependence |
| What must a `304` never contain? | A body |
| What must a `304` still contain? | `Cache-Control`, `ETag`, and other freshness-relevant headers (RFC 7232 §4.1) |
| Strong vs weak ETag comparison | Strong: byte-identical. Weak (`W/"..."`): semantically equivalent only |
| Why do inode-based ETags break under load balancing? | Inode is per-machine filesystem metadata, not content-derived — replicas disagree on identical content |
| What does `Vary` do? | Extends the cache key with named request headers' values |
| Safe vs dangerous `Vary` headers | Safe: `Accept-Encoding`/`Accept-Language` (bounded). Dangerous: `User-Agent`/`Cookie` (huge cardinality) |
| What is anycast, in one line? | Same IP advertised from many locations; BGP routing delivers each client to the nearest one |
| Where else does anycast appear in this course? | DNS root servers (Day 12) |
| What new problem does a CDN have vs. one reverse proxy? | Fan-in: many independent PoPs miss and hit origin simultaneously |
| What solves CDN fan-in? | Origin shielding — one tier absorbs every edge miss before it reaches the true origin |
| CloudFront / Fastly / Cloudflare shield feature names | Origin Shield / Shielding / Tiered Cache (verify current) |
| Is a CDN purge instant? | No — real, vendor-documented, non-zero propagation delay (order of tens of ms to tens of seconds depending on vendor/method) |
| Why does a content-hashed URL remove the need to invalidate? | The filename is a function of content — different content ⇒ different URL; the old URL's content can never change again |
| The HTML shell problem, one line | The file that names the hashed assets must be short-lived even though the assets themselves can be cached forever |
| Cache stampede, one line | Many concurrent requests for the same expired key all independently recompute at once |
| Stampede vs. origin-shielding fan-in — the difference | Stampede = many REQUESTS, one cache. Fan-in = many CACHES (PoPs), one origin |
| What does a single-flight lock do? | First miss recomputes and holds a lock; everyone else waits, then re-checks cache and reuses the winner's result |
| Why does an in-process lock fail with multiple workers? | Each process has its own memory — a lock in one process is invisible to another |
| Three stampede-prevention strategies | In-process lock, distributed lock/lease, probabilistic early expiration (XFetch) |
| When to pick a distributed lock over probabilistic early expiration | Recomputation is side-effecting/expensive-per-call and must not run concurrently more than necessary |
| Facebook Memcache "lease," in one line | First miss gets a token to recompute; other concurrent misses are told to retry shortly instead of hitting the DB |
| Facebook's measured improvement from leases | ~17,000 queries/sec peak reduced to ~1,300 queries/sec (Nishtala et al., NSDI 2013) |
| XFetch's core idea | Each read randomly, with rising probability near true expiry, treats an entry as expired early and refreshes solo |
| Who actually published XFetch? | Vattani, Chierichetti, Lowenstein (Goodreads/Amazon, Sapienza, Bugsnag) — VLDB 2015, not a big-company internal system |
| Fastly, June 8 2021 — root cause | A latent bug from a May 12 deploy, triggered by one customer's valid config change; 85% of network errored |
| Fastly, June 8 2021 — recovery time | Detected in ~1 minute; 95% of network normal within 49 minutes |
| Netflix Open Connect, in one line | Netflix embeds its own cache appliances physically inside ISPs' own networks, closer than any rented PoP could be |
| Netflix's own quoted traffic figure (verify current) | "Close to 90%" of traffic delivered via direct Open Connect–ISP connections |
| Web Cache Deception, in one line | A crafted URL tricks a cache into storing a personalized page, later served to a different, unauthenticated user |
| What's the fix Web Cache Deception argues for? | Explicit `private, no-store` on anything personalized — never rely on a cache's inferred cacheability |

---

## Primary sources

**Core HTTP caching and conditional-request specifications**
- **RFC 9111** — HTTP Caching (obsoletes RFC 7234, which obsoleted the original RFC 2616 caching text). The current normative definition of `Cache-Control`, freshness computation, and shared-vs-private cache rules.
- **RFC 7232** — HTTP/1.1 Conditional Requests. Defines ETag, `If-None-Match`, `If-Modified-Since`, `If-Match`, `If-Unmodified-Since`, and strong vs. weak comparison.
- **RFC 5861** — HTTP Cache-Control Extensions for Stale Content. Defines `stale-while-revalidate` and `stale-if-error`.
- **RFC 8246** — HTTP Immutable Responses. Defines the `immutable` Cache-Control extension.
- **RFC 9110** §12.5.5 — HTTP Semantics: the `Vary` header field.

**CDN and infrastructure documentation (verify current — these drift)**
- AWS documentation, "Use Amazon CloudFront Origin Shield."
- Fastly documentation, "Shielding" and "Lifetime and revalidation" (concepts/edge-state/cache/stale).
- Cloudflare documentation, "Vary," "Tiered Cache," and "Instant Purge" (developers.cloudflare.com/cache; blog post "Instant Purge: invalidating cached content in under 150ms").
- jsDelivr and unpkg's own served response headers and public GitHub issue discussions on pinned-version vs. `@latest` caching behavior.

**Incidents and case studies**
- Fastly, "Summary of June 8 outage" (fastly.com/blog/summary-of-june-8-outage, 2021) — primary account of the outage, timeline, and impact figures used in concept 5.
- Netflix, "How Netflix Works With ISPs Around the Globe to Deliver a Great Viewing Experience" (about.netflix.com/en/news) and Netflix Open Connect technical overview (openconnect.netflix.com).
- Omer Gil, "Web Cache Deception Attack" (omergil.blogspot.com, February 2017) and Black Hat USA 2017 whitepaper/presentation.
- Mirheidari, Arshad, Onarlioglu, Crispo, Kirda, Robertson, "Cached and Confused: Web Cache Deception in the Wild," 29th USENIX Security Symposium (2020).
- GitHub Docs, "Best practices for using the REST API" (docs.github.com/rest/guides/best-practices-for-using-the-rest-api) — conditional-request rate-limit exemption.
- Nishtala et al., "Scaling Memcache at Facebook," 10th USENIX Symposium on Networked Systems Design and Implementation (NSDI 2013) — leases and the thundering-herd fix.
- Vattani, Chierichetti, Lowenstein, "Optimal Probabilistic Cache Stampede Prevention," Proceedings of the VLDB Endowment, Vol. 8, No. 8 (2015) — the XFetch algorithm.

**Fast-drifting facts — verify before relying on any of these**
- Any specific CDN vendor's purge/invalidation propagation-time figures (this note cited Cloudflare's ~30-second general clear and sub-150ms Instant Purge as of its own drafting date — check current vendor docs).
- CDN vendor feature names for origin shielding/tiered caching (Origin Shield / Shielding / Tiered Cache) and their exact configuration surface.
- Netflix's specific quoted traffic percentage and Open Connect deployment footprint (location/country counts).
- Default ETag composition (inode-inclusion or not) for any specific web server version.
- Exact npm-CDN (`jsDelivr`/`unpkg`) `max-age` values for pinned vs. `@latest` URLs.

---

## Layered explanations

**10 seconds.** Caching an HTTP response means storing a copy closer to (or cheaper than) the origin instead of regenerating or re-fetching it every time. `Cache-Control` says how long to trust it; an ETag lets you cheaply confirm "nothing changed" instead of re-downloading; a CDN does both at hundreds of locations near users; and when a hot cached entry expires under heavy concurrent load, a single-flight lock stops everyone from recomputing it at once.

**1 minute.** Every HTTP response costs something to produce and something to transmit, and most of that cost is wasted redoing an answer that hasn't changed. `Cache-Control` — `max-age`, `s-maxage`, `private`/`public`, `no-cache`/`no-store` — is the origin's explicit, in-advance policy for how long and by whom a response may be trusted, following exactly the TTL-as-a-promise logic Day 12 taught for DNS. Where HTTP goes further than DNS is validators: an ETag, usually a content hash, lets a client ask "did this change?" and get back an empty `304 Not Modified` when it didn't, saving bandwidth (and, for an agent's fetch tool, context tokens) on the common case. A CDN runs these same rules at hundreds of physical locations near users instead of one cache near the origin, using anycast — the identical mechanism behind DNS's root servers — to route users to the nearest one, and needs an extra trick, origin shielding, because many independent edge locations can all miss simultaneously in a way a single cache never can. A CDN purge is real, unlike DNS's total lack of invalidation, but it isn't instant across a distributed network — so the more robust strategy is versioned, content-hashed URLs that make a given URL's content permanently fixed, removing the need to ever invalidate it, with the one exception being the small HTML shell that names which hashed files to load, which must stay short-lived. And whenever a hot cache key expires under concurrent load, every waiting caller can independently decide to recompute at once — a cache stampede — fixed by coordinating so exactly one caller recomputes while the rest wait for its answer, whether the expensive thing behind the cache is a database query or a call to a language model.

**5 minutes.** HTTP caching exists because Day 1's latency pyramid doesn't stop at your machine — a network round trip and a database query are both real, avoidable costs, and a cache at any layer (private browser cache, shared proxy, CDN edge, origin-side application cache) is a way to avoid paying them again for an unchanged answer. `Cache-Control` governs three separate questions in one header: freshness (`max-age`/`s-maxage`, which diverge because a private browser and a shared CDN may want different trust windows for the identical response), scope (`private`/`public`, which exists because a shared cache leaking one user's response to another is a real, repeatedly-documented vulnerability class — Web Cache Deception, both Omer Gil's original 2017 research and the 2020 USENIX "Cached and Confused" study confirming it's widespread rather than a one-off), and staleness policy (`no-cache` forces revalidation every time; `must-revalidate` forbids ever serving stale even during an origin outage; `stale-while-revalidate` and `stale-if-error`, from RFC 5861, deliberately serve a stale answer immediately while quietly refreshing in the background, which doubles as one of the two legitimate defenses against a stampede). Where a freshness lifetime alone leaves a gap — "is it actually different, or did the clock just run out" — validators close it: ETag, ideally a content hash, replaced Last-Modified's timestamp because a hash sidesteps both one-second granularity and clock-skew problems, and a `304 Not Modified` response saves the entire body on the common case where nothing changed, with the honest caveat that filesystem-derived ETags (inode+mtime) break the instant you scale horizontally, because two replicas assign different inodes to byte-identical content. `Vary` extends the cache key beyond the URL for content that genuinely differs by a request header, and it's safe exactly when that header has a small, bounded set of values (`Accept-Encoding`) and a well-documented mistake when it doesn't (`User-Agent`, which fragments a cache into thousands of near-single-use entries). A CDN takes all of this and runs it at hundreds of PoPs positioned near users via anycast, the same mechanism as DNS's root servers, and introduces origin shielding specifically because many independent PoPs missing at once is a fan-in problem no single cache faces. Unlike DNS, a CDN gives you a real purge mechanism — but real vendor documentation shows propagation across a distributed edge network takes measurable, non-zero, and vendor-specific time, which is why the more robust pattern for anything under your own build's control is versioned, content-hashed URLs: content that's structurally guaranteed never to change at a given URL can be cached `immutable` for a year with zero invalidation risk, provided you remember the one asymmetric exception — the short-lived HTML shell that names the current build's hashes. Finally, any cache — CDN, database cache, or an LLM-completion cache in an agent framework — faces a cache stampede the instant a hot key's expiry coincides with high concurrent read load, because a naive check-then-compute sequence has no way to know another caller is already recomputing; the fix is coordination, scaled to topology: an in-process lock for a single process, a distributed lock or Facebook-style lease (which cut Facebook's own peak database load from ~17K to ~1.3K queries/second) for multiple processes with a side-effecting recompute, or lock-free probabilistic early expiration (the XFetch algorithm) for very high fan-in with a cheap, idempotent recompute — and the Fastly outage of June 2021 is the standing reminder that once a caching layer sits in front of your system, its outage is your outage, regardless of how healthy everything behind it is.

**Expert summary.** HTTP caching is a consistency-freshness trade negotiated explicitly, per response, through three orthogonal axes — freshness lifetime, storage scope, and staleness-serving policy — layered on top of the same no-active-invalidation, TTL-as-promise model Day 12 established for DNS, but extended with two mechanisms DNS structurally lacks: validators, which convert an expensive full re-fetch into a cheap existence check against an opaque, ideally content-derived identifier, and a genuine (if non-instantaneous) purge primitive at the CDN layer. The validator mechanism's correctness is entirely a function of the identifier's derivation: content-hash-based tags are consistent under horizontal scaling and represent-equality by construction, while any validator derived from per-replica state (filesystem metadata, in-process counters) silently reintroduces exactly the cross-node divergence the mechanism exists to avoid — a specific instance of the general distributed-systems principle that any value used for equality comparison across independent nodes must be a pure function of shared, canonical state, not of node-local incidentals. CDNs generalize single-node caching along a geographic axis using anycast for placement — structurally identical to DNS's own root-server routing — and introduce origin shielding as the direct consequence of that generalization: fan-in from N independent, uncoordinated cache locations is a failure mode no single-cache topology can exhibit, and it's solved by imposing a hierarchy (a shield tier) that bounds the origin's exposure to O(shields) rather than O(edges), the same hierarchical-delegation logic that makes DNS's root scale. Invalidation strategy bifurcates cleanly along a single axis — whether the URL is under the caller's own naming authority — because a content-addressed URL is, by construction, immune to the staleness problem entirely (there is no "old version at this address" to correct, only a new address), while any URL whose name is fixed by an external contract (a CMS slug, a stable API path) requires active, imperfectly-propagating purge as its only lever, a genuine asymmetry that should drive architecture rather than being treated as an implementation detail. Finally, the cache-stampede family of failures is best understood as a race condition (Day 3) specific to the check-then-recompute sequence under concurrent miss conditions, and the choice among its three canonical mitigations — in-process locking, distributed leasing, and lock-free probabilistic early refresh — reduces to two independent variables: the coordination topology (single process vs. distributed fleet) and the cost profile of recomputation (side-effecting/expensive-per-call, which demands an absolute at-most-one guarantee, versus cheap/idempotent, which tolerates a probabilistic one) — a decision structure that transfers unmodified to any expensive, concurrently-read, TTL-or-invalidation-governed cache, including an LLM completion or embedding cache in an agentic system, where the "origin" being protected is a metered, latency-bearing model API call rather than a database.
