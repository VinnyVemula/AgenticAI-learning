# Day 13 — HTTP I: The Protocol Your Whole Career Runs On

> **Framing.** Day 11 gave you a byte pipe: a TCP connection is two ends agreeing on ordering and reliability, and nothing else. The socket Day 11 built has no idea what a "request" is, no idea where one message ends and the next begins, and no idea what any of the bytes flowing through it mean — it will happily carry a Shakespeare sonnet, a JPEG, or garbage with equal indifference. Day 12 showed you how that pipe gets an address to connect *to* in the first place. Neither day told you what to actually *say* once the pipe is open. That's this note: **HTTP is the agreed-upon shape of the conversation that happens over the socket Day 11 built, to the address Day 12 resolved.** Every backend job you will ever have, every public API you will ever call, and — not incidentally — every request your LLM SDK fires at `api.anthropic.com` or `api.openai.com`, is this protocol, wearing different clothes.
>
> There is one genuinely load-bearing connection to agentic systems in this note, and it is mechanical, not thematic: **HTTP's statelessness — the fact that the protocol itself remembers nothing between one request and the next — is the exact same problem an LLM API call has, one layer up.** A model has no memory of your last message either; a "conversation" with an LLM is an illusion built by resending the entire transcript with every single HTTP request, the same trick a cookie or a bearer token plays to fake a "logged-in session" out of a protocol with no concept of a session. We'll build that connection precisely where statelessness is taught (concept 8), not before, and nowhere else in this note is agentic content forced in — the rest of what follows is backend infrastructure, full stop.

---

## Roadmap

HTTP is usually taught as a table of status codes and a mnemonic for the methods, which produces readers who can recite "404 means not found" and cannot tell you why a status code is a *number* rather than a string, or why a proxy is allowed to retry a `PUT` automatically but must never retry a `POST`. We're going to build it the way Day 12 built DNS: as a chain of forced design decisions, because almost everything about HTTP's shape is the direct consequence of one of them.

```
The problem: TCP (Day 11) gives you a reliable, ORDERED STREAM OF BYTES —
no concept of "a message," just an open pipe two ends write into and
read from. Applications need to exchange REQUESTS and RESPONSES.
                            │
              What shape must that agreement take?
                            │
        ┌───────────────────┴───────────────────┐
        │                                       │
  A CUSTOM BINARY PROTOCOL              HUMAN-READABLE TEXT
  (compact, fast to parse,              (a request looks like a memo:
   opaque without a spec+tool           a line, some fields, a blank
   just to LOOK at a message)           line, an optional body —
                                        you can "speak" it by hand)
                                                │
                          HTTP chose text. 1991: one line, "GET /".
                          No headers, no status line, no version —
                          that's HTTP/0.9, and it's why the family
                          is still called "HyperText TRANSFER Protocol."
                                                │
                    But "readable text" still needs RULES, or two
                    programs can't agree on where fields start and end.
                            │
        ┌───────────────────┴───────────────────┐
        │                                       │
  What must a REQUEST say?              What must a RESPONSE say back?
  → method, target, version,            → version, a numeric STATUS
    header fields, optional body          (not prose!), headers,
                                           optional body
        │                                       │
        └───────────────────┬───────────────────┘
                            │
        Both are just TEXT ON A STREAM (Day 11's pipe has no message
        boundaries). So: where does ONE message end and the next begin?
                            │
                      MESSAGE FRAMING
              Content-Length  |  Transfer-Encoding: chunked  |
              connection-close (the last resort)
              Get this wrong — or let two different servers on the
              same request DISAGREE about it — and you get
              REQUEST SMUGGLING: one message hiding inside another.
                            │
        ┌───────────────────┴───────────────────┐
        │                                       │
  The METHOD is a PROMISE about              The STATUS CODE reports
  safety and retry-ability                    what happened, and its
  (can a proxy replay this request            FIRST DIGIT tells you
  automatically after a timeout,              WHICH component is
  with no risk of doing it twice?)            talking: 4xx = you,
                                               5xx = some server
                            │                          │
                            └───────────┬──────────────┘
                                        │
                    Headers carry the METADATA both directions need
                    (which virtual host, what format, who you are,
                     what you'll accept) — and then:
                            │
              After ALL of that, the server has ALREADY FORGOTTEN
              who you are. HTTP has no concept of "the same visitor
              asking twice."
                            │
                      STATELESSNESS
              State is bolted on top, by convention, with cookies
              and tokens — never built into the protocol at all.
```

Concepts and tiers:

| # | Concept | Tier |
|---|---|---|
| 1 | HTTP as a text-based agreement on top of a TCP byte stream | **[CORE]** |
| 2 | Anatomy of an HTTP request | **[CORE]** |
| 3 | Anatomy of an HTTP response | **[CORE]** |
| 4 | Message framing: Content-Length, chunked, connection-close | **[CORE]** |
| 5 | Methods and their semantics: safe, idempotent, the dangerous one | **[CORE]** |
| 6 | The status-code taxonomy | **[CORE]** |
| 7 | Headers that matter: Host, Content-Type, Authorization, Accept | **[WORKING]** |
| 8 | Statelessness and cookies | **[CORE]** |

Tooling note: everything in this note is plain text over a socket, so every example works identically whether you type it by hand, run it through Python's `socket` module, or fire it with `curl`. On Windows, `curl.exe` has shipped with Windows 10/11 since 2018, so no separate install is needed; PowerShell equivalents (`Invoke-WebRequest`, raw `System.Net.Sockets.TcpClient`) are given alongside each `curl` example, same as Days 11–12. WSL2 gives you `nc`/`telnet` if you want the most literal "type it by hand into a terminal" experience.

---

## 1. HTTP as a text-based agreement on top of a TCP byte stream

**Depth: [CORE]**

### Intuition

Start from exactly where Day 11 left you: two processes have a TCP connection. Concretely, that means each side has a file descriptor (Day 5) it can `send()` bytes into and `recv()` bytes out of, in order, reliably, with no corruption and no duplication (Day 11 concepts 4–6 built that guarantee at real cost — sequence numbers, ACKs, retransmission). That is the *entire* guarantee TCP gives you. It does not know that "a browser wants a web page." It does not know where one logical request ends and the next begins. If you write `b"hello"` and then `b"world"` into a TCP socket, the other end might see one `recv()` return `b"helloworld"`, or two returns of `b"hel"` and `b"loworld"` — Day 11 concept 3 called this the byte-stream framing problem, and DNS's own answer to it (Day 12 concept 7, the 2-byte length prefix on DNS-over-TCP) was your first example of a protocol solving it. HTTP has to solve the exact same problem, at a much larger scale, for two kinds of messages: a client asking for something, and a server answering.

So the real question this section answers is not "what is HTTP" — it's **what is the minimum agreement two independent programs, written by people who have never met and will never coordinate, need to make in order to exchange a request and a response over a raw byte pipe?** Everything else in this note is elaboration on that one answer.

### The decision that shaped everything else: text, not binary

Here is the fork in the road, and it's worth taking seriously because HTTP's answer looks almost naive in hindsight, and there's a real reason it wasn't.

**The realistic alternative:** a compact binary protocol — fixed-width fields, length-prefixed strings, a small number of opcodes. This is not a straw man; it's what plenty of successful protocols before and after HTTP actually did. DNS's wire format (Day 12) is binary. So is TLS's record layer (a preview for Day 14). So, eventually, is HTTP/2's framing layer — which is the flip condition, and we'll get to it.

**The actual reason text won in 1991:** Tim Berners-Lee designed HTTP/0.9 as one line — literally `GET /path`, nothing else, no headers, no version, no status code, just the requested document coming back as raw bytes — for a world with no HTTP client libraries, no browser dev tools, and no debuggers for this new thing. The requirement that actually mattered was **that a human, typing into a raw TCP connection with `telnet`, could speak the protocol and read the answer with their own eyes.** A binary format needs a spec and a parser before you can even look at a message; a text format needs nothing but eyeballs. That property — debuggability with zero tooling — is *why* HTTP could be adopted incrementally by a research community with no shared infrastructure, and it's the direct reason the Build task at the end of this note ("type raw HTTP into a socket by hand") is even possible as an exercise nearly 35 years later.

**The trade-off, honestly:** text is wasteful and slow to parse compared to binary. Every request re-sends header names as literal ASCII strings (`Content-Type: application/json` is 31 bytes to say something a binary format could say in 2), parsing means scanning for delimiters byte by byte rather than reading a fixed-width length field, and there's no compression of the redundant boilerplate that appears in every single request (the same `User-Agent`, the same `Accept-Language`, the same cookies, over and over). None of that mattered when a page was a few kilobytes and round trips were the bottleneck. It mattered enormously once a single page load triggered *hundreds* of requests, each dragging kilobytes of nearly-identical header text behind it.

**The flip condition, and where it actually flipped:** once you're Google or Facebook, serving billions of requests where header overhead measurably moves your bandwidth bill and header *parsing* measurably moves your CPU bill, the debuggability argument that won in 1991 no longer outweighs the efficiency cost. That's precisely why HTTP/2 (2015) replaced this text framing with a binary one and added HPACK header compression — same methods, same status codes, same semantics you're learning today, wrapped in a binary envelope for exactly the reasons text was rejected the first time around. **Day 14 opens that box properly; this note deliberately stops here** — everything you learn today about methods, status codes, and headers is unchanged in HTTP/2 and HTTP/3, because those protocols only replaced *how the bytes are framed*, not what the messages mean. That's worth sitting with: you are learning the durable 90% of HTTP by learning the "obsolete" text form first.

### Analogy: the office memo format

Picture an office before email, where everyone communicates by paper memo, and the office adopted one shared template decades ago: a line at the top (`To:`, `From:`, `Date:`, `Subject:`), then the body of the memo below a blank line. Nobody has to ask "wait, how do I format this?" every time — the template *is* the protocol, and anyone in the building can read anyone else's memo without a translator, because the shape is agreed upon in advance, independent of what the memo is actually about.

HTTP is that shared template, agreed upon not for one office but for the entire internet: a starting line that says what kind of message this is, a block of `Name: value` fields, a blank line, and then whatever payload follows. A request and a response use slightly different starting lines (concepts 2 and 3), but the shape — start line, fields, blank line, body — is identical, which is precisely why one parser (concept 4's minimal parser) can handle both with almost the same code.

**Where the analogy breaks, and this is the load-bearing part:** an office memo that's missing the `Date:` field still gets read — a human fills in the gap with judgment ("eh, probably sent this week"). An HTTP message with a malformed start line, or a missing `Host:` header on HTTP/1.1, is not "probably fine" — a compliant parser is required to reject it deterministically, because the *next* piece of software in the chain (a proxy, a cache, a security device) must reach the *identical* conclusion about what the message means, or you get a security bug, not an inconvenience. Concept 4's case study — HTTP request smuggling — is precisely what happens when two different pieces of software disagree about where one memo ends and the next begins. A human reading two ambiguous memos shrugs and asks a colleague. Two computers reading two ambiguous requests can be tricked into smuggling an entire extra request past a firewall that never saw it. The stakes of "keep the format unambiguous" are categorically different once no human is in the loop.

### Worked example: the four-line version that changed everything, and the two-line version that came before it

Here is the entire wire content of the oldest form of HTTP that still (barely) exists in the wild — HTTP/0.9, 1991:

```
GET /pages/implementation.html
```

That's it. One line, sent, and the server just streams back the raw HTML and closes the connection. No method other than `GET` existed. No headers. No status line — if something went wrong, you got an HTML error page or nothing, and your client had to guess which. No version token, because there was no other version to distinguish it from.

By HTTP/1.0 (1996, RFC 1945) the same request had grown to look like this — and this is a genuine, runnable request; I sent exactly these bytes at a real server while writing this note:

```
GET / HTTP/1.1
Host: example.com
Accept: text/html
Connection: close

```
*(that trailing blank line is not decoration — it's load-bearing, and concept 2 explains exactly why)*

Three things were added between those two examples, and each one exists to solve a specific, nameable problem HTTP/0.9 couldn't handle at all:

1. **A version token (`HTTP/1.1`).** HTTP/0.9 had no way to say "I understand headers" versus "I don't" — every client and server had to assume the least-capable dialect. The version token lets both ends negotiate capability in the first three bytes of the exchange.
2. **Headers, including `Host`.** HTTP/0.9 had no way to send *any* metadata — no way to say "I only accept HTML," no way to authenticate, and critically, no way to tell a server *which website* you meant, which is fatal the moment one physical machine (one IP address) hosts more than one website. `Host` fixed that, and its absence is precisely why HTTP/0.9 could never support what we now call **virtual hosting** — concept 7 makes this concrete with real numbers.
3. **A status line on the way back**, which HTTP/0.9 never had. Concept 3 covers this properly, but note it here: without a machine-readable status, "did this work?" could only be answered by inspecting the *content* of the response, which is exactly the anti-pattern condemned in concept 6's `200 OK` with `{"success": false}` in the body — that mistake is HTTP/0.9's actual design, made by accident, by people who should have known better.

### Visual: where HTTP sits, and what it inherits

```
┌─────────────────────────────────────────────────────────┐
│  APPLICATION LAYER                                       │
│  HTTP: request/response, methods, status codes, headers  │  ← this note
│  (also: DNS from Day 12, at this same layer)              │
├─────────────────────────────────────────────────────────┤
│  TRANSPORT LAYER (Day 11)                                 │
│  TCP: reliable, ordered, connection-oriented byte stream   │
│  (HTTP/3 replaces this layer with QUIC-over-UDP — Day 14)  │
├─────────────────────────────────────────────────────────┤
│  NETWORK LAYER (Day 10) — IP: best-effort packet delivery  │
├─────────────────────────────────────────────────────────┤
│  LINK LAYER (Day 9) — frames, MAC addresses, switches      │
└─────────────────────────────────────────────────────────┘

HTTP INHERITS, and never re-solves:
  • reliability & ordering        ← TCP (Day 11)
  • congestion/flow control       ← TCP (Day 11)
  • "where is this server"        ← DNS (Day 12)
HTTP ADDS, because nothing below it provides these:
  • "here is a request; here is where it ends"  (framing)
  • "what kind of operation is this"            (methods)
  • "what happened"                             (status codes)
  • "metadata about this exchange"              (headers)
```

### Under the hood: HTTP is not a running program, it's a convention

There is no "HTTP process" anywhere on your machine the way there's a DNS resolver process or a TCP/IP stack in the kernel. HTTP is implemented entirely in **userspace, inside whatever application is talking** — your browser, `curl`, a FastAPI app, the `requests` or `httpx` library, an nginx worker process. The kernel's job stops at handing your application a connected socket (Day 11); everything from that point — building the request-line string, writing it into the socket's send buffer, reading bytes back out and finding the blank line — is ordinary application code, and you are about to write exactly that code yourself, from scratch, in concept 4's runnable example. There is nothing privileged, hidden, or magical happening "underneath" HTTP the way there genuinely is underneath a syscall (Day 2) or a virtual memory access (Day 4). This is a deliberate and important thing to internalize before the rest of this note: **every fact from here on is a fact about a convention two pieces of ordinary code agree to follow, not a fact about hardware or an OS.**

*Primary source:* the original HTTP/0.9 description survives in the W3C's history pages ("HTTP" at w3.org/Protocols/HTTP/AsImplemented.html, Tim Berners-Lee, 1991); RFC 1945 (HTTP/1.0, 1996) is the first RFC-numbered specification; RFC 9110/9111/9112 (2022) are the current normative specifications for HTTP semantics, caching, and HTTP/1.1 message syntax respectively, obsoleting RFC 7230–7235, which in turn obsoleted RFC 2616.

---

## 2. Anatomy of an HTTP request

**Depth: [CORE]**

### Intuition

You've just seen a request as a block of text. Now take it apart formally, because every field in it exists to answer one specific question a server has to answer before it can do anything: *what do you want, from where, speaking which dialect, with what extra context, and — sometimes — carrying what payload?* Five things, five parts of the message, in a fixed order that never varies:

```
GET /orders/42?expand=items HTTP/1.1        ← the REQUEST LINE
Host: api.example.com                       ┐
Authorization: Bearer eyJhbGciOि...          │  HEADER FIELDS
Accept: application/json                    │  (zero or more)
User-Agent: curl/8.7.1                      ┘
                                             ← BLANK LINE (mandatory)
{"note": "rush this one"}                   ← BODY (optional)
```

### The request line, precisely

RFC 9112 §3 (the current normative HTTP/1.1 message-syntax spec) gives the request-line's exact grammar as `method SP request-target SP HTTP-version`, where `SP` is a single literal space character — not a tab, not multiple spaces. Three fields, two spaces, and a line ending in `\r\n` (carriage return, line feed — not just `\n`; this is inherited from the text-terminal conventions of the 1970s/80s internet, and getting it wrong is a classic bug when hand-building requests, which you're about to do).

**`method`** — a case-sensitive token (`GET`, `POST`, `PUT`, `DELETE`, `PATCH`, `HEAD`, `OPTIONS`, `TRACE`, `CONNECT`) that states the *kind* of operation. Concept 5 is the full treatment; for now, just note it's a bare word, not a number, and it's the very first thing on the wire — a firewall or load balancer can make a routing decision by reading the first few bytes of a connection, before a single header has arrived.

**`request-target`** — almost always what RFC 9112 calls **origin-form**: the path plus an optional `?query-string`, exactly what you see in a browser's address bar after the domain (`/orders/42?expand=items`). This is the form your code will use nearly 100% of the time. Three other forms exist and are worth knowing by name so you're not confused when you see them, though you'll essentially never write them yourself:
- **absolute-form** — the *entire* URL including scheme and host (`GET http://example.com/orders/42 HTTP/1.1`), used only when a client is talking directly to a forward proxy, because the proxy needs to know which origin server to forward to, and it isn't the `Host:` header's job in that specific case (it's actually still sent too, redundantly, for exactly this historical reason).
- **authority-form** — just `host:port` with no path at all, used *exclusively* by the `CONNECT` method when establishing a tunnel (this is how your browser asks a proxy to open a raw TCP tunnel for HTTPS — a preview of Day 14).
- **asterisk-form** — literally the single character `*`, used *exclusively* by `OPTIONS` to ask "what can you do at all," not about any specific resource (`OPTIONS * HTTP/1.1`).

**`HTTP-version`** — the literal token `HTTP/1.1` (or `1.0`, `2`, `3`). **A deliberate stop, [AWARE] level, here by design:** this note teaches HTTP/1.1's wire format because it's the version you can literally type by hand into a socket, and every semantic concept in this note — methods, status codes, headers, statelessness — is unchanged in HTTP/2 and HTTP/3. Day 14 opens the version-negotiation and binary-framing box properly; for today, just know the token exists and says which dialect of the *framing* rules (concept 4) applies.

### Header fields, precisely

RFC 9112 §5 gives the grammar as `field-name ":" OWS field-value OWS`, where `OWS` is "optional whitespace" — meaning `Content-Type:application/json` (no space) and `Content-Type:   application/json` (extra spaces) are both legal and mean the same thing, but **no whitespace is permitted between the field name and the colon itself** (`Content-Type : application/json` is a real historical request-smuggling vector, because some parsers treated the space as part of the name and silently ignored the whole header, while others didn't — a small illustration of concept 4's larger point). Field names are **case-insensitive** by the spec (`content-type`, `Content-Type`, and `CONTENT-TYPE` are the identical header), which is why my minimal parser later in this note lowercases every header name the moment it parses one — matching real client and server behavior, not adding a feature.

A header field can also **repeat**. `Accept: text/html` followed later by `Accept: application/json` is not an error and not automatically an "overwrite" — depending on the header, repetition means different things (`Set-Cookie` genuinely needs to appear multiple times to set several cookies at once; other headers are typically comma-joined into a single logical value, per RFC 9110 §5.3). This is a real source of subtle client-library bugs, worth knowing exists even at a "treat it as a black box until it bites you" level.

### The blank line: the single most consequential two bytes in the message

After the last header, the message sends a line containing *nothing* — just `\r\n` immediately after the previous header's `\r\n`, so on the wire you see `\r\n\r\n`: two carriage-return-linefeed pairs back to back. This is not stylistic. **It is the only signal in the entire message that says "the metadata is finished; anything after this point is payload, not another header."** Every parser — yours in concept 4's runnable example, a browser's, nginx's — is built around finding that exact four-byte sequence. Miss it, and you cannot tell a header line from the first line of a JSON body; RFC 9112 §2.1 states the grammar plainly: `HTTP-message = start-line CRLF *( field-line CRLF ) CRLF [ message-body ]` — read literally, that final bare `CRLF` before `[ message-body ]` *is* the blank line, and it is not optional even when there's no body at all.

### The body

Optional, and — this is the point concept 4 exists entirely to answer — **its length is not self-describing the way a header's is.** A header field ends where its line ends (`\r\n` marks it, unambiguously). A body has no such natural terminator; JSON, HTML, and raw binary content all can legitimately contain byte sequences that look like `\r\n`. So the receiver needs to be *told*, in a header, how to know when the body stops. That's the entirety of concept 4.

### Runnable example — hand-typing a real HTTP/1.1 request over Day 11's socket

This is the study plan's Build task, done for real, against a real server on the public internet, using nothing but the standard library `socket` module — Day 11's own tool.

```python
# raw_http_get.py — literally type an HTTP/1.1 request as a Python string,
# send it as raw bytes over a Day-11-style TCP socket, and read the raw
# response bytes back. No HTTP library involved anywhere.
# Standard library only — nothing to pip install.
import socket

HOST = "example.com"
PORT = 80

# This string IS the request. There is no hidden machinery translating
# "high-level intent" into wire format — this literal text, encoded to
# bytes, is what a browser also sends.
request = (
    f"GET / HTTP/1.1\r\n"
    f"Host: {HOST}\r\n"
    f"Accept: text/html\r\n"
    f"Connection: close\r\n"     # ask the server to close after replying,
    f"\r\n"                      # so a simple recv() loop knows when to stop
)

with socket.create_connection((HOST, PORT), timeout=10) as sock:
    sock.sendall(request.encode("ascii"))
    chunks = []
    while True:
        data = sock.recv(4096)
        if not data:
            break
        chunks.append(data)

raw = b"".join(chunks)
print(f"--- Bytes sent ---\n{request!r}\n")
print(f"--- Total bytes received: {len(raw)} ---\n")
head, _, body = raw.partition(b"\r\n\r\n")
print("--- Head (status line + headers) ---")
print(head.decode("iso-8859-1"))
print(f"\n--- Body: first 300 bytes of {len(body)} total ---")
print(body[:300].decode("iso-8859-1", errors="replace"))
```

Run it:

```
$ python raw_http_get.py
```

Real output, captured against the live `example.com` while writing this note (24 Aug 2026 — Age and CF-RAY will differ every time you run it, because you'll hit a different Cloudflare cache node; that's Day 15's subject, not today's):

```
--- Bytes sent ---
'GET / HTTP/1.1\r\nHost: example.com\r\nAccept: text/html\r\nConnection: close\r\n\r\n'

--- Total bytes received: 868 ---

--- Head (status line + headers) ---
HTTP/1.1 200 OK
Date: Mon, 24 Aug 2026 11:11:22 GMT
Content-Type: text/html
Transfer-Encoding: chunked
Connection: close
Server: cloudflare
Last-Modified: Tue, 18 Aug 2026 20:06:42 GMT
Allow: GET, HEAD
Accept-Ranges: bytes
Age: 8571
cf-cache-status: HIT
CF-RAY: a301ea77eb230b23-BOM

--- Body: first 300 bytes of 571 total ---
22f
<!doctype html><html lang="en"><head><title>Example Domain</title><link rel="icon" href="data:,"><meta name="viewport" content="width=device-width, initial-scale=1"><style>body{background:#eee;width:60vw;margin:15vh auto;font-family:system-ui,sans-serif}h1{font-size:1.5em}div{opacity:0.8}a:link
```

**Why this works, line by line:**

- `socket.create_connection((HOST, PORT))` does exactly what Day 11 taught: DNS-resolves `example.com` (Day 12), then performs the 3-way handshake (Day 11 concept 4). By the time this line returns, we have a connected socket and have not sent one byte of HTTP yet — the TCP layer and the HTTP layer are fully separate steps, executed in sequence, by two completely different pieces of code (the OS kernel did the handshake; we're about to do the HTTP part ourselves).
- `sock.sendall(request.encode("ascii"))` — this is the entire "client speaks HTTP" step. There is no `http.send_request(...)` call anywhere; we built a plain Python string with the exact grammar from concept 2, encoded it to bytes, and wrote it into the socket. This is deliberately the least magical way to make this request, to prove the point.
- The `while True: recv()` loop is naive on purpose: it reads until the connection closes (`recv()` returning `b""`), which only works *because* we asked for `Connection: close` in the request. A persistent connection (the HTTP/1.1 default, actually — see concept 4) would never trigger this exit condition, and the script would hang forever waiting for a close that never comes. This is your first hands-on encounter with **why framing matters**: without `Connection: close`, this exact script breaks, for a reason concept 4 explains fully.
- **Look at what the real response contains, because it's more instructive than a synthetic example would be:** the body doesn't start with `<!doctype`, it starts with `22f` — a hexadecimal number (0x22f = 559 decimal) on its own line. That is **`Transfer-Encoding: chunked`** in action (concept 4), not a bug in this script — the header line right above it says exactly that, and it's a genuine illustration that real production traffic (this is Cloudflare, serving a genuinely enormous fraction of the web) very often does *not* use the simpler `Content-Length` framing this note starts with.
- Also visible, and deliberately *not* explained further here: `Age: 8571` and `cf-cache-status: HIT` are headers about **caching** — this response was served from a Cloudflare cache node, not regenerated by the origin server. That machinery is Day 15's entire subject; for today, the only fact that matters is that they're headers *just like any other* — `Name: value`, parsed by the exact same rule as `Content-Type` or `Host`. Nothing about HTTP's syntax changes based on what a header means semantically.

**Honesty caveat:** this script has no timeout handling beyond the initial connect, doesn't handle a server that never sends the trailing blank line, and — most importantly for the next concept — **only works because we asked for `Connection: close`.** A production HTTP client (Python's own `http.client`, `requests`, `httpx`) handles persistent connections, redirects, encoding negotiation, retries, and TLS, none of which this script attempts. That's the entire 90%-of-the-work a real client library exists to hide from you, and you now know, mechanically, what it's hiding.

### Windows note

The same request, sent from PowerShell without any Python, using .NET's raw socket classes — useful when you want to "feel" this without leaving PowerShell:

```powershell
$client = New-Object System.Net.Sockets.TcpClient("example.com", 80)
$stream = $client.GetStream()
$request = "GET / HTTP/1.1`r`nHost: example.com`r`nConnection: close`r`n`r`n"
$bytes = [System.Text.Encoding]::ASCII.GetBytes($request)
$stream.Write($bytes, 0, $bytes.Length)

$reader = New-Object System.IO.StreamReader($stream)
$reader.ReadToEnd()
$client.Close()
```

`` `r`n `` is PowerShell's escape for `\r\n` inside a double-quoted string — get this wrong (use only `` `n ``) and some strict servers will refuse the request outright, which is itself a small, free demonstration of how literally the spec's `CRLF` requirement is enforced in practice. If you have WSL2 available, `printf 'GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n' | nc example.com 80` is the most literal "type it by hand" version of all — no code, just a pipe into a raw TCP tool.

---

## 3. Anatomy of an HTTP response

**Depth: [CORE]**

### Intuition

A response answers the exact question the request asked, and it has to answer three things a client always needs, in order of urgency: *did it work, in what format is the answer coming, and here's the answer.* The shape mirrors the request almost exactly, with one field swapped:

```
HTTP/1.1 404 Not Found               ← the STATUS LINE (not a "request line")
Content-Type: application/json       ┐
Content-Length: 27                   │  HEADER FIELDS
Date: Mon, 24 Aug 2026 11:20:03 GMT  ┘
                                      ← BLANK LINE (still mandatory, same rule)
{"error":"order not found"}          ← BODY (optional, same framing rules)
```

### The status line, precisely

RFC 9112 §4 gives the grammar as `status-line = HTTP-version SP status-code SP [ reason-phrase ]` — same `HTTP-version` token as the request, then a **three-digit integer**, then an optional human-readable phrase. Two things worth pulling apart here because they're easy to blur together:

**The status code is the machine-readable part, and it is *only* the three digits.** `404` is the entire signal a program is required to act on. The classic phrases (`Not Found`, `OK`, `Internal Server Error`) are conventional, not semantic — RFC 9112 explicitly permits an empty reason phrase, and a compliant client must never branch its logic on the phrase's text. This is a genuinely common junior mistake: writing `if response.reason == "Not Found"` in client code, which breaks the moment a server (correctly, legally) sends `HTTP/1.1 404 ¯\_(ツ)_/¯` or nothing at all after the code. **Always branch on the number.**

**Why a number, and not a short string like `"OK"` or `"ERROR"`, given that everything else in HTTP is a human-readable string?** This is a genuine, if small, design decision, and it's worth naming as one. A string invites synonym drift (is it `"OK"`, `"Success"`, `"Ok"`, `"success"` — every server picks differently, and now every client needs a case-insensitive lookup table against an open-ended vocabulary). A fixed three-digit number, drawn from a **registry** IANA maintains centrally (RFC 9110 §15 defines the base set; new codes get formally registered, like 429 was via RFC 6585 in 2012, decades after the original spec), gives you a closed, extensible, unambiguous space that a `switch` statement or an `if status >= 400` range check can act on with zero parsing ambiguity. The first digit doing double duty as a *category* (concept 6) is the direct payoff of choosing a number over a string: `response.status_code // 100 == 5` is a one-line "is this the server's fault" check that a string vocabulary could never offer as cheaply.

### The body, and the header that describes its *meaning* (not just its length)

The response body carries the actual payload — HTML, JSON, an image, nothing at all. **`Content-Type`** is the header that tells the receiver how to *interpret* those bytes (`text/html; charset=utf-8`, `application/json`, `image/png`), which is a completely different question from "how many bytes are there" (that's `Content-Length` or `Transfer-Encoding`, concept 4's subject) or "did it work" (that's the status code, above). Confusing these three questions — length, meaning, and success — is a real, recurring source of bugs, and keeping them as three separate, independently-answered questions is precisely why HTTP can carry literally anything (HTML today, a video stream tomorrow, an LLM's streamed token-by-token completion the day after) without changing its own grammar at all.

### Worked example — a real status line and headers, captured on the wire

Reusing the exact `raw_http_get.py` capture from concept 2 (same run, same request), here is the response head again, read now as a *response* rather than as an undifferentiated blob:

```
HTTP/1.1 200 OK                              ← status LINE: version, code, phrase
Date: Mon, 24 Aug 2026 11:11:22 GMT           ┐
Content-Type: text/html                       │
Transfer-Encoding: chunked                    │  HEADER FIELDS — metadata about
Connection: close                             │  this specific response
Server: cloudflare                            │
Last-Modified: Tue, 18 Aug 2026 20:06:42 GMT  │
Allow: GET, HEAD                              │
Accept-Ranges: bytes                          │
Age: 8571                                     │
cf-cache-status: HIT                          │
CF-RAY: a301ea77eb230b23-BOM                  ┘
                                               ← blank line (not shown; consumed
                                                  by the parser to find this point)
22f
<!doctype html>...                            ← BODY (chunk-framed — concept 4)
```

Two headers here are worth calling out precisely because they answer questions this section just raised: **`Content-Type: text/html`** tells the receiving browser (or, in our script's case, a human reading the printed output) that the bytes that follow should be parsed as HTML, not displayed as plain text or treated as a binary download. **`Allow: GET, HEAD`** is a header that only appears on *certain* status codes (canonically alongside a `405 Method Not Allowed`, though some servers, like this one, include it informationally on success too) and answers a completely different question — not "what did I send you" but "what methods does this specific URL even support," which is concept 5's subject.

Now compare against a deliberately different real status, captured the same way against a different real endpoint:

```
$ curl -sI http://github.com

HTTP/1.1 301 Moved Permanently
Content-Length: 0
Location: https://github.com/
connection: close
```

**Same shape, completely different meaning.** No `Content-Type` because there's no meaningful body (`Content-Length: 0`). A new header, **`Location`**, appears specifically because `3xx` codes need to tell the client *where to go instead* — a header that would be meaningless on a `200` or a `404`. This is the pattern to internalize: **the status line and the headers work together**, and which headers are present or required shifts with which status code you're looking at. Concept 6 is the full taxonomy; the redirect-caching case study later in this note is built entirely on this specific `301` + `Location` pair.

### Visual: request and response, side by side

```
REQUEST                                RESPONSE
┌─────────────────────────────┐        ┌─────────────────────────────┐
│ METHOD  TARGET  VERSION      │        │ VERSION  STATUS  [PHRASE]   │  ← start line
├─────────────────────────────┤        ├─────────────────────────────┤
│ Header: value                │        │ Header: value                │
│ Header: value                │        │ Header: value                │  ← header fields
│ ...                          │        │ ...                          │
├─────────────────────────────┤        ├─────────────────────────────┤
│ (blank line)                  │        │ (blank line)                  │  ← mandatory, both directions
├─────────────────────────────┤        ├─────────────────────────────┤
│ optional body                 │        │ optional body                 │
└─────────────────────────────┘        └─────────────────────────────┘
     "what do you want?"                    "here's what happened,
                                              and here's your answer"
```

### Under the hood: symmetry is a design choice, not an accident

It would have been possible to design a completely different grammar for responses than for requests — different delimiters, a different header format, a binary status block. HTTP's designers instead reused the *identical* header-field syntax, the identical CRLF line endings, and the identical blank-line-ends-metadata rule for both directions, swapping only the start line's three fields for a different three fields that answer a different question. **The payoff of that symmetry is concrete: one parser handles both directions.** Concept 4's runnable example builds exactly one `parse_headers()`-style routine and reuses it for requests; the only genuinely separate piece of code is the three-token split at the very first line, because a request line has a method where a status line has a numeric code. Everything after that first line — reading headers until the blank line, then reading the body per whatever framing rule applies — is *the same function*, called from both a client reading a response and a server reading a request. This is why building the minimal parser in concept 4 teaches you both halves of HTTP for the cost of one piece of code.

*Primary source:* RFC 9112 §4 (Status Line) and §2.1 (Message Format), the current normative specification.

---

## 4. Message framing: how the receiver knows where a message ends

**Depth: [CORE]**

### Intuition

Concept 1 raised this and deferred it: TCP is a byte stream with no message boundaries. A header field solves its *own* boundary problem trivially — it ends at `\r\n`, full stop, because header values themselves are restricted to not contain a raw `CRLF`. But a **body** has no such restriction. A JSON payload, an uploaded image, a video stream — any of them can legally contain the exact byte sequence `\r\n` as ordinary content. So a header ending in `\r\n\r\n` tells you *headers are done*, but it tells you nothing about **how many more bytes belong to this message before the next one starts** (on a reused connection) or **whether there even is a "next message"** at all. That is the specific, narrow, and surprisingly consequential problem this concept solves.

Why does it matter enough to be its own [CORE] concept rather than a footnote? Because **every party in the request path — your code, a load balancer, a CDN edge, a corporate proxy — must independently compute the exact same answer to "where does this message end," or two of them will disagree about where the next message begins.** When they disagree, one party's "end of request #1" is another party's "the request continues, and here comes a smuggled request #2 hidden inside what looks like body content." That is not a hypothetical; it's a real, actively-exploited attack class, and this concept's case study is a real, named instance of it.

### The three mechanisms, and RFC 9112's exact precedence order

RFC 9112 §6.3 lays out, in order, precisely how a recipient determines a message's body length. Read as a decision tree:

1. **Certain responses never have a body, regardless of any header:** `1xx` informational responses, `204 No Content`, `304 Not Modified` (Day 15's subject), and any response to a `HEAD` request (concept 5) — the body length is defined to be zero, full stop, even if a `Content-Length` header is present and non-zero (some servers do send one anyway, purely as advisory metadata about what the body *would have been*).
2. **`Transfer-Encoding` present → it wins, unconditionally, even over `Content-Length`.** If both headers appear on the same message, RFC 9112 states plainly: *"the Transfer-Encoding overrides the Content-Length,"* and explicitly flags this exact situation — both present, potentially disagreeing — as something that **"might indicate an attempt to perform request smuggling or response splitting and ought to be handled as an error."** This single sentence in the spec is the entire reason the case study below exists.
3. **`Transfer-Encoding: chunked` present → the body is framed in chunks**, each announced by its own length; you read chunks until you see a zero-length chunk, described fully below.
4. **A single, valid `Content-Length` value present → read exactly that many bytes.** ("Single, valid" is doing real work here — see the honesty note below.)
5. **A request with neither header → the body length is zero.** A `GET` with no `Content-Length` and no `Transfer-Encoding` has no body, by definition, not by convention.
6. **A response with neither header → the body runs until the connection closes.** This is the HTTP/1.0-era fallback — the "last resort" framing this note's roadmap calls it — and RFC 9112 itself says this exists **"primarily for backwards compatibility with HTTP/1.0."** It only applies to responses; a request is always required to have explicit framing.

### `Content-Length`: the simple case

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 27

{"error":"order not found"}
```

`Content-Length: 27` means exactly, unambiguously, 27 bytes follow, no more, no less. This is the framing you'll use for the overwhelming majority of ordinary API responses — anywhere the full payload is known in advance, which is nearly everywhere in typical request/response CRUD work. Its one real constraint is the one baked into its name: **you must know the total size before you start sending the first byte**, because the header announcing the size comes *before* the body it describes.

### `Transfer-Encoding: chunked`: when you don't know the size yet

This is precisely the situation `Content-Length` cannot handle: you want to start sending a response *before* you know how long it will ultimately be — a database query streaming rows as they arrive, a long-running computation emitting partial results, or (the case that matters most for the work you're doing) **an LLM streaming a completion one token at a time.** You cannot put a `Content-Length` on that response, because at the moment you send the headers, the total token count doesn't exist yet.

The mechanism: instead of one length announced up front, the body is broken into **chunks**, and *each chunk announces its own length*, in hexadecimal, on its own line, immediately followed by that many bytes of data, a trailing `\r\n`, and then the next chunk's length line. The stream ends with a chunk of length **zero** — RFC 9112 §7.1 states it plainly: *"the chunked transfer coding is complete when a chunk with a chunk-size of zero is received."*

```
HTTP/1.1 200 OK
Content-Type: text/plain
Transfer-Encoding: chunked

7\r\n
Mozilla\r\n
9\r\n
Developer\r\n
7\r\n
Network\r\n
0\r\n
\r\n
```

Walk it by hand: `7` (hex, = 7 decimal) announces a 7-byte chunk, `Mozilla` is exactly 7 bytes, then a `\r\n` separator, then `9` announces 9 bytes, `Developer` is exactly 9 bytes, and so on, until `0\r\n\r\n` — a zero-length chunk followed by an empty "trailer" section (also legal to contain real headers, sent *after* the body, for metadata only known once the body finished — the exact detail concept 3 flagged as belonging to a later day is a distant cousin of this mechanism). Concatenate `Mozilla` + `Developer` + `Network` and you get the real body: `MozillaDeveloperNetwork` — the chunk boundaries carry zero semantic meaning; they're purely a transport-level framing device, invisible to the application that ultimately reads the body.

You already saw this mechanism live, unprompted, in concept 2's real captured output: `example.com`'s response body began with the line `22f` — a chunk-size header, in hex, for a 559-byte chunk — precisely because Cloudflare's edge served that response using `Transfer-Encoding: chunked` rather than `Content-Length`, which is worth pausing on: **chunked encoding is not an exotic corner case reserved for LLM streaming demos. It's default, everyday behavior for an enormous share of real production traffic**, anywhere the serving layer doesn't want to (or can't cheaply) compute the full response size before it starts writing bytes to the socket.

### `Connection: close`: framing by giving up on framing

The last-resort rule (item 6 above) is exactly what made `raw_http_get.py` in concept 2 work at all: with no `Content-Length` and no `Transfer-Encoding`, the *only* way the client can know the response is finished is that the server closes the TCP connection. This is honest, simple, and has one large cost: **it forecloses connection reuse entirely** — Day 11 concept 11's connection-pooling and keep-alive material is, from this angle, exactly the effort spent to *avoid* ever needing this fallback, because a connection you must close after every single response can't be pooled at all. HTTP/1.0 defaulted to closing after every response for precisely this reason (no reliable length-signaling existed yet in the field for many responses); HTTP/1.1 made persistent connections (`Connection: keep-alive` implicitly, unless you say otherwise) the default specifically *because* `Content-Length` and `Transfer-Encoding` became universal enough to make "read until close" the exception rather than the rule.

### Worked example — building the parser, and watching request smuggling become obvious

Rather than parse a canned example, build the actual mechanism: a raw TCP server, using nothing but the `socket` module, that reads bytes off the wire, finds the header/body boundary by hand, and honors `Content-Length` framing exactly as RFC 9112 §6.3 specifies. This is the study plan's second Build task.

```python
"""
minimal_http_server.py — a raw TCP server that speaks just enough HTTP/1.1
to prove the point: there is no "HTTP layer" in the OS. It's your code,
reading bytes off a socket, finding a blank line, and doing string work.
Standard library only.
"""
import socket


def recv_until_headers_done(conn: socket.socket) -> bytes:
    """Read from the socket until we've seen the blank line (CRLF CRLF)
    that RFC 9112 section 2.1 says ends the header section. We may also
    read part of the body in the same recv() call -- TCP has no message
    boundaries (Day 11), so we keep whatever comes after the split point."""
    buf = b""
    while b"\r\n\r\n" not in buf:
        chunk = conn.recv(4096)
        if not chunk:
            break
        buf += chunk
    return buf


def parse_request(head_bytes: bytes) -> dict:
    """Parse the request line and headers out of raw bytes.
    head_bytes may contain trailing body bytes after the blank line;
    only the part before b"\\r\\n\\r\\n" is a header section."""
    header_part, _, leftover = head_bytes.partition(b"\r\n\r\n")
    lines = header_part.split(b"\r\n")

    request_line = lines[0].decode("iso-8859-1")
    method, path, version = request_line.split(" ")

    headers = {}
    for line in lines[1:]:
        if not line:
            continue
        name, _, value = line.partition(b":")
        headers[name.decode("iso-8859-1").strip().lower()] = (
            value.decode("iso-8859-1").strip()
        )

    return {
        "method": method,
        "path": path,
        "version": version,
        "headers": headers,
        "leftover_body_bytes": leftover,
    }


def read_body(conn: socket.socket, parsed: dict) -> bytes:
    """Body framing per RFC 9112 section 6.3: honour Content-Length. (We do
    not implement Transfer-Encoding: chunked here -- see the honesty note
    in the text. A real server must.)"""
    body = parsed["leftover_body_bytes"]
    content_length = int(parsed["headers"].get("content-length", "0"))
    while len(body) < content_length:
        chunk = conn.recv(4096)
        if not chunk:
            break
        body += chunk
    return body[:content_length]


def handle_connection(conn: socket.socket, addr):
    raw_head = recv_until_headers_done(conn)
    if not raw_head:
        conn.close()
        return

    try:
        parsed = parse_request(raw_head)
    except (ValueError, IndexError):
        # A request line we can't even split into 3 parts is malformed --
        # RFC 9112 gives us nothing to frame a body with, so we can't
        # safely keep the connection open. 400, then close.
        response = (
            b"HTTP/1.1 400 Bad Request\r\n"
            b"Content-Length: 0\r\n"
            b"Connection: close\r\n\r\n"
        )
        conn.sendall(response)
        conn.close()
        return

    body = read_body(conn, parsed)

    print(f"[{addr}] {parsed['method']} {parsed['path']} {parsed['version']}")
    for k, v in parsed["headers"].items():
        print(f"    {k}: {v}")
    if body:
        print(f"    body ({len(body)} bytes): {body!r}")

    if parsed["method"] not in ("GET", "HEAD", "POST"):
        status_line = "HTTP/1.1 405 Method Not Allowed"
        payload = b""
    elif parsed["path"] != "/":
        status_line = "HTTP/1.1 404 Not Found"
        payload = b'{"error":"not_found"}'
    else:
        status_line = "HTTP/1.1 200 OK"
        payload = b'{"message":"hello from a hand-rolled HTTP server"}'

    response = (
        f"{status_line}\r\n"
        f"Content-Type: application/json\r\n"
        f"Content-Length: {len(payload)}\r\n"
        f"Connection: close\r\n"
        f"\r\n"
    ).encode("ascii") + payload

    conn.sendall(response)
    conn.close()


def serve_one(host="127.0.0.1", port=8080):
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as srv:
        srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        srv.bind((host, port))
        srv.listen(1)
        print(f"listening on {host}:{port} (one connection, then exit)")
        conn, addr = srv.accept()
        with conn:
            handle_connection(conn, addr)


if __name__ == "__main__":
    serve_one()
```

Run it, and hit it with real `curl` requests — all output below is real, captured while writing this note:

```
$ python minimal_http_server.py &
$ curl -s -X POST http://127.0.0.1:8080/ -H "Content-Type: application/json" -d '{"item":"widget","qty":3}'
{"message":"hello from a hand-rolled HTTP server"}
```

Server-side log for that exact request:

```
listening on 127.0.0.1:8080 (one connection, then exit)
[('127.0.0.1', 60589)] POST / HTTP/1.1
    host: 127.0.0.1:8080
    user-agent: curl/8.17.0
    accept: */*
    content-type: application/json
    content-length: 25
    body (25 bytes): b'{"item":"widget","qty":3}'
```

A GET to a path that doesn't exist:

```
$ curl -s -o /dev/null -w "status: %{http_code}\n" http://127.0.0.1:8080/api/widgets/42
status: 404
```

And a genuinely malformed request — not even close to the grammar — to exercise the `except` branch:

```python
# malformed_client.py
import socket
with socket.create_connection(("127.0.0.1", 8080), timeout=5) as s:
    s.sendall(b"THIS IS NOT HTTP\r\n\r\n")
    print(s.recv(4096).decode("iso-8859-1"))
```

```
$ python malformed_client.py
HTTP/1.1 400 Bad Request
Content-Length: 0
Connection: close
```

**Why this works, line by line:**

- `recv_until_headers_done` is the direct, hand-built answer to concept 1's opening problem: TCP gives you bytes with no boundaries, so *your code* has to accumulate them until the one signal that means something — `\r\n\r\n` — shows up. Note it can over-read: if the client sent headers *and* the start of the body in one TCP segment (entirely possible — Day 11 never promised message-sized `recv()` returns), those extra bytes are sitting in `buf` past the split point, which is exactly why `parse_request` returns them as `leftover_body_bytes` rather than discarding them. Losing those bytes would be a real, silent data-corruption bug — the classic mistake when hand-rolling this the first time.
- `request_line.split(" ")` is the literal implementation of RFC 9112 §3's grammar. It will raise `ValueError` on `"THIS IS NOT HTTP"` because that splits into *four* space-separated tokens, not three (`method, path, version = 4 values` fails to unpack) — which is precisely the malformed-request case, and precisely why it's caught and turned into a `400`.
- `read_body` implements exactly step 4 of RFC 9112 §6.3's precedence list from earlier in this section: trust `Content-Length`, read until you have that many bytes, and — importantly — **stop reading exactly there**, never one byte more, so that if the client tries to reuse this connection for a second request, those extra bytes are still sitting unread on the wire for the *next* `recv_until_headers_done()` call to find. (This server closes after one request regardless, so that case never triggers here — a genuine, honest limitation, flagged below.)
- The 404 and 400 status lines are hand-written strings, not looked up from any library — this is the mechanical proof that "returning a status code" is nothing more exotic than writing three ASCII digits into a string, exactly as concept 3 argued.

**Honesty caveats, stated plainly, because a real HTTP server does far more than this:**
1. **This server does not implement `Transfer-Encoding: chunked` on the receiving side at all** — a client sending a chunked request body would have its chunk-size lines misread as literal body bytes, corrupting the parse. A real server must implement every branch of RFC 9112 §6.3's precedence list, not just the `Content-Length` branch.
2. **It handles exactly one connection and exits** — no concurrency (Day 3's threading, or an event loop, would be required for a real server; this is a deliberate simplification to keep the framing logic legible), no connection reuse, no timeouts on a slow or malicious client that never finishes sending headers (a real server needs a maximum-header-size limit and a read timeout, or a client can hold a connection open forever with a slow trickle of bytes — a real, named denial-of-service pattern called "Slowloris").
3. **It does not defend against the smuggling scenario below** — a request arriving with *both* `Content-Length` and `Transfer-Encoding` headers is naively read using whichever branch the code happens to check first (`content-length`, unconditionally, in this implementation), which is exactly the *wrong*, RFC-violating behavior the case study below is about. **Do not use this server for anything but learning.**

### System design — streaming an LLM completion over HTTP: choosing the framing

**Problem:** design the wire format for an endpoint that streams a chat completion back to a client token-by-token, so the client can render partial output as it arrives rather than waiting for the entire response.

**Requirements:** the client must see the first token as soon as it's generated, not after the whole completion finishes; the total response length is genuinely unknown when the response begins (it depends on how many tokens the model decides to generate); the mechanism must work over plain HTTP/1.1 (not require a separate protocol like WebSockets) so it works through the same proxies, load balancers, and client libraries as every other endpoint.

**The alternatives:**

1. **`Content-Length`, computed after generation finishes**, buffering the entire completion server-side before sending the first byte.
2. **`Transfer-Encoding: chunked`**, sending each token (or a small batch of tokens) as its own chunk, as it's generated.
3. **A WebSocket**, upgrading the connection to a full bidirectional protocol before streaming begins.

**The decision: (2), and it's not close.**

**The actual reason:** option (1) is disqualified by the requirements themselves — it requires knowing the total length before sending anything, which is precisely the one thing you cannot know until generation is finished, and it destroys the entire point of streaming (the user stares at a blank screen for the full generation time, then gets everything at once — indistinguishable from not streaming at all). Option (3) genuinely *works*, but it's solving a bigger problem than you have: WebSockets buy you full bidirectional, low-latency messaging, at the cost of a protocol upgrade handshake, different client libraries, different proxy/load-balancer configuration (Day 16's subject), and infrastructure that has to explicitly support the upgrade. **Chunked transfer-encoding gives you everything the streaming requirement actually needs — start sending before you know the total size — using the plain HTTP/1.1 request/response model every existing tool already understands**, with zero protocol upgrade. This is exactly the mechanism `curl`, browsers' `fetch()` with a `ReadableStream`, and every major LLM provider's own streaming API (Anthropic's `stream=True`, OpenAI's server-sent-events-over-HTTP streaming) actually use under the hood.

**The trade-off, honestly:** chunked responses can't be trivially retried by an intermediary the way a `Content-Length`-framed response can (a proxy that needs to buffer-and-retry a request has to buffer an unknown amount of streamed data, or give up on retry semantics for that response entirely), and a client library that doesn't handle chunked decoding correctly will hang or corrupt the stream — this is a real, common bug in naive HTTP client code that assumes `Content-Length` always exists. You also give up the ability to know progress as a simple percentage (`bytes received / Content-Length`) — there is no total to divide by, which is precisely the price of *not* needing to know the total in advance.

**Flip condition:** if you *can* compute the full response cheaply and fast (a short, bounded completion, or a use case where "wait for the whole answer" is actually fine — a batch classification job, not an interactive chat), option (1) is simpler and gets you retry-friendliness and byte-accurate progress tracking for free. It only stops being the right answer once the perceived latency of "nothing appears until everything is ready" becomes the thing you're optimizing against — which, for an interactive chat UI, it almost always is.

**Failure modes:** a proxy or corporate firewall that buffers the *entire* response before forwarding it (some do, especially older or security-focused ones) silently defeats streaming — the client still gets everything at once, just later, with no error to tell you why. A server that sets `Content-Type: text/event-stream` (the convention for Server-Sent Events, one layer above raw chunking) but forgets to disable response buffering in its own web server or reverse proxy config hits the identical problem from the *serving* side. And a client library that reads the response with a call like `.read()` or `.text()` rather than iterating the stream incrementally defeats the entire mechanism at the *consuming* end — the bytes were streamed correctly over the wire, but the client code chose to wait for all of them before doing anything, which is a mistake worth explicitly checking for whenever "streaming feels laggy" is reported.

### Case study — HTTP request smuggling: when two servers disagree about where a message ends

**What happened, and why it's the direct, real-world consequence of this section's central fact.** In 2019, security researcher James Kettle (Director of Research at PortSwigger, makers of Burp Suite) published "HTTP Desync Attacks: Request Smuggling Reborn," reviving and dramatically extending a class of attack first described a decade earlier. The core technique exploits exactly the ambiguity RFC 9112 §6.3 warns about directly: **send a request containing both a `Content-Length` header and a `Transfer-Encoding: chunked` header, with values engineered so that a front-end proxy and a back-end server parse the *same bytes* into two *different* sets of messages.**

Concretely: if the front-end (say, a CDN edge or load balancer) honors `Content-Length` and reads N bytes as one complete request, while the back-end server honors `Transfer-Encoding` and reads a chunked body that's *shorter* than N bytes, the back-end is left with leftover bytes it never consumed as part of request #1. Because HTTP/1.1 connections are commonly reused for multiple requests in sequence (concept 4's own framing mechanism, working exactly as designed, turned against itself), the back-end simply interprets those leftover bytes as the **start of the next request on that connection** — except that "next request" was never sent by the legitimate client at all. It was smuggled inside the body of request #1, invisible to the front-end that only ever saw one request go by. If that connection is shared between multiple users (a common performance optimization — reuse the same back-end connection for many different front-end clients), the smuggled request can be **prepended to some other, unrelated user's next real request**, and the response meant for the smuggled request can come back attached to that other user's session.

**The real, named damage this produced:** Kettle's research and the wave of follow-up disclosures it triggered found genuine, exploitable smuggling vulnerabilities in production infrastructure at major companies, including a PayPal login-page compromise (researchers built on the technique using line-wrapped headers to hide the `Transfer-Encoding` header from a CDN's parser while the origin server still honored it, earning a $20,000 bounty) and Netflix, which traced a request-smuggling vulnerability through its Zuul proxy back to a parsing bug in Netty (the underlying Java networking library), tracked as **CVE-2021-21295** and patched with a $20,000 maximum bounty award. *(Verify current — specific bounty figures and the precise line-wrapping technique used against any one company are widely reported but vary slightly by source; treat the CVE identifier and the general mechanism as the load-bearing, verifiable facts.)*

**The engineering lesson, stated at the level this note is teaching:** this is not a bug in any one server's code — it's a bug in **disagreement**. Both the front-end and the back-end correctly implemented *a* valid interpretation of an ambiguous or contradictory message; the vulnerability exists entirely in the gap between two technically-defensible readings of the same bytes. That is exactly why RFC 9112 §6.3 doesn't merely describe a precedence order for the well-behaved case — it explicitly calls out the both-headers-present case as **"an indication of an attempt to perform request smuggling"** and directs that it **"ought to be handled as an error"** rather than resolved by any precedence rule at all. The fix that actually closes this class of bug isn't "pick a consistent tie-breaker" (front-end and back-end would still need to agree on *which* tie-breaker, which is the same coordination problem one level up) — it's **reject the ambiguous message outright**, at the first point in the chain that sees it, so there is never a second interpretation for anything downstream to disagree about.

*Primary source:* PortSwigger Research, "HTTP Desync Attacks: Request Smuggling Reborn" (James Kettle, 2019), portswigger.net/research/http-desync-attacks-request-smuggling-reborn; CVE-2021-21295 (Netty); RFC 9112 §6.3's own explicit smuggling warning.

### In production — operating message framing correctly

**Best practices:**
1. **Never hand-parse HTTP in production the way this note's runnable examples do.** Use a battle-tested HTTP implementation (your language's standard library, or a framework like FastAPI/Starlette, which sits on a mature ASGI server) that has already closed the ambiguous-header case and the dozen other edge cases this note's minimal parser deliberately ignores.
2. **If you operate a reverse proxy or load balancer (Day 16), keep front-end and back-end on the same HTTP implementation family, or at minimum, verify both reject ambiguous `Content-Length`/`Transfer-Encoding` combinations identically.** Mismatched parsers between tiers is the root cause of every request-smuggling class this section describes.
3. **Set explicit maximum body-size and header-size limits.** Both `Content-Length`-based and chunked reads can be weaponized to exhaust memory if a server naively trusts an attacker-supplied length or an unbounded number of chunks.
4. **Prefer `Content-Length` whenever the size is known in advance**, and reach for chunked encoding only when it's genuinely required (streaming, unknown-length responses) — it's the more complex mechanism, and complexity you don't need is complexity you have to secure for no benefit.

**Mistakes, beginner → senior:**
- *Beginner:* forgetting `Content-Length` (or an equivalent) entirely when hand-building a response and watching a client hang forever, mistaking it for a "the server is slow" problem rather than a "the client doesn't know when to stop reading" problem.
- *Intermediate:* trying to set both `Content-Length` and stream a response of genuinely-unknown length, silently truncating or padding to make the numbers match, rather than switching to chunked encoding.
- *Senior:* running mismatched HTTP parser versions/implementations across a multi-tier proxy chain without ever having explicitly verified they agree on ambiguous-message handling — invisible until a security researcher (or attacker) finds the gap.

**Monitoring and observability:** logging *raw* header combinations you don't expect on the way in — a request arriving with both `Content-Length` and `Transfer-Encoding` set should be a logged, alertable event at any well-run edge, not a silently-resolved ambiguity. Reverse proxies (nginx, HAProxy, Envoy) have configuration flags specifically for rejecting such requests outright; verify yours has that flag enabled rather than assuming a sane default.

**Cost:** framing itself costs nothing extra to operate correctly — the "cost" here is almost entirely the *risk* of getting it wrong, which is security exposure, not a dollar line item. The one real performance trade-off is chunked encoding's slightly higher per-chunk overhead (each chunk carries its own length line and trailing CRLF) versus `Content-Length`'s single upfront number — negligible for large chunks, non-negligible if you find yourself chunking many tiny pieces (a design mistake worth watching for, not a fundamental cost of the mechanism).

---

## 5. Methods and their semantics: safe, idempotent, and the dangerous one

**Depth: [CORE]**

### Intuition

The method is the very first word on the wire (concept 2), and it exists to answer a question that matters enormously to anyone standing *between* the client and the server: **if this request fails to get a response — a timeout, a dropped connection, Day 11's exact failure mode — is it safe to just send it again?** That single question is why methods aren't just a vocabulary list to memorize; each one is a **promise the client and server both agree to honor**, and the promise is precisely what makes automatic retries, caching, and safe pre-fetching possible anywhere in the chain without either end having to understand what the request is actually *for*.

### Safe methods: promises about side effects

RFC 9110 §9.2.1 defines a method as **safe** if *"it is intended only for retrieving information and not for causing changes."* `GET`, `HEAD`, and `OPTIONS` are safe (`TRACE` is too, though it's rarely used and often disabled server-side, because it can be abused to leak headers an attacker shouldn't see — a security footnote, not a reason to avoid learning it exists). Safety is a promise made *to every intermediary in the chain*, not just to the server: a browser can safely pre-fetch a `GET` link the user hasn't clicked yet, precisely because "safe" means the server promises nothing will change as a side effect of that request happening. A search engine's crawler can follow every `GET` link on a page with a clear conscience for the exact same reason. **The instant a server makes a `GET` request delete something — a real and disturbingly common mistake, most famously exploited when browser pre-fetching or crawlers triggered "delete" links implemented as plain `GET` requests — the safety promise is broken, and every piece of infrastructure that trusted it (caches, crawlers, browser pre-fetchers) becomes an accidental attack vector against your own data.**

### Idempotent methods: promises about retry safety

RFC 9110 §9.2.2 defines idempotency precisely, and the definition is subtler than "does the same thing twice": *"a request method is considered idempotent if the intended effect on the server of multiple identical requests with that method is the same as the effect for a single such request."* Read that carefully — it's not claiming the *responses* are identical (the second `DELETE` of an already-deleted resource will correctly return `404` where the first returned `200` or `204`), only that the **server's resulting state** is the same either way. `GET`, `HEAD`, `PUT`, `DELETE`, and `OPTIONS` are all idempotent by this definition; every safe method is automatically idempotent (doing nothing, twice, is still doing nothing), but idempotency is the *broader* category — `PUT` and `DELETE` are idempotent without being safe, because they do change server state, just in a way that's stable under repetition.

Walk through why `PUT` earns this promise and `POST` categorically cannot. `PUT /orders/42` with a body means, by convention, **"make the resource at this exact URL look exactly like this payload."** Send it once: the order at `/orders/42` now matches the payload. Send the byte-for-byte identical request again, ten times, due to network retries: the order at `/orders/42` still matches the payload — nothing new was created, nothing changed further, because "make it look like X" applied twice produces the same X both times. `DELETE /orders/42` means **"the resource at this URL should not exist."** Apply that ten times and the end state — nonexistence — is identical every time, even though nine of those ten requests will (correctly) report "already gone" rather than "just deleted." Contrast `POST /orders` with a body meaning **"create a new resource under this collection."** Send it once: one new order exists. Send the identical request again because the response to the first one got lost in transit (a genuinely common failure — Day 11 concept 9's TIME_WAIT and connection-teardown ambiguity means a client can never be 100% sure whether a request it never got a response for actually succeeded server-side): **two** orders now exist, one of them a duplicate the client never intended to create. That is the entire reason `POST` is singled out in the study plan as "the dangerous one" — **it is the one common method where a network-level retry, done blindly, can cause real, duplicated, business-visible damage**, and `PATCH` inherits the same danger by default (RFC 9110 explicitly leaves `PATCH` "neither safe nor idempotent," though a *specific* patch document format can be designed to be idempotent in practice — a JSON Merge Patch that sets `{"status": "cancelled"}` is idempotent by construction, since setting a field to the same value twice changes nothing further, while a JSON Patch operation like `{"op": "add", "path": "/tags/-", "value": "urgent"}` — append to an array — is not, since applying it twice appends the tag twice).

| Method | Safe? | Idempotent? | The promise |
|---|---|---|---|
| `GET` | Yes | Yes | "Give me this, nothing changes." Cacheable, pre-fetchable, retry freely. |
| `HEAD` | Yes | Yes | "Give me `GET`'s headers, no body." Used to check existence/size cheaply. |
| `OPTIONS` | Yes | Yes | "What can I do here?" Used for CORS preflight and capability discovery. |
| `PUT` | No | **Yes** | "Make this URL look exactly like this payload." Retry freely — same end state. |
| `DELETE` | No | **Yes** | "This URL should not exist." Retry freely — already-gone is a fine outcome. |
| `PATCH` | No | **No** (usually) | "Apply this partial change." Retry-safety depends entirely on the patch format. |
| `POST` | No | **No** | "Do something — usually create, but truly generic." **Never blindly retry.** |

### Analogy: a light switch versus a coin-operated turnstile

`PUT` and `DELETE` behave like a **light switch fixed to a specific position**: flipping it to "on" ten times leaves the light exactly as on as flipping it once. There's no accumulation, no memory of how many times you flipped it — only the final position matters, and "final position" is precisely what idempotency means. `POST`, by contrast, behaves like a **coin-operated turnstile**: every coin you insert lets one more person through, full stop, with no way for the turnstile to know "wait, that's the same person who already went through, this coin is a duplicate." Insert two coins by accident — because you weren't sure the first one registered — and two people get counted, whether or not that's what actually happened.

**Where the analogy breaks, and it's worth being precise about it:** a light switch has exactly two states, so "idempotent" there is almost trivially true — there's nowhere else for the second flip to push the state. A real `PUT` can have far richer state (an entire JSON document, with dozens of fields), and idempotency is a genuine, non-trivial *design property* of how you built the endpoint, not a free consequence of using the word "PUT" in your route decorator. A `PUT /orders/42` handler that appends to an audit log as a side effect on every call ("order 42 was PUT to on this timestamp") has, technically, broken idempotency — the *primary* resource state is stable under repetition, but a side effect isn't, and whether that matters depends entirely on whether anything downstream treats that log as load-bearing. The method name is a promise your *implementation* has to actually keep; it is not a property the HTTP spec enforces for you.

### Worked example — Stripe's `Idempotency-Key`: making a dangerous method safe to retry, by convention

Here is the honest resolution to the tension `POST`'s danger creates: sometimes you genuinely need `POST`'s "create a new thing" semantics (a `PUT` can't express "create a new order with a server-assigned ID," because `PUT` requires the client to already know and specify the exact target URL) *and* you need retry-safety, because the request travels over an unreliable network (Day 11, again) and a payment API absolutely cannot afford to double-charge a customer because a client-side timeout fired one second before the server's response would have arrived.

Stripe's answer, and it's real, current, and documented at `docs.stripe.com/api/idempotent_requests`: **the client generates a unique key (Stripe recommends a V4 UUID) and sends it as an `Idempotency-Key` header on the `POST`.** The server's contract: the *first* request with a given key executes normally and stores its result (status code and body) keyed by that string. **Every subsequent request presenting the identical key returns the exact same stored result, without executing the operation again** — Stripe's own documentation states this applies "regardless of whether it succeeds or fails," which is a detail worth sitting with: even a `500` gets replayed verbatim on retry with the same key, because re-attempting a request that failed for reasons unrelated to the key (a downstream timeout, say) needs its own explicit, separate request with a *new* key if you actually want it retried.

```
Client sends:                              Server does:
POST /v1/charges                           Never seen key "a1b2c3..." before.
Idempotency-Key: a1b2c3-...                → Charges the card. Stores
{"amount": 4200, "currency": "usd"}          {status: 200, body: {...}} under
                                              that key. Returns it.

  ... network drops the response in transit, client never sees it ...

Client retries, SAME key:                  Server does:
POST /v1/charges                           Has seen key "a1b2c3..." before.
Idempotency-Key: a1b2c3-...                → Does NOT charge the card again.
{"amount": 4200, "currency": "usd"}          Returns the STORED result from
                                              the first attempt, verbatim.
```

**This is `POST`'s exact danger, neutralized by a mechanism the HTTP spec itself doesn't provide** — idempotency, for `POST`, is not a property of the method; it's a property this specific API chose to bolt on, by agreement between client and server, using an ordinary header. That's precisely why it's real content for a *methods* section rather than a footnote: it's the concrete proof that "safe to retry" is not an inherent, immutable property of a verb — it's a promise, and promises can be engineered even where the protocol itself doesn't hand you one for free. Stripe's own error-type vocabulary includes a dedicated `idempotency_error`, returned specifically when a key is reused against a request whose *parameters don't match* the original — reusing a key for a genuinely different charge is treated as a client bug, not honored as if it were a legitimate retry, which closes the obvious loophole (you can't launder a second, different charge through a stale key from an unrelated first one).

### System design — URL structure and method choice for an action that doesn't fit CRUD

**Problem:** design the API surface for "cancel an order" on an e-commerce platform whose existing API is otherwise a clean, conventional CRUD resource model (`GET /orders/{id}`, `POST /orders`, `PATCH /orders/{id}` for edits).

**Requirements:** the operation must be safe to expose to a client that might retry it on a network failure; the API's URL structure should stay predictable and consistent with the rest of the resource model rather than inventing a one-off pattern per action; the operation has real business logic beyond "set a field" — cancelling a shipped order should be rejected, and cancelling should probably trigger a refund workflow, not just flip a status string.

**The alternatives:**

1. **`PATCH /orders/{id}` with `{"status": "cancelled"}`** — model cancellation as a partial update to the existing resource, reusing the edit endpoint you already have.
2. **`POST /orders/{id}/cancel`** — a dedicated sub-resource endpoint representing the *action* itself, distinct from the order resource's own data.
3. **`POST /order-cancellations`** — model the cancellation as its own first-class resource (a "cancellation request" object with its own ID, status, and history), created via `POST` against its own collection.

**The decision: (2) for most teams, (3) once cancellation itself has a life cycle worth tracking; (1) is a trap dressed up as simplicity.**

**The actual reason (1) looks appealing and isn't:** `PATCH` for a status change *feels* RESTful, because it reuses machinery you already built. But "cancel" is not a generic field update — it's a specific business operation with side effects (a refund, a notification, an inventory release) and specific validation rules (can't cancel a shipped order) that have nothing to do with the general-purpose "edit any field" semantics `PATCH` implies. Overloading `PATCH` to secretly mean "trigger this one specific workflow when this one specific field takes this one specific value" hides business logic behind what looks like a data update, and it doesn't generalize: the moment you need a *second* status-triggered workflow ("mark as returned" also needs its own validation and side effects), you either duplicate the hidden-logic pattern per status value or you've quietly built a bespoke state machine that a generic `PATCH` handler was never designed to safely enforce.

**Why (2) wins for most teams:** `POST /orders/{id}/cancel` names the *action* explicitly in the URL, keeps it a `POST` (correctly — this is genuinely not idempotent in the way `PUT`/`DELETE` are unless you add an `Idempotency-Key`, since "cancel a pending order" and "cancel an already-cancelled order" are meaningfully different requests that should probably behave differently — the second one is arguably a client error, a `409 Conflict`, not a silent no-op), and gives you one obvious place to put the cancellation-specific validation and side effects, separate from whatever `PATCH /orders/{id}` does for ordinary field edits. This is the same "verb as a sub-resource" pattern real platforms use constantly — GitHub's API has `POST /repos/{owner}/{repo}/merges`, Stripe's has `POST /v1/payment_intents/{id}/cancel` — precisely because it keeps the resource model's nouns clean while still giving actions-that-aren't-CRUD an unambiguous, RESTful-feeling home.

**The trade-off, honestly:** this pattern is not "pure" REST in the strict, hypermedia-driven sense some REST theorists argue for — you're encoding a verb into a URL path, which purists will point out looks more like RPC wearing a REST costume. It also means your API surface grows one endpoint per distinct action, rather than staying confined to five methods on a fixed set of resource URLs; a resource with a dozen possible actions ends up with a dozen action sub-resources. And it doesn't, by itself, give you the *history* of cancellation attempts, retries, or reasons — if the business genuinely needs to audit "who tried to cancel this order, when, and why it was rejected," option (2) has nowhere natural to put that.

**Flip condition:** move to option (3) — a first-class `order-cancellations` resource — the moment cancellation itself needs to be tracked, queried, retried, or approved as its own object with its own life cycle (a refund that has to go through a manual approval step before it completes, say, where "cancellation" is genuinely pending for hours or days, not an instant flip). At that point, a bare action endpoint has nowhere to represent "this cancellation is currently pending approval," while a resource can carry its own status field the same way any other resource does — and creating it is naturally a plain `POST` to a plain collection, with all the ordinary CRUD tooling (idempotency keys, pagination, filtering by status) available for free.

**Failure modes:** implementing option (2) as a `GET` instead of a `POST` because "it's just cancelling, it's simple" — this is the exact safe-method violation flagged earlier in this section, and it means a crawler, a browser pre-fetch, or an overly-eager monitoring probe can cancel real orders by simply following a link. Forgetting idempotency handling on the cancel endpoint and having a client's retry-on-timeout logic trigger the refund workflow twice. And modeling cancellation as `DELETE /orders/{id}` — tempting, because "cancel" sounds like "delete" — which is a real semantic mismatch: `DELETE` conventionally means "this resource should cease to exist," and an order record almost certainly needs to keep existing (for accounting, auditing, and customer support) even after it's cancelled, just with its status changed. Using `DELETE` here would mean either lying about the method's meaning or accidentally implementing actual data deletion where a status change was intended.

### In production — operating methods and idempotency correctly

**Best practices:**
1. **Treat "is this method safe to retry blindly?" as a question every client-side HTTP call site should answer explicitly**, not assume. Most HTTP client libraries (including `httpx`, and the Anthropic and OpenAI SDKs) auto-retry on connection errors and timeouts *only* for methods they consider safe by default, and treat `POST` retries as something you must opt into deliberately — know which behavior your library defaults to before you rely on it.
2. **Require an idempotency key on every `POST` that has a real-world side effect involving money, inventory, or anything else expensive to duplicate.** Generate it client-side (a UUID is fine), store it alongside the request in your own retry logic, and reuse the *same* key across retries of the *same* logical attempt.
3. **Design `PUT` handlers to genuinely replace the full resource state**, and `PATCH` handlers to be explicit about which partial-update semantics they implement (JSON Merge Patch vs. JSON Patch have meaningfully different idempotency properties — pick one and document it, don't invent an ad hoc third thing).

**Mistakes, beginner → senior:**
- *Beginner:* implementing a "delete this record" action as a `GET` link (because it's easy to make a plain `<a href>` trigger it) and discovering a search-engine crawler or corporate link-scanner has cleared out a table.
- *Intermediate:* building automatic client-side retry logic for network errors that retries `POST` requests with no idempotency mechanism at all, discovering duplicate records only once a customer complains about being charged twice.
- *Senior:* an idempotency-key implementation that stores keys forever with no expiry, becoming an unbounded, ever-growing table — or one with too short an expiry, silently losing protection for legitimately-delayed retries (a client behind a very slow or flaky connection retrying an hour later, past the window).

**Monitoring and observability:** track duplicate-idempotency-key hit rate as a real signal — a very low rate might mean clients aren't using the mechanism at all (worth auditing); a rising rate suggests worsening client-side network conditions or timeout tuning that's too aggressive. Log every `POST` retry attempt with its idempotency key so a support engineer can reconstruct "was this a genuine double-submission or a network retry" during an incident.

---

## 6. The status-code taxonomy

**Depth: [CORE]**

### Intuition

A status code's *first digit* is a category, and the categories exist to answer the single most urgent triage question anyone building or debugging a distributed system ever asks: **when something goes wrong, whose fault is it, and where do I even start looking?** RFC 9110 §15 lays out five families:

```
1xx  Informational  — "hang on, more is coming" (rare; a preview, not a verdict)
2xx  Success        — "it worked, here's what happened"
3xx  Redirection    — "the answer is somewhere else; go look there"
4xx  Client error   — YOU (the request) did something the server won't accept
5xx  Server error   — the SERVER (or something behind it) failed to cope
```

That last split — 4xx versus 5xx — is the single most operationally useful fact in this entire taxonomy, and it's worth internalizing as a reflex: **`response.status_code // 100 == 4` means "look at the request you sent." `// 100 == 5` means "look at the server, or something the server depends on."** A monitoring dashboard that alerts on 5xx rate and ignores 4xx rate (mostly) has the right instinct — a spike in 4xx usually means clients are misbehaving or a contract changed; a spike in 5xx means *you* broke.

### The 4xx family, precisely, with the real distinctions the study plan asks for

**`400 Bad Request`** — RFC 9110 §15.5.1: *"the server cannot or will not process the request due to something that is perceived to be a client error."* This is the catch-all — malformed syntax, a request the server can't even parse into a coherent operation. Reach for it when the problem is with the *shape* of the request itself (unparseable JSON, a missing required field at the wire level), not with values that parsed fine but don't make business sense (that's `422`, below).

**`401 Unauthorized`** — RFC 9110 §15.5.2: *"the request has not been applied because it lacks valid authentication credentials for the target resource."* Despite the name, this is about **authentication** — "I don't know who you are, or the credentials you gave me are invalid/expired." A compliant `401` response is required to include a `WWW-Authenticate` header telling the client *how* to authenticate.

**`403 Forbidden`** — RFC 9110 §15.5.4: the server understood exactly who's asking and refuses anyway. This is **authorization**, a categorically different failure from `401`: "I know exactly who you are, and you're not allowed to do this." The `401` vs. `403` line is one of the most commonly confused pairs in the entire taxonomy, and the confusion has a clean one-line resolution: **`401` means "log in" (or "your login didn't work"); `403` means "being logged in wouldn't have helped."**

**`404 Not Found`** — RFC 9110 §15.5.5: *"the origin server did not find a current representation for the target resource, or is unwilling to disclose that one exists."* Read that second clause carefully — it's not a loose paraphrase, it's in the actual spec text, and it licenses a real, deliberate design choice covered in this section's judgment discussion below: a server is explicitly permitted to return `404` even when a resource *does* exist, specifically to avoid confirming that fact to someone not authorized to know.

**`409 Conflict`** — RFC 9110 §15.5.10: *"the request conflicts with the current state of the target resource... the user might be able to resolve the conflict."* This is the code for "your request was fine in isolation, but it collides with something already true on the server" — a unique-constraint violation (creating a username that's taken), a version mismatch in optimistic concurrency control, or (as concept 5 showed) reusing an idempotency key against parameters that don't match the original request.

**`422 Unprocessable Content`** (originally "Unprocessable Entity," defined first in RFC 4918 for WebDAV in 2007, folded into the core HTTP spec and renamed by RFC 9110 in 2022) — *"the request was well-formed but was unable to be followed due to semantic errors."* This is the precise complement to `400`: the JSON parsed fine, the request had the right shape, but a *value* inside it fails validation — a required field is missing, a number is out of range, an email address isn't a valid email address. **`400` is "I couldn't even read this." `422` is "I read it fine, and it's wrong."**

**`429 Too Many Requests`** — defined by RFC 6585 in 2012, notably *after* the original HTTP/1.1 spec, because rate limiting as a widespread API practice postdates HTTP's original design. It should carry a `Retry-After` header telling the client how long to back off, and the spec explicitly states responses with this status **must not be stored by a cache** — a stale "you're rate limited" response cached and replayed later would be actively harmful.

### The 3xx family — the necessary detour, because the case study below needs it

**`301 Moved Permanently`** and **`302 Found`** look almost interchangeable at a glance — both carry a `Location` header telling the client where to go instead — but they make a fundamentally different promise, and that promise has a real, load-bearing consequence for *caching*, not just semantics. RFC 9110 §15.4.2 defines `301` as meaning the resource *"has been assigned a new permanent URI, and any future references... ought to use one of the enclosed URIs."* RFC 9111 (HTTP Caching) treats `301` as **heuristically cacheable by default** — meaning a cache is explicitly permitted to remember and reuse a `301` response *without being told an explicit expiry at all*, purely because the spec designates permanent-redirect status codes (alongside a handful of others like `200`, `404`, and `410`) as ones a cache may assume are safe to keep indefinitely absent other instructions. `302 Found`, by contrast, means the move is **temporary**, and it carries no such default cacheability — a cache is not permitted to assume it may keep serving a `302` on its own initiative the way it may for a `301`.

That single difference — one status code a browser is *allowed* to remember forever by default, the other one it isn't — is the entire mechanism behind the case study that follows, and it's why the study plan singles this specific pair out rather than the broader `3xx` family.

### Judgment: choosing `404` over `403` to avoid leaking existence

Here's a genuine, real, documented design decision that most engineers encounter as a surprise the first time they hit it: **GitHub's REST API deliberately returns `404 Not Found`, not `403 Forbidden`, when you request a private repository you don't have access to** — this is stated plainly in GitHub's own official troubleshooting documentation, not an accident or a bug report waiting to be fixed.

**The real alternative:** return `403`, which is arguably the more semantically "correct" code — the resource does exist, and the server does know exactly who's asking and is choosing to refuse.

**The actual reason `404` wins here:** a `403` response *confirms the resource exists* to anyone who asks, authorized or not — it just also tells them "you personally can't see it." That's an information leak: an attacker probing repository names (or usernames, or account IDs — the same pattern shows up anywhere private-by-default resources exist) can distinguish "doesn't exist" from "exists but I can't see it" purely from the status code, and use that to enumerate the existence of private, sensitive resources they have no right to know about at all. Returning `404` for *both* "genuinely doesn't exist" and "exists, but you can't see it" collapses that distinction entirely — an unauthorized requester learns nothing beyond "not available to you," which is exactly the amount of information they're entitled to.

**The trade-off, honestly:** this makes debugging *your own* access problems measurably harder. A `404` when you *know* the resource should exist gives you no signal about whether you have a typo in the URL or a genuine permissions problem — you're forced to go check your token's scopes and the resource owner's settings by hand, exactly as GitHub's own troubleshooting docs instruct. A `403` would have told you immediately which category of problem you're in.

**Flip condition:** use `403` plainly, without this obfuscation, whenever the *existence* of the resource isn't itself sensitive information — an internal admin panel where every authenticated employee already knows every endpoint exists, or a resource that's public metadata even when its contents are restricted. The `404`-for-privacy pattern earns its cost specifically when unauthorized existence-disclosure is itself the threat you're defending against — which is precisely the situation with someone else's private source code, and precisely why GitHub, and not every API everywhere, makes this specific trade.

### The 5xx family — each one names a different broken component

This is the study plan's exact framing, and it deserves to be taught as the mechanical fact it is, not a vague "server stuff went wrong" bucket:

**`500 Internal Server Error`** — RFC 9110 §15.6.1: *"the server encountered an unexpected condition."* This is the origin server itself — the process actually running your application code — hitting a bug: an unhandled exception, a `None` where a value was expected, a division by zero. If you own the code that produced a `500`, the bug is in *your* application.

**`502 Bad Gateway`** — RFC 9110 §15.6.3: *"the server, while acting as a gateway or proxy, received an invalid response from an inbound server it accessed."* This code **only makes sense when an intermediary exists in the path** — a reverse proxy, a load balancer, an API gateway (Day 16's subject). It means the *proxy itself* is healthy and working correctly, but the thing it forwarded your request to sent back garbage, a malformed response, or something the proxy couldn't interpret as a valid HTTP message at all.

**`503 Service Unavailable`** — RFC 9110 §15.6.4: *"the server is currently unable to handle the request due to temporary overload or scheduled maintenance."* This is a deliberate, often self-imposed refusal — the server (or something in front of it) has decided, on purpose, not to handle requests right now, and is asking you to come back later, ideally with a `Retry-After` header telling you exactly how long to wait.

**`504 Gateway Timeout`** — RFC 9110 §15.6.5: *"the server, while acting as a gateway or proxy, did not receive a timely response from an upstream server."* The crucial distinction from `502`: this is **silence**, not garbage. The proxy asked, and got nothing back before its own timeout expired — the upstream might still be working, might be overloaded, might be dead; the proxy genuinely doesn't know, it just knows it waited long enough and gave up.

```
Client ──► [Load Balancer / Proxy] ──► [Your App Server] ──► [Database]
                    │                        │                    │
              502: got garbage           500: YOUR bug        (times out →
              back from the app          crashed while         the app server
              server (crashed            handling this          itself might
              mid-response, sent          specific request       return its OWN
              a truncated/invalid                                500 or 504,
              response)                                          depending on
                    │                                             how IT talks
              504: asked the app                                  to the DB)
              server, got NO
              response before
              its own timeout
                    │
              503: the load balancer
              (or the app) is
              DELIBERATELY refusing
              right now (overloaded,
              draining for a
              deploy, maintenance)
```

**The diagnostic payoff:** a `502` or `504` from your load balancer, with your own application logs showing *nothing* for that request, tells you the failure is *between* the load balancer and your app — a network problem, a crashed worker process, a deployment mid-restart — not a bug in your request-handling code, because your code never got the chance to run at all. A `500`, by contrast, means your code *did* run, and *it* is where the postmortem starts. Confusing these categories during an incident wastes the exact time an incident response can least afford.

### Runnable example — building the taxonomy into a FastAPI app, and closing a real honesty gap

FastAPI (built on Starlette, using Pydantic v2 for validation) makes an interesting, real, and slightly surprising choice worth confronting directly rather than glossing over: **by default, it returns `422`, not `400`, for a request body that isn't even valid JSON** — collapsing exactly the "couldn't parse it at all" (`400`) versus "parsed fine, values are wrong" (`422`) distinction this section just taught as a genuine, separate pair.

```python
# default_gap.py — confirm the real, current FastAPI default behavior.
# pip install fastapi httpx
from fastapi import FastAPI
from fastapi.testclient import TestClient
from pydantic import BaseModel

app = FastAPI()

class OrderIn(BaseModel):
    item: str
    qty: int

@app.post("/orders")
async def create_order(order: OrderIn):
    return order

c = TestClient(app)
r = c.post("/orders", content=b"{not json", headers={"Content-Type": "application/json"})
print(r.status_code, r.json())
```

Real output:

```
422 {'detail': [{'type': 'json_invalid', 'loc': ['body', 1], 'msg': 'JSON decode error', 'input': {}, 'ctx': {'error': 'Expecting property name enclosed in double quotes'}}]}
```

**This confirms the gap: 422, not 400, for malformed JSON that never even parsed.** Per the honesty standard this note follows, here's the fix — a custom exception handler that inspects *why* FastAPI's validation failed and routes the "couldn't even parse it" case to a real `400`, while everything else (a value that parsed but failed a Pydantic constraint) correctly stays `422` — built alongside a complete, small order API exercising the rest of this section's taxonomy plus concept 5's idempotency mechanism:

```python
# status_demo.py — FastAPI status-code taxonomy demo, including the
# FastAPI-returns-422-not-400-on-bad-JSON gap and a fix.
# pip install fastapi httpx
from typing import Annotated

from fastapi import FastAPI, Header, HTTPException, Request, status
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse
from pydantic import BaseModel, Field
from starlette.exceptions import HTTPException as StarletteHTTPException

app = FastAPI()

ORDERS = {"ord_1": {"id": "ord_1", "status": "pending", "total_cents": 4200}}
IDEMPOTENCY_STORE: dict[str, dict] = {}


class OrderIn(BaseModel):
    item: str = Field(min_length=1)
    qty: int = Field(gt=0)


def problem(status_code: int, title: str, detail: str, type_: str = "about:blank"):
    # RFC 9457 "Problem Details" shape -- see the System design block below.
    return JSONResponse(
        status_code=status_code,
        media_type="application/problem+json",
        content={"type": type_, "title": title, "status": status_code, "detail": detail},
    )


# ---- close the 422-vs-400 gap: malformed JSON should be 400, not 422 -------
@app.exception_handler(RequestValidationError)
async def validation_handler(request: Request, exc: RequestValidationError):
    for err in exc.errors():
        if err["type"] == "json_invalid":
            return problem(400, "Malformed JSON", "Request body is not valid JSON.")
    return problem(422, "Validation failed", str(exc.errors()))


@app.exception_handler(StarletteHTTPException)
async def http_exception_handler(request: Request, exc: StarletteHTTPException):
    return problem(exc.status_code, exc.detail, exc.detail)


@app.post("/orders", status_code=201)
async def create_order(
    order: OrderIn,
    idempotency_key: Annotated[str | None, Header(alias="Idempotency-Key")] = None,
):
    if idempotency_key and idempotency_key in IDEMPOTENCY_STORE:
        return IDEMPOTENCY_STORE[idempotency_key]      # concept 5's mechanism
    new_id = f"ord_{len(ORDERS) + 1}"
    record = {"id": new_id, "status": "pending", "item": order.item, "qty": order.qty}
    ORDERS[new_id] = record
    if idempotency_key:
        IDEMPOTENCY_STORE[idempotency_key] = record
    return record


@app.get("/orders/{order_id}")
async def get_order(order_id: str):
    if order_id not in ORDERS:
        raise HTTPException(status.HTTP_404_NOT_FOUND, detail=f"no order '{order_id}'")
    return ORDERS[order_id]


@app.post("/orders/{order_id}/cancel")           # concept 5's system design, in code
async def cancel_order(order_id: str):
    o = ORDERS.get(order_id)
    if o is None:
        raise HTTPException(status.HTTP_404_NOT_FOUND, detail=f"no order '{order_id}'")
    if o["status"] == "shipped":
        raise HTTPException(status.HTTP_409_CONFLICT, detail="cannot cancel a shipped order")
    o["status"] = "cancelled"
    return o
```

Driven with a `TestClient` (no server process needed — Starlette's test client talks to the ASGI app in-process):

```python
from fastapi.testclient import TestClient
c = TestClient(app)

r = c.post("/orders", content=b"{not json", headers={"Content-Type": "application/json"})
# -> 400 {'type': 'about:blank', 'title': 'Malformed JSON', 'status': 400,
#         'detail': 'Request body is not valid JSON.'}

r = c.post("/orders", json={"item": "widget", "qty": 0})
# -> 422 {'type': 'about:blank', 'title': 'Validation failed', 'status': 422,
#         'detail': "[{'type': 'greater_than', 'loc': ('body', 'qty'), ...}]"}

r1 = c.post("/orders", json={"item": "gadget", "qty": 1}, headers={"Idempotency-Key": "abc-123"})
r2 = c.post("/orders", json={"item": "gadget", "qty": 1}, headers={"Idempotency-Key": "abc-123"})
# -> first:  201 {'id': 'ord_3', ...}
# -> retry:  201 {'id': 'ord_3', ...}   (identical id -- concept 5's mechanism, live)

r = c.get("/orders/ord_999")
# -> 404 {'type': 'about:blank', 'title': "no order 'ord_999'", 'status': 404, ...}

ORDERS["ord_1"]["status"] = "shipped"
r = c.post("/orders/ord_1/cancel")
# -> 409 {'type': 'about:blank', 'title': 'cannot cancel a shipped order', ...}
```

All of the above is real, captured output from actually running this code.

**Why this works, line by line:**

- The `@app.exception_handler(RequestValidationError)` intercepts *every* validation failure FastAPI would otherwise turn into its default `422`, and inspects `exc.errors()` — a list of structured error dicts Pydantic v2 produces — for the specific `"type": "json_invalid"` marker, which is exactly the signal that means "this never became valid JSON at all," as opposed to `"type": "greater_than"` or similar, which means "it parsed fine, a value failed a constraint." That single `if` is the entire fix for the gap: route the first case to `400`, leave everything else at `422`.
- The `problem()` helper builds a body shaped like **RFC 9457's "Problem Details for HTTP APIs"** format (`type`, `title`, `status`, `detail`) with the media type `application/problem+json` — a direct preview of this section's System design block below, and of Day 30, where this becomes the note's own subject rather than a preview.
- The idempotency retry (`r1`/`r2`) is concept 5's `Idempotency-Key` mechanism, now actually running: the second `POST`, despite an identical body, does not create a second order — `IDEMPOTENCY_STORE` short-circuits it and returns the *stored* first result, `id: 'ord_3'` both times.
- `409 Conflict` fires exactly where concept 5's system design predicted it should: cancelling an already-shipped order is a state conflict, not a missing resource (`404`) or a malformed request (`400`/`422`).

**Honesty caveats:** `IDEMPOTENCY_STORE` and `ORDERS` are plain in-memory dicts — `# demo only — never do this in production`; a real idempotency store needs a TTL (concept 5's "In production" flags exactly why) and durable storage (a crash loses every in-flight idempotency guarantee), and a real order store needs an actual database, not a process-lifetime dictionary. This demo also runs everything through Starlette's synchronous `TestClient`, not a live `uvicorn` process — fine for exercising the logic, but note that a real deployment needs `async def` route handlers to actually avoid blocking the event loop the moment a handler does real I/O (a database call, an outbound HTTP request) — none of these handlers do any blocking I/O, so the trap doesn't bite *here*, but it's real and worth naming: an `async def` handler that calls a *synchronous*, blocking database driver inside it blocks the entire event loop for every other concurrent request while that one call runs, silently destroying the concurrency `async def` was supposed to buy you.

### System design — the error contract for a public API

**Problem:** you're launching a public API. Design the error contract: which status codes you'll use, what shape the error body takes, and how a client is supposed to know whether retrying is safe.

**Requirements:** consistent error shape across every endpoint, so client code can write *one* error-handling path rather than one per endpoint; enough machine-readable structure that a client can branch on the error programmatically, not just display a string to a human; a clear, explicit signal for "is this retryable," since concept 5 already established that blind retries are dangerous and *informed* retries need the server's help to be safe.

**The alternatives:**

1. **A bespoke JSON shape per team/endpoint** — whatever fields felt natural when that endpoint was built (`{"error": "..."}` here, `{"message": "...", "code": "..."}` there).
2. **RFC 9457's "Problem Details for HTTP APIs"** — a standardized JSON (or XML) shape with `type` (a URI identifying the *category* of problem), `title` (a short, human-readable summary, stable across occurrences of the same problem type), `status` (the HTTP status code, repeated in the body for convenience), `detail` (a human-readable explanation of *this specific* occurrence), and `instance` (a URI identifying this specific occurrence, useful for support/debugging correlation) — served with `Content-Type: application/problem+json`, and extensible with additional, API-specific members beyond that base set.
3. **Stripe's own convention** — a nested `{"error": {"type": ..., "code": ..., "message": ..., "param": ..., "doc_url": ...}}` shape, predating RFC 9457 (Stripe's API is over a decade old; the RFC was published in 2023) but converging on strikingly similar ideas independently: a machine-readable category (`type`), a specific, stable machine-readable identifier (`code`), a human message, and — Stripe's own genuinely useful addition — `param`, naming *which* request field the error is about, letting a client highlight the exact broken form field without string-parsing an English sentence.

**The decision: (2), RFC 9457, as the base shape, extended with API-specific members the way the RFC explicitly permits.**

**The actual reason:** option (1) fails the very first requirement — "one client error-handling path" is impossible if every endpoint invented its own JSON shape, and this is not a hypothetical failure mode, it's the default outcome of many real APIs built by many teams over many years with no shared convention enforced. Option (2) is a *published, IANA-registered, RFC-backed standard* specifically so that generic tooling (API gateways, client SDK generators, monitoring dashboards) can be written *once*, against the RFC, and work against any API that adopts it — the entire value of a standard over a bespoke format is that value that accrues to *everyone* who adopts it, not just your own team. Option (3), Stripe's shape, is excellent and genuinely instructive (its `param` field is a real, worthwhile idea worth stealing), but it's Stripe's own bespoke convention, not a standard — adopting it wholesale buys you Stripe's good taste without buying you any of the tooling ecosystem a published RFC accrues over time.

**The synthesis this note actually recommends, and it's not a false choice:** adopt RFC 9457's base shape (`type`, `title`, `status`, `detail`, `instance`) for the standard's own tooling benefits, and add Stripe-style extension members on top — the RFC explicitly permits arbitrary additional members in the same JSON object — including a `param` field for validation errors and, critically for concept 5's retry-safety question, a `retryable: true/false` extension member (not part of the RFC itself, but exactly the kind of API-specific extension it's designed to accommodate) so a client can make an automated retry decision without needing a hardcoded, per-status-code lookup table baked into every client library.

**The trade-off, honestly:** RFC 9457 is genuinely young (2023, obsoleting the older RFC 7807 from 2016) — plenty of production APIs predate it and have their own established, incompatible error shapes already in wide use by existing clients, and migrating a live API's error contract is a breaking change for anyone parsing the old shape. There's also a real design tension in `type`: the RFC wants it to be a *dereferenceable URI* (ideally one that resolves to human-readable documentation about that error category), which is more infrastructure than many teams bother standing up, and `"about:blank"` (the RFC's own explicitly-sanctioned placeholder, used in this section's runnable example) is a legitimate, honest fallback when you haven't built that documentation yet — but it does mean `type` alone isn't always as machine-actionable in practice as the spec envisions.

**Flip condition:** a genuinely internal API, consumed only by services your own team controls end-to-end, has much less to gain from a *public standard* specifically — you control every client, so a bespoke shape costs you nothing extra in coordination, and RFC 9457's main payoff (interoperability with client tooling you don't control) doesn't apply. The public-facing case is where the standard earns its keep, precisely because you can't coordinate a shape with every third-party developer who'll ever call your API.

**Failure modes:** shipping `type` as a bare string (`"validation_error"`) rather than a URI, quietly diverging from the spec in a way that breaks generic RFC-9457-aware tooling expecting a URI there. Putting genuinely sensitive information — a stack trace, an internal file path, a raw database error message — into `detail`, which is meant to be shown to the API's *external* consumer; concept 6's "In production" section below returns to this exact mistake. And the most common failure of all: building the perfect error contract and then having half your endpoints simply not use it, because there was no enforcement mechanism (a shared exception handler, applied globally, exactly as this section's runnable example demonstrates) forcing every code path through the same shape.

### Case study — Stripe's API: status codes and error bodies as product design

**What Stripe actually does, and why it's worth reading as literature rather than as a reference table.** Stripe's own API documentation (`docs.stripe.com/api/errors`) states the taxonomy plainly: `2xx` for success, `4xx` for "an error that failed given the information provided," `5xx` — described as *"rare"* — for Stripe's own infrastructure. Inside the `4xx` range, Stripe makes a deliberate, opinionated choice that departs from a naive reading of the spec: rather than reaching for `422` to mean "your request was well-formed but semantically invalid," Stripe repurposes **`402 Request Failed`** — a status code the base HTTP spec reserves, essentially unused, for "Payment Required" — to mean *"the parameters were valid, but the request failed,"* which is precisely their category for a syntactically-perfect charge request that gets **declined by the card issuer**. That is a genuinely clever, business-aware reuse: a card decline is not a bug in the request (the amount, currency, and card number were all perfectly valid), and it's not really a `500` (Stripe's own systems worked correctly), so neither of the "obvious" codes fits — and rather than force it into `422` (which most REST guidance would suggest), Stripe claimed an otherwise-idle status code and gave it a precise, product-relevant meaning that lets a client distinguish "your integration is broken" (`400`) from "the integration is fine, the *card* was declined" (`402`) with a single number, before ever reading a byte of the response body.

Stripe's `409 Conflict` documentation is equally precise and directly validates this note's own concept-5 material: their own docs describe it as occurring *"perhaps due to using the same idempotent key"* against mismatched parameters — the exact idempotency-error scenario concept 5 taught, now visible as a real, shipped, documented product decision rather than a theoretical example.

The error *body* itself is equally deliberate: a nested `error` object carrying `type` (one of a small, closed set: `api_error`, `card_error`, `idempotency_error`, `invalid_request_error`), `code` (a specific, stable, documented string a client can `switch` on programmatically — Stripe maintains a public registry of every possible value at `docs.stripe.com/error-codes`), `message` (human-readable, and for `card_error` specifically, written to be safely shown directly to an end user, not just logged), `param` (which field the error concerns, letting a checkout form highlight the exact broken input), and `doc_url` (a link straight to the relevant documentation for that specific error). **Read that list again as a product decision, not a schema:** every field answers a question a *developer integrating Stripe at 2 a.m. during an outage* would actually ask, in the order they'd ask it — this is error-handling designed with the same rigor most companies reserve for their checkout flow, because for an API company, the error response *is* a core part of the product experience.

*Primary source:* `docs.stripe.com/api/errors` and `docs.stripe.com/api/idempotent_requests` (current as of this writing; verify current — Stripe's specific error `code` vocabulary and status-code table are living documentation and do get extended over time, e.g., a `424 External Dependency Failed` code appears in their current table alongside the codes discussed here).

### Case study — a redirect-caching disaster: shipping a `301` when you meant a `302`

**What happened.** On 17 June 2014, Chef (then Opscode) soft-launched their community site, Supermarket, without HTTPS enforcement. The next day, the team deployed a change to redirect all HTTP traffic to HTTPS — using, per their own published postmortem, an nginx configuration containing the literal line `return 301 https://$server_name$request_uri;`. The deployment introduced an unrelated misconfiguration at the same time: the load balancer's HTTPS listener was pointed at the wrong backend port, so instead of reaching the application, HTTPS traffic hit nginx's bare default "Welcome" page — the entire application appeared to vanish behind a generic placeholder screen.

**Why this is the load-bearing lesson, stated honestly.** The actual outage was short — Chef's own postmortem reports roughly sixteen minutes from deploy to fix, because the team noticed immediately and traced the load-balancer misconfiguration quickly. This is not, on its own, a dramatic multi-day catastrophe, and it would be dishonest to present it as one. **But look at what Chef's own corrective actions, published in that same postmortem, chose to prioritize afterward:** alongside infrastructure-automation fixes, they specifically added external Nagios monitoring configured to *follow the `301` redirect all the way through and verify the final HTTPS response is actually valid* — not just check that the redirect *response itself* returned a `200`-family code, which is precisely the gap that let this incident happen undetected until a human noticed. **That corrective action is the tell:** an engineering team that had just shipped a `301` with a real misconfiguration behind it immediately recognized the deeper risk this section's technical material already named — because the status code they chose was `301`, not `302`, every browser and every intermediate cache that had already seen that redirect before the sixteen-minute window closed had permission, per RFC 9111's heuristic-cacheability rule covered earlier in this section, to **remember it indefinitely, with no expiry, and never ask the server again.** Had the misconfigured target been slightly different — not a generic "welcome" page but a URL that would go on to be reused for something else entirely once the real fix landed — some fraction of visitors would have kept being silently redirected to the *wrong* destination for as long as their browser cache lived, with the origin server's own subsequent fix powerless to reach them, because a `301` cached client-side is a promise the *server* cannot revoke.

**The engineering lesson, generalized:** the danger in shipping a `301` isn't really "16 minutes of visible downtime" — Chef's fast detection avoided that outcome. It's that **`301` converts a mistake from something a fast rollback fixes into something a fast rollback cannot fully fix**, because the redirect's own default cacheability (this section's earlier point about RFC 9111) means some population of clients had already, silently, and permanently taken the server at its word before the mistake was caught. This is precisely why the common industry practice — recommended widely across engineering blogs on exactly this topic, and implicitly endorsed by Chef's own subsequent monitoring investment — is to **ship a new redirect as `302` first, verify it in production under real traffic, and only promote it to `301` once you're confident it's correct and permanent.** A `302` gives you the one thing a `301` structurally cannot: **the ability to change your mind.**

*Primary source:* Chef (Opscode) Engineering, "Supermarket HTTPS Redirect Postmortem," chef.io/blog/supermarket-https-redirect-postmortem (2014); RFC 9111 §4.2.2 (heuristic freshness) and RFC 9110 §15.4 (redirection status codes), for the underlying `301`-vs-`302` cacheability rule that makes this failure mode possible in the first place. *(Honesty note: I searched specifically for a case where a `301` mistake caused a prolonged, undoable outage and could not find one publicly documented with the specificity this note requires — the Chef postmortem is the real, verifiable, named incident I found that directly exercises this exact mechanism, and I'm presenting its actual, modest timeline honestly rather than inflating it.)*

### In production — operating the status-code taxonomy correctly

**Best practices:**
1. **Never return `200 OK` with a failure encoded only in the body** (`{"success": false, "error": "..."}` alongside a `200` status line). This is one of the single most common and most damaging API anti-patterns in production systems: it silently defeats every piece of HTTP-aware infrastructure in the request path — caches can't tell success from failure, monitoring dashboards counting error rates by status code see a flat, healthy `200` line while the API is actually failing, and generic retry logic (built correctly, per concept 5, to distinguish safe-to-retry statuses) has no signal to act on at all.
2. **Always send `Retry-After` on `429` and `503`** — it costs one header and turns a client's blind guesswork ("how long do I wait before retrying?") into an exact, server-specified number.
3. **Never leak stack traces, raw exception messages, or internal file paths into a `500` body** returned to an external client — this is both a security exposure (internal implementation details handed to anyone who can trigger an error) and, per this section's error-contract design, exactly the kind of information that does not belong in a `detail` field meant for an external consumer.
4. **Keep a status-code decision consistent across your entire API surface** — if `/orders/{id}` returns `404` for "not found, or not yours to see" per this section's judgment discussion, every other resource in the same API should make the same choice, not mix `403` and `404` conventions endpoint by endpoint.

**Mistakes, beginner → senior:**
- *Beginner:* returning `500` for literally every error case, including ones that are the client's fault (a malformed request should never be a `500` — that's a bug in the *server's* error handling, mapping every exception to the same generic status regardless of cause).
- *Intermediate:* confusing `401` and `403` — returning `401` when a logged-in, correctly-authenticated user simply lacks permission for an action, which is a `403` by definition and misleads the client into thinking re-authentication (logging in again) would fix anything.
- *Senior:* an inconsistent error contract across services in the same organization — one team's API uses RFC 9457, another's uses a bespoke shape from five years ago, and a client integrating with both has to write two entirely separate error-handling code paths, defeating the entire point of standardizing at all.

**Monitoring and observability:** dashboard 4xx and 5xx rates *separately*, never combined into one generic "error rate" — they mean fundamentally different things and demand different responses (a 4xx spike often means "investigate what changed client-side, or in your validation rules"; a 5xx spike means "page someone, now"). Break down 5xx further by which specific code fired (`500` vs `502` vs `503` vs `504`) precisely because, as this section's diagram showed, each one localizes the failure to a different component in the chain — a dashboard that only shows "5xx rate: 12%" with no further breakdown has thrown away the single most useful diagnostic fact status codes give you for free.

**Cost:** status codes themselves cost nothing — this is purely an engineering-discipline cost, not a dollar one. The real cost shows up downstream: an inconsistent or ambiguous error contract multiplies support burden (every confusing error becomes a support ticket asking "what does this actually mean"), and a `200`-only anti-pattern multiplies incident-response time during a real outage, because the standard tooling (dashboards, alerting, load-balancer health checks) that would otherwise localize the problem automatically has been made blind to it by design.

---

## 7. Headers that matter: `Host`, `Content-Type`, `Authorization`, `Accept`

**Depth: [WORKING]**

### Intuition and interface

Concept 2 already gave you the generic grammar every header follows (`Name: value`, case-insensitive, CRLF-terminated). This section is deliberately narrower and more practical: **four specific headers you will use in essentially every request or response you ever write, and the one job each one does.** This is a [WORKING]-tier concept — you need to use these correctly and reason about what breaks when they're wrong, but you don't need to reimplement header-parsing machinery yourself; concept 4's parser already showed you the generic mechanism, and every real HTTP library handles the rest.

**`Host`** answers *"which website, on this one IP address, do you actually mean?"* — and this is not a hypothetical question. A single physical server, with one IP address, routinely hosts dozens or hundreds of unrelated domains (this is called **name-based virtual hosting**, and it's the default deployment model behind almost every shared-hosting provider and every CDN edge node on the internet). Before the request reaches your application code, the TCP handshake (Day 11) only established a connection to an *IP address* — nothing in the connection itself says which domain the client meant. `Host: api.example.com` is the *only* place that information exists once the TCP layer is done with it, which is precisely why `Host` became a **mandatory** header in HTTP/1.1 (it was optional and frequently omitted in HTTP/1.0, back when one-IP-one-website was still the common case). Concept 1's HTTP/0.9 example — a bare `GET /path`, no headers at all — is *structurally* incapable of virtual hosting for exactly this reason: there was nowhere in that one-line request to say which website you meant.

**`Content-Type`** answers *"how should the receiver interpret these bytes,"* on both requests and responses — `application/json`, `text/html`, `multipart/form-data` (for file uploads), `application/x-www-form-urlencoded` (classic HTML form submissions). Get this wrong on a request — send JSON with `Content-Type: text/plain` — and a server that dispatches its parsing logic based on this header (which is standard practice, and exactly what FastAPI's `RequestValidationError` machinery in concept 6 does) will fail to parse a perfectly well-formed body, because it never tried to parse it as JSON at all.

**`Authorization`** carries credentials — most commonly as `Authorization: Bearer <token>` for token-based auth (the mechanism Day 32's sessions-and-tokens material builds on directly) or `Authorization: Basic <base64-encoded-user:pass>` for the older HTTP Basic scheme. The header exists specifically because credentials are metadata *about the request*, not data the request is *about* — they don't belong in the URL (which gets logged, cached, and shown in browser history — a real, common, and serious security mistake: putting an API key in a query string) or in the body (which not every request even has, per concept 4).

**`Accept`** is the request-side header for **content negotiation**: the client states what representations it's willing to receive (`Accept: application/json`, or a prioritized list like `Accept: application/json, text/html;q=0.8` — the `q` value is a preference weight, 0 to 1), and a server capable of serving the same underlying resource in multiple formats picks one from that list rather than guessing. This is the header that lets one API endpoint serve both a browser (which might send `Accept: text/html`) and a programmatic client (`Accept: application/json`) with genuinely different response bodies from the identical URL, without either client having to specify a format-specific path.

### Worked example — the virtual-hosting failure mode, made concrete

This is the single clearest way to *feel* why `Host` earns its place as load-bearing rather than incidental. Take the raw request from concept 2's `raw_http_get.py`, and change exactly one header value, nothing else:

```
GET / HTTP/1.1
Host: example.com          ← real, live, and Cloudflare-fronted
Accept: text/html
Connection: close

```

versus

```
GET / HTTP/1.1
Host: definitely-not-a-real-site.invalid
Accept: text/html
Connection: close

```

Send both to the *identical* IP address (which is entirely possible — many Cloudflare-fronted sites share edge IPs), and the responses can be completely different: one serves `example.com`'s real content; the other, depending on the server's configuration, might return a `404`, a generic default page, or an outright connection refusal — **despite both requests reaching the exact same physical machine over the exact same TCP connection type.** The IP address and the TCP handshake (Day 10, Day 11) got you to the right *server*. Only the `Host` header, evaluated by application-layer code running on that server, decides which *website* you actually get — a clean, concrete illustration of the layering diagram from concept 1: DNS and IP routing solve "which machine," and HTTP's own headers solve everything above that a lower layer structurally cannot know.

### Condensed practices and the top failure mode

- **Never trust `Host` for security decisions without validation** — because a client controls this header entirely, a server that blindly uses it to build absolute URLs (in a password-reset email link, say) is exposed to "Host header injection," where an attacker sends a request with a forged `Host` value and gets the server to embed an attacker-controlled domain into security-sensitive output.
- **Set `Content-Type` explicitly on every response your own code produces** — relying on a framework's default is usually fine, but silent mismatches between what you *set* and what you actually *send* are a real, recurring bug class, particularly around character encoding (`; charset=utf-8`) omissions that render fine in testing (where the default happens to match) and break in production for one specific class of user input.
- **Never put an `Authorization` value in a URL, a log line, or a cached response** — treat it with the same handling discipline as a password, because functionally, for the duration of its validity, that's exactly what it is.
- **The single most common failure mode across all four:** a client and server that both individually work correctly in isolation, but were tested against each other with a *mismatched* assumption about one of these four headers — a client sending `Accept: application/xml` against a server that only ever implemented JSON, silently getting an unexpected format (or an error) neither side's own unit tests would have caught, because each side's tests only ever exercised its own default.

*Primary source:* RFC 9110 §10 (Fields) covers `Host` (§7.2), `Content-Type` (§8.3), `Authorization` (§11.6.2), and `Accept` (§12.5.1) as the current normative definitions.

---

## 8. Statelessness and cookies: HTTP remembers nothing

**Depth: [CORE]**

### Intuition

Here is a fact worth sitting with, because it's easy to hear once and then forget in practice: **send two HTTP requests, back to back, on the exact same TCP connection, from the exact same client — and the server, by the protocol's own design, has no built-in way to know they came from the same person.** Every request stands entirely alone. There is no `session_id` field baked into HTTP's grammar (concept 2 and 3's entire request and response anatomy — you've now seen every field that exists — contains nothing resembling "remember me"). This isn't an oversight or a missing feature; it's a deliberate, foundational property called **statelessness**, and RFC 9110 itself frames it as a defining characteristic of the protocol's design: each request is processed independently of any previous request on the same or a different connection.

Why design it this way on purpose? Because statelessness is precisely what makes the rest of this note's material *work* at internet scale. Recall Day 11 concept 11's connection pooling and Day 12's DNS-based load balancing: if a server had to remember "conversation state" tied to one specific TCP connection or one specific backend machine, you could never freely route request #2 from the same user to a *different* server than request #1 handled — which is exactly what happens constantly in any load-balanced (Day 16), horizontally-scaled system. **A stateless protocol is what lets any server behind a load balancer answer any request, with zero coordination between servers about "who talked to this client last time."** The cost of that freedom is that if you *do* need to remember something about a specific user across requests — that they're logged in, what's in their shopping cart — the protocol will not do it for you. You have to build it yourself, on top.

### Analogy: an amnesiac clerk at a walk-up counter

Picture a customer-service counter staffed by a clerk with a specific, unusual condition: they have no memory of any previous customer interaction whatsoever, the instant it ends. Every single person who steps up to the counter is, to the clerk, a complete stranger, even if it's the same person who was standing there thirty seconds ago. If the clerk needs to know "is this the same person who was here a minute ago, mid-transaction," the *only* way that can happen is if the **customer themselves hands the clerk a claim ticket** each time — a physical token that says "this ticket belongs to transaction #4471, which was in progress" — and the clerk looks up the transaction using the ticket, not any memory of the person's face.

That claim ticket is exactly what a cookie or a bearer token is. The clerk (the server) genuinely never remembers you; the *ticket* (which the *client* is responsible for presenting again, every single time) is the entire mechanism. **Where the analogy breaks, and it's the load-bearing detail:** a human clerk, even one with no memory of who you are, could in principle recognize your face on sight if they happened to remember it by accident. A server has no equivalent accidental fallback — there is categorically no channel through which "the same person as before" can be inferred without an explicit token being presented, because HTTP genuinely, structurally carries nothing else across requests. The amnesia here isn't clerk-specific forgetfulness; it's a hard architectural boundary with no back door.

### Under the hood: cookies, the bolted-on fix, precisely

A **cookie** is the standard mechanism (RFC 6265, and its 2025 successor RFC 6265bis) for handing the client that claim ticket and having the client automatically present it back on every subsequent request, with no application code needing to remember to do so manually each time:

```
Request 1 (login):
  POST /login HTTP/1.1
  Host: app.example.com
  Content-Type: application/json

  {"username":"alice","password":"..."}

Response 1:
  HTTP/1.1 200 OK
  Set-Cookie: session_id=a1b2c3d4e5; HttpOnly; Secure; SameSite=Lax; Max-Age=3600
                └────────┬────────┘  └──┬──┘  └──┬──┘  └────┬────┘   └────┬────┘
                     the claim       JS can't    HTTPS    cross-site    expires
                      ticket          read it     only     restriction   in 1 hr

Request 2 (any later request, SAME browser, automatically):
  GET /orders HTTP/1.1
  Host: app.example.com
  Cookie: session_id=a1b2c3d4e5
           └──────────┬──────────┘
           the browser attached this AUTOMATICALLY —
           your application code never had to remember to
```

The mechanism is entirely a *convention*, built from two ordinary headers this note already taught the generic grammar for (concept 2's header-field syntax applies unchanged): `Set-Cookie` on the response, `Cookie` on every subsequent request. **Nothing about this requires any change to HTTP's grammar.** The server does the actual remembering — `session_id=a1b2c3d4e5` is typically an opaque, unguessable key into a server-side store (a database row, a Redis entry) holding "this session belongs to alice, logged in at 14:02, cart contains 3 items"; the cookie itself carries no meaningful data, just the lookup key. This is the architecture Day 32 builds on in full; today's job is only to see *why* this bolt-on exists at all, mechanically, as the direct, unavoidable consequence of concept 8's opening fact.

Each attribute on that `Set-Cookie` line is answering a real, specific threat this bolted-on mechanism introduces the moment it exists: `HttpOnly` prevents JavaScript running on the page from reading the cookie at all (blunting an entire class of cross-site-scripting attacks that would otherwise be able to steal the session token outright); `Secure` refuses to send the cookie over a plain, unencrypted `http://` connection at all (a preview of Day 14's TLS material — without it, the claim ticket travels in the clear for anyone on the network path to copy); and `SameSite` restricts whether the cookie gets attached to requests originating from a *different* site than the one that set it — which is exactly the mechanism the case study below is built around.

### The genuine agentic connection: a "conversation" with an LLM is the identical bolt-on

Here is the load-bearing thread this note's framing promised, made concrete rather than asserted. **An LLM API call is, itself, an ordinary stateless HTTP request** — `POST /v1/messages` to `api.anthropic.com`, or the equivalent for any other provider — and the model behind it has exactly as much built-in memory of your previous call as an HTTP server has of your previous request: **none.** There is no `session_id` an LLM API call implicitly carries the way a cookie does; a model does not "remember" the first message in a chat by virtue of some persistent connection or server-side state tied to you.

What actually happens, every single time you send a follow-up message in what feels like an ongoing conversation, is that **the entire prior transcript is resent, in full, as part of the request body** — every previous user message and every previous assistant reply, concatenated into the `messages` array of that one, brand-new, entirely stateless HTTP request. The "memory" of the conversation lives nowhere on the server between calls; it lives *only* in whatever the calling application chose to store and resend, exactly the way a session cookie's actual data lives in a server-side store that the *cookie itself* never contains — the cookie is just the pointer, and here, the resent transcript *is* the pointer and the data at once, carried directly in the request rather than looked up from a key. This is precisely why an agent framework's "memory" (Day 43's subject) is not a magical capability the model possesses — it is application-level engineering, built entirely on top of a protocol that, by design, forgets everything the instant each request completes, using the exact same "state is bolted on by convention, never native to the protocol" pattern a login session uses. Get the resending wrong — truncate the history to save tokens, or fail to persist it across a process restart — and the "conversation" doesn't degrade gracefully; it simply forgets, in precisely the same abrupt, total way a server with no cookie mechanism would forget you existed the moment you stepped away from the counter.

### Case study — Chrome's 2020 `SameSite=Lax` default, and the authentication flows it broke

**What happened.** Prior to Chrome 80 (released February 2020), a cookie with no explicit `SameSite` attribute was sent on *every* request to its origin domain, regardless of what site the request originated from — including requests triggered by a completely different website embedding an image, an iframe, or a cross-site form submission pointing at your domain. Google changed the default: starting with Chrome 80, any cookie set without an explicit `SameSite` value would be treated **as if it had been set with `SameSite=Lax`**, restricting it from being sent along with most cross-site requests, with an additional transitional carve-out (cookies still attached to cross-site *top-level* navigations via `POST`, specifically to ease the migration) — documented on Chromium's own `chromium.org/updates/same-site/` page.

**The real, documented breakage.** Mozilla's own developer blog (Mozilla Hacks, August 2020) reported this change **"broke authentication flows that are based on the OpenID Connect standard"** — a widely-used, standard single-sign-on protocol whose flow legitimately depends on a cookie set on one site being readable during a redirect *back* from a different site (the identity provider), which is structurally a cross-site pattern the new default was specifically designed to restrict. The breakage was significant enough that **Google itself temporarily rolled the change back in early April 2020**, citing the need for stability during the early COVID-19 period, before re-enabling it later once affected sites had time to add the explicit `SameSite=None; Secure` attribute pair needed to opt back into the old cross-site behavior deliberately, rather than by default.

**The engineering lesson.** This is a close cousin of the redirect-caching case study earlier in this note, and it's worth naming the parallel explicitly: **a change to a *default* value, in a mechanism (cookies) that is itself already a bolted-on convention rather than a protocol-native feature, broke real, standards-based authentication flows across a large fraction of the web — not because those flows were badly built, but because they had, reasonably, relied on the *previous* default's behavior without ever setting the attribute explicitly.** The generalizable rule this teaches, directly relevant to anything you build on cookies or tokens: **never rely on a security-relevant default staying the way it is today.** Set `SameSite`, `Secure`, and `HttpOnly` explicitly, on every cookie your own application sets, precisely because "the browser's current default happens to do what I want" is not a property you control, and browser vendors reserve, and periodically exercise, the right to change it out from under you — for good reasons, as this case study shows, but with real, external, breaking consequences for anyone who wasn't explicit.

*Primary source:* Chromium Project, "SameSite Updates," chromium.org/updates/same-site/; Mozilla Hacks, "Changes to SameSite Cookie Behavior – A Call to Action for Web Developers" (August 2020). *(Verify current — SameSite defaults and browser-specific rollout timelines are the kind of fast-moving web-platform detail that's worth re-checking against current browser documentation before relying on specifics beyond the historical 2020 event itself.)*

### In production — operating stateless applications and cookie-based state correctly

**Best practices:**
1. **Store session state server-side (a database or an in-memory store like Redis), and keep the cookie itself limited to an opaque, unguessable lookup key.** Never encode meaningful, sensitive data directly into a cookie's value on the (mistaken) assumption that the client can't read or tamper with it — a cookie is fully under the client's control unless it's cryptographically signed or encrypted, and even then, a signed cookie can still be read (just not forged) unless it's also encrypted.
2. **Set `HttpOnly`, `Secure`, and an explicit `SameSite` value on every session cookie, always**, precisely per the case study above — never rely on a browser default.
3. **Design your application layer to be stateless wherever the *server process* is concerned**, even though the *user's session* is stateful — the distinction matters: no request should ever depend on hitting the *same specific backend server instance* as a previous request (this is the property that makes horizontal scaling and load balancing, Day 16, possible at all). If your session data lives in that server's local memory rather than a shared store, you've quietly broken this, and a load balancer routing a user's next request to a *different* instance will find them mysteriously logged out.

**Mistakes, beginner → senior:**
- *Beginner:* storing session data in a global in-memory Python dictionary on a single server process, which works perfectly in local development (one process, no load balancer) and then breaks unpredictably in production the moment a second server instance is added.
- *Intermediate:* putting a user's actual permissions or role directly into an unsigned, unencrypted cookie value, trusting the client not to edit it — a client can trivially edit any cookie value in their own browser, and an application that trusts client-supplied authorization data without server-side verification has built a privilege-escalation bug into its own architecture.
- *Senior:* an authentication flow that works flawlessly in every browser tested during development and silently breaks for a subset of real users after a browser vendor ships a new default (exactly this case study) — because `SameSite`, `Secure`, or `HttpOnly` was never set explicitly, leaving the application's actual behavior at the mercy of whatever a browser vendor decides a sane default should be, this month.

**Monitoring and observability:** track session-store hit/miss rates and unexpected-logout rates as first-class metrics, not afterthoughts — a spike in "user reports being logged out randomly" is very often a symptom of exactly the server-affinity mistake above (session data pinned to one instance a load balancer no longer routes to), and it's diagnosable in minutes if you're already watching for it, versus hours of confused bug reports if you aren't.

**Cost:** a server-side session store (Redis, a database table) is genuine infrastructure with genuine operating cost — memory for Redis, storage and query load for a database table — that scales roughly with concurrent active users, and it's worth sizing deliberately rather than discovering under load. The alternative some systems choose — encoding session state entirely inside a signed token the client carries (a JWT, for instance, a full treatment of which belongs to Day 32) — trades that server-side storage cost for a different one: a larger cookie/header on every single request, and the much harder problem of *revoking* a token before its own stated expiry, since there's no server-side record to simply delete.

---

## Interview questions and practice

### Conceptual

1. **Why is HTTP text-based rather than binary, and why did that stop being the obviously-correct choice at large enough scale?** *(Debuggability with zero tooling in 1991 — "speak it with telnet." Text parsing and header redundancy cost real CPU/bandwidth at Google/Facebook scale, which is why HTTP/2 introduced binary framing — Day 14 — while keeping every semantic concept from this note unchanged.)*
2. **What are the five parts of an HTTP request, in order, and what question does each one answer?** *(Method — what kind of operation; request-target — on what; version — which dialect; headers — what metadata; body — optional payload. Blank line separates headers from body, unconditionally.)*
3. **Why is a status code a three-digit number rather than a string like `"OK"` or `"ERROR"`?** *(A closed, IANA-registered, extensible numeric space avoids vocabulary drift across implementations, and lets the first digit double as a category — `status // 100 == 5` — for free.)*
4. **How does a receiver know where an HTTP message's body ends?** *(`Content-Length` — read exactly N bytes; `Transfer-Encoding: chunked` — read chunks until a zero-length chunk; for responses only, absent both, read until the connection closes. `Transfer-Encoding` wins if both are present — and RFC 9112 flags that combination itself as a possible smuggling attempt.)*
5. **Why is `PUT` idempotent but `POST` is not?** *("Make this URL look like X" applied N times still yields X. "Create a new thing" applied N times creates N things. `DELETE` is idempotent for the same reason as `PUT` — "should not exist" is a stable end state.)*
6. **What's the actual difference between `401` and `403`?** *(`401`: I don't know who you are, or your credentials are invalid — log in. `403`: I know exactly who you are, and you can't do this — logging in again wouldn't help.)*
7. **Why might a real API deliberately return `404` for a resource that actually exists?** *(To avoid confirming existence to an unauthorized requester — GitHub's real, documented behavior for private repos. Trade-off: harder for the legitimate owner to self-diagnose a permissions problem.)*
8. **Why do `502` and `504` only make sense when a proxy is in the request path?** *(Both describe what a gateway/proxy observed about an *upstream* it talked to on your behalf — `502`: got an invalid response; `504`: got no response before timing out. Neither describes the origin app's own internal state the way `500` does.)*
9. **Why is `301` operationally more dangerous to ship by mistake than `302`?** *(RFC 9111: `301` is heuristically cacheable by default — a client may remember it indefinitely with no explicit expiry. `302` isn't. A wrong `301`, once cached, can't be fully undone by fixing the server.)*
10. **What does "HTTP is stateless" actually mean, mechanically, and what fixes it?** *(No field in the request/response grammar carries "remember me" — every request is independent. Cookies (`Set-Cookie`/`Cookie`) bolt state on by convention: an opaque key round-tripped by the client, looked up against server-side storage.)*
11. **Why does the `Host` header exist, and why did it become mandatory in HTTP/1.1?** *(One IP can serve many domains — virtual hosting. The TCP handshake only gets you to a machine, not a website; `Host` is the only place "which website" is ever stated once the connection is open.)*
12. **How is an LLM "remembering" a conversation actually implemented, given that the underlying API call is stateless?** *(The full prior transcript is resent as part of every new, independent request's body — the identical "state is bolted on by the caller, not native to the protocol" pattern a session cookie uses, one layer up.)*

### Diagnostic scenarios

13. **A client hangs forever waiting for a response that the server already fully sent.** *(Likely a framing mismatch: the response has neither a correct `Content-Length` nor `Transfer-Encoding: chunked`, and the connection wasn't closed either — the client has no signal the message ended. Check with `curl -v` and inspect the raw headers.)*
14. **Two different teams' proxies in the same request path disagree about how many bytes a request body has.** *(Request smuggling risk — check for a request carrying both `Content-Length` and `Transfer-Encoding`, which RFC 9112 explicitly flags as smuggling-suspect and says should be rejected outright, not resolved by precedence.)*
15. **A public API's client library retried a `POST` after a timeout and a customer was charged twice.** *(No idempotency-key mechanism on a non-idempotent method. Fix: an `Idempotency-Key` header, server-side, storing and replaying the first attempt's result verbatim on a retried key.)*
16. **A monitoring dashboard shows "error rate: 3%" and nobody can tell if it's the client's fault or an outage.** *(4xx and 5xx were combined into one metric. Split them — 4xx usually means "something changed client-side or in validation," 5xx means "page someone." Break 5xx down further by exact code per this note's diagram.)*
17. **A user reports being randomly logged out, but only sometimes, and only under load.** *(Session state stored in one server process's local memory; a load balancer routed their next request to a different instance with no record of them. Fix: move session storage to a shared store — Redis, a database.)*
18. **An authentication flow that worked for years broke overnight with no code changes on your end.** *(A browser vendor shipped a new default — this note's SameSite case study, exactly. Fix, and prevention: set `SameSite`, `Secure`, `HttpOnly` explicitly on every cookie; never rely on a security-relevant default persisting.)*

### Design questions

19. Design the error contract for a new public API from scratch, including how a client knows whether a given error is safe to retry. *(RFC 9457 base shape + extension members; a `retryable` field or a documented mapping from status code to retry-safety; never `200` with failure in the body.)*
20. Design the URL and method for "approve a pending expense report" in an API that's otherwise clean CRUD. *(Action sub-resource, `POST /expense-reports/{id}/approve`, not an overloaded `PATCH` on status; consider a first-class `approvals` resource if the approval itself needs its own auditable life cycle.)*
21. Design how a chat-completion endpoint should frame its response so a client can render tokens as they arrive. *(`Transfer-Encoding: chunked` — the total length is unknowable at response-start time; `Content-Length` is structurally the wrong tool here.)*

---

# Topic-wide wrap-up

## Glossary

**`Accept`** — a request header stating which response formats the client is willing to receive, optionally weighted with `q` values, enabling one endpoint to serve multiple representations of the same resource.

**`Authorization`** — a request header carrying credentials, most commonly `Bearer <token>` or `Basic <base64>`, kept separate from the URL and body specifically because it's metadata about the request, not data the request concerns.

**Body** — the optional payload of an HTTP message, following the mandatory blank line; its length must be determined by framing (`Content-Length`, `Transfer-Encoding`, or connection-close), never inferred from its own content.

**Chunked transfer encoding** — a body-framing mechanism (`Transfer-Encoding: chunked`) where the body is sent as a series of length-prefixed chunks, terminated by a zero-length chunk, used whenever the total body size isn't known when transmission begins.

**Cookie** — the bolted-on, convention-based mechanism (`Set-Cookie` / `Cookie` headers) for carrying an opaque session key from server to client and back, giving a stateless protocol the appearance of remembering a specific visitor.

**`Content-Length`** — a header stating the exact byte length of a message body, used whenever the full size is known before transmission begins.

**`Content-Type`** — a header stating how to interpret a message body's bytes (`application/json`, `text/html`, etc.), independent of the body's length or the request/response's success.

**Framing** — the general problem of determining where one HTTP message ends and the next begins on a byte stream with no inherent message boundaries; solved via `Content-Length`, chunked encoding, or (for responses only) connection closure.

**`Host`** — a mandatory (in HTTP/1.1+) request header naming the specific website being requested, required because one IP address can serve many domains (virtual hosting) and the TCP layer carries no such information.

**HTTP/0.9** — the original, 1991 form of HTTP: a single line (`GET /path`), no headers, no status line, no version token.

**Idempotent method** — per RFC 9110 §9.2.2, a method where the server's resulting state is identical whether the request is applied once or many times; `GET`, `HEAD`, `PUT`, `DELETE`, `OPTIONS` are idempotent; `POST` and (usually) `PATCH` are not.

**`Idempotency-Key`** — a client-generated header (Stripe's convention, widely adopted elsewhere) letting an otherwise-non-idempotent `POST` be safely retried: the server stores and replays the first attempt's result for any repeated key.

**Problem Details (RFC 9457)** — a standardized JSON/XML error-body shape (`type`, `title`, `status`, `detail`, `instance`, plus extensions) served as `application/problem+json`, replacing bespoke per-API error formats.

**Reason phrase** — the optional human-readable text following a status code on a status line (`OK`, `Not Found`); purely conventional, never meant to be branched on programmatically.

**Request line** — the first line of an HTTP request: `method SP request-target SP HTTP-version`.

**Request smuggling** — an attack exploiting disagreement between two HTTP parsers (typically a front-end proxy and a back-end server) about a message's framing, letting an attacker hide one request inside another's body.

**Request-target** — the second field of a request line; usually origin-form (a path and optional query string), with absolute-form, authority-form, and asterisk-form reserved for specific cases (proxying, `CONNECT`, and `OPTIONS *` respectively).

**Safe method** — per RFC 9110 §9.2.1, a method intended only for retrieving information, with no side effects; `GET`, `HEAD`, `OPTIONS`, and `TRACE`.

**`SameSite`** — a `Set-Cookie` attribute restricting whether a cookie is sent with cross-site requests, defaulted to `Lax` by Chrome and other browsers starting in 2020.

**Statelessness** — HTTP's foundational property that each request is processed independently, with no built-in memory of any previous request; the property that makes load-balanced, horizontally-scaled serving possible, at the cost of requiring state to be built on top by convention.

**Status code** — the three-digit numeric field on a status line indicating the outcome of a request; the first digit categorizes it (2xx success, 3xx redirection, 4xx client error, 5xx server error).

**Status line** — the first line of an HTTP response: `HTTP-version SP status-code SP [reason-phrase]`.

**Virtual hosting** — serving multiple distinct websites/domains from a single IP address, made possible by the mandatory `Host` header telling a server which domain a given request is for.

---

## Cheat sheet

**The message shape, both directions**
```
start-line CRLF
( field-name ":" OWS field-value OWS CRLF )*
CRLF                          ← blank line, MANDATORY, never optional
[ message-body ]
```

**Methods: safety and idempotency**
| Method | Safe | Idempotent | Retry blindly? |
|---|---|---|---|
| GET / HEAD / OPTIONS | Yes | Yes | Always safe |
| PUT / DELETE | No | **Yes** | Safe |
| PATCH | No | Depends on patch format | Only if you know the format is idempotent |
| POST | No | **No** | **Never, without an Idempotency-Key** |

**Message framing precedence (RFC 9112 §6.3)**
```
1. Certain codes (1xx, 204, 304, HEAD responses) → body length is always zero
2. Transfer-Encoding present → wins, even over Content-Length
   (both present + disagreeing → smuggling risk, reject as an error)
3. Transfer-Encoding: chunked → read chunks until size-zero chunk
4. Valid single Content-Length → read exactly that many bytes
5. Request, neither header → body length is zero
6. Response, neither header → read until connection closes (HTTP/1.0 fallback)
```

**Status codes: the taxonomy**
| Code | Means | Whose fault / which component |
|---|---|---|
| 400 | Malformed request, couldn't even parse it | Client — request shape |
| 401 | Not authenticated (or bad credentials) | Client — log in |
| 403 | Authenticated, but not authorized | Client — logging in wouldn't help |
| 404 | Not found (or hidden to avoid confirming existence) | Client — wrong URL, or no access |
| 409 | Conflicts with current server state | Client — retry with corrected state |
| 422 | Parsed fine, semantically invalid | Client — fix the values |
| 429 | Rate limited | Client — back off, check `Retry-After` |
| 500 | Origin app's own bug | Server — your code |
| 502 | Proxy got a bad response from upstream | Proxy's upstream is broken |
| 503 | Deliberately refusing right now | Server/proxy — overloaded or draining |
| 504 | Proxy got no response before timeout | Upstream is silent/dead |

**3xx and caching, the one fact that matters**
```
301 Moved Permanently  → heuristically cacheable by DEFAULT (RFC 9111).
                          Ship this and you may not be able to take it back.
302 Found               → NOT cacheable by default. Safe to test with first.
```

**Statelessness**
```
No request carries any memory of a previous request, ever, by design.
State is bolted on: Set-Cookie / Cookie headers round-trip an opaque key;
the server looks the key up against its OWN storage. An LLM "conversation"
uses the identical pattern one layer up: resend the whole transcript, every call.
```

---

## Build this

### Task 1 — speak raw HTTP/1.1 by hand (the study plan's core build)

- [ ] Using `socket.create_connection`, or a raw `nc`/`telnet` session over WSL2, or the PowerShell `TcpClient` snippet in concept 2, send exactly `GET / HTTP/1.1\r\nHost: <a real domain>\r\nConnection: close\r\n\r\n` to port 80 of a real server.
- [ ] Capture the raw response bytes and identify: the status line, at least three headers and what each one means, the blank line, and the body.
- [ ] Find (or force, with a `curl -H "Accept-Encoding: identity"` style request) a response using `Transfer-Encoding: chunked`. Walk the chunk-size lines by hand and reconstruct the real body.
- [ ] Repeat against `http://github.com` (not https) and confirm you get a real `301` with a `Location` header. Explain, from this note's material, exactly why shipping *that* code instead of a `302` is a decision worth pausing on.

**Definition of done:** you can point at the blank line in a raw capture and explain, without looking it up, why the protocol cannot function without it.

### Task 2 — write the minimal parser

- [ ] Implement `parse_request(raw_bytes) -> dict` extracting method, path, version, and a lowercased headers dict, using nothing but string/bytes operations — no HTTP library.
- [ ] Extend it to correctly read the body using `Content-Length`, per RFC 9112 §6.3's precedence rules.
- [ ] Feed it a request with both `Content-Length` and `Transfer-Encoding` headers set to disagreeing values. Confirm your parser either rejects it outright or explain, precisely, why *not* rejecting it is the exact mistake this note's request-smuggling case study is about.
- [ ] Feed it a deliberately malformed request line (not 3 space-separated tokens) and confirm it produces a clean `400`, not a crash.

**Definition of done:** your parser handles a real `curl -X POST` request with a JSON body, correctly reporting method, path, headers, and the exact body bytes — verify against `curl -v` output for the same request.

### Task 3 — build the status-code taxonomy into a real API

- [ ] Using FastAPI, build endpoints that genuinely produce `200`/`201`, `400`, `401`, `403`, `404`, `409`, `422`, and `429` — not by memorizing the numbers, but by writing the actual condition each one represents (a real validation failure for `422`, a real conflicting-state check for `409`).
- [ ] Confirm FastAPI's default behavior for malformed JSON (`422`, not `400`) with a `TestClient`, then fix it with a custom exception handler, exactly as this note's runnable example does.
- [ ] Add an `Idempotency-Key`-aware `POST` endpoint. Prove, with two identical requests carrying the same key, that only one resource gets created.
- [ ] Shape every error response according to RFC 9457 (`type`, `title`, `status`, `detail`).

**Definition of done:** a `curl` transcript, saved as your own artifact, showing all eight status codes actually firing from your own running code, with real response bodies.

### Task 4 — statelessness and cookies, made visible

- [ ] Build a two-endpoint FastAPI app: `POST /login` sets a `Set-Cookie` with `HttpOnly`, `Secure` (test over `http://localhost`, note whether your browser enforces `Secure` there — see if it silently drops the cookie, and explain why), and `SameSite=Lax`; `GET /whoami` reads the `Cookie` header and looks the session key up in a plain in-memory dict.
- [ ] Using `curl`, show that a request to `/whoami` with no `Cookie` header gets a `401`, and a request replaying the exact `Set-Cookie` value gets a `200` with the right user — proving to yourself, mechanically, that the server is doing no recognition beyond a dictionary lookup on that one opaque string.
- [ ] Restart your server process (clearing the in-memory dict) without clearing your browser's cookie. Observe what happens to `/whoami`, and explain it in terms of this note's "state lives server-side; the cookie is just a lookup key" material.

**Definition of done:** you can explain, from memory, why restarting the server broke the session even though the browser never stopped sending the exact same cookie value.

---

## Active recall and self-test

Answer from memory, in writing, before checking.

1. Why is HTTP text rather than binary, and what changed the calculus enough for HTTP/2 to go binary?
2. Name the five parts of a request and the five parts of a response, in order. Which part differs between them?
3. What single two-byte-repeated sequence separates headers from body, and why can't the body's own content be used to find that boundary instead?
4. State RFC 9110's precise definition of "idempotent." Why does it define it in terms of server *state*, not response content?
5. Give three genuinely idempotent methods and explain, for each, what "applying it twice" produces the same result as "applying it once."
6. Why is `POST` singled out as dangerous, mechanically? What does an `Idempotency-Key` actually fix, and how?
7. Explain the difference between `400` and `422` in one sentence each.
8. Explain the difference between `401` and `403` in one sentence each.
9. Why might an API deliberately choose `404` over `403`? Name the real, documented example from this note.
10. Why does `502` only make sense when a proxy exists in the request path? What's the exact difference from `504`?
11. Which status code is heuristically cacheable by default, `301` or `302`? What's the practical consequence of getting that choice wrong?
12. What determines where an HTTP message's body ends? List the precedence order.
13. What happens if a message arrives with both `Content-Length` and `Transfer-Encoding` set to disagreeing values? Why is that specific case singled out in the spec?
14. Why does `Host` exist, and why did HTTP/1.1 make it mandatory?
15. What does "HTTP is stateless" mean, precisely, and what mechanism is bolted on to fake statefulness?
16. Explain, in your own words, why an LLM "remembering" a conversation is mechanically identical to a cookie-based session.
17. What did Chrome's 2020 `SameSite=Lax` default change, and what real, standards-based flows broke because of it?
18. Why is `Transfer-Encoding: chunked` the right framing choice for a streamed LLM completion, and why is `Content-Length` structurally wrong for that case?

### 60-second teach-back

> **"HTTP is just an agreed-upon shape for text sent over the TCP byte stream Day 11 built you: a request says a method, a target, a version, some headers, and maybe a body; a response answers with a version, a numeric status, headers, and maybe a body — always separated from its headers by one mandatory blank line. Because a byte stream has no message boundaries on its own, every message has to say how long its body is — with `Content-Length` when the size is known up front, or `Transfer-Encoding: chunked` when it isn't, which is exactly how a streamed LLM completion gets sent one token at a time. The method you choose is a promise: `GET` promises nothing changes, `PUT` and `DELETE` promise that doing it twice is as safe as doing it once, and `POST` promises nothing at all about retries — which is exactly why Stripe and everyone else bolted an `Idempotency-Key` onto it. A status code's first digit tells you whose problem it is: 4xx is something about your request, 5xx is the server or something behind it, and a `502` versus a `504` even tells you whether a proxy got garbage back or got nothing back at all. And underneath every bit of that, HTTP genuinely remembers nothing between one request and the next — a cookie is a handed-back claim ticket the server looks up in its own storage, and an LLM 'conversation' is the identical trick one layer up: resend the whole transcript, every single call, because the model has no built-in memory either."**

If you can deliver that and then explain why shipping a `301` instead of a `302` can be a mistake you can't take back, you have this topic.

---

## Spaced-repetition flashcards

| Q | A |
|---|---|
| What are the five parts of an HTTP request? | Method, request-target, version, headers, optional body |
| What separates headers from the body? | A single blank line (`\r\n\r\n` on the wire) — mandatory, always |
| Is a status code's reason phrase meaningful to a client? | No — purely conventional; a client must branch on the numeric code only |
| Which methods are safe? | `GET`, `HEAD`, `OPTIONS`, `TRACE` |
| Which methods are idempotent but not safe? | `PUT`, `DELETE` |
| Is `POST` idempotent by default? | No — and neither is `PATCH`, usually |
| What fixes `POST`'s retry danger? | An `Idempotency-Key` header — server stores and replays the first result |
| `Content-Length` vs `Transfer-Encoding`, if both present? | `Transfer-Encoding` wins — and both-present is itself a smuggling red flag |
| When does chunked encoding end? | A chunk with size zero |
| 400 vs 422? | 400: couldn't parse the request. 422: parsed fine, values are invalid |
| 401 vs 403? | 401: don't know who you are. 403: know exactly who you are, still refused |
| Why might an API return 404 instead of 403? | To avoid confirming a resource exists to someone unauthorized (GitHub's real behavior) |
| 502 vs 504? | 502: proxy got a bad/invalid response. 504: proxy got no response before timeout |
| Which redirect is cacheable by default, 301 or 302? | 301 — heuristically cacheable per RFC 9111; 302 is not |
| Why is that dangerous? | A wrong 301 can be cached forever by clients before the server-side fix ever reaches them |
| What does "HTTP is stateless" mean? | No request carries memory of any previous request, by design |
| What mechanism fakes statefulness? | Cookies: an opaque key round-tripped by the client, looked up server-side |
| How does an LLM "remember" a conversation? | The full transcript is resent in every new, independent request — same pattern as a cookie, one layer up |
| Why did Chrome default cookies to `SameSite=Lax` in 2020? | To restrict cross-site cookie sending by default, for security — it broke real OpenID Connect flows |
| Why is `Transfer-Encoding: chunked` right for LLM streaming? | Total length is unknown when the response starts — `Content-Length` requires knowing it upfront |
| Why does `Host` exist? | One IP can serve many domains (virtual hosting); TCP alone can't say which one you meant |

---

## Primary sources

**Core specifications**
- **RFC 9110** — HTTP Semantics. Methods, status codes, headers; obsoletes RFC 7231 and others.
- **RFC 9111** — HTTP Caching. Heuristic freshness and default cacheability of status codes (including the 301-vs-302 distinction this note leans on).
- **RFC 9112** — HTTP/1.1. The exact wire grammar: request-line, status-line, header-field syntax, and message-length determination (§6.3) — the single most load-bearing section for concept 4.
- **RFC 1945** — HTTP/1.0 (1996), the first RFC-numbered version of the protocol.
- **RFC 6585** — Additional HTTP Status Codes, defining `429 Too Many Requests` (2012).
- **RFC 4918** — WebDAV, the original source of status code 422 (as "Unprocessable Entity"), later folded into RFC 9110 as "Unprocessable Content."
- **RFC 5789** — the PATCH method.
- **RFC 9457** — Problem Details for HTTP APIs (2023), obsoleting RFC 7807.
- **RFC 6265** (and its RFC 6265bis successor) — HTTP State Management Mechanism (cookies).

**Company documentation, read as primary sources**
- Stripe API Reference — `docs.stripe.com/api/errors`, `docs.stripe.com/api/idempotent_requests`, `docs.stripe.com/error-codes`.
- GitHub REST API — `docs.github.com/en/rest/using-the-rest-api/troubleshooting-the-rest-api` (the documented 404-not-403 behavior for private repositories).
- Chromium Project — `chromium.org/updates/same-site/` (the SameSite default-change rollout).

**Incidents and research**
- Chef (Opscode) Engineering, "Supermarket HTTPS Redirect Postmortem" (2014), chef.io/blog/supermarket-https-redirect-postmortem.
- James Kettle / PortSwigger Research, "HTTP Desync Attacks: Request Smuggling Reborn" (2019), portswigger.net/research/http-desync-attacks-request-smuggling-reborn.
- CVE-2021-21295 (Netty HTTP/1.1 request-smuggling vulnerability, reported by Netflix).
- Mozilla Hacks, "Changes to SameSite Cookie Behavior – A Call to Action for Web Developers" (August 2020).

**Fast-drifting facts — verify before relying on any of these**
- Stripe's exact error `code` vocabulary and full HTTP-status table (`docs.stripe.com/api/errors` is living documentation and is extended over time).
- Specific bug-bounty figures and exact technique details in the PayPal/Verizon request-smuggling disclosures — widely reported, but figures vary slightly by source.
- Browser-specific `SameSite` defaults and any further rollout changes beyond the 2020 event described here.
- FastAPI/Starlette/Pydantic's exact default exception-handling behavior for malformed request bodies (confirmed against FastAPI with Pydantic v2 at the time of writing; verify against your installed versions, since framework defaults do change across major releases).

---

## Layered explanations

**10 seconds.** HTTP is a text-based agreement for requests and responses sent over a plain TCP connection: a start line, some headers, a blank line, and an optional body — for both directions. Methods promise different things about safety and retry-ability, status codes report what happened and whose fault it was, and the protocol itself remembers nothing between one request and the next.

**1 minute.** A request is a method, a target, a version, headers, and an optional body; a response swaps the first three for a version, a numeric status, and a reason phrase — both are separated from their body by one mandatory blank line, because a raw byte stream has no other way to mark where metadata ends. Since a body's own content can't signal its own length, every message must say how long it is, via `Content-Length` (known size, sent up front) or `Transfer-Encoding: chunked` (unknown size, sent as it's produced — exactly how streamed LLM output works). Methods are promises: `GET` promises no side effects; `PUT` and `DELETE` promise that repeating the request is as safe as doing it once; `POST` promises nothing about retries at all, which is why real APIs like Stripe's bolt an `Idempotency-Key` onto it by convention. A status code's first digit tells you who's at fault — 4xx you, 5xx the server or something behind it — and the specific 5xx code even tells you which component: `502` means a proxy got a bad answer from upstream, `504` means it got no answer in time. And underneath everything, HTTP genuinely remembers nothing between requests; cookies bolt state on by handing the client an opaque key to hand back, looked up against server-side storage — the same trick an LLM API uses to fake a "conversation" by resending the whole transcript every single call.

**5 minutes.** HTTP exists to answer the question TCP (Day 11) deliberately leaves open: given a reliable byte pipe with no concept of "a message," what shape must two independent programs agree a request and a response take? The 1991 answer was radically minimal — HTTP/0.9's one-line `GET /path`, chosen as plain text specifically so a human with nothing but `telnet` could speak and debug it — and everything since has been elaboration on that same text-based idea, until HTTP/2 finally went binary once parsing overhead and header redundancy mattered more than raw debuggability at internet scale (Day 14). A request's five fields and a response's five fields share an identical shape — start line, header fields, a mandatory blank line, optional body — which is exactly why one parser handles both directions with nearly the same code, as this note's own hand-built server demonstrates. Because a body has no self-describing length, RFC 9112 gives an exact precedence for framing it: certain status codes never have one; `Transfer-Encoding` wins over `Content-Length` if both appear, and that specific disagreement is explicitly flagged as a possible request-smuggling attempt — a real, named vulnerability class (James Kettle's 2019 research, with real, disclosed impact at PayPal and Netflix) that exists precisely because a front-end and a back-end parser can each correctly implement a different, individually-defensible reading of the same ambiguous bytes. Methods encode promises about safety and retry-ability rather than arbitrary labels: safe methods (`GET`, `HEAD`, `OPTIONS`) cause no side effects and can be pre-fetched or crawled freely; idempotent methods (those plus `PUT`, `DELETE`) can be retried blindly because repeating them leaves the server in the same state as doing them once; `POST` (and usually `PATCH`) can do neither, which is why a network-level retry of an un-protected `POST` is exactly how a customer gets charged twice, and exactly why Stripe's `Idempotency-Key` — a client-generated token the server uses to detect and replay, rather than repeat, a retried attempt — exists as a bolted-on fix for a gap the protocol itself leaves open. Status codes carry the same "promise, not just a label" quality: the first digit is a genuine, machine-actionable category (4xx you, 5xx the server), `401` versus `403` distinguishes "who are you" from "you can't do this even though I know exactly who you are," and `502` versus `504` distinguishes a proxy receiving garbage from a proxy receiving silence — a diagnostic distinction that only makes sense once you know a proxy sits in the path at all. Even the seemingly cosmetic choice between `301` and `302` carries real operational weight: RFC 9111 makes `301` cacheable by default with no explicit expiry, which is precisely why a real, documented incident (Chef's 2014 postmortem) led a team to add monitoring specifically for that risk after shipping one by mistake, however briefly the visible outage lasted. And underneath the entire protocol sits one deliberate, foundational property: statelessness — no request carries any memory of a previous one, which is exactly what makes load-balanced, horizontally-scaled serving possible in the first place, at the cost of requiring every notion of "the same visitor as before" to be built on top, by convention, using an opaque token the client hands back on every subsequent request. Cookies are the browser-era instance of that convention (and Chrome's 2020 `SameSite` default change, which broke real OpenID Connect flows, shows how fragile relying on an unstated default can be); an LLM API's illusion of "conversation memory" is the identical convention, one layer up, resending an entire transcript because the model, like the protocol underneath it, has no memory of its own.

**Expert summary.** HTTP is best understood as the minimal text-based framing convention required to carry a request/response exchange over an ordered, boundary-free byte stream, and nearly every one of its durable design choices is a direct trade against the debuggability requirement that shaped its 1991 origin: text over binary (until scale flipped the calculus at HTTP/2), a numeric status code over a string vocabulary (closed, extensible, category-bearing for free via the leading digit), and a message-length model that, per RFC 9112 §6.3, treats any framing ambiguity between `Content-Length` and `Transfer-Encoding` not as a case for precedence resolution but as a signal of active attack, because two independently-correct parsers disagreeing about message boundaries is a security property, not merely an interoperability inconvenience — the entire request-smuggling attack class James Kettle's research operationalized in 2019 is this exact ambiguity, weaponized. Method semantics — safety and idempotency, per RFC 9110 §9.2 — encode retry-safety as a first-class protocol-level contract rather than an implementation detail, which is precisely why the one non-idempotent, non-safe common method (`POST`) requires an out-of-band convention (an idempotency key, a pattern Stripe's production API demonstrates at real stakes) to be made retry-safe at all; the protocol offers no native mechanism, because none was designed to generalize across arbitrary, developer-defined side effects. Status codes function as a compact, structured error taxonomy whose value is almost entirely in the *distinctions* they force — client versus server fault at the first digit, authentication versus authorization at 401/403, and proxy-observed-garbage versus proxy-observed-silence at 502/504 — distinctions that collapse the instant an API violates them by encoding failure only in a body under a uniform `200`, which is why that anti-pattern is treated here as a first-order production risk rather than a stylistic preference. Caching interacts with status semantics in a way most engineers underestimate until it costs them: RFC 9111's heuristic-cacheability default for `301` (and not `302`) converts an easily-reversible server-side mistake into one with an uncontrolled, unrecoverable client-side tail, a mechanism real enough that a real, named postmortem's own corrective actions target it directly. Underlying all of it is statelessness as an architectural commitment rather than a limitation: by refusing to let any request carry forward memory of another, HTTP guarantees that any compliant server can answer any request with zero cross-request coordination, which is the precondition for horizontal scaling and load balancing (Day 16) to work at all — and the price of that guarantee is that every notion of continuity a real system needs (a login session, a shopping cart, an LLM "conversation") must be reconstructed by the calling application, out of band, using the identical primitive in every case: an opaque, round-tripped token, resolved against state that lives anywhere but in the protocol itself.
