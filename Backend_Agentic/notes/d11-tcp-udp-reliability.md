# Day 11 — TCP & UDP: Making Packets Trustworthy

> **Framing.** Day 9 taught you that data moves as *frames* on a wire, and Day 10 taught you that *IP packets* hop from router to router, each hop a fresh decision, nobody holding the whole path in mind. Both layers share one property that should alarm you: **neither of them promises anything.** A packet may arrive, or not. Twice, or not. In order, or scrambled. Corrupted, or clean. Day 11 is the day that gap gets closed — or deliberately left open. TCP is the answer "close it, in the kernel, for everyone"; UDP is the answer "leave it open, I'll handle it myself, and I have a good reason."
>
> This topic has one genuinely load-bearing connection to agentic systems, and it's not a stretch: **an LLM API call is a long-lived, mostly-idle TCP connection that streams a trickle of bytes for tens of seconds.** That shape — open early, silent for seconds while the model thinks, then a slow drip of tokens, then silence again — sits exactly on top of every TCP behaviour in this note: connection reuse, idle timeouts, keep-alive, head-of-line blocking, and the difference between "the socket is open" and "the peer is still there." The most common mystery failure in agent infrastructure ("it works, but every so often the first call after a quiet period dies") is a pure TCP-lifetime bug. We'll teach that inline, in the section where it belongs, with a runnable reproduction.

---

## Roadmap: what we're building, and in what order

We're going to build TCP the way it was actually invented — as a stack of answers to specific, concrete failures. Each concept exists because the previous layer broke in a particular way.

```
The problem:  IP delivers "best effort". Four distinct failure modes.
                    │
    ┌───────────────┼────────────────────────────────────────┐
    │               │                                        │
 Which app?     Did it arrive?                    How fast may I send?
    │               │                                        │
  PORTS        SEQ + ACK + RETRANSMIT              ┌─────────┴─────────┐
    │               │                              │                   │
 4-tuple       (needs: a shared starting      FLOW CONTROL      CONGESTION CONTROL
 identity       point → HANDSHAKE)            "don't drown       "don't drown
    │                                          the receiver"      the network"
    │                                              │                   │
    └──────────────► SOCKET = FD + buffers + state machine ◄────────────┘
                            │
                    …and then how does it end?
                            │
                    TEARDOWN, TIME_WAIT, RST
                            │
                    …and what does all this reliability COST?
                            │
                 HEAD-OF-LINE BLOCKING → which is why QUIC/UDP exists
```

Concepts, with their depth tier stated up front so you know how hard to study each one:

| # | Concept | Tier |
|---|---|---|
| 1 | IP's four failure modes (what "unreliable" concretely means) | **[CORE]** |
| 2 | Ports and the 4-tuple: connection identity | **[CORE]** |
| 3 | What a socket really is (FD + kernel buffers + state machine) | **[CORE]** |
| 4 | The 3-way handshake, SYN backlog, SYN floods | **[CORE]** |
| 5 | Sequence numbers, ACKs, retransmission, RTO, SACK | **[CORE]** |
| 6 | Flow control: the receive window | **[CORE]** |
| 7 | Congestion control: slow start, AIMD, CUBIC, BBR, BDP | **[CORE]** |
| 8 | Nagle's algorithm × delayed ACK (the 40ms mystery) | **[WORKING]** |
| 9 | Teardown, TIME_WAIT, RST, half-open connections | **[CORE]** |
| 10 | Head-of-line blocking | **[CORE]** |
| 11 | Connection lifetime: keep-alive, idle timeouts, pools | **[CORE]** |
| 12 | UDP: what you get when you remove all of the above | **[CORE]** |
| 13 | QUIC and HTTP/3: reliability rebuilt in userspace | **[WORKING]** |
| 14 | Bufferbloat | **[AWARE]** |

A note on tooling before we start, because you're on Windows and half the internet's TCP tutorials assume Linux. Every Python program in this note runs unchanged on Windows, macOS, and Linux. Where I use a diagnostic command, I'll give you the Windows form (`Get-NetTCPConnection`, Wireshark) *and* the Linux form (`ss`, `tcpdump`), because you will read Linux-shaped output in production logs for the rest of your career even if you never type it yourself. Where a *kernel tunable* differs between the two — and several do, importantly — I'll say so rather than pretending one command works everywhere.

---

## 1. What "unreliable" actually means — IP's four failure modes

**Depth: [CORE]**

### Intuition

Here is the sentence that every networking textbook uses and that teaches almost nothing: *"IP provides best-effort delivery."* It sounds like a reassurance. It is the opposite. "Best effort" is what a courier says when they will not sign anything: *we'll try*.

The reason to care is that "we'll try" is not one failure. It is four, and they are genuinely different problems requiring genuinely different machinery. If you only ever internalise "packets can get lost," you will build a system that handles loss and then falls over on reordering — which is exactly what happens to people who write their own protocols over UDP for the first time.

Let's be precise about all four, and about *why each one physically happens*. That last part matters: a failure mode you can only recite is one you'll forget, and a failure mode whose physical cause you understand is one you'll predict.

### The four failure modes, with their physical causes

**Failure mode 1: loss.** A packet enters a router and never comes out. The overwhelming majority of packet loss on the modern internet is not radio static or a chewed cable — it is a **full queue**. A router has a link out; that link has a fixed bit rate; packets arrive faster than the link drains; they pile into a memory buffer; the buffer fills; the next arriving packet has nowhere to go and is discarded. That's it. Loss is mostly *congestion*, which is why loss is TCP's primary congestion signal (concept 7) — the network's only way to say "too fast" is to throw your data away.

Secondary causes: a bit error flips somewhere and the checksum catches it, so the packet is dropped rather than delivered corrupt (common on wireless, rare on fibre); TTL hits zero because of a routing loop (Day 10); a firewall rule drops it silently.

**Failure mode 2: duplication.** The same packet arrives twice. Physical causes: a retransmission timer fired *just* before the original's acknowledgement came back, so both copies are in flight; a routing change briefly created two paths and a packet took both; a misconfigured device duplicates frames. Duplication is nastier than loss for a subtle reason — loss is *visible* (something is missing) but duplication is *invisible* unless you're tracking identity. If your protocol has no notion of "which byte is this," a duplicate silently corrupts your data by inserting a repeated chunk.

**Failure mode 3: reordering.** Packet #2 arrives before packet #1. Physical cause: **the internet has no single path.** Day 10's central lesson was that each router forwards independently, hop by hop. Two packets from the same sender to the same receiver can take different routes — because a routing table changed between them, or because the router is load-balancing across parallel links (ECMP — equal-cost multi-path). Different route, different queue depths, different latency. The one sent first can arrive second. This is *routine*, not exotic; it happens continuously on real internet paths.

**Failure mode 4: corruption.** The bits change in flight and IP does not fully catch it. IP has a header checksum, but classic IPv4's checksum covers the **header only** — not your payload. So IP itself will happily deliver a packet whose payload has been mangled, as long as the addresses survived. (IPv6 removed the header checksum entirely, on the theory that the layers below and above both check.) This is why TCP and UDP have their *own* checksums covering the payload.

### Analogy: the postcard courier

Imagine you want to send someone a 400-page manuscript, and the only delivery mechanism available is a courier company with these rules: you may hand them one postcard at a time; each postcard is delivered by whichever driver is free; drivers pick their own route; a driver who hits traffic may give up and bin the postcard; and occasionally a driver, unsure whether they delivered it, drives back and delivers it again.

You will receive your manuscript as a heap of postcards on a doormat: some missing, a few duplicated, none in order, one or two with a smudged word. **Everything TCP does is the work of turning that heap back into a manuscript** — and notice that you can't even *start* that work unless every postcard is numbered. That numbering is TCP's sequence number, and it's the single idea from which the rest follows.

**Where the analogy breaks — and this one matters a lot.** In the postcard picture, *you* (the recipient) do the reassembly, and the sender learns nothing. In real TCP the flow of information is **bidirectional and continuous**: the receiver constantly tells the sender what it has received, and the sender uses that stream of feedback to decide what to resend, how fast to send, and whether the network is congested. TCP is not "sender sprays, receiver sorts." It is a *control loop* — a conversation where the acknowledgements are as important as the data. Anyone who pictures TCP as one-directional will find flow control and congestion control (concepts 6 and 7) inexplicable, because both are entirely built out of the reverse channel.

### Worked example: trace a 5-packet transfer through all four failures

Let's make this mechanical. The sender wants to transmit 5000 bytes, chunked into five 1000-byte packets, labelled by the byte range each carries:

| Packet | Byte range |
|---|---|
| A | 0–999 |
| B | 1000–1999 |
| C | 2000–2999 |
| D | 3000–3999 |
| E | 4000–4999 |

Now a plausible, unexceptional internet path does the following:

```
Sender sends:     A    B    C    D    E
                  │    │    │    │    │
              route1 route1 route2 route1 route1
                  │    │    │    │    │
                  │    │  (route2 is 30 ms faster)
                  │    │    │    │    │
                  │  (router queue full → B dropped)
                  │         │    │    │
                  │         │    │  (D duplicated by a flapping link)
                  ▼         ▼    ▼    ▼

Receiver's doormat, in arrival order:
      C   A   D   D   E
      │   │   │   │   └── bytes 4000–4999
      │   │   │   └────── bytes 3000–3999  (duplicate!)
      │   │   └────────── bytes 3000–3999
      │   └────────────── bytes 0–999
      └────────────────── bytes 2000–2999   (arrived FIRST — reordered)

Missing entirely: B (bytes 1000–1999)
```

What must the receiving side do to hand the application a clean 5000-byte stream?

1. **Buffer out-of-order arrivals.** C arrives first, but bytes 2000–2999 cannot be given to the application yet — the application must see bytes in order, so C sits in a holding area (the *out-of-order queue*) until 0–1999 exist.
2. **Deliver in order as gaps fill.** A arrives. Now 0–999 is contiguous from the start, so it can be handed up. But 1000–1999 is still missing, so C *still* can't be delivered even though it's sitting right there. **This is head-of-line blocking, and you're seeing it in its natural habitat** — concept 10 is just this observation taken seriously.
3. **Discard the duplicate.** D arrives twice. The second copy covers bytes 3000–3999, which the receiver already holds. It must be silently dropped — not appended. Byte-range identity is what makes this possible.
4. **Tell the sender about the hole.** The receiver must communicate "I have 0–999 contiguously, and separately I'm holding 2000–2999 and 3000–4999." The sender must then resend *only* B. Doing this efficiently is exactly what fast retransmit and SACK are for (concept 5).

Notice what fell out of a single traced example: buffering, in-order delivery, duplicate suppression, gap reporting, selective retransmission, and head-of-line blocking. Every one of TCP's mechanisms is a direct answer to a line in that trace. TCP is not an arbitrary pile of features; it is the minimum machinery that turns that doormat into a manuscript.

### Under the hood: where the checks actually live

Layered, concretely (building on Day 9's encapsulation):

```
┌─────────────────────────────────────────────────────────┐
│ Ethernet frame                                          │
│  ┌───────────────────────────────────────────────────┐  │
│  │ IPv4 packet   [header checksum: header ONLY]      │  │
│  │  ┌─────────────────────────────────────────────┐  │  │
│  │  │ TCP segment [checksum: header + payload +   │  │  │
│  │  │              pseudo-header w/ IP addrs]     │  │  │
│  │  │  ┌───────────────────────────────────────┐  │  │  │
│  │  │  │ your bytes                            │  │  │  │
│  │  │  └───────────────────────────────────────┘  │  │  │
│  │  └─────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────┘  │
│  [Ethernet FCS: whole frame, link-local only]           │
└─────────────────────────────────────────────────────────┘
```

Three separate integrity checks, each with a different scope, and each one *per-hop or end-to-end* in a different way:

- **Ethernet FCS (frame check sequence)**, a 32-bit CRC over the whole frame, is checked and *regenerated at every hop*. It protects one cable. A frame corrupted on the wire is dropped by the receiving NIC. But because it's regenerated, corruption that happens *inside a router's memory* is invisible to it — the router recomputes a valid CRC over the now-wrong bytes.
- **IPv4 header checksum** protects addresses and TTL so a packet doesn't get delivered to the wrong host. Recomputed at each hop (TTL changes). Says nothing about your payload.
- **TCP/UDP checksum** is the only **end-to-end** check that covers your actual data. It's computed by the sending host and verified by the receiving host, and it includes a "pseudo-header" containing the source and destination IP addresses — a clever detail, because it means a packet delivered to the wrong host fails the checksum even though the transport header itself is intact.

**Deliberate stop:** TCP's checksum is a 16-bit one's-complement sum — weak by modern standards (it misses some multi-bit error patterns, and there is real measured data on undetected TCP corruption in large transfers). Understanding that it is *weak but end-to-end* is the load-bearing part. If you need stronger guarantees you add an application-level hash (concept: the file-upload design scenario later uses exactly this). We are not opening the arithmetic of one's-complement checksums; treat it as a black box that catches most but not all corruption.

### Judgment: was "unreliable IP + reliable TCP" the right split?

This is a real design decision with a real alternative, and it's worth pausing on, because the answer shaped the entire internet.

**The alternative that a competent engineer would have chosen:** make the *network* reliable. Have each router store a packet until the next router acknowledges it, retransmitting hop by hop. This is not a straw man — it is precisely what the telephone network and X.25 (the dominant packet-switching standard of the 1970s–80s, backed by national telecoms) did. Reliable links, intelligent network, dumb endpoints.

**Why the internet's answer won here:** the specific constraint was ARPANET's founding requirement (Day 9) — *survive the loss of arbitrary nodes*. A network that holds per-connection reliability state inside routers cannot survive losing a router, because the state dies with it. Pushing reliability to the endpoints means the network's job shrinks to "forward this if you can," which any surviving path can do. This is the **end-to-end principle**: put function at the endpoints unless there's a compelling reason not to, because only the endpoints know what the application actually needs.

**The trade-off, honestly.** X.25 bought something real, and pretending otherwise is how you get engineers who think the internet's design is self-evidently correct. Hop-by-hop reliability recovers from loss *locally and fast*: a packet lost on hop 3 of 12 is retransmitted by hop 3's router in one link-latency, instead of the sender noticing a whole round-trip later and resending from the far end. On a lossy link, end-to-end retransmission is dramatically less efficient. We gave up local recovery, and we gave up the network's ability to help — and we pay for it every day in high-latency loss recovery.

**The flip condition — when the alternative becomes right:** when the link is bad enough that end-to-end recovery is unworkable. And this is not hypothetical; it's what actually happens at the edges:
- **Wi-Fi and cellular** do link-layer retransmission (802.11 ARQ, LTE/5G RLC). Your laptop's Wi-Fi retransmits lost frames locally rather than letting TCP notice — precisely the X.25 idea, applied to the one hop where loss is common.
- **Deep-space networking** abandons end-to-end entirely. Round-trip time to Mars is 8–40 minutes; end-to-end retransmission is absurd. NASA's Delay/Disruption Tolerant Networking uses store-and-forward custody transfer between hops — X.25's philosophy, vindicated by a flip in the constraint.

So the honest statement is not "end-to-end is correct" but "end-to-end is correct when links are good enough that local recovery isn't worth the router state, and we hybridise at the edges where that stops being true." A reader who can state the flip condition understands the decision; a reader who can only recite "the end-to-end principle" has memorised a slogan.

---

## 2. Ports and the 4-tuple: how a machine knows which conversation this is

**Depth: [CORE]**

### Intuition

IP gets a packet to a *machine*. That is genuinely not enough, and the gap is easy to feel: your laptop right now has a browser with a dozen tabs, a Slack client, a code editor talking to a language server, and Windows checking for updates — all of them talking over the network, all through *one* IP address. When a packet arrives, something must decide which of those programs it belongs to. IP has no field for that. It has "which machine," not "which conversation on that machine."

So TCP and UDP add a 16-bit number at each end: the **port**. Source port and destination port sit in the first four bytes of the TCP header, before anything else, because they're what the receiving kernel needs first in order to know whose data this is.

Two things follow immediately from "16-bit":
- There are 65536 possible ports (0–65535).
- Port numbers are *per-protocol and per-host*, not global. TCP port 80 and UDP port 80 are different things. Your port 8000 and mine are unrelated.

### Analogy: the office building

The classic analogy, and it's a good one: the IP address is the building's street address; the port is the apartment or suite number. The postal service (IP) gets mail to the building; the number on the envelope (port) gets it to the right door.

**Where the analogy breaks:** in a building, apartment 5 is *one place*, and two letters to apartment 5 go to the same recipient. On a server, port 443 has one *listener* but potentially **fifty thousand simultaneous, completely independent conversations** happening through it. That is the thing the building analogy actively hides, and it's the single most important fact in this section. A port is not the identity of a conversation. It's a *rendezvous point*. Which leads to the real answer:

### The 4-tuple: what actually identifies a connection

A TCP connection is identified by **four** values together:

```
(source IP, source port, destination IP, destination port)
```

Plus, implicitly, the protocol (TCP vs UDP) — some texts call it the 5-tuple for that reason. This quadruple is what the kernel hashes on when a segment arrives, to find the right connection state.

Here is why that matters, made concrete. Suppose two browser tabs on your laptop (`192.168.1.50`) both open a connection to `example.com` (`93.184.216.34`) on port 443:

```
Connection 1:  (192.168.1.50, 51402, 93.184.216.34, 443)
Connection 2:  (192.168.1.50, 51403, 93.184.216.34, 443)
                              ^^^^^
                     only this differs — and it's enough
```

Three of the four values are identical. The **source port** distinguishes them. And because your machine chose those source ports, and the server never had to coordinate, the server can accept an unlimited number of connections on a single listening port — each arriving segment's 4-tuple tells it which one.

This is why the port-as-apartment analogy is dangerous and the 4-tuple is load-bearing. Get this and a whole class of "how does a server handle 50,000 users on port 443?" confusion evaporates.

### Ephemeral ports: where your source port comes from

When your program says "connect to example.com:443" it doesn't specify a source port. The kernel picks one from the **ephemeral port range** — a reserved band of high-numbered ports for exactly this.

- **Linux:** default `32768`–`60999` (about 28,000 ports). Check with `cat /proc/sys/net/ipv4/ip_local_port_range`.
- **Windows:** default `49152`–`65535` (about 16,000 ports). Check with `netsh int ipv4 show dynamicport tcp`.

These defaults drift between OS versions — verify on your box rather than trusting a number in a note (including this one).

Now the consequence that bites people in production. Your client can have at most ~16k–28k *simultaneous* connections **to the same destination IP and port**, because the only field free to vary is the source port. Beyond that, the kernel has no unused 4-tuple to hand you and `connect()` fails with "address already in use" or "cannot assign requested address."

Read that carefully, because the limit is per-destination, not global:
- 28,000 connections to `10.0.0.5:5432`? At the ceiling.
- 28,000 to `10.0.0.5:5432` **and** 28,000 to `10.0.0.6:5432`? Fine — different destination IP means a fresh space of 4-tuples.

This is the actual reason a busy service in front of a single database or a single load-balancer VIP hits a wall that looks like a mysterious connection failure, and it's why the fix is usually "add destination IPs" (multiple LB addresses, DNS returning several A records) or "stop opening so many connections" (pooling — concept 11). It also interacts brutally with `TIME_WAIT`, which we'll get to in concept 9: sockets in `TIME_WAIT` still occupy their 4-tuple for a minute after you're done with them.

### Well-known ports, and the one privilege rule worth knowing

Ports 0–1023 are the **well-known** or *system* ports, assigned by IANA: 22 SSH, 53 DNS, 80 HTTP, 443 HTTPS, 5432 PostgreSQL (actually 5432 is *registered*, not well-known — it's above 1023). On Unix, binding to a port below 1024 historically requires root. That single rule is why:

- Development servers use 8000, 8080, 3000, 5000 — no privileges needed.
- Production web servers either start as root and drop privileges after binding, or run behind something that does (a load balancer, an ingress controller), or use `CAP_NET_BIND_SERVICE` on Linux to grant just that one capability.

Windows has no such rule; any user can bind port 80 if it's free. Worth knowing because it's a common source of "works on my Windows machine, permission denied in the Linux container."

### Runnable example — watching the 4-tuple with your own eyes

This is a case where seeing it beats reading it: three connections to the same server, and we print all four values from both ends.

```python
# fourtuple_demo.py — no installs needed, stdlib only.
# Shows that (src IP, src port, dst IP, dst port) is what identifies a connection.
import socket
import threading

HOST, PORT = "127.0.0.1", 9101


def server():
    listener = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    listener.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    listener.bind((HOST, PORT))
    listener.listen(8)
    # The LISTENING socket has no peer — it is a rendezvous point, not a conversation.
    print(f"[server] listening on {listener.getsockname()}  (peer: none)")
    conns = []
    for _ in range(3):
        conn, addr = listener.accept()
        # Each ACCEPTED socket is a distinct connection with a full 4-tuple.
        print(f"[server] accepted: local={conn.getsockname()}  remote={conn.getpeername()}")
        conns.append(conn)          # keep them open so all 3 coexist
    for c in conns:
        c.close()
    listener.close()


threading.Thread(target=server, daemon=True).start()

clients = []
for i in range(3):
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((HOST, PORT))
    print(f"[client {i}] local={s.getsockname()}  remote={s.getpeername()}")
    clients.append(s)               # hold open, so we don't reuse ports

input("\nThree connections are open. Press Enter to close them...\n")
for s in clients:
    s.close()
```

Run it:

```
python fourtuple_demo.py
```

Real output (your ephemeral port numbers will differ — that's the point):

```
[server] listening on ('127.0.0.1', 9101)  (peer: none)
[client 0] local=('127.0.0.1', 55418)  remote=('127.0.0.1', 9101)
[server] accepted: local=('127.0.0.1', 9101)  remote=('127.0.0.1', 55418)
[client 1] local=('127.0.0.1', 55419)  remote=('127.0.0.1', 9101)
[server] accepted: local=('127.0.0.1', 9101)  remote=('127.0.0.1', 55419)
[client 2] local=('127.0.0.1', 55420)  remote=('127.0.0.1', 9101)
[server] accepted: local=('127.0.0.1', 9101)  remote=('127.0.0.1', 55420)

Three connections are open. Press Enter to close them...
```

**Why this works, line by line:**

- `socket.socket(socket.AF_INET, socket.SOCK_STREAM)` — `AF_INET` means IPv4 addressing, `SOCK_STREAM` means "give me a reliable ordered byte stream," which on IPv4 means TCP. (`SOCK_DGRAM` would give you UDP; we use it in concept 12.) The kernel creates the socket data structure and hands your process a descriptor for it — concept 3 is about what's inside that structure.
- `setsockopt(SOL_SOCKET, SO_REUSEADDR, 1)` — lets us re-bind port 9101 immediately even if a previous run left sockets in `TIME_WAIT`. Without it, re-running this script within a minute can fail with "address already in use." We'll fully explain *why* in concept 9; for now note that we needed it before we'd even met it, which tells you how unavoidable `TIME_WAIT` is.
- `bind((HOST, PORT))` — claims the local (address, port) pair. This is what makes the port *ours*.
- `listen(8)` — flips the socket from "a thing I could connect with" to "a thing that accepts connections," and creates the accept queue. The `8` is the backlog (concept 4).
- `getsockname()` / `getpeername()` — the two halves of the 4-tuple. Printing both from both ends is the whole demonstration: **the listener has a local address and no peer; each accepted socket has both.** The listening socket and the accepted sockets all share local port 9101, and are distinguished only by the remote port.
- We deliberately keep all three client sockets in a list. If we closed each one before opening the next, the kernel might reuse the same ephemeral port and the output would be less convincing.

The clinching observation: all three server-side sockets have local port 9101. One port, three conversations. Only the 4-tuple distinguishes them.

### Cross-check on a real machine

Windows PowerShell, while the script is paused at the prompt:

```powershell
Get-NetTCPConnection -LocalPort 9101 | Select-Object LocalAddress,LocalPort,RemoteAddress,RemotePort,State
```

```
LocalAddress LocalPort RemoteAddress RemotePort State
------------ --------- ------------- ---------- -----
127.0.0.1         9101 0.0.0.0                0 Listen
127.0.0.1         9101 127.0.0.1          55418 Established
127.0.0.1         9101 127.0.0.1          55419 Established
127.0.0.1         9101 127.0.0.1          55420 Established
```

Linux equivalent:

```
$ ss -tan '( sport = :9101 or dport = :9101 )'
State   Recv-Q Send-Q  Local Address:Port   Peer Address:Port
LISTEN  0      8           127.0.0.1:9101        0.0.0.0:*
ESTAB   0      0           127.0.0.1:9101      127.0.0.1:55418
ESTAB   0      0           127.0.0.1:9101      127.0.0.1:55419
ESTAB   0      0           127.0.0.1:9101      127.0.0.1:55420
```

One `LISTEN` row with no peer; three `ESTAB` rows differing only in peer port. The kernel's connection table *is* a 4-tuple table, and you're looking at it.

### Judgment: 16 bits for a port — a decision worth questioning

**The choice:** ports are 16 bits, giving 65536 values, and roughly a third of that is usable as an ephemeral range.

**The realistic alternative:** 32 bits. It was on the table — the header had room to be designed either way, and 32-bit port fields would have given effectively unlimited connection multiplicity per destination.

**Why 16 won:** header bytes were genuinely expensive in 1981. TCP's header is 20 bytes minimum; going to 32-bit ports adds 4 bytes to *every segment ever sent*. On a 56 kbit/s link carrying 536-byte segments, that's real overhead, and 65536 conversations per host was an unimaginable number at the time.

**The trade-off we live with:** the ~28,000-connections-per-destination ceiling described above is a direct consequence, and it is a live production constraint today. Every load balancer that needs multiple VIPs, every "use more destination IPs" workaround, every `TIME_WAIT` exhaustion incident traces back to this field width.

**Flip condition:** if you were designing a transport *today*, for datacentre-internal traffic where a single client fleet talks to a single service VIP at enormous connection counts, you would give yourself more bits — and note that QUIC (concept 13) did something adjacent: it identifies connections by a **Connection ID** in the QUIC header rather than by the 4-tuple, precisely so that a connection can survive a change of IP or port. QUIC didn't widen the port field (it can't; it rides on UDP), but it decoupled *connection identity* from *4-tuple identity*, which solves the more painful half of the problem. That's the flip in action: when your constraint becomes "connections must survive address changes and there are too many of them," you stop using the 4-tuple as the identity.

---

## 3. What a socket really is: a file descriptor, two buffers, and a state machine

**Depth: [CORE]**

### Intuition

Day 5 taught you that a file descriptor is a small integer indexing into your process's table of open things, and that "everything is a file" means `read()` and `write()` work on far more than files. A socket is the most interesting instance of that idea, and it's worth being precise about what the integer actually points at, because *every* confusing TCP behaviour you will ever debug lives in that structure.

The naive mental model is "a socket is a pipe to the other machine." That model is wrong in a way that makes half of TCP inexplicable. Here's what a socket actually is:

```
Your process                          Kernel
┌──────────────┐    ┌──────────────────────────────────────────────┐
│              │    │  struct socket / struct tcp_sock             │
│  fd table    │    │  ┌────────────────────────────────────────┐  │
│  ┌────┬────┐ │    │  │ 4-tuple: 127.0.0.1:55418               │  │
│  │ 0  │stdin│ │    │  │         → 127.0.0.1:9101              │  │
│  │ 1  │stdout│    │  ├────────────────────────────────────────┤  │
│  │ 2  │stderr│    │  │ STATE: ESTABLISHED                     │  │
│  │ 3  │ ●──────────► ├────────────────────────────────────────┤  │
│  └────┴────┘ │    │  │ send buffer  [....data awaiting ACK...]│  │
│              │    │  │ recv buffer  [..data app hasn't read..]│  │
│  s.send(b"x")│    │  ├────────────────────────────────────────┤  │
│  s.recv(1024)│    │  │ snd_nxt, snd_una, rcv_nxt (seq nums)   │  │
│              │    │  │ cwnd, ssthresh (congestion state)      │  │
│              │    │  │ rwnd (peer's advertised window)        │  │
│              │    │  │ srtt, rttvar, RTO (timers)             │  │
│              │    │  │ out-of-order queue                     │  │
│              │    │  └────────────────────────────────────────┘  │
└──────────────┘    └──────────────────────────────────────────────┘
                                    │
                              NIC ──┴── wire
```

Three things, and the whole rest of this note is about them:

1. **A file descriptor** — your handle. `send()`/`recv()` (or `write()`/`read()`) on it are syscalls: a user→kernel transition, exactly as Day 2 described.
2. **Two buffers** — a **send buffer** holding bytes you've handed over but that haven't been acknowledged yet, and a **receive buffer** holding bytes that arrived but your application hasn't read yet.
3. **A state machine plus a pile of counters** — the connection's state (`LISTEN`, `ESTABLISHED`, `TIME_WAIT`…), sequence numbers, window sizes, congestion variables, retransmission timers, and a queue for out-of-order data.

### The single most important consequence: `send()` does not send

Internalise this sentence: **`send()` returning successfully means "the kernel copied your bytes into the send buffer." It does not mean the bytes left your machine, and it certainly does not mean the peer received them.**

This is not pedantry. It is the reason for an entire genre of production bug. Consider:

```python
sock.send(b"critical audit record\n")
sock.close()
# Was the record delivered?  You have NO IDEA.
```

The `send()` succeeded. The bytes are in a kernel buffer. Then `close()` happens. If the peer crashed thirty seconds ago and the network hasn't noticed, those bytes will be retransmitted a few times and then discarded when the connection is reset — and your process exited believing it had done its job. **TCP gives you reliable delivery or an error, but it gives the error to whoever is still listening**, and if you've closed the socket and moved on, nobody is.

The corollary, which is the actionable form: *if you need to know that the other side got it, the other side has to tell you.* An application-level acknowledgement — a response body, a 200 status, a row written — is the only proof. TCP's ACKs acknowledge receipt *by the peer's kernel*, not processing by the peer's application, and they aren't visible to you anyway.

### The mirror image: `recv()` and short reads

The receive side has its own trap, and it's the one that breaks the most beginner code. `recv(4096)` does not mean "give me 4096 bytes." It means "give me *up to* 4096 bytes, and block until at least one is available." If 12 bytes are in the receive buffer, you get 12.

Why? Because **TCP is a byte stream, not a message protocol.** There is no such thing as "a TCP message." You called `send(b"HELLO")` and then `send(b"WORLD")`; the peer may see `HELLOWORLD` in one `recv()`, or `HEL` then `LOWORLD`, or ten one-byte reads. The kernel is free to coalesce your two sends into one segment (that's Nagle's algorithm, concept 8), and the network is free to fragment a segment.

Every text protocol on earth exists partly to solve this. HTTP uses `\r\n\r\n` to end headers and `Content-Length` to delimit bodies. Line-based protocols use `\n`. Binary protocols prefix a length. **If your protocol has no framing, your code has a bug that will show up under load** — because at low volume your sends usually happen to land as single segments, and at high volume they stop doing so. This is the classic "worked in dev, corrupted data in prod."

### Worked example — watching a byte stream refuse to respect your message boundaries

```python
# stream_not_messages.py — proves TCP has no message boundaries.
import socket
import threading
import time

HOST, PORT = "127.0.0.1", 9102
received_chunks = []


def server():
    lis = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    lis.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    lis.bind((HOST, PORT)); lis.listen(1)
    conn, _ = lis.accept()
    time.sleep(0.4)                    # let the client's sends pile up in our recv buffer
    while True:
        chunk = conn.recv(4096)        # ask for 4096 — see what we actually get
        if not chunk:                  # b"" means the peer closed (EOF)
            break
        received_chunks.append(chunk)
    conn.close(); lis.close()


t = threading.Thread(target=server); t.start()
time.sleep(0.1)

c = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
c.connect((HOST, PORT))
for msg in [b"MSG-1", b"MSG-2", b"MSG-3", b"MSG-4"]:
    c.send(msg)                        # four separate send() calls, four "messages"
c.close()
t.join()

print("four send() calls became", len(received_chunks), "recv() results:")
for i, chunk in enumerate(received_chunks):
    print(f"  recv #{i}: {chunk!r}  ({len(chunk)} bytes)")
```

Output:

```
four send() calls became 1 recv() results:
  recv #0: b'MSG-1MSG-2MSG-3MSG-4'  (20 bytes)
```

**Why this works:** the four `send()` calls each copied 5 bytes into the same send buffer. Because the connection was idle and the amounts tiny, the kernel coalesced them into one segment on the wire. On the receiving side, the `time.sleep(0.4)` gave all 20 bytes time to arrive before we called `recv()`, so a single `recv(4096)` drained all of them. **Your four messages became one read.** Nothing in TCP preserved your boundaries, because TCP does not know you had any.

Now delete the `time.sleep(0.4)` and re-run a few times: you'll sometimes see `b'MSG-1'` alone, sometimes `b'MSG-1MSG-2'`, sometimes all four — because now you're racing the network and the boundary you get is an accident of timing. **A protocol whose correctness depends on that timing is broken; it just hasn't failed yet.** The fix is framing:

```python
# The minimal fix: length-prefix each message. 4-byte big-endian length, then payload.
import struct

def send_msg(sock, payload: bytes) -> None:
    sock.sendall(struct.pack("!I", len(payload)) + payload)   # !I = network-order uint32

def recv_exactly(sock, n: int) -> bytes:
    """recv() may return fewer bytes than asked — loop until we have exactly n."""
    buf = b""
    while len(buf) < n:
        chunk = sock.recv(n - len(buf))
        if not chunk:
            raise ConnectionError(f"peer closed after {len(buf)} of {n} bytes")
        buf += chunk
    return buf

def recv_msg(sock) -> bytes:
    (length,) = struct.unpack("!I", recv_exactly(sock, 4))
    return recv_exactly(sock, length)
```

Note `sendall` versus `send`: **`send()` may write fewer bytes than you gave it** and returns how many it took (if the send buffer is nearly full, it takes what fits). `sendall()` loops for you. Using `send()` and ignoring the return value is the same class of bug as assuming `recv()` fills your buffer, and in Python it is the more common of the two because `send()` usually succeeds fully on small writes — until it doesn't.

**Honesty caveat on this example:** it runs over loopback (`127.0.0.1`), where there is no real network, no loss, and an MTU of 65535 on Linux / 4GB-ish on Windows. Loopback exaggerates coalescing (everything fits in one segment) and eliminates reordering entirely. That's *fine* for demonstrating "no message boundaries" — the point is that framing is your job — but do not use loopback to reason about timing, throughput, or loss behaviour. For that you need a real path or a deliberately impaired one (`tc netem` on Linux; Clumsy or the Network Emulator Toolkit on Windows).

### Under the hood: buffer sizes, and where `send()` blocks

The send buffer is finite. So:

- If you `send()` more than fits, and the socket is **blocking** (the default), your process *blocks inside the syscall* until space frees up. Space frees up when the peer acknowledges data (letting the kernel discard it) — so **`send()` blocking is your application being back-pressured by the network**, transmitted to you through the ACK stream.
- If the socket is **non-blocking**, `send()` instead returns immediately, either writing partially or failing with `EWOULDBLOCK`/`EAGAIN`. This is the foundation of async I/O (`asyncio`, `epoll`, `select`) — you never block, you get told "not now" and come back when the socket is writable.

Typical default buffer sizes (verify on your machine; they've grown over the years and Linux auto-tunes):

```
Linux:   net.ipv4.tcp_wmem = 4096  16384  4194304    # min / default / max (send)
         net.ipv4.tcp_rmem = 4096  131072 6291456    # min / default / max (receive)
```

Those triples mean Linux *auto-tunes* between min and max based on observed conditions — a detail worth knowing because it means "the default buffer is 16 KB" is wrong; the default is a starting point.

Windows also auto-tunes (receive window auto-tuning has been on by default since Vista); inspect with `Get-NetTCPSetting`. Deliberately setting `SO_SNDBUF`/`SO_RCVBUF` yourself **disables auto-tuning on both platforms**, which is usually a pessimisation — a specific and commonly-made mistake, because setting a buffer size *feels* like tuning.

You can read the sizes from Python:

```python
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
print("send buffer:", s.getsockopt(socket.SOL_SOCKET, socket.SO_SNDBUF))
print("recv buffer:", s.getsockopt(socket.SOL_SOCKET, socket.SO_RCVBUF))
# On Linux the reported value is doubled by the kernel (it reserves overhead),
# so a "16384" setting reads back as 32768. This is documented, not a bug.
```

### Why the buffers matter: they are where your latency hides

A worked latency accounting, because "there are buffers" is not a lesson until you can compute with it.

Suppose your service sends a 200 KB response over a connection with a 40 ms round-trip time and a send buffer of 64 KB. What does `send()` do?

- Bytes 0–65535: copied into the send buffer. `send()` returns immediately. **From your application's point of view, this was instant.**
- Byte 65536: no room. `send()` blocks. It stays blocked until ACKs arrive for earlier data, and the first ACK cannot arrive sooner than one RTT (40 ms) after the first data left.
- So your application sees: a fast `send()`, then a mysterious 40 ms stall, then progress in ~40 ms-spaced bursts.

If you were timing "how long did my handler take" you'd see a bimodal distribution and wonder why. **The buffer is why.** Small responses fit and look instant; large ones don't and expose the RTT. This is also why a single slow client can pin a thread in a thread-per-connection server — the thread is asleep in `send()`, waiting for someone on a bad connection to acknowledge — and therefore why async I/O exists at all.

### Runnable example — feeling the send buffer fill up

```python
# send_blocks.py — demonstrates that send() blocks when the peer stops reading.
# Shows the send buffer is finite and that back-pressure reaches your code.
import socket
import threading
import time

HOST, PORT = "127.0.0.1", 9103


def lazy_server():
    """Accept a connection and then deliberately never read from it."""
    lis = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    lis.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    lis.bind((HOST, PORT)); lis.listen(1)
    conn, _ = lis.accept()
    time.sleep(30)          # never recv() — receive buffer fills, window closes
    conn.close(); lis.close()


threading.Thread(target=lazy_server, daemon=True).start()
time.sleep(0.2)

c = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
c.connect((HOST, PORT))
c.settimeout(3.0)           # so this demo terminates instead of hanging forever

chunk = b"x" * 65536        # 64 KB per write
total = 0
try:
    while True:
        c.send(chunk)
        total += len(chunk)
        print(f"buffered {total // 1024} KB", end="\r", flush=True)
except socket.timeout:
    print(f"\nsend() BLOCKED after {total // 1024} KB — buffers are full, "
          f"peer isn't reading, back-pressure has reached my code.")
c.close()
```

Output (numbers vary by OS and buffer auto-tuning — that's expected):

```
buffered 2496 KB
send() BLOCKED after 2496 KB — buffers are full, peer isn't reading, back-pressure has reached my code.
```

**Why this works, and what each part is proving:**

- The server accepts and then sleeps without ever calling `recv()`. Data still arrives at its kernel and lands in the **receive buffer** — the application not reading doesn't stop the network.
- Once that receive buffer fills, the receiver advertises a **zero window** (concept 6) — literally "stop, I have no room." The sender's TCP stops transmitting.
- With transmission stopped, ACKs stop coming, so the sender's **send buffer** can't be drained and fills too.
- Only *then* does `send()` block. So the ~2.5 MB we managed to buffer is roughly *sender send buffer + receiver receive buffer*, both of which auto-tuned upward as we hammered them.
- `settimeout(3.0)` converts the block into a `socket.timeout` exception so the demo ends. **This is also the honest version of a real production decision**: a blocking `send()` with no timeout is an unbounded hang, and "my service froze" incidents are frequently exactly this.

The lesson to carry: **flow control is not an abstraction — it is a chain of finite buffers, and when the far end stops consuming, the pressure propagates backwards until it reaches your `send()` call.** A system with no timeout at that point has no way to shed load.

### System design — sizing the buffers for a streaming service

**Problem:** you're building a service that streams large model responses (or video segments, or CSV exports) to thousands of concurrent clients over connections with RTTs from 5 ms (same region) to 250 ms (intercontinental). How much kernel memory will connections consume, and what do you set?

**Requirements:** support 20,000 concurrent streaming connections per host; don't let a slow client stall a fast one; don't OOM the box.

**The core arithmetic — bandwidth-delay product.** To keep a pipe full you must have in flight at least `bandwidth × RTT` bytes. For 10 Mbit/s to a client at 250 ms RTT:

```
10,000,000 bits/s × 0.250 s = 2,500,000 bits = 312,500 bytes ≈ 305 KB
```

So a *single* far-away client needs ~305 KB in flight to sustain 10 Mbit/s. If your send buffer is 64 KB, you cannot exceed `65536 / 0.250 = 262 KB/s ≈ 2 Mbit/s` to that client no matter how much bandwidth exists. **The buffer, not the link, is the bottleneck.** This is the single most useful calculation in this note — commit the formula, not the numbers.

**The alternatives considered:**

1. **Set `SO_SNDBUF` to 512 KB explicitly on every socket.** Guarantees the far-away client can go fast.
2. **Leave auto-tuning on, raise the kernel's `tcp_wmem` maximum.** The kernel grows each socket's buffer only for connections that demonstrate they need it.
3. **Cap per-connection buffers small and rely on application-level chunking with explicit pacing.**

**The decision: (2).** Leave auto-tuning enabled and raise the ceiling.

**The actual reason:** option (1) multiplies by connection count. 20,000 × 512 KB = **10 GB of kernel memory**, committed regardless of whether a connection is a 5 ms local client that needs 8 KB or a 250 ms one that needs 305 KB. Auto-tuning gives the large buffer *only to the sockets whose measured RTT and throughput justify it*, so your 5 ms clients cost a few KB each and your handful of intercontinental streams get their 305 KB. And critically, **explicitly setting `SO_SNDBUF` disables auto-tuning** — option (1) doesn't just over-allocate, it forfeits the mechanism that would have gotten this right.

**The trade-off honestly:** auto-tuning is reactive. A connection *starts* with the default buffer and grows, which means the first second or so of a new far-away stream is buffer-limited even though the kernel will eventually fix it. If your workload is many short far-away transfers that never live long enough to be auto-tuned upward, option (1) genuinely wins — you're paying memory to skip the ramp. That's a real cost you're accepting.

**Flip condition:** set buffers explicitly when (a) you have few connections and know their characteristics — a datacentre replication link, a fixed set of peers — so the memory maths is bounded, or (b) you are on a platform whose auto-tuning you've measured to be bad, or (c) you must *limit* rather than enable, deliberately capping a buffer to bound worst-case memory or latency. "Explicit is better" is not the rule; "explicit when you know something the kernel can't measure" is.

**Failure modes:** raising `tcp_wmem`'s max without watching memory means 20,000 connections all deciding they need big buffers during a traffic spike and the OOM killer taking your process. Monitor total socket memory (`ss -m`, `/proc/net/sockstat`) not just process RSS — **kernel socket buffers are not in your process's RSS**, which is why "my process uses 400 MB but the box is out of memory" happens.

### In production — what actually goes wrong at the socket layer

**Best practices, in rough order of how much pain they save:**

1. **Always set timeouts.** Blocking `recv()`/`send()` with no timeout is an unbounded hang waiting for a network event that may never come. In Python, `sock.settimeout()`; in `httpx`/`requests`, the `timeout` parameter; in the Anthropic SDK, the client's `timeout` (default 10 minutes — deliberately generous because a long thinking turn is legitimate, but *not* what you want for a health check).
2. **Frame your messages.** Length-prefix or delimiter. Never assume `recv()` boundaries.
3. **Use `sendall`, and check `send`'s return value if you use `send`.**
4. **Close sockets deterministically** (context managers, `try/finally`). A leaked socket is a leaked FD, and FD exhaustion presents as a bewildering "too many open files" from code nowhere near the leak.
5. **Don't set `SO_SNDBUF`/`SO_RCVBUF` unless you've measured** — you're disabling auto-tuning.

**Mistakes, beginner → senior:**

- *Beginner:* assuming `recv(n)` returns `n` bytes; assuming one `send` = one `recv`; no timeouts; treating `send()` success as delivery.
- *Intermediate:* leaking FDs in error paths; blocking calls inside an async event loop (this one is so common in FastAPI that it deserves the callout it gets on Day 7 — a blocking `socket.recv()` in an `async def` handler stalls *every* concurrent request on that worker, because the event loop thread is stuck in the syscall).
- *Senior:* not distinguishing "connection established" from "application healthy" in health checks (concept 11's design scenario); sizing buffers without doing the BDP arithmetic; monitoring process memory but not kernel socket memory.

**Observability:**

- Windows: `Get-NetTCPConnection`, `Get-NetTCPSetting`, `netstat -ano`, Resource Monitor, Wireshark.
- Linux: `ss -tanmi` (the `m` shows socket memory, the `i` shows internal TCP info: cwnd, rtt, retransmits — enormously useful), `/proc/net/sockstat`, `nstat -az | grep -i tcp`.
- Both: connection count by state is your single most valuable TCP metric. A rising `TIME_WAIT` count, a rising `CLOSE_WAIT` count, and a rising `SYN_RECV` count each mean something specific and different (concepts 9 and 4).

**Cost:** each connection costs kernel memory (buffers, typically several KB to several hundred KB), an FD, and a slot in the connection hash table. At scale connections are your dominant per-client cost, which is the entire economic argument for connection pooling and multiplexing.

---

## 4. The 3-way handshake: agreeing where to start counting

**Depth: [CORE]**

### Intuition

Concept 1 showed that reassembly is impossible without numbering every byte. But there's a prerequisite hiding in that: **both sides must agree on what the numbers mean.** If I start counting my bytes at 1 and you assume I started at 0, every byte lands one position off and the stream is garbage.

So before a single byte of your data moves, TCP does one thing: the two sides exchange starting numbers. That's what the handshake *is*. Everything else people say about it — "establishing the connection," "negotiating options" — is secondary. The primary job is **agreeing on two initial sequence numbers**, one for each direction, and confirming that each side heard the other's.

Why does it take *three* messages? Because there are two independent directions to establish, and each side needs to know that its own number was received. Count the required facts:

1. Client tells server its starting number. (1 message)
2. Server tells client its starting number, **and** confirms it got the client's. (1 message — these can ride together)
3. Client confirms it got the server's number. (1 message)

Three. Not two (the server would never know its number arrived), not four (step 2's two jobs genuinely combine). The handshake is minimal, not arbitrary. This is worth dwelling on because "why three?" is a stock interview question and the answer people memorise ("SYN, SYN-ACK, ACK") is the *what*, not the *why*.

### Analogy: two people agreeing on a page number over a bad phone line

You're reading a document to a colleague over a crackly line. Before you start, you must agree where you're each starting:

> **You:** "I'll start from page 100." *(SYN, seq=100)*
> **Them:** "Got it, you're at 100. I'll start from page 500." *(SYN-ACK, seq=500, ack=101)*
> **You:** "Got it, you're at 500." *(ACK, ack=501)*

Now both of you can talk, and either of you can say "I didn't get page 103" and be understood.

**Where the analogy breaks:** the phone call has a *circuit* — a physical connection that exists whether or not you speak. TCP has nothing of the kind. After the handshake, "the connection" is purely **two sets of matching numbers in two kernels' memory.** There is no reserved path, no resource in any router, nothing on the wire. This is the deepest thing to understand about TCP and it explains an otherwise baffling family of behaviours: if one side reboots, the other side has a "connection" to nobody and *will not find out until it tries to use it*; a firewall can silently forget your connection while both sides remain convinced it exists; a connection can survive a network outage lasting minutes because neither side's memory changed. **A TCP connection is a shared hallucination maintained by two state machines.** Concept 11 is entirely about what happens when the hallucination diverges.

### Worked example — a real handshake, byte by byte

Here's an actual `tcpdump` capture of a handshake to a local server on port 9009, annotated. (Windows equivalent: the same three packets in Wireshark with filter `tcp.port == 9009`.)

```
$ sudo tcpdump -i lo -n -S 'tcp port 9009'

12:04:11.104291 IP 127.0.0.1.51402 > 127.0.0.1.9009: Flags [S],
    seq 2837465901, win 65495,
    options [mss 65495,sackOK,TS val 3910284 ecr 0,nop,wscale 7], length 0

12:04:11.104312 IP 127.0.0.1.9009 > 127.0.0.1.51402: Flags [S.],
    seq 1029384756, ack 2837465902, win 65483,
    options [mss 65495,sackOK,TS val 3910284 ecr 3910284,nop,wscale 7], length 0

12:04:11.104329 IP 127.0.0.1.51402 > 127.0.0.1.9009: Flags [.],
    ack 1029384757, win 512, options [nop,nop,TS val 3910284 ecr 3910284], length 0
```

Read it slowly; every field earns its place.

**Packet 1 — `Flags [S]` (SYN):**
- `seq 2837465901` — the client's **Initial Sequence Number (ISN)**. Not 0, and not sequential across connections. It's *randomised*, and that's a security property, not an implementation detail (see below).
- `win 65495` — "I can currently accept 65495 bytes." The receive window (concept 6).
- `mss 65495` — **Maximum Segment Size**: "don't send me a TCP payload bigger than this." It's 65495 only because this is loopback, whose MTU is 65536; on a real Ethernet path you'd see `mss 1460` (1500 MTU − 20 IP − 20 TCP). MSS is announced here, in the SYN, and *only* here — it cannot be renegotiated later.
- `sackOK` — "I support Selective ACK." Concept 5.
- `wscale 7` — **window scale factor**: multiply my advertised window by 2⁷ = 128. Concept 6 explains why this option exists and why the connection is crippled without it.
- `TS val ... ecr 0` — timestamps, used for RTT measurement and for `TIME_WAIT` safety (concept 9).
- `length 0` — **no data.** A SYN carries no payload (classically; TCP Fast Open changes this — see the deliberate stop below).

**Packet 2 — `Flags [S.]` (SYN-ACK; tcpdump writes ACK as `.`):**
- `seq 1029384756` — the server's own independent ISN for the server→client direction. **The two directions have completely separate numbering.** This is the thing people most often get wrong.
- `ack 2837465902` — "I have received everything up to and including sequence 2837465901; the next byte I expect is 2837465902." Note it's the client's ISN **+ 1**. The SYN carried no data, yet it consumed one sequence number. That's deliberate: the SYN itself must be acknowledgeable, so it occupies a number. (FIN does the same — concept 9.)
- The server echoes its own options: its MSS, its `sackOK`, its `wscale`. **Options are per-direction and each side announces its own.** If the server hadn't sent `wscale`, window scaling would be off for *both* directions — it requires mutual support.

**Packet 3 — `Flags [.]` (ACK):**
- `ack 1029384757` — the server's ISN + 1. "Got your number."
- `win 512` — and here's the detail that trips everyone: 512 looks *smaller* than the 65495 in packet 1, but window scaling is now active, so the real window is 512 × 128 = **65536**. Scaling applies from the third packet onward, never to the SYNs themselves (the scale factor isn't known yet when the SYN is sent). Reading a capture and thinking the window collapsed is a classic misdiagnosis.

**Elapsed time: 38 microseconds**, because this is loopback. On a real path each arrow costs one one-way latency, so the handshake costs **one full round-trip before any data flows.** Recall Day 1's latency pyramid: to a server 100 ms away, you have burned 100 ms before sending byte one. Add TLS (Day 14) and it's 2–3 RTTs. This is the entire reason connection reuse exists, and the reason HTTP/2 and QUIC were designed the way they were.

### Under the hood: the state machine and the two queues

```
        CLIENT                                  SERVER
                                            (socket(), bind(), listen())
      CLOSED                                    LISTEN
        │                                          │
        │  ──── SYN seq=X ───────────────────────► │
   SYN_SENT                                        │  ┌──────────────────┐
        │                                          │  │ SYN queue        │
        │  ◄─── SYN-ACK seq=Y ack=X+1 ──────────── │  │ (half-open,      │
        │                                    SYN_RECV  │  SYN_RECV state) │
   ESTABLISHED                                     │  └──────────────────┘
        │  ──── ACK ack=Y+1 ─────────────────────► │  ┌──────────────────┐
        │                                    ESTABLISHED│ accept queue    │
        │                                          │  │ (handshake done, │
        │  ──── your data ──────────────────────►  │  │  awaiting        │
        │                                          │  │  accept())       │
                                                      └──────────────────┘
                                                             │
                                                      accept() returns
                                                      a NEW socket fd
```

Two separate queues on the server, and knowing they're separate is what lets you read the symptoms:

1. **The SYN queue** (`SYN_RECV` state) holds **half-open** connections: a SYN arrived, a SYN-ACK went out, the final ACK hasn't come back. Sized by `net.ipv4.tcp_max_syn_backlog` on Linux.
2. **The accept queue** holds connections that completed the handshake and are waiting for your application to call `accept()`. Sized by the `backlog` argument to `listen()`, capped by `net.core.somaxconn`.

The symptoms differ, and this is diagnostic gold:

- **SYN queue full** → new SYNs are dropped. The client sees `connect()` hang and retry (SYN retransmissions at roughly 1 s, 3 s, 7 s…) and eventually time out. Looks like *"the server is unreachable."*
- **Accept queue full** → the handshake *completes* and then the connection sits there. The client believes it's connected and sends a request; the server's application never accepts it. Looks like *"the server accepted my connection and then ignored me"* — a hung request, not a connection failure. (Linux's default is to drop the final ACK so the client retransmits it; with `tcp_abort_on_overflow=1` it sends an RST instead, converting a hang into a fast error.)

**The load-bearing insight: the handshake is done by the kernel, not your application.** Your process can be completely wedged — deadlocked, stuck in a slow database query, garbage-collecting for four seconds — and the kernel will *still* happily complete handshakes and stack connections in the accept queue. This is why "the port is open" is a near-worthless health check, and it's the basis of the design scenario in concept 11.

**`accept()` returns a new socket.** The listening socket stays listening. The new fd has its own 4-tuple and its own buffers. This is exactly what you saw in concept 2's output: one `LISTEN` row, three `ESTAB` rows.

### Why the ISN is random: a security property, not trivia

Early TCP implementations chose ISNs predictably (often derived from a clock). If an attacker can predict the ISN a server will choose, they can perform **TCP sequence prediction**: forge a SYN from a spoofed source IP (say, a trusted internal host), never see the SYN-ACK (it goes to the spoofed address), but *guess* the server's ISN and send the third-packet ACK with the right number. The server now believes it has an established connection from a trusted host, and the attacker can inject data blind. This was the technique in the 1994 attack on Tsutomu Shimomura's machines attributed to Kevin Mitnick.

RFC 6528 specifies the modern fix: ISNs are generated from a cryptographic hash of the 4-tuple plus a secret, so they're unpredictable across connections but still increase over time for the same 4-tuple (which `TIME_WAIT` logic wants). *Primary source:* RFC 6528, "Defending against Sequence Number Attacks."

The generalisable lesson: **the randomness of a number that looks like an implementation detail can be the only thing standing between you and blind injection.** The same reasoning appears in DNS transaction IDs and source-port randomisation, which is Day 12's material — and the same class of attack (Kaminsky) exists there for the same reason.

### SYN flood, and why SYN cookies are a beautiful hack

The SYN queue is a resource an attacker can consume without proving anything. Send a flood of SYNs from spoofed source addresses; the server allocates SYN-queue state for each and dutifully sends SYN-ACKs into the void; the final ACKs never come; the queue fills; legitimate SYNs are dropped. **The asymmetry is what makes it work: the attacker sends 40 bytes and never completes anything, while the server allocates memory and holds it for tens of seconds.**

**SYN cookies** are the defence, and the idea is genuinely elegant: *stop keeping state at all.* When the SYN queue overflows, instead of allocating an entry, the server encodes the connection's essential information — a coarse timestamp, the MSS, and a MAC (keyed hash) over the 4-tuple — **into the ISN it sends in the SYN-ACK.** Then it forgets everything.

If a real client responds, its ACK carries `ISN + 1`. The server subtracts 1, verifies the MAC, and reconstructs the connection state *from the number in the packet*. No state was held between the SYN and the ACK. An attacker who never sends the ACK costs the server nothing.

```
Normal:      SYN → [allocate SYN-queue entry] → SYN-ACK → ACK → match entry
SYN cookie:  SYN → [allocate NOTHING]         → SYN-ACK(ISN = MAC(4-tuple, t, mss))
                                              → ACK → verify MAC, rebuild state
```

Enable on Linux with `net.ipv4.tcp_syncookies=1` — the common default, which means "use them only when the queue overflows," exactly right since they're a degraded mode. Windows has equivalent SYN attack protection, on by default.

**The trade-off, honestly:** the ISN is only 32 bits, and you must fit a timestamp, an MSS encoding, and a MAC into it. So SYN cookies **cannot preserve TCP options.** MSS is squeezed into 3 bits (8 quantised values), and window scaling, SACK, and timestamps are lost for cookie-established connections. A connection established via SYN cookies is therefore *measurably worse* — no window scaling means the throughput ceiling of concept 6, no SACK means the retransmission inefficiency of concept 5. That's why they're a fallback, not a default mode.

**Flip condition:** if you're under sustained SYN flood, degraded connections beat no connections, so cookies win. If you're not under attack, the option loss is pure cost, so you want overflow-only mode. And if you're facing volumetric floods large enough to saturate your *link* rather than your SYN queue, cookies don't help at all — the packets never reach you — and the answer moves upstream to your provider's scrubbing, a different layer of defence entirely.

**Deliberate stop:** TCP Fast Open (RFC 7413) lets a client send data *in* the SYN on repeat visits, using a cryptographic cookie from a previous connection to skip a round trip. It exists; deployment is patchy (middlebox interference is the usual culprit); QUIC's 0-RTT largely supersedes the idea. Know it exists; treat as a black box unless a project forces you deeper.

### Runnable example — watching the handshake and the accept queue

```python
# handshake_and_backlog.py — shows that the kernel completes handshakes
# even when the application never calls accept().
import socket
import time

HOST, PORT = "127.0.0.1", 9104

listener = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
listener.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
listener.bind((HOST, PORT))
listener.listen(2)          # backlog of 2 — deliberately tiny
print(f"listening on {HOST}:{PORT} with backlog=2; NOT calling accept()")

# Open five connections. The application never accepts any of them.
clients = []
for i in range(5):
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.settimeout(2.0)
    try:
        s.connect((HOST, PORT))
        s.send(b"hello?")            # send a request nobody will ever read
        print(f"  client {i}: connect() SUCCEEDED, request sent")
        clients.append(s)
    except (socket.timeout, OSError) as e:
        print(f"  client {i}: connect() failed: {e!r}")

print("\nAll of those 'successes' are connections the app never saw.")
print("Inspect with:  Get-NetTCPConnection -LocalPort 9104     (Windows)")
print("               ss -tan 'sport = :9104'                  (Linux)")
input("Press Enter to exit...\n")
for s in clients:
    s.close()
listener.close()
```

Output on Linux:

```
listening on 127.0.0.1:9104 with backlog=2; NOT calling accept()
  client 0: connect() SUCCEEDED, request sent
  client 1: connect() SUCCEEDED, request sent
  client 2: connect() SUCCEEDED, request sent
  client 3: connect() SUCCEEDED, request sent
  client 4: connect() SUCCEEDED, request sent

All of those 'successes' are connections the app never saw.
```

And in another terminal:

```
$ ss -tan 'sport = :9104 or dport = :9104'
State    Recv-Q Send-Q  Local Address:Port    Peer Address:Port
LISTEN   4      2           127.0.0.1:9104         0.0.0.0:*
ESTAB    6      0           127.0.0.1:9104       127.0.0.1:44102
ESTAB    6      0           127.0.0.1:9104       127.0.0.1:44104
ESTAB    0      6           127.0.0.1:44102      127.0.0.1:9104
...
```

**Why this works, and what to notice:**

- **Every `connect()` succeeded**, including the ones beyond the backlog. From the client's perspective these are healthy, established connections — it sent its request and is now waiting for a reply that will never come. This is the *hung request* signature of accept-queue overflow.
- On the `LISTEN` row, `Recv-Q` is the **number of completed connections waiting in the accept queue** and `Send-Q` is the **configured backlog**. Seeing `Recv-Q 4 / Send-Q 2` on a LISTEN row is the single clearest "my application is not accepting fast enough" signal there is — and note that these two columns mean something *different* on a LISTEN row than on an ESTAB row (where they're actual buffered byte counts). This inconsistency confuses people constantly.
- On the ESTAB rows, `Recv-Q 6` means six bytes (`hello?`) sitting in the server's receive buffer with nobody reading them.
- The kernel over-delivered on our backlog of 2 — Linux allows some slack. Don't rely on exact counts; rely on the *shape*.

**Honesty caveat:** the exact number of connections that succeed past the backlog varies by kernel version and by `tcp_abort_on_overflow`. The point isn't the number, it's that **the handshake and `accept()` are decoupled**, so "connected" tells the client nothing about whether the application is alive.

### System design — sizing `listen()` backlog for a burst-prone API

**Problem:** you run an HTTP API behind a load balancer. Normally 200 req/s, but a partner's batch job fires 5,000 requests in one second, four times a day. Your app takes 50 ms per request with 16 workers. What backlog do you set, and what happens in the burst?

**Requirements:** don't drop partner requests; don't let the burst make normal traffic time out; fail fast rather than hanging if you truly can't cope.

**The arithmetic first.** 16 workers × (1 / 0.050 s) = **320 requests/second** of service capacity. The burst delivers 5,000 in one second. So 4,680 requests must queue somewhere, and draining them takes 4680 / 320 ≈ **14.6 seconds**. Whatever you do about the backlog, *the last request in that burst waits about 15 seconds.* Establish this before touching a config value: **the backlog does not add capacity. It converts rejection into latency.**

**Alternatives:**

1. **Large backlog (e.g. 8192).** Accept everything, queue it in the kernel.
2. **Small backlog (e.g. 128).** Reject the excess immediately; let the load balancer retry or the client see an error fast.
3. **Moderate backlog (512) plus an explicit application-level concurrency limit that returns `429` with `Retry-After`.**

**The decision: (3).**

**The actual reason:** with (1), all 5,000 requests are accepted and the last one is answered ~15 s later — but the client's timeout is probably 10 s, so it gave up at 10 s, and your worker spends its 50 ms answering *nobody*. You've burned capacity producing responses for abandoned requests, which is how a burst turns into a sustained overload: the queue never drains because everything in it is stale. This is the classic **queue of death**, and a large backlog is how you build one. With (2) you fail fast but *opaquely* — the client sees a connection reset with no information, no `Retry-After`, and typically retries immediately, amplifying the burst. (3) gives you a shallow kernel queue for genuinely short bursts plus an application-level signal (`429`, `Retry-After: 15`) that tells a well-behaved client *when* to come back.

**The trade-off honestly:** (3) means you deliberately reject requests you could theoretically have served. If the partner's batch job is genuinely important, its client has a 60-second timeout, and it honours `Retry-After`, then option (1) with a big backlog serves all 5,000 successfully and (3) rejects thousands unnecessarily. You are trading throughput-under-burst for latency predictability and load-shedding safety.

**Flip condition:** go with a large backlog when (a) clients have timeouts comfortably longer than your worst-case drain time, (b) the work is idempotent and queueable, and (c) you'd rather be slow than wrong — batch ingestion, log shipping, webhook receivers. Go small when latency is the product (interactive APIs) and a stale response is worthless.

**Failure modes:** setting `listen(8192)` while `net.core.somaxconn` is 4096 silently caps you at 4096 — **the kernel does not warn you.** Verify with `ss -tanl` and compare `Send-Q` on the LISTEN row to what you asked for. On the other side, a backlog of 5 in a container that receives a thundering herd on startup produces a burst of connection resets exactly at deploy time, which people misdiagnose as a bad image.

### Case study — the SYN flood as a named historical event

**Panix, September 1996.** Panix, one of the oldest ISPs in New York, was taken offline for roughly a week by a SYN flood from spoofed source addresses. It's the incident that pushed SYN floods from a theoretical paper attack (they had been described in *2600* and *Phrack* that same month) into an operational emergency the whole industry had to answer. The response was the development and deployment of SYN cookies (Bernstein and Schenk, 1996) and of ingress-filtering recommendations that became **BCP 38 / RFC 2827** — "don't forward packets whose source address couldn't legitimately come from that direction."

**The engineering lesson, tied to the concept:** the vulnerability was not a bug in any implementation. It was a *protocol-level asymmetry* — the responder commits resources at step 1 of a 3-step handshake, before the initiator has proven anything. Any protocol with that shape has this problem, which is why the fix generalises: **make the responder stateless until the initiator has proven round-trip reachability.** QUIC's Retry mechanism and Retry token are the same idea, designed in from the start rather than bolted on 15 years later.

The second lesson is bleaker and still current: BCP 38 was published in 2000 and source-address spoofing *still works* in much of the internet, because ingress filtering benefits everyone except the network that has to implement it. That's a tragedy of the commons in protocol deployment, and it's why reflection/amplification attacks (concept 12's UDP material, and Day 12's DNS material) remain viable a quarter-century later.

*Primary sources:* RFC 4987 ("TCP SYN Flooding Attacks and Common Mitigations") documents the attack and defences comprehensively; RFC 2827/BCP 38 for ingress filtering; D. J. Bernstein's SYN cookies page for the original design. Verify current per-OS defaults before relying on them.

---

## 5. Sequence numbers, ACKs, retransmission: how loss actually gets fixed

**Depth: [CORE]**

### Intuition

Now we have agreed starting numbers. The reliability machinery is what we build on top, and it reduces to three questions the sender must answer continuously:

1. **What have I sent?**
2. **What has been acknowledged?**
3. **What have I sent that hasn't been acknowledged for suspiciously long?**

That's it. Everything — retransmission timers, fast retransmit, SACK, congestion control — is refinement of question 3. The naive answer ("wait a while, resend") works but is slow and wasteful; each refinement makes the sender detect loss sooner or resend less.

### The three pointers

The sender's view of the byte stream is three markers:

```
        already ACKed          in flight              not yet sent
   ─────────────────────┬──────────────────────┬──────────────────────►
                        │                      │
                     snd_una                snd_nxt
                 (oldest unacked)         (next to send)

   in-flight bytes = snd_nxt − snd_una      ← this is what congestion control caps
```

And the receiver's view is simpler:

```
   contiguous data received     hole      out-of-order data held
   ────────────────────────┬───────────┬─────────────────────────
                        rcv_nxt
                  (next byte expected — this is what ACKs report)
```

**TCP ACKs are cumulative.** An ACK carrying `ack=5000` means "I have received every byte up to 4999; send me 5000 next." It does *not* mean "I got the segment starting at 5000." This one design choice has large consequences:

- **Robustness:** a lost ACK is harmless. If ACK 3000 is lost but ACK 4000 arrives, the sender learns everything it needed. ACKs are self-healing.
- **Ambiguity:** a cumulative ACK cannot express "I have 0–999 and 2000–2999 but not 1000–1999." It can only say `ack=1000` and keep saying it. The sender knows where the hole *starts* but not what's beyond it — so it may retransmit data the receiver already has. **SACK exists to fix exactly this**, and it was added 15 years later.

### Worked example — a hole, and three ways to fix it

The sender has sent segments 1000–1999, 2000–2999, 3000–3999, 4000–4999. Segment 2000–2999 is lost.

```
Sender                                          Receiver
  │── seq=1000 (1000 bytes) ───────────────────►│  rcv_nxt: 1000→2000
  │◄─────────────────────── ack=2000 ───────────│
  │── seq=2000 ─────────── X  LOST              │
  │── seq=3000 ────────────────────────────────►│  hole! hold 3000-3999
  │◄─────────────────────── ack=2000 ───────────│  "still want 2000"  (dup ACK #1)
  │── seq=4000 ────────────────────────────────►│  hold 4000-4999
  │◄─────────────────────── ack=2000 ───────────│  (dup ACK #2)
  │── seq=5000 ────────────────────────────────►│  hold 5000-5999
  │◄─────────────────────── ack=2000 ───────────│  (dup ACK #3)
  │                                             │
  │  ★ three duplicate ACKs = FAST RETRANSMIT   │
  │── seq=2000 (retransmit) ───────────────────►│  hole filled!
  │◄─────────────────────── ack=6000 ───────────│  everything up to 5999
```

Three distinct mechanisms could have recovered that loss, and understanding why TCP uses the middle one is the whole point.

**(a) Retransmission timeout (RTO) — the fallback.** The sender started a timer when it sent 2000–2999. If no ACK covering it arrives before the timer fires, resend. This *always* works — it's the mechanism of last resort that handles every loss including the case where *everything* is lost. But it's slow: the RTO is at least one RTT plus safety margin, and Linux's minimum RTO is **200 ms** (`TCP_RTO_MIN`), which on a 1 ms datacentre link is 200× the RTT. A single loss recovered by RTO on a fast network is a catastrophic latency outlier.

**(b) Fast retransmit — the workhorse.** The receiver's duplicate ACKs are *information*: each one means "another segment arrived, but it wasn't the one I need." Three duplicate ACKs strongly suggest a single loss rather than reordering, so the sender resends immediately without waiting for the timer. Recovery in about 1 RTT instead of ≥200 ms.

Why three, and not one? **Because reordering exists** (failure mode 3). A single duplicate ACK is much more likely to mean "packets arrived out of order" than "packet lost." Three is an empirical compromise: enough to be confident, few enough to be fast. If TCP retransmitted on the first duplicate ACK, every reordering event on the internet would cause a spurious retransmission *and* a spurious congestion-window reduction, and throughput would collapse on paths with ECMP load balancing.

**(c) SACK — the precision instrument.** With `sackOK` negotiated in the handshake (you saw it in concept 4's capture), the receiver can attach a TCP option listing the *blocks it actually holds*:

```
ack=2000, sack: [3000-5999]
   ↑              ↑
"want 2000"   "and I already have 3000 through 5999, don't resend those"
```

Without SACK, a sender facing multiple holes must essentially retransmit from the hole onward and re-learn, which on a lossy path means re-sending large amounts of data the receiver already has. With SACK it retransmits precisely the missing bytes. *Primary sources:* RFC 2018 (SACK), RFC 6675 (SACK-based loss recovery).

The judgment here: **all three mechanisms coexist because each covers a case the others can't.** Fast retransmit needs *subsequent segments to arrive* in order to generate duplicate ACKs — so it's useless if the loss was the last segment in a burst, or if the whole burst was lost. That's exactly when RTO saves you. And SACK is an optimisation on top of both, not a replacement. A reader who says "SACK made fast retransmit obsolete" has the wrong model.

### How the RTO is computed — and why it can't be a constant

The RTO must adapt: 200 ms is far too long for a datacentre and far too short for a satellite link. TCP measures RTT continuously and maintains two smoothed estimates (Jacobson/Karels 1988; standardised in RFC 6298):

```
SRTT    = (1 − α) × SRTT    + α × RTT_sample              α = 1/8
RTTVAR  = (1 − β) × RTTVAR  + β × |SRTT − RTT_sample|     β = 1/4
RTO     = SRTT + max(G, 4 × RTTVAR)        (G = clock granularity)
RTO     = clamp(RTO, 200 ms, 120 s)        (Linux: min 200 ms, max 120 s)
```

Two features worth understanding rather than memorising:

**Why include variance, not just the mean?** Because a path with a stable 50 ms RTT and one whose RTT bounces between 20 ms and 300 ms need very different timeouts. Setting `RTO = 2 × SRTT` would cause constant spurious retransmits on the jittery path. The `4 × RTTVAR` term makes the timeout proportional to how *unpredictable* the path is. **This is a broadly reusable idea: any timeout you set on a variable-latency operation should be a function of both its mean and its variance, not a constant you picked.** Timeouts in your HTTP clients, database drivers, and agent tool calls all have this shape, and almost nobody does it.

**Exponential backoff on repeated failure.** Each time the RTO fires without an ACK, the RTO doubles: 200 ms, 400 ms, 800 ms, 1.6 s… This is what makes a TCP connection survive a multi-second network outage (it keeps trying, less and less often) and what makes a dead peer take a long time to detect (Linux retries roughly 15 times via `net.ipv4.tcp_retries2`, taking on the order of 13–30 minutes before giving up). **That number is why a service can sit "connected" to a dead machine for a quarter of an hour**, and it's the setup for concept 11.

**Karn's algorithm** — one subtlety worth naming: RTT samples from *retransmitted* segments must be discarded, because you can't tell whether the ACK was for the original or the copy, and guessing wrong corrupts your estimator in exactly the wrong direction. Timestamps (the `TS val` option in the capture) sidestep this by letting the receiver echo which send the ACK corresponds to.

### Runnable example — watching retransmissions and the two kinds of death

Loss doesn't reliably happen on demand, so we manufacture it the honest way: kill the peer mid-transfer and observe what the sender does.

```python
# retransmit_observe.py — start a transfer, kill the receiver, watch what happens.
# Capture alongside:  Wireshark filter  tcp.port == 9105
#                     or  sudo tcpdump -i lo -n 'tcp port 9105'
import socket
import subprocess
import sys
import threading
import time

HOST, PORT = "127.0.0.1", 9105


def start_receiver():
    """Run the receiver in a child process so we can hard-kill it mid-transfer."""
    code = f"""
import socket, time
lis = socket.socket()
lis.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
lis.bind(("{HOST}", {PORT})); lis.listen(1)
conn, _ = lis.accept()
while True:
    b = conn.recv(65536)
    if not b: break
    time.sleep(0.05)          # read slowly so data stays in flight
"""
    return subprocess.Popen([sys.executable, "-c", code])


proc = start_receiver()
time.sleep(0.6)

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect((HOST, PORT))
print("connected; streaming...")


def killer():
    time.sleep(1.5)
    print("\n>>> hard-killing the receiver process now <<<")
    proc.kill()          # no FIN, no clean close — the process just vanishes


threading.Thread(target=killer, daemon=True).start()

chunk = b"y" * 32768
sent = 0
try:
    while sent < 200 * 1024 * 1024:
        s.send(chunk)
        sent += len(chunk)
except OSError as e:
    print(f"\nsend() failed after {sent // 1024} KB")
    print(f"  exception: {e!r}")
    print(f"  errno: {getattr(e, 'errno', None)}")
s.close()
```

Typical output on Windows:

```
connected; streaming...

>>> hard-killing the receiver process now <<<

send() failed after 3168 KB
  exception: ConnectionResetError(10054, 'An existing connection was forcibly closed by the remote host', None, 10054, None)
  errno: 10054
```

On Linux you'd see `ConnectionResetError(104, 'Connection reset by peer')`.

**Why this works, and what the packet capture shows:**

- Killing the process kills it without closing the socket. But the **OS still owns the socket**, and an OS whose process has died responds to further data with an **RST** — so we get a fast, clean error. This is the *good* case.
- In the capture you'll see the sender's segments continue briefly, then an `R` (RST) flag from the receiver's kernel, then `send()` failing.
- **The reason we got an error at all is that the receiving OS was alive to send the RST.** Change the experiment so the *machine* disappears (pull the cable, `docker kill` the container, add a firewall DROP rule) and the behaviour is completely different: no RST, so the sender retransmits with exponentially increasing backoff for **many minutes** before giving up.

Try the black-hole variant on Windows (as Administrator):

```powershell
New-NetFirewallRule -DisplayName "blackhole-9105" -Direction Inbound -LocalPort 9105 -Protocol TCP -Action Block
# ...run the sender, watch it retransmit for minutes with no error...
Remove-NetFirewallRule -DisplayName "blackhole-9105"
```

**This is the single most important distinction in TCP failure handling, and it deserves to be stated flatly:**

| Peer failure | What the peer's OS does | What your `send()`/`recv()` sees | How long |
|---|---|---|---|
| Process crashes, OS alive | Sends RST | `ConnectionResetError` | milliseconds |
| Process closes cleanly | Sends FIN | `recv()` returns `b""` | milliseconds |
| Machine powers off / cable pulled | Nothing | Retransmits, then error | **~13–30 min** |
| Firewall silently drops | Nothing | Retransmits, then error | **~13–30 min** |
| Load balancer forgets connection | Nothing (or a late RST) | Hangs, or RST on next write | seconds to minutes |

The bottom three rows are indistinguishable from "the network is briefly slow," which is precisely why TCP waits so long — and precisely why **you cannot rely on TCP to tell you the peer is gone in a useful timeframe.** Your application needs its own timeouts and its own liveness signal. Concept 11 is about building those.

### In production (condensed) — retransmission as a metric

**Best practice:** treat the retransmission *rate* as a first-class network health metric. A healthy internet path retransmits well under 1% of segments; a healthy datacentre path essentially 0%. Rising retransmits mean congestion or a failing link, and they mean it *before* users complain, because TCP absorbs the damage as latency first.

**Top failure mode:** a small, persistent retransmission rate (0.5–2%) on one path silently destroying throughput. Because loss drives congestion control (concept 7), a path with 1% loss has a hard mathematical throughput ceiling that no amount of bandwidth fixes. People buy bigger links and see no improvement; the correct diagnosis was a dodgy cable or an overloaded intermediate queue.

**How to see it:**

```powershell
netstat -s -p tcp        # includes segments retransmitted
Get-NetTCPConnection | Group-Object State | Select-Object Name,Count
```

```bash
nstat -az | grep -iE 'retrans|TCPLostRetransmit'
ss -tin                  # per-connection: rtt, cwnd, retrans, bytes_retrans
```

---

## 6. Flow control: "don't drown *me*"

**Depth: [CORE]**

### Intuition

The sender now knows how to fix loss. But there's a failure it should *prevent* rather than fix: sending faster than the receiver can absorb. A laptop on Wi-Fi asking a datacentre server for a large file could easily be sent data 100× faster than it can process, and every byte beyond the receive buffer's capacity is discarded — retransmitted, discarded again, in a pointless loop that wastes the whole path's capacity.

The fix is almost embarrassingly direct: **the receiver tells the sender how much room it has, in every single ACK.** That's the 16-bit `window` field in the TCP header, and it means "I currently have this many bytes of free receive-buffer space." The sender is forbidden from having more unacknowledged data in flight than that.

```
in flight (snd_nxt − snd_una)  ≤  min(receiver's advertised window, congestion window)
                                       ↑                              ↑
                                  flow control                  congestion control
                                  (concept 6)                     (concept 7)
```

Two independent limits, both enforced simultaneously, and confusing them is the most common conceptual error in this whole topic. **Flow control protects the receiver's buffer. Congestion control protects the network's queues.** They're computed by different parties, respond to different signals, and either can be the binding constraint. Keep them separate in your head from the start and concept 7 becomes easy.

### Analogy: the conveyor belt and the packer

A factory conveyor delivers boxes to a packer. The packer has a small table — space for ten boxes. Every time they finish a box, they hold up a card showing how many free slots remain: "7", "4", "1", "0". The conveyor operator watches the card and never sends more boxes than the number shown.

**Where the analogy breaks:** the card is not instantaneous. It travels back along the belt and takes *half a round-trip* to reach the operator. By the time the operator reads "4", the packer may have cleared the table (so the real answer is 10) or received four more boxes already in transit (so the real answer is 0). **Every window value the sender sees is stale by half an RTT.** This is why TCP is conservative about windows, why a zero window needs an explicit probe to escape, and why the receiver must avoid advertising tiny windows (silly window syndrome, below) — a stale small number throttles the sender far longer than the condition that caused it.

### Worked example — the window closing, and the deadlock it nearly causes

A client with a 64 KB receive buffer downloads from a server, and the client's application is slow to read:

```
Client recv buffer: 65536 bytes free

Server ──── 16 KB data ─────►  buffer: 49152 free
Client ◄─── ack, win=49152 ───
Server ──── 16 KB data ─────►  buffer: 32768 free
Client ◄─── ack, win=32768 ───
Server ──── 16 KB data ─────►  buffer: 16384 free
Client ◄─── ack, win=16384 ───
Server ──── 16 KB data ─────►  buffer:     0 free    ← app hasn't read ANYTHING
Client ◄─── ack, win=0 ────────  "STOP"
Server: transmission halted. Nothing more may be sent.

        ... app finally calls recv(32768) ...

Client ──── ack, win=32768 ───►  "window update — you may send again"
Server ──── resumes ─────────►
```

Two mechanisms to name:

**The zero-window probe.** When the window is zero, the sender stops. But now consider: the window update ("you may send again") is an ACK, and **pure ACKs are not retransmitted** — TCP has no reliability mechanism for them. If that window update is lost, the sender waits forever for permission that will never come and the receiver waits forever for data that will never arrive. **Deadlock.** The fix is the *persist timer*: while the window is zero, the sender periodically sends a 1-byte probe (or a zero-length segment) purely to elicit a fresh window advertisement. It's a deadlock-breaker, and it illustrates a general principle worth stealing: **any protocol where one side waits for an unreliable notification needs a poll as a backstop.** You will write this exact pattern by hand in agent systems, webhook receivers, and job queues.

**Silly window syndrome.** Suppose the receiving application reads 1 byte at a time. After each read, one byte of space frees, and a naive receiver advertises `win=1`. The sender dutifully sends a 1-byte segment — 1 byte of payload inside a 40-byte IP+TCP header, so **2.4% efficiency**. The connection technically works and is catastrophically wasteful.

Two fixes, one on each side:

- **Receiver side (Clark's solution):** don't advertise a window increase until it's at least one MSS or half the buffer. Lie about having no room until you have *useful* room.
- **Sender side (Nagle's algorithm):** don't send a small segment while an earlier small segment is still unacknowledged — coalesce instead. That's concept 8, and it has an unfortunate interaction we'll get to.

### The window scale option: why 16 bits stopped being enough

The window field is 16 bits, so the maximum advertisable window is **65535 bytes**. Now do the bandwidth-delay arithmetic:

```
max throughput = window / RTT
```

| Path | RTT | Ceiling with a 64 KB window |
|---|---|---|
| Same datacentre | 0.5 ms | 131 MB/s ≈ 1 Gbit/s |
| Same country | 20 ms | 3.3 MB/s ≈ 26 Mbit/s |
| Transatlantic | 80 ms | 819 KB/s ≈ 6.5 Mbit/s |
| Geostationary satellite | 600 ms | 109 KB/s ≈ 0.9 Mbit/s |

**Look at the transatlantic row.** On a gigabit link across the Atlantic, a 64 KB window caps you at 6.5 Mbit/s — **0.65% of the available bandwidth**. The link sits idle 99% of the time, waiting for ACKs. This is the "long fat network" problem, and it is not a rounding error; it's the difference between a transfer taking a minute and taking two and a half hours.

The fix is the **window scale** option (RFC 7323, formerly RFC 1323): a value 0–14 negotiated in the SYN, applied as a left-shift to every subsequent window advertisement. `wscale 7` (which you saw in concept 4's real capture) multiplies by 128, allowing windows up to 65535 × 128 ≈ **8 MB**. The maximum scale of 14 gives about 1 GB.

Three properties that matter operationally:

1. **It is negotiated only in the SYN.** If either side omits it, scaling is off for the whole connection, permanently. There is no later opportunity.
2. **It applies to *every* window field except the SYNs'.** This is why packet 3 in our capture said `win 512` — meaning 65536. Misreading scaled windows is a classic capture-analysis error.
3. **Middleboxes have historically broken it.** A firewall that rewrites windows without understanding the scale factor produces a connection where each side computes a different window size. The symptom is a connection that establishes fine and then stalls or trickles. Rarer now than in 2005, but still the reason some enterprise networks have bizarre throughput.

### Runnable example — measuring the window ceiling yourself

You can watch flow control bite by *shrinking the window on purpose* and measuring the throughput ceiling. The number falling out of the formula is the whole lesson.

```python
# window_ceiling.py — show that throughput ≈ window / RTT, by capping the window.
# Stdlib only. Uses SO_RCVBUF to force a small receive window — which DOES
# disable auto-tuning, and that is deliberately part of the demonstration.
import socket
import threading
import time

HOST, PORT = "127.0.0.1", 9106
PAYLOAD = 40 * 1024 * 1024          # 40 MB


def receiver(rcvbuf_bytes: int, results: dict):
    lis = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    lis.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    # Set SO_RCVBUF BEFORE bind/listen so accepted sockets inherit it.
    # Setting it after accept() is too late: the window was already advertised
    # in the SYN-ACK.
    lis.setsockopt(socket.SOL_SOCKET, socket.SO_RCVBUF, rcvbuf_bytes)
    lis.bind((HOST, PORT)); lis.listen(1)
    conn, _ = lis.accept()
    actual = conn.getsockopt(socket.SOL_SOCKET, socket.SO_RCVBUF)
    got = 0
    t0 = time.perf_counter()
    while got < PAYLOAD:
        b = conn.recv(1 << 20)
        if not b:
            break
        got += len(b)
    results.update(bytes=got, seconds=time.perf_counter() - t0, rcvbuf=actual)
    conn.close(); lis.close()


for requested in (8 * 1024, 64 * 1024, 1024 * 1024):
    results: dict = {}
    t = threading.Thread(target=receiver, args=(requested, results))
    t.start(); time.sleep(0.3)

    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((HOST, PORT))
    buf = b"z" * (1 << 16)
    remaining = PAYLOAD
    while remaining > 0:
        remaining -= s.send(buf[:min(len(buf), remaining)])
    s.close(); t.join()

    mb = results["bytes"] / 1e6
    print(f"asked SO_RCVBUF={requested:>8}  kernel gave {results['rcvbuf']:>8}  "
          f"→ {mb:6.1f} MB in {results['seconds']:5.2f}s = {mb / results['seconds']:7.1f} MB/s")
```

Output on loopback (Linux):

```
asked SO_RCVBUF=    8192  kernel gave    16384  →   40.0 MB in  1.94s =    20.6 MB/s
asked SO_RCVBUF=   65536  kernel gave   131072  →   40.0 MB in  0.51s =    78.4 MB/s
asked SO_RCVBUF= 1048576  kernel gave  2097152  →   40.0 MB in  0.19s =   210.5 MB/s
```

**Why this works, line by line:**

- `setsockopt(SOL_SOCKET, SO_RCVBUF, n)` on the **listening** socket, before `listen()`, is the trick: accepted sockets inherit the buffer size. This is a real and commonly-missed detail.
- **The kernel doubled our request.** We asked for 8192 and got 16384. Linux reserves roughly half the buffer for bookkeeping overhead and reports the doubled figure — documented behaviour, not a bug. Windows does not double. Always read the value back rather than trusting what you set.
- Throughput scales with the buffer, roughly as `window / RTT` predicts. On loopback the RTT is microseconds so the absolute numbers are huge, but **the proportionality is the point** — 8× the buffer bought roughly 10× the throughput.
- **Honesty caveat, twice over.** (1) Loopback has no meaningful RTT, so this measures buffer-copy efficiency as much as flow control; on a real 50 ms path the small-window run would be dramatically worse and the formula would predict it almost exactly. (2) By calling `setsockopt` we **disabled auto-tuning**, which is why even the large-buffer case isn't as fast as leaving the kernel alone — a live demonstration of the pessimisation warned about in concept 3.

To do this properly on Linux, add real latency and rerun:

```bash
sudo tc qdisc add dev lo root netem delay 25ms    # 25 ms each way → 50 ms RTT
python window_ceiling.py
sudo tc qdisc del dev lo root
```

Now the numbers track `window / 0.050` closely, and the 16 KB run lands near 320 KB/s. **Run this.** Watching a formula predict a measurement is what converts it from something you've read into something you know.

### System design — choosing a window strategy for cross-region replication

**Problem:** you replicate a write-ahead log from `us-east` to `ap-southeast` — RTT 220 ms, provisioned 500 Mbit/s, and about 40 MB/s of log to ship. It's running at 6 MB/s and falling behind. Nothing looks saturated: CPU idle, link at 10% utilisation, retransmits near zero.

**Diagnosis first, config second.** Do the arithmetic before touching anything:

```
observed: 6 MB/s × 0.220 s = 1.32 MB in flight
needed:  40 MB/s × 0.220 s = 8.80 MB in flight      ← required BDP
```

You need 8.8 MB in flight and you have 1.32 MB. The window is the bottleneck, and no amount of extra bandwidth changes that. (If `wscale` were absent entirely you'd be capped at 65535 / 0.220 = 298 KB/s — so scaling is clearly on, just not scaled far enough.)

**Alternatives:**

1. **Raise the kernel's `tcp_rmem`/`tcp_wmem` maxima to ≥16 MB and let auto-tuning use them.**
2. **Open N parallel TCP connections** and stripe the log across them, so N windows sum to the needed BDP.
3. **Switch to a UDP-based transport** with congestion control tuned for long fat networks (QUIC, or something like UDT/Aspera).

**The decision: (1), then (2) if (1) proves insufficient.**

**The actual reason:** (1) is a one-line change that directly addresses the measured cause, and Linux's auto-tuning will grow *only this connection* to the 8.8 MB it demonstrably needs while leaving thousands of short local connections small. It's the minimal intervention that matches the diagnosis.

**The trade-off honestly:** a single connection with an 8.8 MB window is a single congestion-control instance and a single point of failure. If loss appears, that one connection halves its window and your replication throughput halves with it; option (2)'s N connections degrade more gracefully (only the affected stream backs off) and recover faster (N connections in slow start ramp faster than one — which is exactly why download accelerators use parallel streams). You're trading robustness-under-loss for simplicity. Option (2) is also, bluntly, a way of taking more than your fair share of a congested path — fine on a private link, antisocial on a shared one.

**Flip condition:** go to parallel connections when (a) the path has non-trivial loss so a single congestion window can't stay open, (b) you've hit the kernel's practical buffer ceiling, or (c) you need to survive one connection resetting without a throughput cliff. Go to (3) — a userspace transport — when you additionally need loss recovery *not* to head-of-line-block (concept 10), which on a 220 ms path is a large win: with TCP, one lost segment stalls everything behind it for at least 220 ms.

**Failure modes:** raising buffer maxima globally and then hitting a connection surge → kernel memory exhaustion (watch `/proc/net/sockstat`, not process RSS). Setting `SO_RCVBUF` explicitly instead of raising the max → auto-tuning disabled, and you now own the sizing for every path forever. Enabling parallel streams without ensuring the receiver can reassemble out-of-order chunks → data corruption that presents as a storage bug.

### Case study — window scaling and the "fast link, slow transfer" class of incident

There is no single famous named public postmortem for window-scale misconfiguration in the way there is for BGP or DNS outages — it's a chronic, unglamorous problem rather than a headline event, and I'd rather say so than invent one. What *is* well documented is the underlying phenomenon and its measurement:

**The documented artefact:** RFC 7323 and its predecessor RFC 1323 ("TCP Extensions for High Performance," 1992) exist precisely because measured throughput on high-bandwidth long-distance paths was a small fraction of link capacity, and the RFCs state the window-size limitation explicitly as the motivation. The operational literature around research and education networks documents the same class of problem extensively under the name **"the wizard gap"** — the measured gap between the throughput a network expert can achieve on a path and what a default-configured host achieves, historically an order of magnitude or more, almost entirely attributable to buffer and window configuration. ESnet's `fasterdata.es.net` knowledge base is the standard practical reference and documents case after case of scientific data transfers running at 1–5% of link capacity until buffers were fixed.

**The engineering lesson tied to the concept:** throughput on a long path is a function of `window / RTT`, and *both* terms are usually invisible in the dashboards people look at. Link utilisation looks fine (it's low!), CPU looks fine, error rates look fine — because none of those is the constraint. **The lesson is to make the BDP calculation a reflex whenever you hear "the link is fast but the transfer is slow."** It costs thirty seconds and it identifies the cause in a case where every conventional metric reads "healthy."

*Verify current:* modern OS defaults are far better than 2005's, and Linux/Windows auto-tuning handles most paths. The residual cases are (a) explicitly-set buffers overriding auto-tuning, (b) middleboxes mangling the scale option, and (c) genuinely extreme BDPs — satellite, or intercontinental 10 Gbit/s.

---
## 7. Congestion control: "don't drown the *network*"

**Depth: [CORE]**

### Intuition

Flow control solved a problem where the constrained resource had a spokesperson: the receiver can *tell you* its buffer is full, because it owns that buffer. Congestion is harder for one brutal reason: **the queues that overflow belong to routers in the middle, and routers do not talk to you.**

A router with a full queue drops your packet. It sends no message, files no complaint, and has no idea who you are. The only way you learn that you were sending too fast is that your data disappeared. So TCP's congestion control is built on a startling premise:

> **Packet loss is the congestion signal.** The sender infers the state of queues it cannot see, from the absence of acknowledgements it expected.

That is an inference from silence, and it explains both TCP's ingenuity and its Achilles heel. When loss has a *different* cause — radio interference on Wi-Fi, a flaky cable — TCP misreads it as congestion, slows down, and makes things worse for no reason. Every "why is my Wi-Fi download slow when the signal is fine?" question is downstream of that.

### The historical motivation: congestion collapse actually happened

This is not a theoretical concern, and the history is the best possible motivation.

In **October 1986**, the link between Lawrence Berkeley Laboratory and UC Berkeley — a few hundred metres apart — dropped from its nominal 32 kbit/s to **40 bits per second**. A factor of 1000. Van Jacobson investigated and found something worse than a broken link: the network was busy, the link was saturated, and almost none of the traffic was *useful*. It was retransmissions of data that had already been sent, competing with the retransmissions of other senders, all of them retransmitting because their packets were being dropped by a queue that was full of retransmissions.

This is **congestion collapse**: a stable, self-sustaining state where a network carries traffic at capacity and delivers almost no goodput. It's a positive feedback loop —

```
queue fills → packets dropped → senders retransmit → more traffic → queue fills more
     ▲                                                                      │
     └──────────────────────────────────────────────────────────────────────┘
```

— and the crucial observation is that **it does not resolve itself.** Every sender, acting in its own rational interest (my packet didn't arrive, so I should resend it), makes the aggregate worse. The internet nearly died of it.

Jacobson's 1988 paper *Congestion Avoidance and Control* introduced slow start, congestion avoidance, fast retransmit, and the RTT variance estimator we saw in concept 5. It is one of the most consequential papers in computing, and it works by adding one rule: **every sender must react to loss by slowing down.** *Primary source:* Jacobson, SIGCOMM '88; standardised in RFC 5681 (TCP Congestion Control).

The generalisable lesson, and it's one that reappears in every distributed system you will build: **an unbounded retry policy is not resilience, it is an attack on your own infrastructure.** Retry storms in microservices, thundering-herd cache stampedes, and agent loops that retry a failing tool call forever are the same failure with different labels. Jacobson's answer — back off multiplicatively, probe back up gently — is the template.

### The congestion window

The sender maintains a second limit alongside the receiver's window:

```
cwnd  = congestion window   — the sender's own estimate of what the network can take
rwnd  = receive window      — what the receiver said it can take

bytes in flight ≤ min(cwnd, rwnd)
```

`rwnd` is *told* to the sender. `cwnd` is *guessed* by the sender, continuously, from the pattern of ACKs and losses. All of congestion control is the algorithm for adjusting `cwnd`.

### Analogy: driving into fog with no speedometer

You're driving on an unfamiliar road in thick fog. You cannot see the road ahead and you have no speedometer. Your only feedback is: if you go too fast you clip a kerb (loss). So you accelerate gently, and every time you clip something you halve your speed and start creeping up again.

**Where the analogy breaks — two ways, both important.** First, you are not the only driver: dozens of cars share the road, and the kerb you clipped may have been caused by someone else's overcrowding. TCP congestion control is fundamentally a **distributed** algorithm whose correctness depends on *everyone* running something compatible — which is why a "greedy TCP" that never backed off would work brilliantly for one user and destroy the network for everyone. Second, in the fog analogy the kerb is a real obstacle at a real position; in networking, the loss might not have been congestion at all (wireless noise), so you slowed down for nothing. Both breaks are load-bearing: the first explains TCP-friendliness as a design constraint, the second explains why BBR exists.

### Phase 1: slow start (which is not slow)

A new connection knows nothing about the path. It could be 10 Mbit/s or 10 Gbit/s. Starting at full speed would blast a burst into an unknown network; starting at one packet and adding one per RTT would take forever on a fast path.

Slow start's answer: **start small, double every RTT.**

```
cwnd starts at IW (initial window) = 10 segments on Linux (RFC 6928)
Each ACK received → cwnd += 1 segment
Effect: cwnd DOUBLES every round trip.

RTT 1:  cwnd = 10 segments  ≈  14 KB
RTT 2:  cwnd = 20           ≈  29 KB
RTT 3:  cwnd = 40           ≈  58 KB
RTT 4:  cwnd = 80           ≈ 116 KB
RTT 5:  cwnd = 160          ≈ 232 KB
RTT 6:  cwnd = 320          ≈ 464 KB
```

The name is a historical accident and actively misleads people: **slow start is exponential growth.** It's "slow" only relative to the alternative it replaced (send everything immediately). It's called slow start because it *starts* slow, not because it *is* slow.

The consequence that matters for web performance: a small response finishes *inside* slow start and never reaches the path's capacity. With `IW = 10` and MSS 1460, the first round trip can carry about **14.6 KB**. So:

| Response size | Round trips needed (in slow start) |
|---|---|
| 14 KB | 1 |
| 44 KB | 2 |
| 100 KB | 3 |
| 220 KB | 4 |

On a 100 ms RTT connection, a 100 KB response takes at *minimum* 300 ms of transfer time regardless of bandwidth — plus 100 ms handshake, plus 200 ms TLS. **This is why "make the critical response fit in the first 14 KB" is real web-performance advice and not superstition**, and it's why the initial window was raised from 3 to 10 segments (RFC 6928, based on Google's measurements) — a change that measurably sped up the web.

It's also the reason connection reuse matters so much: a warm connection has already grown its `cwnd`, so a request on it starts fast. **A new connection is slow not mainly because of the handshake but because it has forgotten everything it learned about the path.**

### Phase 2: congestion avoidance (AIMD)

Exponential growth cannot continue — it would overshoot enormously. So at some threshold (`ssthresh`), the sender switches from doubling per RTT to adding *one segment* per RTT:

```
if cwnd < ssthresh:   cwnd += 1 per ACK          (slow start — exponential)
else:                 cwnd += 1 per RTT          (congestion avoidance — linear)
```

And when loss is detected:

```
Loss via 3 duplicate ACKs (fast recovery):     ssthresh = cwnd / 2 ;  cwnd = ssthresh
Loss via RTO (something is badly wrong):       ssthresh = cwnd / 2 ;  cwnd = 1 segment
```

Notice the asymmetry, because it's a deliberate judgment call: a loss detected by duplicate ACKs means *data is still flowing* (you received three ACKs!), so halve and carry on. A loss detected by timeout means *nothing is getting through*, which is a far more serious signal, so collapse to one segment and restart slow start. TCP is calibrating its panic to its evidence.

This is **AIMD — Additive Increase, Multiplicative Decrease.** The shape it draws over time is the famous sawtooth:

```
cwnd
  │                                    ╱│
  │                    ╱│            ╱  │
  │        ╱│        ╱  │          ╱    │
  │      ╱  │      ╱    │        ╱      │
  │    ╱    │    ╱      │      ╱        │
  │  ╱      │  ╱        │    ╱          │
  │╱ slow   │╱          │  ╱            │
  │  start  ▼ loss      ▼ loss          ▼ loss
  └──────────────────────────────────────────► time
     exponential   linear (additive increase), halve on loss
```

**Why multiplicative decrease and additive increase, rather than the reverse?** This is the deepest result in the area, and it's worth stating because it explains why TCP is *fair*. Chiu and Jain (1989) proved that AIMD converges to a fair, efficient allocation among competing flows, while AIAD (additive both ways) and MIMD (multiplicative both ways) do not. The intuition: multiplicative decrease means a flow with a *large* window gives up more absolute bandwidth on loss than a flow with a small window, so repeated loss events push unequal flows toward equality. Additive increase means everyone gains at the same absolute rate, which doesn't re-introduce inequality. **Fairness is an emergent property of the specific arithmetic, not a rule anyone enforces**, and that's a genuinely beautiful piece of design.

### Modern algorithms: CUBIC and BBR

Classic Reno AIMD has a known scaling problem. Consider a 10 Gbit/s link with 100 ms RTT: the BDP is 125 MB, roughly 83,000 segments. After a single loss, `cwnd` halves to 41,500 and then grows by *one segment per RTT*. Recovering to 83,000 segments takes 41,500 RTTs — at 100 ms each, that's **69 minutes.** Reno cannot use a modern fat long path at all.

**CUBIC** (RFC 8312, the Linux default since 2.6.19) fixes this by making window growth a **cubic function of time since the last loss**, rather than a linear function of RTTs. The shape: grow fast, flatten out as you approach the window size where loss last occurred (the "plateau" — probably the path's real capacity), then grow fast again if you get past it. Two properties fall out: growth is *RTT-independent* (so short-RTT and long-RTT flows compete more fairly than under Reno, where short-RTT flows grow much faster), and recovery on fat pipes takes seconds rather than an hour.

**BBR** (Google, 2016) is a genuinely different philosophy and worth understanding as a *rejection of the founding premise*. Instead of treating loss as the congestion signal, BBR continuously estimates two things — the path's **bottleneck bandwidth** and its **minimum RTT** — and paces sending at exactly `bandwidth × min_RTT`. It aims to keep the pipe full and the queues *empty*.

Why that's a big deal: loss-based control *must fill a queue to detect it*, so it necessarily operates with a standing queue, which means added latency for everyone sharing that queue (this is bufferbloat, concept 14). BBR's model-based approach can achieve high throughput without filling buffers, and it doesn't collapse on paths with random (non-congestive) loss — which makes it dramatically better on lossy wireless and long international paths.

**The honest trade-off, and this one is genuinely contested:** BBRv1 was measured to be aggressive against CUBIC flows sharing a bottleneck — it can take a disproportionate share, because it isn't backing off on the loss signal CUBIC responds to. BBRv2/v3 address this (they incorporate loss and ECN signals), but "BBR is strictly better" is a claim you should not make. It's better *for the flow that runs it*, on lossy or high-BDP paths; whether it's better for the network depends on what else is sharing the bottleneck.

**Flip condition, stated practically:** switch to BBR when your traffic crosses long or lossy paths (CDN edge to origin, intercontinental replication, mobile clients) and you control both ends or at least the sender. Stay on CUBIC when you're inside a datacentre with negligible loss (there's little to gain), or when you're a good-citizen service on a shared bottleneck with many CUBIC flows and you'd rather not be the flow that takes more than its share.

```bash
# Linux — inspect and change
sysctl net.ipv4.tcp_congestion_control          # cubic (usual default)
sysctl net.ipv4.tcp_available_congestion_control
sudo sysctl -w net.ipv4.tcp_congestion_control=bbr
```

Windows: `Get-NetTCPSetting` shows `CongestionProvider`. Modern Windows defaults to CUBIC for internet-facing templates (it was NewReno historically, and Windows 11/Server 2022 moved to CUBIC). *Verify on your build* — this genuinely changed recently and any specific claim ages fast.

### Worked example — computing what a connection can actually achieve

Two calculations that will serve you for a whole career. Do them by hand once.

**(1) The BDP requirement.** Sustaining throughput `T` over RTT `R` requires `T × R` bytes in flight:

```
Target: 100 Mbit/s over an 80 ms transatlantic path
  = 12.5 MB/s × 0.080 s = 1 MB in flight

Required: cwnd ≥ 1 MB  AND  rwnd ≥ 1 MB  AND  send buffer ≥ 1 MB
          (all three, because the effective limit is the minimum)
```

**(2) The Mathis equation — the loss ceiling.** This is the one that surprises people, and it is the most useful single formula in practical networking:

```
                       MSS
Throughput  ≈  ─────────────────
                RTT × √p            (p = packet loss probability)
```

Plug in MSS 1460 bytes, RTT 80 ms:

| Loss rate | Max throughput |
|---|---|
| 0.001% (1 in 100,000) | ~577 Mbit/s |
| 0.01% | ~182 Mbit/s |
| 0.1% | ~58 Mbit/s |
| 1% | ~18 Mbit/s |
| 3% | ~10 Mbit/s |

**Read the 1% row and sit with it.** On an 80 ms path with 1% loss, a single TCP connection cannot exceed roughly 18 Mbit/s — *even on a 10 Gbit/s link*. You could upgrade to 100 Gbit/s and see no change whatsoever. The bottleneck is the mathematics of AIMD interacting with the loss rate, and the only fixes are: reduce loss, reduce RTT, use more parallel connections, or use a congestion controller that doesn't treat loss this way (BBR).

This is the formula to reach for whenever someone says "we have plenty of bandwidth, why is it slow?" It converts a vague complaint into a testable hypothesis in one line. *Primary source:* Mathis, Semke, Mahdavi, Ott, "The Macroscopic Behavior of the TCP Congestion Avoidance Algorithm," ACM SIGCOMM CCR 1997. (It's an approximation for Reno; CUBIC does better on high-BDP paths, so treat it as a lower bound and an order-of-magnitude tool, not a precise prediction.)

### Runnable example — reading real congestion state from a live connection

This is Linux-only (Windows doesn't expose `TCP_INFO` in the same way), and it's worth showing because *reading `cwnd` from a running connection* is the single most useful diagnostic skill in this section.

```python
# tcp_info.py — Linux only. Reads the kernel's TCP_INFO struct from a live socket
# to display cwnd, RTT estimates, and retransmit counts as they evolve.
import socket
import struct
import sys
import threading
import time

if not sys.platform.startswith("linux"):
    sys.exit("TCP_INFO layout is Linux-specific; on Windows use `ss` in WSL "
             "or Get-NetTCPConnection (which exposes far less).")

HOST, PORT = "127.0.0.1", 9107

# Layout of the first fields of struct tcp_info (linux/tcp.h). Field order is
# stable for these early members across kernel versions; later members have been
# appended over time, which is why we only parse the prefix we need.
FMT = "BBBBBBBI" + "IIIIIIIIIIIIIIIIIIIIIIIIIIIII"


def show(sock, label):
    raw = sock.getsockopt(socket.IPPROTO_TCP, socket.TCP_INFO, 256)
    vals = struct.unpack_from(FMT, raw)
    # Indices from struct tcp_info after the 8-byte header of u8 fields:
    #  ... rto, ato, snd_mss, rcv_mss, unacked, sacked, lost, retrans, fackets,
    #      last_data_sent, last_ack_sent, last_data_recv, last_ack_recv,
    #      pmtu, rcv_ssthresh, rtt, rttvar, snd_ssthresh, snd_cwnd, advmss, ...
    rto, ato, snd_mss, rcv_mss, unacked, sacked, lost, retrans = vals[8:16]
    rtt, rttvar, snd_ssthresh, snd_cwnd, advmss = vals[21:26]
    print(f"{label:>10} | cwnd={snd_cwnd:>6} segs  ssthresh={snd_ssthresh:>10}  "
          f"rtt={rtt/1000:6.2f}ms  var={rttvar/1000:5.2f}ms  "
          f"mss={snd_mss:>5}  retrans={retrans}  lost={lost}")


def sink():
    lis = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    lis.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    lis.bind((HOST, PORT)); lis.listen(1)
    conn, _ = lis.accept()
    while conn.recv(1 << 20):
        pass
    conn.close(); lis.close()


threading.Thread(target=sink, daemon=True).start()
time.sleep(0.3)

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect((HOST, PORT))
show(s, "connected")

chunk = b"q" * (1 << 16)
sent = 0
t0 = time.perf_counter()
next_report = 0.05
while time.perf_counter() - t0 < 1.0:
    sent += s.send(chunk)
    if time.perf_counter() - t0 > next_report:
        show(s, f"{next_report:.2f}s")
        next_report += 0.15
s.close()
print(f"\nsent {sent / 1e6:.1f} MB")
```

Representative output:

```
 connected | cwnd=    10 segs  ssthresh=2147483647  rtt=  0.01ms  var= 0.01ms  mss=32741  retrans=0  lost=0
     0.05s | cwnd=   163 segs  ssthresh=2147483647  rtt=  0.07ms  var= 0.03ms  mss=32741  retrans=0  lost=0
     0.20s | cwnd=   322 segs  ssthresh=2147483647  rtt=  0.11ms  var= 0.04ms  mss=32741  retrans=0  lost=0
     0.35s | cwnd=   484 segs  ssthresh=2147483647  rtt=  0.13ms  var= 0.05ms  mss=32741  retrans=0  lost=0

sent 2314.6 MB
```

**Why this works, and what to read in it:**

- `cwnd=10` at connect time is `IW`, the initial window — RFC 6928's 10 segments, exactly as described above. **You are looking at the constant that determines how fast the first 14 KB of every HTTP response can be sent.**
- `ssthresh=2147483647` is `INT_MAX`, meaning "no threshold set yet" — the connection is in slow start and has not yet seen loss. **Watching `ssthresh` drop from `INT_MAX` to a real number is watching the exact moment TCP first inferred congestion.** That transition is the most informative single event in a connection's life.
- `cwnd` climbing 10 → 163 → 322 → 484 is slow start followed by congestion avoidance, live.
- `mss=32741` is a loopback artefact (huge MTU). On a real interface expect 1448–1460.
- `retrans=0 lost=0` on loopback, as expected — there's no network to lose anything in.

**Honesty caveats, and they matter here:**
1. `struct tcp_info` has grown over kernel versions by *appending* fields. Parsing a prefix as we do is safe; hard-coding a total size or parsing later fields without checking the returned length is not. If you need this in production, use `ss -tin` (which the kernel maintainers keep in sync) rather than hand-rolled struct parsing.
2. This is loopback, so the congestion behaviour is not representative — there's no bottleneck to congest. To see a genuine sawtooth you need a constrained path: `tc qdisc ... netem delay 50ms loss 0.5%` plus a rate limit, or a real internet transfer.
3. On Windows this script exits immediately by design rather than pretending to work. The nearest Windows equivalents (`Get-NetTCPConnection`, `netsh interface tcp show`) do not expose per-connection `cwnd`. If you want this visibility on Windows, run it in WSL2 or use Wireshark's TCP Stream Graphs, which infer window behaviour from the capture — genuinely excellent for this.

### System design — congestion control for a global CDN's origin fetches

**Problem:** your CDN has 200 edge PoPs worldwide fetching from origins in `us-east`. Edge-to-origin RTTs range 5 ms to 300 ms. Some edges (South America, parts of Asia) see 0.5–2% packet loss on transit paths. Origin fetches are 50 KB to 500 MB. Fetch throughput from the lossy PoPs is terrible and users there see slow first-byte times.

**Requirements:** maximise origin-fetch throughput from far/lossy PoPs; don't destabilise well-behaved paths; don't require changes at the origin application layer.

**Alternatives:**

1. **Keep CUBIC everywhere, and add more parallel connections from lossy PoPs.**
2. **Switch the edge fleet's sender-side congestion control to BBR.**
3. **Terminate at regional mid-tier caches so no single TCP connection spans a long lossy path** (a "tiered cache" / split-TCP architecture).

**The decision: (3) as the architecture, (2) within it.**

**The actual reason:** apply the Mathis equation to the worst case — MSS 1460, RTT 300 ms, loss 1%:

```
1460 / (0.300 × √0.01) = 1460 / 0.030 = ~48.7 KB/s ≈ 0.39 Mbit/s
```

**Under 400 kbit/s per connection.** A 500 MB origin fetch at that rate takes about three hours. No amount of bandwidth purchasing changes it. So option (1) is not tuning, it's arithmetic denial — and while parallel connections do multiply the ceiling (N connections ≈ N × 0.39 Mbit/s), you'd need dozens, which is both antisocial and fragile.

Option (3) attacks *both* terms in the denominator simultaneously, which is why it dominates: splitting a 300 ms 1%-loss path into two hops of 150 ms and ~0.3% loss each changes the ceiling on each hop by roughly `(2) × (√(1/0.3)) ≈ 3.6×` — and because the hops are independent TCP connections, a loss on the far hop no longer causes the near hop to back off. Option (2) then improves each hop further, because BBR doesn't interpret the residual random loss as congestion at all.

**The trade-off honestly:** option (3) is a real architectural cost — mid-tier caches are machines to run, cache-consistency surface to reason about, an extra hop of latency on cache misses at the mid-tier, and a new failure domain. Option (2) alone is a `sysctl` change with a deploy. If you can only do one thing this quarter, do (2); it's cheap and it helps. The reason (3) is the *answer* rather than the *nice-to-have* is that it fixes the RTT term, and BBR does not.

**Flip condition:** if loss on your transit paths is negligible (say under 0.01%) then the Mathis ceiling is above your link capacity, congestion control stops being the bottleneck, and both (2) and (3) are solving a non-problem — spend the effort on cache hit rate instead. Conversely, if you *cannot* deploy mid-tier caches (regulatory data-residency constraints, cost), then (2) plus a bounded number of parallel connections is the honest fallback, and you accept the lower ceiling.

**Failure modes:** deploying BBR fleet-wide without measuring its effect on *co-resident* CUBIC traffic (your own or your host's) can starve those flows — measure, don't assume. Parallel connections interact badly with server-side connection limits and with per-IP rate limiting, which presents as random 429s or resets. And mid-tier caches introduce the classic cache-stampede risk on origin: one mid-tier miss for a hot object can fan out to the origin from many edges at once unless you add request coalescing.

### Case study — the 1986 NSFNET congestion collapse

**What happened:** in October 1986, throughput between LBL and UC Berkeley collapsed from 32 kbit/s to 40 bit/s. The proximate cause was that the network's aggregate offered load exceeded capacity, and the *dominant* traffic became retransmissions: every sender, on losing a packet, retransmitted; those retransmissions filled the queues that caused the loss. The state was stable and self-sustaining, which is what makes it "collapse" rather than "congestion."

**Why it was possible:** pre-1988 TCP had no congestion control at all. Senders were limited only by the receiver's advertised window (flow control), which by design says nothing about the network. A sender with a generous receive window would happily saturate any bottleneck, and on loss would retransmit at the same rate. **The protocol had no mechanism by which the network could ask anyone to slow down.**

**The engineering lesson, and it generalises far beyond networking:** the fix was not more capacity, better hardware, or smarter routers. It was **making every endpoint responsible for restraining itself in response to failure.** Jacobson's insight was that a distributed system with no back-pressure has a stable failure mode, and that the back-pressure must be built into the *client's* retry behaviour, because the shared resource has no way to push back.

Carry this into every system you build: a service under load, a database with a connection pool, an agent loop calling a rate-limited API. If your retry policy is "retry immediately, forever," you have built a 1986 NSFNET. The correct shape — **exponential backoff with jitter, plus a circuit breaker** — is AIMD with a different vocabulary.

**Second, less-told part of the story:** the fix was deployed by *changing the endpoints*, not the network — 4.3BSD shipped Jacobson's changes and the internet's hosts gradually adopted them. This is the end-to-end principle paying off: a protocol-level pathology was fixed by upgrading software at the edges, with no router changes required. A network with reliability logic in the middle could not have been fixed that way.

*Primary source:* Van Jacobson, "Congestion Avoidance and Control," ACM SIGCOMM 1988 — the paper opens with exactly this incident. RFC 5681 is the current standard specification; RFC 8312 for CUBIC; the BBR papers and the `tcp_bbr` Linux source for BBR.

### In production — congestion control operationally

**Best practices:**
1. **Know your algorithm.** `sysctl net.ipv4.tcp_congestion_control` on every fleet you operate. A surprising number of teams don't know, and it materially affects their throughput on long paths.
2. **Measure the loss rate on paths that matter**, and apply Mathis before buying bandwidth.
3. **Consider BBR for long/lossy paths** — CDN edges, mobile-facing services, cross-region replication — after measuring impact on co-resident flows.
4. **Keep responses small enough to fit early slow-start windows** where latency is the product. The first ~14 KB is free; the next 30 KB costs a round trip.
5. **Reuse connections.** A warm connection has a grown `cwnd`; a cold one starts at 10 segments and has to relearn the path.

**Mistakes, beginner → senior:**
- *Beginner:* thinking bandwidth is the only throughput determinant; believing "slow start" means slow.
- *Intermediate:* conflating flow control and congestion control; "fixing" slowness by raising `SO_SNDBUF` when the constraint was `cwnd` (which you cannot set).
- *Senior:* deploying BBR without measuring fairness impact; ignoring that a 0.5% loss rate has a hard mathematical throughput ceiling; not realising that a load balancer terminating TCP *resets* congestion state, so `cwnd` is relearned per hop (which is exactly the mechanism that makes split-TCP fast).

**Observability:** `ss -tin` (per-connection `cwnd`, `ssthresh`, `rtt`, `retrans`, `bbr:...` state if BBR is on) is the single best tool. `nstat`, and Wireshark's *Statistics → TCP Stream Graphs → Window Scaling / Time Sequence* for visual sawtooth analysis. Windows: `Get-NetTCPSetting`, and Wireshark.

**Cost:** congestion control is free in dollars and expensive in *understanding*. The cost of ignorance is buying bandwidth that cannot help you, which is the most common form of wasted networking spend there is.

---

## 8. Nagle's algorithm × delayed ACK: the 40-millisecond mystery

**Depth: [WORKING]**

### Intuition

Two optimisations, each individually sensible, each solving a real problem, which **interact to produce a pathological delay.** This is the canonical example of a class of bug worth learning to recognise: two correct components whose composition is wrong. You will hit this general pattern far more often than you'll hit this specific TCP case, which is why it's worth studying even though the fix is one line.

**Nagle's algorithm** (RFC 896, 1984) exists because of telnet. Every keystroke was a 1-byte TCP segment inside a 40-byte header: 41 bytes on the wire to transmit one character, and thousands of tiny packets swamping 1980s networks. Nagle's rule:

> Do not send a small segment if there is already unacknowledged small data in flight. Buffer it, and send when the outstanding data is acknowledged (or when you accumulate a full MSS).

Result: many keystrokes coalesce into one segment. Efficiency restored.

**Delayed ACK** (RFC 1122) exists because pure ACKs are pure overhead — a 40-byte packet carrying zero bytes of data. So:

> Do not acknowledge immediately. Wait up to ~40 ms (Linux; RFC permits up to 500 ms) in the hope that (a) you'll have data to send and can piggyback the ACK on it, or (b) more data arrives and one ACK can cover several segments.

Result: fewer ACK packets. Efficiency restored.

### The interaction

Now put them together in a request/response protocol where the client writes a request in **two** `send()` calls — a header, then a body. This is extremely common: any framing that writes a length prefix separately, any HTTP client that writes headers and body separately, any code doing `sock.send(header); sock.send(payload)`.

```
Client                                          Server
  │
  │ send(header)   → Nagle: nothing outstanding, send it
  │── header ─────────────────────────────────►│
  │                                             │ delayed ACK: "I'll wait ~40 ms;
  │                                             │  maybe I'll have data to piggyback"
  │ send(body)     → Nagle: small data IS       │
  │                  outstanding (the header,   │  server needs the BODY before it
  │                  unacked) → BUFFER IT       │  can produce a response, so it
  │                                             │  has nothing to piggyback
  │        ╎                                    │        ╎
  │        ╎  ← both sides waiting for the other ─►      ╎
  │        ╎     Nagle waits for the ACK                 ╎
  │        ╎     delayed ACK waits for data               ╎
  │        ╎                                    │        ╎
  │        ╎  ~40 ms elapse                     │        ╎
  │                                             │
  │◄────────── ACK (delayed-ACK timer fires) ───│
  │ Nagle unblocks                              │
  │── body ───────────────────────────────────►│  finally!
  │                                             │  processes request
  │◄───────── response ────────────────────────│
```

**A fixed ~40 ms penalty on every request**, on a path whose RTT might be 0.5 ms. It's a **deadlock broken only by a timer**, which is why the delay is so suspiciously consistent — the tell-tale symptom is latency clustered at almost exactly 40 ms (or 40 ms multiples) with near-zero variance.

The reason this is such a good teaching example: **neither algorithm is buggy.** Nagle correctly avoids small packets. Delayed ACK correctly avoids bare ACKs. The pathology is entirely in the composition, and it only manifests for a specific traffic shape (small write, then small write, then wait for a response). Change any one of those and it vanishes — which is why it's so hard to reproduce in a test and so easy to hit in production.

### Worked example — the fix, three ways

**Fix 1 — `TCP_NODELAY` (disable Nagle). The usual answer.**

```python
sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)
```

This is what essentially every modern library does by default: Redis clients, gRPC, Nginx (`tcp_nodelay on;` is the default), Node.js, and the Go standard library (which sets `TCP_NODELAY` on every connection unless you opt out). If you're writing a request/response protocol, set it.

**Fix 2 — write once (`sendall` with a single buffer). Often the better answer.**

```python
# Bad: two sends → Nagle can interleave with delayed ACK
sock.send(header)
sock.send(body)

# Good: one send → nothing to coalesce, no interaction possible
sock.sendall(header + body)
```

Or use `writev`/scatter-gather to avoid the copy:

```python
sock.sendmsg([header, body])      # single segment, no concatenation copy
```

This fix is *better* than `TCP_NODELAY` where it applies, because it eliminates the small-write pattern rather than disabling a safeguard against it. `TCP_NODELAY` says "let me send small packets"; a single write says "I don't have small packets." One of those is a workaround and the other is a correction.

**Fix 3 — `TCP_QUICKACK` (Linux, disable delayed ACK).** Available, but *not sticky*: the kernel re-enables delayed ACK after use, so you must set it repeatedly. Almost never the right tool; know it exists so you recognise it in someone else's code.

### Judgment: should you just disable Nagle everywhere?

**The apparent answer:** yes — every modern library does, so it must be right.

**The real alternative worth taking seriously:** leave Nagle on, and fix your write pattern.

**Why `TCP_NODELAY` usually wins in practice:** you often don't control the write pattern. It's inside a framework, an ORM, a driver, a TLS library that writes records in pieces. You can't audit every write in your dependency tree, but you *can* set one socket option. Pragmatism wins.

**The trade-off, honestly stated — and this is the part usually omitted:** Nagle exists for a reason, and disabling it means you can and will emit tiny packets. A logging library that writes one line per `send()` with `TCP_NODELAY` set will put each line in its own segment: 100 bytes of log in a 40-byte-header packet, 29% overhead, and packet-rate rather than byte-rate becomes your bottleneck. On a high-packet-rate service this is a real cost, and "we set `TCP_NODELAY` and our packet rate tripled" is a genuine incident shape. The correct posture is `TCP_NODELAY` **plus** application-level batching — you take responsibility for not emitting tiny writes, rather than delegating it to the kernel.

**Flip condition:** keep Nagle enabled when you're bulk-streaming (it can only help — you're writing full-MSS chunks anyway, so Nagle never triggers) and when you're on a metered or packet-rate-limited path where packet count matters more than a 40 ms tail. Disable it whenever latency of small request/response exchanges is the product, which is most interactive systems.

### Runnable example — reproducing and fixing the stall

**Honesty note up front:** this is genuinely hard to reproduce reliably on loopback, because loopback ACKs are essentially instantaneous and Linux has heuristics that suppress Nagle in some cases. What the script below *does* reliably show is the latency distribution difference and the mechanism; on a real network path with the two-write pattern you'll see the clean 40 ms plateau.

```python
# nagle_demo.py — compare two-write vs one-write vs TCP_NODELAY latencies.
import socket
import statistics
import threading
import time

HOST, PORT = "127.0.0.1", 9108
ROUNDS = 200


def echo_server():
    lis = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    lis.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    lis.bind((HOST, PORT)); lis.listen(8)
    while True:
        try:
            conn, _ = lis.accept()
        except OSError:
            return
        # Server must read the FULL 24-byte request before replying — this is
        # what makes it unable to piggyback an ACK on a response.
        with conn:
            while True:
                buf = b""
                while len(buf) < 24:
                    part = conn.recv(24 - len(buf))
                    if not part:
                        return
                    buf += part
                conn.sendall(b"OK")


threading.Thread(target=echo_server, daemon=True).start()
time.sleep(0.3)

header, body = b"HDR:0016", b"payload-16-bytes"     # 8 + 16 = 24 bytes


def measure(label, nodelay: bool, single_write: bool):
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    if nodelay:
        s.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)
    s.connect((HOST, PORT))
    samples = []
    for _ in range(ROUNDS):
        t0 = time.perf_counter()
        if single_write:
            s.sendall(header + body)          # ONE segment
        else:
            s.sendall(header)                 # two writes: the risky pattern
            s.sendall(body)
        # read the 2-byte reply
        got = b""
        while len(got) < 2:
            got += s.recv(2 - len(got))
        samples.append((time.perf_counter() - t0) * 1000)
    s.close()
    samples.sort()
    print(f"{label:<34} median={statistics.median(samples):7.3f} ms   "
          f"p95={samples[int(0.95 * len(samples))]:7.3f} ms   "
          f"max={samples[-1]:7.3f} ms")


measure("two writes, Nagle ON  (risky)", nodelay=False, single_write=False)
measure("two writes, TCP_NODELAY (fix 1)", nodelay=True,  single_write=False)
measure("one write,  Nagle ON  (fix 2)", nodelay=False, single_write=True)
```

Output on loopback (Windows):

```
two writes, Nagle ON  (risky)        median=  0.169 ms   p95=  0.291 ms   max=  1.204 ms
two writes, TCP_NODELAY (fix 1)      median=  0.121 ms   p95=  0.198 ms   max=  0.673 ms
one write,  Nagle ON  (fix 2)        median=  0.118 ms   p95=  0.176 ms   max=  0.582 ms
```

**Why this works, and what to take from it:**

- **On loopback the 40 ms stall does not appear**, and I'm showing you that rather than faking the numbers. Loopback ACKs return in microseconds, so the delayed-ACK timer rarely gets a chance to matter, and the kernel has additional heuristics. What you *do* see is a consistent ordering: two-writes-with-Nagle is measurably slowest, and both fixes improve median and tail.
- To see the real 40 ms plateau, add latency so the delayed-ACK timer becomes decisive. On Linux:

  ```bash
  sudo tc qdisc add dev lo root netem delay 5ms
  python nagle_demo.py
  sudo tc qdisc del dev lo root
  ```

  With that in place the two-write/Nagle-on row jumps to a median in the tens of milliseconds while the fixes stay near 10 ms (2 × 5 ms RTT). **That gap is the bug.**
- The server deliberately reads exactly 24 bytes before replying. That's what creates the deadlock: it cannot send a response (and therefore cannot piggyback an ACK) until it has the body, and the body is being withheld by Nagle pending an ACK.
- Notice fix 2 is as fast as fix 1 *and* keeps Nagle's protection. That's the argument for preferring it where you control the writes.

### In production (condensed)

**Best practice:** set `TCP_NODELAY` on any socket carrying interactive request/response traffic, *and* batch at the application layer so you aren't emitting tiny segments. Prefer a single `sendall`/`sendmsg` over multiple small writes.

**Top failure mode:** latency histograms with a hard cluster at ~40 ms and almost no variance, on a path whose RTT is sub-millisecond. That signature is nearly diagnostic — real network variance doesn't produce a spike that tight. If you see it, look for a two-write pattern before you look at anything else.

**Where it commonly hides:** custom binary protocols that write a length prefix separately from the payload; TLS libraries writing records in pieces; ORMs and drivers that build a query in fragments; anything doing `write(header); write(body)` in a language whose socket API makes that natural.

---

## 9. Teardown, TIME_WAIT, and RST: how connections end (and refuse to)

**Depth: [CORE]**

### Intuition

Closing a TCP connection is harder than opening one, and the reason is a genuine problem, not an accident of design: **each side must independently signal that it has no more data to send, and neither side can know whether its final message arrived.**

The second half of that is the "two-army problem" (or two generals problem) and it is provably unsolvable in general: two parties communicating over a lossy channel cannot both be *certain* they've agreed. TCP doesn't solve it — nothing can — it *engineers around it* with a timer, and that timer is `TIME_WAIT`, the single most misunderstood state in networking.

### The four-way close

TCP connections are **full-duplex**: two independent byte streams. So closing is two independent shutdowns.

```
   CLIENT (calls close() first)                    SERVER
   ESTABLISHED                                     ESTABLISHED
        │                                               │
        │── FIN seq=X ────────────────────────────────► │
   FIN_WAIT_1                                       CLOSE_WAIT   ← app must close()!
        │                                               │
        │◄──────────────────── ACK ack=X+1 ──────────── │
   FIN_WAIT_2                                           │
        │                                               │  server may still SEND data
        │◄──────── (optional data) ──────────────────── │  — this direction is open
        │                                               │
        │                             (server app calls close())
        │◄──────────────────── FIN seq=Y ────────────── │
        │                                            LAST_ACK
        │── ACK ack=Y+1 ──────────────────────────────► │
   TIME_WAIT                                          CLOSED
        │  (wait 2×MSL — Linux: 60 s)
    CLOSED
```

**Half-close is a real, usable feature and not a curiosity.** Between the client's FIN and the server's FIN, the connection is *half-closed*: the client has said "I'm done sending" and the server can still send. This is what `shutdown(SHUT_WR)` does, and it's how you implement "send a request, signal end-of-request, then read the whole response until EOF" without a length header:

```python
sock.sendall(request)
sock.shutdown(socket.SHUT_WR)     # "I'm done sending; you'll see EOF" — but keep reading
response = b""
while chunk := sock.recv(65536):
    response += chunk
```

That's how the classic `nc host port < request > response` pattern works, and it's a genuinely useful tool.

**`CLOSE_WAIT` is an application bug, essentially always.** The peer sent FIN; the kernel acknowledged it; the connection is now waiting for **your application** to call `close()`. The kernel cannot do it for you — you might still want to send. So a pile of `CLOSE_WAIT` connections means:

> Your code received EOF (`recv()` returned `b""`) and did not close the socket.

That's a leaked file descriptor with a specific, unmistakable signature. Rising `CLOSE_WAIT` → you will run out of FDs → your service will start failing with "too many open files" in code nowhere near the leak. The fix is always the same: find the path where you read EOF without closing, usually an error branch or a `break` that skips your cleanup.

```powershell
Get-NetTCPConnection -State CloseWait | Group-Object OwningProcess | Select Name,Count
```
```bash
ss -tan state close-wait | wc -l
```

**Diagnostic pairing worth memorising:**

| State piling up | Meaning | Who's at fault |
|---|---|---|
| `CLOSE_WAIT` | Peer closed; **your app** didn't call `close()` | **Your code** — FD leak |
| `FIN_WAIT_2` | You closed; **peer's app** hasn't closed | The peer's code |
| `TIME_WAIT` | You closed first; normal, self-clearing | Nobody — but see below |
| `SYN_RECV` | Handshakes not completing | SYN flood, or peer disappearing |
| `SYN_SENT` (client, many) | Your connects aren't completing | Firewall, wrong port, dead peer |

### TIME_WAIT: what it's actually for

The side that closes **first** (sends the first FIN) enters `TIME_WAIT` and stays there for **2 × MSL** — twice the Maximum Segment Lifetime. MSL is nominally 2 minutes, so 2×MSL is 4 minutes; **Linux hard-codes 60 seconds** (`TCP_TIMEWAIT_LEN` in `include/net/tcp.h`, not tunable via sysctl), and Windows historically used 240 seconds with a registry-configurable `TcpTimedWaitDelay`.

There are exactly **two** reasons, and knowing both is what separates understanding from folklore:

**Reason 1: absorb the retransmitted FIN.** The final ACK might be lost. If it is, the peer's `LAST_ACK` timer fires and it retransmits its FIN. Somebody has to be there to re-ACK it. If the closing side had gone straight to `CLOSED`, that retransmitted FIN would arrive at a nonexistent connection and get an **RST** — and the peer would report an error on a connection that closed perfectly normally. `TIME_WAIT` is a *tombstone that answers the phone*.

**Reason 2 (the subtle and more important one): prevent old duplicates from poisoning a new connection.** Suppose you close a connection on the 4-tuple `(1.2.3.4:51000, 5.6.7.8:443)` and immediately open a *new* connection using the identical 4-tuple. Now imagine a packet from the *old* connection was delayed in some router queue and arrives late. Its sequence number could plausibly fall inside the new connection's valid window — and the new connection would accept it as legitimate data. **Silent data corruption.**

`TIME_WAIT` blocks reuse of that 4-tuple for long enough that any packet from the old connection has certainly expired (this is what MSL *means* — the longest a packet can survive, enforced by TTL). It is a **quarantine on an address, not on a connection.**

This is why the folk wisdom "TIME_WAIT is a wasteful legacy thing, just turn it off" is dangerous. Reason 1 causes spurious errors; reason 2 causes **silent wrong data**, which is categorically worse.

### The problem TIME_WAIT causes, and the correct fixes

A busy client — an API gateway, a proxy, a service making many short-lived outbound calls — closing thousands of connections per second accumulates thousands of `TIME_WAIT` entries, each holding a 4-tuple hostage for 60 seconds. Combine that with concept 2's ephemeral-port ceiling:

```
28,000 ephemeral ports ÷ 60 seconds of TIME_WAIT = ~466 new connections/second
                                                    to a single destination
```

Exceed that and `connect()` starts failing with "cannot assign requested address." **This is one of the most common scaling walls in service-to-service architectures**, and it looks like a mysterious intermittent connection error rather than a resource limit.

The fixes, ranked from best to worst:

**1. Reuse connections (keep-alive / pooling). The actual fix.** If you don't close the connection, you don't get a `TIME_WAIT`. This addresses the cause, eliminates handshake cost, *and* keeps `cwnd` warm (concept 7). Every other item on this list is a mitigation for failing to do this. Concept 11 is about doing it properly.

**2. Make the *server* close first, where you can choose.** `TIME_WAIT` lands on whoever closes first. A server has one port and many peers, so its 4-tuples vary in the client's fields — the server's `TIME_WAIT` entries don't exhaust anything. A client has one destination and must vary its own port, so `TIME_WAIT` there is expensive. Shifting who closes first moves the cost to where it's harmless. (This is why HTTP's `Connection: close` semantics and server-side idle timeouts matter architecturally, not just operationally.)

**3. Add destination diversity.** More server IPs, multiple LB VIPs, multiple ports. Each new destination is a fresh 4-tuple space of ~28,000.

**4. `net.ipv4.tcp_tw_reuse=1` (Linux, client side).** Allows a new *outgoing* connection to reuse a `TIME_WAIT` 4-tuple **when TCP timestamps confirm the old connection's packets are strictly older**. This is safe because it doesn't blindly ignore the quarantine — it uses timestamps to prove the quarantine is unnecessary for this specific case. Requires timestamps on both ends. This is the acceptable knob.

**5. Widen the ephemeral range.** `net.ipv4.ip_local_port_range = 10240 65535` buys you roughly 2× headroom. A fine cheap mitigation, not a fix.

**What NOT to do — and this deserves a callout:** `net.ipv4.tcp_tw_recycle` was the knob people reached for, and it was **removed from Linux in 4.12** because it was broken. It dropped connections from clients behind NAT, since it made per-IP timestamp assumptions that are false when many hosts share one source address. If you find `tcp_tw_recycle=1` in a config file or a StackOverflow answer, it is either a no-op (modern kernel) or a source of baffling intermittent failures for NATed clients (old kernel). Delete it. *Primary source:* the Linux commit removing it (4396e46187ca, "tcp: remove tcp_tw_recycle") and Vincent Bernat's widely-cited write-up on `TIME_WAIT` state.

### SO_REUSEADDR vs SO_REUSEPORT — two options that sound identical and aren't

Constant source of confusion, and the confusion is worse because the semantics differ *by platform*.

**`SO_REUSEADDR`:**
- **On Unix:** lets you `bind()` to a port that has connections in `TIME_WAIT`. This is why every server example (including all of mine) sets it — otherwise restarting your server within 60 seconds fails with "address already in use." It does **not** let two live sockets bind the same port.
- **On Windows:** genuinely different and genuinely dangerous — it allows a *second live socket* to bind the same address and port, which permits port hijacking. Windows' equivalent of the Unix behaviour is `SO_EXCLUSIVEADDRUSE` (to *prevent* hijacking). **Do not port a Unix `SO_REUSEADDR` assumption to Windows without checking.**

**`SO_REUSEPORT`** (Linux 3.9+, BSD; **not available on Windows**):
- Lets *multiple live sockets* bind the identical address and port, with the kernel load-balancing incoming connections across them by hashing the 4-tuple.
- This is how modern multi-process servers scale: N worker processes each `listen()` on port 80 with `SO_REUSEPORT`, and the kernel distributes connections. No shared listening socket, no accept-lock contention, no single-process bottleneck. It's how Nginx's `reuseport` directive, Envoy, and Go servers with `SO_REUSEPORT` achieve near-linear scaling with cores.
- Bonus: it enables zero-downtime restarts — start the new binary listening on the same port, drain the old one, kill it.

```python
import socket, sys

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)     # everywhere

if hasattr(socket, "SO_REUSEPORT"):                          # Linux/BSD only
    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEPORT, 1)
else:
    print(f"SO_REUSEPORT unavailable on {sys.platform}; "
          "use multiple processes with a shared inherited socket, or IOCP", file=sys.stderr)
```

**Honesty caveat:** on Windows, `hasattr(socket, "SO_REUSEPORT")` is `False`, so the code above degrades correctly instead of raising — and I'm showing the `else` branch rather than pretending the option is portable. The Windows path to the same goal is different (a shared listening socket handle inherited by child processes, or IOCP-based single-process scaling).

### Runnable example — creating and observing TIME_WAIT

```python
# time_wait_demo.py — deliberately create TIME_WAIT entries and count them.
# Shows which side gets TIME_WAIT, and that it's the CLOSER who pays.
import socket
import subprocess
import sys
import threading
import time

HOST, PORT = "127.0.0.1", 9109
N = 300


def server(client_closes_first: bool):
    lis = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    lis.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    lis.bind((HOST, PORT)); lis.listen(128)
    for _ in range(N):
        conn, _ = lis.accept()
        if client_closes_first:
            # wait for the client's FIN (recv returns b"") before closing
            while conn.recv(1024):
                pass
        conn.close()          # server closes second → server gets no TIME_WAIT
    lis.close()


def count_time_wait() -> int:
    if sys.platform == "win32":
        out = subprocess.run(
            ["powershell", "-NoProfile", "-Command",
             f"(Get-NetTCPConnection -State TimeWait -ErrorAction SilentlyContinue "
             f"| Where-Object {{ $_.LocalPort -eq {PORT} -or $_.RemotePort -eq {PORT} }}).Count"],
            capture_output=True, text=True)
        return int((out.stdout.strip() or "0"))
    out = subprocess.run(["ss", "-tan", "state", "time-wait"],
                         capture_output=True, text=True)
    return sum(1 for line in out.stdout.splitlines() if f":{PORT}" in line)


print(f"TIME_WAIT on port {PORT} before: {count_time_wait()}")

t = threading.Thread(target=server, args=(True,))
t.start(); time.sleep(0.3)

for _ in range(N):
    c = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    c.connect((HOST, PORT))
    c.sendall(b"hi")
    c.close()               # CLIENT closes first → client accrues TIME_WAIT
t.join()

time.sleep(0.5)
print(f"TIME_WAIT on port {PORT} after {N} client-closed connections: {count_time_wait()}")
print("\nThese entries hold their 4-tuples for 60 s (Linux) / up to 240 s (Windows).")
print("Watch them drain:  ss -tan state time-wait | grep 9109 | wc -l")
```

Output on Linux:

```
TIME_WAIT on port 9109 before: 0
TIME_WAIT on port 9109 after 300 client-closed connections: 300

These entries hold their 4-tuples for 60 s (Linux) / up to 240 s (Windows).
Watch them drain:  ss -tan state time-wait | grep 9109 | wc -l
```

**Why this works:**

- The server explicitly waits for the client's FIN (`recv` returning `b""`) before closing, guaranteeing the **client** is the one that closed first. The result: 300 `TIME_WAIT` entries on the client side, zero on the server side. That's the "who closes first, pays" rule demonstrated rather than asserted.
- Flip `client_closes_first` to `False` and re-run: the server closes first, and now the `TIME_WAIT` entries sit on the server's 4-tuples instead. Same total count, different owner — and on a real server that placement is *much* cheaper, because the server's `TIME_WAIT` entries vary in the client's IP/port and therefore exhaust nothing.
- Now extrapolate: 300 connections in a fraction of a second held 300 4-tuples for a full minute. At a sustained 500 connections/second to one destination you would hold ~30,000 simultaneously — past the ephemeral ceiling. **You just measured the wall.**

### RST: the emergency exit

`RST` is not a "close." It's an abort, and it means one of these:

1. **Connection refused** — a SYN arrived for a port with no listener. The kernel replies RST immediately, which is why `connect()` to a closed port fails fast rather than timing out. **A fast "connection refused" is good news**: it means the host is up and reachable, and only the service is absent. A *timeout* on connect means something silently dropped your packet — a firewall or a dead host — which is strictly less information.
2. **Data arrived for a nonexistent connection** — e.g. a peer rebooted and lost its state, or a middlebox forgot the connection and one side sent on it anyway. The receiving side has no matching 4-tuple, so it RSTs.
3. **Explicit abort** — the application set `SO_LINGER` with timeout 0 and closed, or a load balancer/proxy actively killed the connection.
4. **A process died** while a connection was open, and its OS is cleaning up.

Two operationally important differences from FIN:

- **RST discards buffered data.** A FIN says "everything I sent has been delivered, and now I'm done." An RST says "stop, immediately," and unread data in the receive buffer is thrown away. That's why `SO_LINGER(0)` closes can lose the response your peer had already sent.
- **RST is not acknowledged.** There's no handshake. It's fire-and-forget, so if it's lost, the peer never learns — and continues believing the connection is fine until it writes and gets another RST. This is exactly why "the first request after an idle period fails, the retry succeeds" is such a common signature.

In application terms: `ConnectionResetError` on Python, `ECONNRESET` (errno 104) on Linux, `WSAECONNRESET` (10054) on Windows.

### System design — zero-downtime deploys without connection errors

**Problem:** you deploy 30× a day. Currently each deploy produces a burst of `ConnectionResetError` in client logs and a visible latency spike. Fix it so deploys are invisible.

**Requirements:** no client-visible errors during deploy; in-flight requests complete; new instance takes over cleanly; works behind a load balancer with a 30-second health-check interval.

**Where the errors come from — diagnose before designing.** Three distinct sources, and they need three different fixes:

1. The old process is SIGKILLed → its OS sends RST for every open connection → clients see `ConnectionResetError` on in-flight requests.
2. The load balancer keeps sending new connections for up to 30 s after the process dies, because its health check hasn't noticed yet → clients get "connection refused."
3. The new process can't bind the port because the old one's sockets are in `TIME_WAIT` → startup failure → restart loop.

**Alternatives:**

1. **Graceful shutdown only:** on SIGTERM, stop accepting, finish in-flight requests, then exit.
2. **Graceful shutdown + LB drain:** on SIGTERM, first fail the health check, wait for the LB to remove you (≥ 2 check intervals), *then* stop accepting and drain.
3. **`SO_REUSEPORT` handoff:** start the new process bound to the same port via `SO_REUSEPORT`, let the kernel split traffic across both, then drain and stop the old one.

**The decision: (2) as the baseline for everyone; (3) additionally where the platform supports it and you need true zero-gap.**

**The actual reason:** (1) alone fixes source 1 but not source 2 — the load balancer is still routing new connections to a process that has stopped accepting, so you've converted resets into connection-refused errors. **The ordering is the whole insight: fail the health check *first*, wait longer than the LB needs to notice, and only then stop accepting.** That's the difference between a graceful shutdown that works and one that just moves the error.

The concrete sequence:

```
SIGTERM received
  ↓
1. Health endpoint starts returning 503        ← LB begins removing us
  ↓
2. sleep(2 × health_check_interval + margin)   ← 30 s interval → sleep ~75 s
     (KEEP SERVING NORMAL TRAFFIC during this window — this is the bit
      people omit, and omitting it is what causes the errors)
  ↓
3. Stop accepting new connections (close the listening socket)
  ↓
4. Wait for in-flight requests, bounded by a drain timeout (e.g. 30 s)
  ↓
5. Close remaining connections; exit
```

And step 3 has a subtlety: closing the *listening* socket is what you want; closing established connections is not. `SO_REUSEADDR` on the new process handles the `TIME_WAIT` bind problem (source 3).

**The trade-off honestly:** step 2 makes every deploy take at least 75 seconds longer per instance, which for 30 deploys a day across a large fleet is real wall-clock and real cost, and it means a rollback is slower too. Teams under deploy-velocity pressure hate it, and the temptation is to shorten the sleep to "probably enough." Shortening it below the LB's detection window silently reintroduces source 2 — intermittently, under load, which is the worst way for a bug to reappear.

Option (3)'s trade-off is different: `SO_REUSEPORT` means two versions of your code serve traffic *simultaneously* for the overlap window. If they disagree about a schema, a cache format, or a feature flag, you get genuinely confusing split-brain behaviour that only appears during deploys. You are trading a clean cutover for a brief period of version ambiguity.

**Flip condition:** skip the LB-drain sleep entirely if your LB supports **active deregistration** — an API call that removes the target immediately, or a Kubernetes `preStop` hook plus readiness gate that the control plane honours before sending SIGTERM. Then you don't need to *wait for a health check to notice*; you tell it. This is strictly better and is why Kubernetes' `terminationGracePeriodSeconds` + `preStop` pattern exists. And skip (3) whenever your versions can't safely coexist — a migration deploy, for instance — where a hard cutover with brief queuing beats split-brain.

**Failure modes:** a drain timeout shorter than your slowest legitimate request → you kill in-flight work that would have completed. `terminationGracePeriodSeconds` shorter than (health-check drain + request drain) → Kubernetes SIGKILLs you mid-drain and every mitigation above is bypassed. **This is the most common Kubernetes deploy bug there is**, and the arithmetic is: `terminationGracePeriodSeconds > preStop sleep + drain timeout + margin`.

### System design — a connection-heavy API gateway hitting the TIME_WAIT wall

**Problem:** your gateway proxies to 4 backend services. At 2,000 req/s it starts failing with "cannot assign requested address" on outbound `connect()`. Each backend is a single Kubernetes Service ClusterIP.

**The arithmetic:**

```
ephemeral ports available:    ~28,000  (Linux default 32768–60999)
TIME_WAIT duration:               60 s
distinct destinations:                4  (one ClusterIP:port each)

max sustainable NEW connections per second per destination:
    28,000 / 60 ≈ 466/s
across 4 destinations:  ~1,866/s        ← and you're at 2,000/s
```

You are not near the wall, you are *through* it. And the diagnosis is available in one command:

```bash
ss -tan state time-wait | wc -l          # expect ~28,000, i.e. saturated
```

**Alternatives:**

1. **HTTP keep-alive with a connection pool** — reuse connections instead of one per request.
2. **Widen `ip_local_port_range` to 10240–65535** — buys about 2×.
3. **`tcp_tw_reuse=1`** — allow safe reuse of `TIME_WAIT` tuples using timestamps.
4. **Add destination diversity** — headless Service so you connect to pod IPs directly, giving one 4-tuple space per pod.

**The decision: (1), with (4) as a structural improvement and (3) as belt-and-braces.**

**The actual reason:** (1) doesn't raise the ceiling, it *removes the requirement*. With a pool of 200 warm connections per backend, 2,000 req/s needs **zero** new connections in steady state — so `TIME_WAIT` accumulation goes to near zero, and you also stop paying a handshake RTT and a cold `cwnd` on every request. It's the only option that makes the problem structurally absent rather than pushed further away.

(4) is genuinely valuable alongside it: connecting to 20 pod IPs instead of 1 ClusterIP multiplies your 4-tuple space by 20 *and* removes a layer of kube-proxy/conntrack indirection. It also improves failure isolation. But note it moves load-balancing responsibility into your client, which is a real cost.

**The trade-off honestly:** (2) and (3) are one-line changes with no code impact, and (1) is a code change with new failure modes — pool exhaustion under burst (requests queue waiting for a connection, which can be worse than opening one), stale pooled connections (concept 11's entire subject), and a new tuning surface (pool size, max idle time, max lifetime). If you're firefighting at 3 a.m., do (2) and (3) tonight and (1) properly next week. That's not a cop-out, it's correct triage — but a team that stops at (2) and (3) hits the wall again at 4,000 req/s.

**Flip condition:** stay with per-request connections if your request rate is genuinely low (tens per second) and your backends are diverse, because pooling adds complexity for no benefit. Also stay per-request if your backends behave badly on reused connections — some legacy servers leak state across requests on a keep-alive connection, and a pool turns that into cross-request contamination.

**Failure modes:** pool size too small → requests queue for a connection and your p99 explodes while CPU sits idle. Pool with no max lifetime → connections pinned to backends that have been replaced, so a deploy silently sends traffic to nothing (concept 11). Pool larger than the backend's connection limit → the backend starts refusing, and you've moved the wall rather than removing it.

### Case study — Nginx, `SO_REUSEPORT`, and the accept-lock era

**What happened, factually:** before `SO_REUSEPORT` (Linux 3.9, 2013), multi-worker servers sharing one listening socket faced two bad options. Either all workers block in `accept()` on the shared socket — causing the **thundering herd**, where every worker wakes on each connection and all but one go back to sleep — or they serialise via an application-level `accept_mutex`, which trades wasted wakeups for lock contention and uneven distribution. Nginx shipped an `accept_mutex` directive precisely to manage this trade-off, and tuning it was a known operational chore.

`SO_REUSEPORT` moved the load-balancing decision into the kernel: each worker gets its *own* listening socket on the same port, and the kernel hashes each incoming connection's 4-tuple to pick a worker. No herd, no lock. Nginx exposed it as `listen ... reuseport;`, and Nginx's own published benchmarks reported substantial improvements in requests/second and, more importantly, dramatic reductions in latency variance at high connection rates. `accept_mutex` was subsequently defaulted off and is now deprecated in favour of `reuseport`.

**The engineering lesson tied to the concept:** `SO_REUSEPORT` is a case where **the kernel could do a job the application was doing badly, because the kernel had information the application didn't** — namely, atomic visibility into which sockets exist and a cheap way to distribute across them. The application-level mutex was a workaround for a missing kernel primitive, and once the primitive existed the workaround became strictly worse.

The transferable heuristic: when you find yourself building coordination machinery around a shared OS resource, check whether the OS has since grown a primitive for it. This same story has repeated with `epoll` (replacing `select` loops), `io_uring` (replacing thread pools for async I/O), and `SO_REUSEPORT` here.

**The honest caveat, because `reuseport` is not free:** kernel 4-tuple hashing is *static*. If one worker gets a connection that turns out to be extremely long-lived and expensive (a big upload, a websocket, an SSE stream), the kernel will keep sending that worker its statistical share of new connections regardless. With `accept_mutex`, a busy worker simply doesn't accept, so distribution self-corrects by load. So `reuseport` gives better distribution of *connection counts* and worse distribution of *work* when per-connection cost varies wildly. For short uniform requests it's clearly better; for heterogeneous long-lived connections the answer is genuinely "measure it."

*Primary sources:* the Linux commit adding `SO_REUSEPORT` (Tom Herbert, 2013); Nginx's engineering blog post "Socket Sharding in NGINX Release 1.9.1"; the Nginx `accept_mutex` documentation.

---
## 10. Head-of-line blocking: what reliability costs you

**Depth: [CORE]**

### Intuition

You saw this in concept 1's trace and I flagged it then. Now it gets taken seriously, because it is the single design flaw that motivated a decade of protocol work and eventually a whole new transport.

**In-order delivery is not free. It means one missing byte can hold back an unlimited amount of data that has already arrived.**

Restate that as a mechanism: the receiver's job is to hand the application a contiguous byte stream. If bytes 1000–1999 are missing and bytes 2000–999,999 are sitting in the receive buffer, the application gets **nothing** — not because the data isn't there, but because delivering it would violate ordering. Everything waits behind the hole.

The hole takes at minimum one RTT to fill (fast retransmit) and at worst an RTO (≥200 ms) to fill. So **one lost packet stalls the stream for at least one full round-trip.** On a 200 ms path that's 200 ms of a fully-loaded receive buffer going undelivered.

### Analogy: the single-file tunnel

Cars must exit a one-lane tunnel in the order they entered. Car #3 breaks down inside. Cars #4 through #400 are behind it, running fine, physically present at the exit — and none of them can leave until #3 is towed. The tunnel isn't full and the road ahead is empty; the *ordering constraint* is the entire problem.

**Where the analogy breaks:** cars physically can't overtake in a tunnel, but data *has already arrived* and is sitting in memory. The constraint is not physical, it is **a promise TCP made** ("you will see bytes in order"). That distinction is what makes head-of-line blocking fixable — you fix it by changing the promise, not the physics. QUIC changes the promise to "bytes within a stream are in order; different streams are independent." Same network, same loss, no cross-stream blocking.

### Why it became a crisis: HTTP/1.1 → HTTP/2 → HTTP/3

This is a genuinely good story about a fix that created a subtler version of the problem it fixed.

**HTTP/1.1's problem: one request at a time per connection.** A response must be fully sent before the next begins. A slow response blocks everything behind it — head-of-line blocking *at the HTTP layer*. The workaround was **6 parallel connections per origin** (a browser convention), which meant 6 handshakes, 6 TLS negotiations, 6 congestion windows all in slow start, and 6× the server-side connection cost. It worked, and it was ugly.

**HTTP/2's fix: multiplex many streams over one TCP connection.** Requests become interleaved frames on a single connection. One connection, one handshake, one warm `cwnd`, unlimited concurrency, header compression. HTTP-layer head-of-line blocking: solved.

**HTTP/2's remaining problem: TCP is still underneath.** And TCP delivers *one* byte stream in order. So:

```
HTTP/2 over TCP — 4 concurrent streams sharing one TCP connection:

TCP byte stream:  [S1][S2][S3][S1][S4][S2][S3][S1] ...
                            ▲
                       this segment lost

TCP receiver: cannot deliver ANY bytes after the hole.
              Streams 1, 2, 3, AND 4 all stall — even though the loss
              only affected stream 3's data.
```

**One lost packet stalls every concurrent request.** HTTP/2 removed application-layer head-of-line blocking and, by concentrating everything onto one TCP connection, made *transport*-layer head-of-line blocking hurt far more than it did before. With HTTP/1.1's six connections, a loss on one stalled roughly one-sixth of your traffic. With HTTP/2's one connection, it stalls all of it.

Google measured this at scale and found that **on lossy networks, HTTP/2 could be slower than HTTP/1.1** — the multiplexing win was outweighed by the shared-fate loss penalty. That measurement is the origin of QUIC.

**HTTP/3's fix: replace TCP.** Move reliability and ordering into a userspace protocol over UDP where ordering is *per-stream*. A loss on stream 3 stalls only stream 3; streams 1, 2, and 4 are delivered immediately. Concept 13.

### Worked example — quantifying the cost

Concrete numbers, because "it's slower" isn't a lesson.

A page loads 20 resources concurrently, RTT 100 ms, 1% packet loss, average resource 30 KB (≈21 segments at MSS 1460). Total ≈ 420 segments.

**Expected losses:** 420 × 0.01 = **~4.2 lost segments** during the page load.

**HTTP/2 over TCP:** each of those ~4 losses stalls the *entire* connection for at least one RTT while it's retransmitted.

```
4.2 losses × 100 ms minimum stall = ~420 ms of added latency
   — and every one of the 20 resources experiences all 4 stalls,
     because they share the single byte stream.
```

**HTTP/3 over QUIC:** each loss stalls only its own stream.

```
4.2 losses distributed across 20 streams → each affected stream stalls ~100 ms;
the other ~16 streams are unaffected.
Added latency for the page (which finishes when the slowest resource does):
   ~100 ms, not ~420 ms.
```

**Roughly a 4× reduction in loss-induced latency, from the same network, purely by changing where the ordering guarantee is enforced.** And the effect scales with concurrency: the more streams you multiplex, the worse TCP's shared-fate penalty gets and the better QUIC's independence looks. That is a beautiful illustration of a general principle — **the cost of a guarantee scales with how much you've bundled under it.**

### Judgment: is in-order delivery worth it?

**The decision TCP made:** guarantee in-order byte delivery, always, at the transport layer.

**The realistic alternative:** deliver bytes as they arrive with sequence information, and let the application reorder if it cares (this is roughly SCTP's message-oriented, optionally-unordered model, and it's what QUIC does per-stream).

**Why TCP's choice was right in 1981:** applications were simple, connections carried one thing at a time (one file, one telnet session, one mail transfer), and pushing reassembly into every application would mean every application reimplementing it — badly, differently, with its own bugs. A single correct kernel implementation was enormously valuable. **In-order delivery was the right default because there was nothing to multiplex.**

**The trade-off we discovered later:** the guarantee is applied at the wrong *granularity*. TCP can only offer "everything in order" because it has no idea your byte stream contains 20 independent things. The information needed to relax the guarantee safely — "these bytes belong to a different request than those" — lives in the application and is invisible to the transport.

**Flip condition — and it's precise:** in-order-everything stops being right the moment you multiplex independent logical streams over one connection. That's exactly what HTTP/2 did, which is exactly when it became a problem, which is exactly why HTTP/3 exists. The general form: **a strong ordering guarantee over a multiplexed channel converts independent failures into shared failures.** You'll meet the same shape in message queues (one poison message blocking a partition — Kafka), in database replication (one slow transaction blocking a single-threaded apply stream), and in agent orchestration (one slow tool call blocking a serialised turn). The fix always has the same shape too: partition the stream so failures don't share fate.

---

## 11. Connection lifetime: keep-alive, idle timeouts, and pools

**Depth: [CORE]**

### Intuition

We now know a connection costs a round trip to establish, starts with a cold congestion window, and consumes an ephemeral port for 60 seconds after it closes. All three arguments say the same thing: **don't close connections you'll need again.**

But a connection you keep open is a connection that can *silently die*. And concept 4 told us why: a TCP connection is two sets of numbers in two kernels' memory, with nothing on the wire. Anything that forgets or discards that state — a reboot, a NAT table eviction, a load balancer's idle timeout, a container being rescheduled — leaves one or both sides with a connection to nowhere, **and no notification.**

That's the tension this concept resolves, and it's where the most confusing production failures in networked services live.

### The core failure: the half-open connection

```
Time 0:    Client ══════════ Established ══════════ Server
Time 100:  Client ══════════ Established ══════════ Server    (idle, no traffic)
Time 400:  Client ══════════      ✗ LB idle timeout — LB silently
                                    drops its connection-tracking entry
Time 401:  Client thinks: ESTABLISHED
           Server thinks: ESTABLISHED
           Reality:       there is no path between them
Time 500:  Client sends a request → travels to LB → LB has no matching entry
                                  → either silently drops it, or sends RST
           Client: ConnectionResetError, or a hang until its own timeout
```

**Neither endpoint did anything wrong and neither was told.** The connection is *half-open* — alive in the endpoints' memory, dead in the middle.

The symptom is unmistakable once you know it: **the first request after a period of inactivity fails; the immediate retry succeeds.** Because the retry opens a fresh connection.

Everyday causes, all with the same shape:

| Middlebox | Typical idle timeout | Consequence |
|---|---|---|
| AWS Classic/Application LB | 60 s (configurable) | connection silently dropped |
| AWS NLB | 350 s, historically non-configurable | same |
| Azure Load Balancer | 4 min default | same |
| Home/office NAT router | 2–30 min, unpredictable | same |
| Corporate stateful firewall | often 5–15 min | same |
| Kubernetes conntrack | `nf_conntrack_tcp_timeout_established` = 5 days, but pods die sooner | pod IP reused → RST or worse |

**Verify these against current provider docs before relying on any number** — they change, and some are per-account configurable. The pattern is what matters: something in the middle has a timer shorter than your idle period, and it doesn't tell you when it fires.

### Two kinds of keep-alive, and people constantly conflate them

This confusion is worth eliminating carefully, because the two operate at different layers, solve different problems, and have wildly different default timings.

**TCP keep-alive (transport layer, RFC 1122).** The kernel periodically sends a probe segment on an *idle* connection to check the peer is still there. Enabled per-socket with `SO_KEEPALIVE`.

Linux defaults, and read these numbers carefully:

```
net.ipv4.tcp_keepalive_time   = 7200      # 2 HOURS of idle before the first probe
net.ipv4.tcp_keepalive_intvl  = 75        # 75 s between probes
net.ipv4.tcp_keepalive_probes = 9         # 9 failed probes before declaring death

Worst-case detection time: 7200 + (9 × 75) = 7875 s ≈ 2 hours 11 minutes
```

**The default is useless for the problem we're describing.** A load balancer with a 60-second idle timeout will kill your connection 7,140 seconds before TCP keep-alive sends its first probe. If you want TCP keep-alive to be useful, you must tune it *below* the shortest idle timeout in your path:

```python
import socket, sys

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.setsockopt(socket.SOL_SOCKET, socket.SO_KEEPALIVE, 1)      # portable

if sys.platform.startswith("linux"):
    s.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPIDLE, 30)   # first probe after 30 s idle
    s.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPINTVL, 10)  # then every 10 s
    s.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPCNT, 3)     # 3 failures → dead
    # detection: 30 + 3×10 = 60 s
elif sys.platform == "darwin":
    TCP_KEEPALIVE = 0x10                                        # macOS name for KEEPIDLE
    s.setsockopt(socket.IPPROTO_TCP, TCP_KEEPALIVE, 30)
elif sys.platform == "win32":
    # Windows exposes per-socket keepalive via an ioctl, not setsockopt.
    # struct tcp_keepalive { u_long onoff; u_long keepalivetime; u_long keepaliveinterval; }
    # times are in MILLISECONDS. Probe count is NOT per-socket configurable.
    import struct
    SIO_KEEPALIVE_VALS = 0x98000004
    s.ioctl(SIO_KEEPALIVE_VALS, struct.pack("LLL", 1, 30_000, 10_000))
```

**Honesty caveat, and it's a real portability trap:** `TCP_KEEPIDLE` does not exist on Windows and the constant name differs on macOS; `TCP_KEEPCNT` has no per-socket Windows equivalent (it's a system-wide registry setting). Code that sets `TCP_KEEPIDLE` unconditionally raises `AttributeError` on Windows. The branching above is the honest version, not defensive clutter.

**HTTP keep-alive (application layer, HTTP/1.1 default).** Completely different thing despite the shared name. It means "don't close this TCP connection after the response; I'll send another request on it." It's the *reason* you have idle connections to worry about, not a solution to them. `Connection: keep-alive` is the (redundant, since HTTP/1.1 defaults to it) header; `Connection: close` opts out.

**A third mechanism you actually want more often than TCP keep-alive: application-level pings.** HTTP/2 has `PING` frames; websockets have ping/pong; gRPC has configurable keepalive pings. These are better than TCP keep-alive for two reasons: they traverse proxies that terminate TCP (a TCP keep-alive probe dies at the proxy and tells you only that the *proxy* is alive), and they prove the *application* is responsive, not just the kernel.

**That distinction is the crux of health checking, and it deserves stating plainly:** a TCP keep-alive probe is answered by the peer's *kernel*. A wedged application, a deadlocked thread pool, a process stuck in a 40-second GC pause — all of those answer TCP keep-alives perfectly. **Only an application-layer round-trip proves the application works.**

### The connection pool, and the three timeouts it needs

A pool holds warm connections and hands them out. The mechanism is simple; getting the *lifecycle* right is where everyone fails.

```
┌────────────────────── Connection Pool ─────────────────────┐
│                                                            │
│  idle:  [conn A][conn B][conn C]                           │
│  busy:  [conn D][conn E]                                   │
│                                                            │
│  acquire() → hand out an idle conn, or open a new one,      │
│              or WAIT if at max_connections                  │
│  release() → return to idle (or close if it looks dead)     │
└────────────────────────────────────────────────────────────┘
```

Three timeouts, and **omitting the second is the single most common cause of the "first request after idle fails" bug:**

1. **`max_connections`** — the ceiling. Bounds your load on the backend and your memory.
2. **`max_idle_time`** — how long an idle connection may sit before the pool closes it proactively. **This must be shorter than the shortest idle timeout in your network path.** If the LB kills at 60 s, set this to 30–45 s. Then *you* close it cleanly, on your terms, instead of discovering later that someone else did. This is the fix, and it's a configuration value, not code.
3. **`max_lifetime`** — a hard cap regardless of use, typically 5–30 minutes. Why, if the connection is being actively used and is healthy? Because a long-lived connection is **pinned to whatever it connected to.** After a deploy, a scale-out, or a DNS change, your still-healthy connection is still talking to yesterday's pod. `max_lifetime` forces periodic rebalancing and re-resolution. **A pool without `max_lifetime` silently defeats your load balancing and your DNS-based failover** — a subtle, high-impact failure that looks like "the new instances aren't getting traffic."

### The agentic dimension: LLM streaming is the worst-case shape for all of this

This is the section where the note's framing pays off, and the connection is mechanical rather than thematic. Consider what an LLM API call looks like on the wire:

```
t=0.0s    TCP handshake (1 RTT) + TLS handshake (1–2 RTT)
t=0.3s    HTTP POST with the request body — a few KB of prompt
t=0.3s    ... silence ...
          ... the model is thinking. With adaptive thinking on a hard task,
              this can be many seconds — or, on a long-horizon task at high
              effort, minutes. Nothing crosses the wire. ...
t=14.2s   first SSE event arrives — a few bytes
t=14.3s   another few bytes
t=14.4s   another few bytes          ← a trickle, for potentially minutes
   ...
t=97.6s   message_stop
t=97.6s   ... connection goes idle, held in the pool for the next call ...
```

Every TCP behaviour in this note is being stressed simultaneously:

- **A long silent gap mid-request.** The connection is established and *in use*, but nothing is transmitted for many seconds. Any middlebox measuring "idle" by packet flow may reap it mid-request — and you'll see the stream die partway through with no error from the model's side. This is why streaming is the *recommended* mode for long requests: the token trickle keeps the connection demonstrably active. Non-streaming requests with a large `max_tokens` can sit silent long enough to hit an HTTP-client or proxy timeout, which is exactly why the Anthropic Python SDK raises a `ValueError` if you request a non-streaming call it estimates will exceed roughly ten minutes.
- **Tiny writes in the response.** SSE events are small. Nagle (concept 8) on the server side, delayed ACK on yours — the trickle shape is precisely the pattern where those interact.
- **Long idle gaps between requests.** An agent that calls the API once a minute has a 60-second idle period, which sits exactly on top of the most common LB idle timeout.
- **Connection reuse is disproportionately valuable.** Handshake + TLS is 2–3 RTTs and the `cwnd` starts cold. For a request whose *response* is a trickle of small events, a cold `cwnd` barely matters — but for the *request*, a large prompt with prompt caching still has to be uploaded, and 2–3 RTTs of setup is pure added latency on every call.
- **Head-of-line blocking matters more than usual.** If your agent issues several concurrent tool-backed calls over one HTTP/2 connection, one lost segment stalls *all* their token streams (concept 10). On a long-lived agent session that's a visible hitch across every concurrent stream at once.

### Runnable example — the agent-shaped connection, and how to configure it

```python
# agent_connection_lifetime.py
# pip install anthropic
#
# Streams a response from Claude, and instruments the TCP-visible behaviour:
# time-to-first-byte, inter-event gaps, and total wall time. The gaps are the
# thing to look at — they are the "idle" periods a middlebox might reap.
import os
import time

import anthropic
import httpx

# --- Client configuration: every value here is a TCP-lifetime decision --------
client = anthropic.Anthropic(
    # Total per-request budget. The SDK default is 10 minutes, which is
    # deliberately generous because a long thinking turn is legitimate.
    # Granular control matters more than the total for our purposes:
    timeout=httpx.Timeout(
        600.0,        # overall
        connect=5.0,  # TCP handshake must complete fast, or the host is unreachable
        read=120.0,   # ← max gap BETWEEN chunks, not total read time (see caveat)
        write=30.0,
    ),
    # Retries connection errors, 408/409/429, and 5xx with backoff. Default is 2.
    # Note this is exactly Jacobson's lesson from concept 7 applied at the
    # application layer: bounded retries with backoff, not unbounded retry.
    max_retries=3,
)

t_start = time.perf_counter()
events = []
last = t_start

with client.messages.stream(
    model="claude-opus-5",
    max_tokens=2000,
    messages=[{"role": "user",
               "content": "In three sentences, explain why TCP head-of-line "
                          "blocking motivated QUIC."}],
) as stream:
    for text in stream.text_stream:
        now = time.perf_counter()
        events.append((now - t_start, now - last, len(text)))
        last = now
    final = stream.get_final_message()

total = time.perf_counter() - t_start
ttfb = events[0][0] if events else float("nan")
gaps = [gap for _, gap, _ in events[1:]]

print(f"\n--- TCP-visible shape of one streamed LLM call ---")
print(f"time to first token : {ttfb:6.2f} s   ← handshake + TLS + thinking, "
      f"all with NO bytes flowing")
print(f"total wall time     : {total:6.2f} s")
print(f"chunks received     : {len(events)}")
print(f"largest inter-chunk gap: {max(gaps):6.3f} s" if gaps else "")
print(f"mean  inter-chunk gap  : {sum(gaps)/len(gaps):6.3f} s" if gaps else "")
print(f"output tokens       : {final.usage.output_tokens}")
print(f"\nIf any middlebox on this path reaps connections after "
      f"{ttfb:.0f}s of no traffic,\nthis request dies before the first token arrives.")
```

Representative output:

```
--- TCP-visible shape of one streamed LLM call ---
time to first token :   4.71 s   ← handshake + TLS + thinking, all with NO bytes flowing
total wall time     :   9.38 s
chunks received     :     61
largest inter-chunk gap:  0.412 s
mean  inter-chunk gap  :  0.077 s
output tokens       :    128
```

**Why this works, and why each configuration line is a TCP decision:**

- **`connect=5.0`** — the TCP handshake is one RTT. Five seconds is generous for a handshake and *stingy* for anything else, which is exactly right: if you can't complete a handshake in 5 s, DNS resolved to something unreachable or the host is down. Distinguishing connect failures from read failures lets you distinguish "can't reach the service" from "the service is slow," which are different incidents.
- **`read=120.0`** — and here is a caveat that matters enormously, and that I want to state rather than let you assume: **`read` in `httpx` (and `requests`) is a per-chunk timeout, not a wall-clock cap.** It resets every time a byte arrives. So a stream that trickles one token every 100 seconds forever will never trip a 120 s read timeout. The `600.0` overall value is what bounds total duration. If you need a hard wall-clock deadline on a stream, track `time.monotonic()` in your loop and break out yourself, or wrap the whole thing in `asyncio.wait_for()`. This trips up people constantly, and it's a direct consequence of TCP being a byte stream with no notion of a request.
- **`max_retries=3`** — the SDK retries connection errors and 429/5xx with exponential backoff. This is your defence against the half-open-connection failure: a stale pooled connection produces a connection error on first write, and the retry opens a fresh one. **Retries are not a substitute for correct pool timeouts** — they turn a user-visible error into an invisible extra round trip, which is good, but you still paid the latency and you're still relying on the request being safe to retry.
- **`ttfb` is the number to stare at.** Between the request body and the first token, *no bytes cross the wire*. That interval is indistinguishable, to a middlebox counting idle time, from a connection nobody is using. On a hard task at high effort it can be tens of seconds or more.

**Retry safety, stated plainly because it's the part people skip:** retrying a streaming LLM call that already emitted tokens means the model generates from scratch and you're billed for both attempts. And a request that had *side effects* — a tool call your loop already executed — is not idempotent, so a blind retry can double-execute it. The transport-layer lesson generalises: **automatic retry is only safe for idempotent operations, and TCP cannot tell you whether yours is.** That's your job, at the application layer, exactly as the end-to-end principle predicts.

### System design — health checks that mean something

**Problem:** your service sits behind a load balancer whose health check is a TCP connect to port 8080. During an incident, the app was deadlocked (thread pool exhausted, no request progressing) for 20 minutes. The LB kept routing traffic to it the entire time, because the health check passed. Design a health-check scheme that would have caught it.

**Requirements:** detect an application-level wedge within 30 s; don't remove healthy instances during normal GC pauses or brief spikes; don't create a cascading failure where a slow dependency marks the whole fleet unhealthy.

**Why the TCP check failed, mechanically** — and this is concept 4 paying off directly. The handshake is performed by the **kernel**. A wedged application is not involved. The kernel completed handshakes and stacked connections in the accept queue, and the LB saw "connection succeeded → healthy." **You measured the kernel and inferred something about the application.**

**Alternatives:**

1. **HTTP check on a `/health` endpoint that returns 200.**
2. **HTTP check on `/health` that also verifies critical dependencies (DB ping, cache ping).**
3. **Two-tier: a shallow `/livez` for the LB (is the process serving?) plus a deeper `/readyz` for the orchestrator (should I receive traffic?).**

**The decision: (3).**

**The actual reason:** (1) fixes the original bug — a wedged app can't serve HTTP, so the check fails and the LB removes it. Good. But (2) introduces a *worse* failure: if the shared database has a hiccup, **every instance's health check fails simultaneously**, the LB removes the entire fleet, and a degraded dependency becomes a total outage. This is the classic health-check cascade, and it has taken down real production systems repeatedly. Making a health check depend on a shared resource means a single shared failure marks everything dead at once.

(3) separates the two questions that (1) and (2) conflate:
- **`/livez`** — "is this process functional?" Touches nothing external. Fails only if *this instance* is broken. Restart on failure.
- **`/readyz`** — "should this instance receive traffic *right now*?" May consider dependencies, warm-up state, or in-flight load. Remove from rotation on failure, don't restart.

That distinction is the whole design, and it's why Kubernetes has separate `livenessProbe` and `readinessProbe` rather than one probe.

**The trade-off honestly:** two endpoints means two things to maintain, two sets of semantics for on-call to remember, and a real risk of getting the split wrong — a `/readyz` that checks a dependency *is* a cascade risk if every instance shares that dependency, so you have to be disciplined about what goes in it (per-instance state: warm-up done, local queue depth, circuit-breaker state — not "can I reach the DB"). Option (1) is simpler and catches the incident in the problem statement. If your team is small and your dependencies are few, (1) plus good alerting is a defensible choice, and pretending otherwise is architecture theatre.

**Flip condition:** if instances are genuinely independent and stateless with no shared dependency (a pure compute service), then (2) is safe and simpler, because there's no shared failure to cascade. And if you have no orchestrator — just an LB — then you can't act on `/readyz` differently from `/livez` anyway, so the split buys you nothing; use (1).

**Failure modes:** a health check that's too aggressive (1-second timeout, 2 failures) removes instances during normal GC pauses, so the fleet flaps. A health check that shares a thread pool with real requests can't respond when the pool is exhausted — which is *the exact condition you're trying to detect* — so run it on a separate thread or event loop. And a `/health` endpoint that does real work becomes a DoS vector if it's reachable externally.

### System design — connection management between services

**Problem:** 40 microservices, each calling 3–8 others, deployed on Kubernetes. Symptoms: sporadic `ConnectionResetError`s (a few hundred a day, no pattern), p99 latency 5× p50, and after each deploy some instances get almost no traffic for 20 minutes.

**Requirements:** eliminate spurious resets; reduce tail latency; ensure traffic rebalances after a deploy within 60 s.

**Diagnose all three, because they have three different causes** — and each maps to a concept in this note:

1. **Sporadic resets** → stale pooled connections. Something (kube-proxy conntrack, an LB, a replaced pod) dropped the connection; the client's pool didn't know; the next request on it fails. *Concept 11's half-open failure.*
2. **p99/p50 = 5×** → either pool exhaustion (requests queueing for a connection) or per-request connection setup (handshake RTT + cold `cwnd`). *Concepts 4 and 7.*
3. **Post-deploy imbalance** → long-lived pooled connections pinned to old pods; new pods get connections only when the pool opens new ones, which it doesn't because the old ones work fine. *Concept 11's `max_lifetime` gap.*

**Alternatives:**

1. **Per-request connections** (no pooling). Always fresh, never stale.
2. **Pooling with the three timeouts properly configured.**
3. **A service mesh sidecar** (Envoy/Linkerd) owning all connection management.

**The decision: (2) as the baseline everywhere; (3) if you also need mTLS, retries, and observability uniformly across 40 heterogeneous services.**

**The actual reason:** (1) does eliminate staleness — you can't reuse a dead connection you never kept — but it reintroduces every cost we've established: a handshake RTT per request, a cold `cwnd` per request, and `TIME_WAIT` accumulation at 466 connections/second per destination (concept 9's arithmetic). At 40 services × 8 dependencies you'd hit the ephemeral-port wall well before you hit CPU. (2) addresses all three symptoms with configuration:

```
max_connections     = 100 per destination     # bounds backend load
max_idle_time       =  30 s                   # < any middlebox idle timeout → symptom 1
max_lifetime        = 300 s                   # forces rebalancing → symptom 3
acquire_timeout     =   2 s                   # fail fast instead of queueing → symptom 2
+ TCP_NODELAY                                 # small request/response → concept 8
+ retry once on connection error (idempotent requests only)
```

Note `max_lifetime = 300 s` directly answers symptom 3: within five minutes of a deploy, every pooled connection has been forced to re-establish and therefore re-resolve and re-balance. And `acquire_timeout` answers symptom 2 by converting an invisible queue into a visible error you can alert on.

**The trade-off honestly:** (2) means 40 services each carry pool configuration that must be kept consistent, in whatever HTTP client each language ecosystem uses, with different knob names and different defaults. That drift is real — it's precisely how you end up with 38 services correct and 2 with the original bug. (3) solves the consistency problem by moving connection management out of application code entirely, at the cost of a sidecar per pod (memory, CPU, an extra network hop, an extra thing that can break, and a substantial operational learning curve). For 40 services the mesh is arguably justified; for 5 it is not.

**Flip condition:** skip pooling entirely when request rates are genuinely low (single digits per second) and simplicity is worth more than latency — a cron job calling an API twice an hour should just open a connection. Adopt the mesh when connection policy needs to be uniform and enforced rather than merely documented, and when you want mTLS and retry policy centrally, not reimplemented 40 times.

**Failure modes:** `max_idle_time` longer than the middlebox timeout → symptom 1 persists and you'll conclude pooling "didn't help." No `max_lifetime` → symptom 3 persists. `acquire_timeout` unset (infinite) → under load, requests queue invisibly for connections and your latency graph explodes while CPU sits at 20%, which is the single most confusing performance signature in this whole area. Pool max larger than the backend's accept capacity → you've relocated the queue rather than removed it.

### System design — file upload over flaky mobile networks

**Problem:** a mobile app uploads 50–500 MB video files. Users are on cellular with intermittent coverage, tunnels, lift shafts, and network handovers (Wi-Fi → LTE). Currently a single `PUT` and any interruption loses the whole upload. Design something that survives.

**Requirements:** survive a 5-minute total connectivity loss; survive an IP address change mid-upload; detect corruption; resume without re-uploading completed data; don't require server-side memory proportional to file size.

**What TCP gives you, and what it categorically doesn't** — this is the framing that makes the design obvious:

| TCP provides | TCP does NOT provide |
|---|---|
| Retransmission of lost segments | Survival of connection death |
| In-order delivery | Survival of an IP address change (4-tuple changes → connection is gone) |
| A weak end-to-end checksum | Strong integrity guarantees |
| Congestion adaptation | Any notion of "resume from byte N" |
| Flow control | Knowledge of what the *application* durably persisted |

**Every "does NOT" is a requirement above.** So the design is not "tune TCP," it's "build the missing layer."

**Alternatives:**

1. **Single `PUT` with a very long timeout and TCP keep-alive tuned down.**
2. **Chunked resumable upload**: split into fixed-size chunks, upload each independently, server tracks which chunks it has, client queries and resumes.
3. **`tus` protocol / S3 multipart upload** — standardised versions of (2).

**The decision: (3) — use a standard resumable protocol rather than inventing (2).**

**The actual reason:** (1) cannot work, and the reason is structural rather than a matter of tuning. A 5-minute connectivity loss exceeds any reasonable timeout, and **an IP address change destroys the 4-tuple**, which *is* the connection's identity (concept 2). There is no timeout value that makes a TCP connection survive its own identity changing. Wi-Fi-to-LTE handover is not an edge case on mobile; it's routine. Option (1) is defeated by physics, not by configuration.

(2) is the right shape: each chunk is an independent, short-lived HTTP request, so a failure loses at most one chunk, and a new connection with a new source IP is fine because chunk identity lives in the *application* protocol (`Content-Range` / part number), not in the transport. Choosing (3) over rolling your own (2) is a boring-but-correct call: resumable upload has real subtleties — chunk-size selection, concurrent chunk limits, expiry of partial uploads, idempotent chunk re-submission, final assembly and verification — and `tus` and S3 multipart have already made those decisions and been debugged in the field.

**The trade-off honestly:** chunking costs overhead. Each chunk is a fresh HTTP request: potentially a fresh TCP handshake and TLS negotiation (mitigated by keep-alive within a connectivity window), a fresh cold `cwnd` (concept 7 — so small chunks never leave slow start and never reach the link's capacity), and a per-chunk round trip. Chunks that are too small mean an upload dominated by per-chunk overhead; too large and a failure loses more work. 5–10 MB is a common sweet spot, and it is genuinely a tuning exercise against your users' actual connectivity distribution.

Second real cost: server-side state. The server must remember which chunks it has for each in-flight upload, and garbage-collect abandoned ones. That's a database table and a cleanup job you didn't previously need.

**Flip condition:** use a single `PUT` when files are small (under a few MB), the network is reliable (desktop on Ethernet, server-to-server inside a datacentre), and re-uploading on failure is cheap. The whole resumable apparatus is dead weight there. Also skip chunking when the client can't checkpoint anyway — a streaming source with no ability to seek backwards can't resume from byte N regardless of what the protocol supports.

**Failure modes:** trusting TCP's checksum for integrity → add a per-chunk hash *and* a whole-file hash the server verifies before declaring success (concept 1's honesty note: TCP's 16-bit checksum is weak). No expiry on partial uploads → unbounded storage growth from abandoned uploads. Chunk re-submission not idempotent → duplicated data in the assembled file. And a subtle one: the client believing a chunk succeeded because `send()` returned (concept 3) rather than because the server acknowledged it — **the only proof a chunk landed is the server's response.**

---

## 12. UDP: what you get when you remove everything

**Depth: [CORE]**

### Intuition

Now that you know precisely what TCP does, UDP is easy to define: **UDP is IP plus ports plus a checksum. That's the entire protocol.**

The header is 8 bytes against TCP's 20+:

```
UDP header (8 bytes):
┌────────────────┬────────────────┐
│  source port   │   dest port    │   4 bytes
├────────────────┼────────────────┤
│  length        │   checksum     │   4 bytes
└────────────────┴────────────────┘

TCP header (20 bytes minimum + up to 40 bytes of options):
  source port, dest port, sequence number, acknowledgement number,
  flags (SYN/ACK/FIN/RST/PSH/URG), window, checksum, urgent pointer,
  options (MSS, window scale, SACK, timestamps)
```

Everything TCP has that UDP lacks is a thing UDP won't do for you:

| Not present | So UDP has no… |
|---|---|
| Sequence number | ordering, duplicate detection, or reassembly |
| Acknowledgement number | delivery confirmation or retransmission |
| Flags | connection setup or teardown — no handshake |
| Window | flow control |
| (no congestion state) | congestion control |

The right way to hold this is: **UDP is not "TCP with the safety off." It is a different tool — a way to send an individual, self-contained datagram and be told nothing about what happened to it.** That framing is important because "unreliable" makes UDP sound like a worse TCP, and for a large class of problems it is straightforwardly the better choice.

### The properties UDP *has*, which are the reason to use it

Stated positively, because the negative framing hides the value:

1. **Message boundaries are preserved.** One `sendto()` = one datagram = one `recvfrom()`. This is the *opposite* of TCP's byte-stream behaviour, and it eliminates the entire framing problem of concept 3. If your data is naturally a message, UDP matches it.
2. **No head-of-line blocking, ever.** Each datagram is independent (concept 10). A lost one blocks nothing.
3. **No connection setup.** Zero RTTs before your first byte. For a single request-response exchange, this alone can halve total latency.
4. **No connection state.** A UDP server can serve millions of clients with one socket and no per-client kernel memory. This is why DNS root servers can answer the world.
5. **No retransmission of stale data.** For real-time media, a 200 ms-old audio frame is *worthless* — you've already played past it. TCP would retransmit it and stall everything behind it; UDP lets it go and you play the next frame. **Here TCP's reliability is actively harmful.**
6. **Multicast and broadcast are possible.** TCP is inherently point-to-point (a 4-tuple has one peer). UDP can address a group.

### Analogy: postcards vs a phone call

TCP is a phone call: you dial, they answer, you both confirm you can hear each other, you converse, you say goodbye, you hang up. Ordered, confirmed, and it costs setup time.

UDP is dropping a postcard in a postbox. No dialling, no confirmation, no goodbye. It might not arrive. But it took a fraction of a second and you can send a thousand to a thousand people just as easily as one.

**Where the analogy breaks:** postcards are slow and phone calls are fast; UDP and TCP travel at identical speed over identical wires. The difference is *not* transmission speed, it's the round trips of overhead and the delays introduced by ordering and retransmission. Also, postcards to the same address arrive roughly in order in practice; UDP datagrams genuinely and routinely arrive out of order. Don't let the analogy suggest "usually fine."

### Where UDP is the right answer — and specifically why

**DNS (Day 12's entire subject).** A query is one small message and the answer is one small message. Using TCP would mean a handshake (1 RTT) before a query that itself takes 1 RTT — **tripling the latency of the most latency-critical operation on the internet**, since DNS is the first hop of every request. And a DNS server would need per-client connection state, which for a root server serving the planet is impossible. Reliability is handled trivially at the application layer: if no answer arrives in a couple of seconds, ask again — possibly a different server. *That's the whole retry protocol.* DNS falls back to TCP only when the response exceeds what fits in a datagram (large DNSSEC responses, zone transfers), which is exactly the case where UDP's advantages stop applying.

**Real-time media (voice, video calls, live streaming).** Retransmitting a lost audio frame is worse than useless: by the time it arrives the playback point has passed, and TCP's in-order delivery would have stalled every *subsequent* frame waiting for it. **You would trade one lost 20 ms of audio for 300 ms of silence.** UDP lets the codec conceal the gap (interpolate, repeat, mute briefly) and keep going. This is a case where TCP's core guarantee is the wrong guarantee.

**Gaming.** Position updates at 60 Hz: update #100 makes update #99 *obsolete*. Retransmitting #99 is pointless. And a stall waiting for #99 while #100 and #101 sit in a buffer is exactly the wrong behaviour for an interactive game.

**Metrics/telemetry at high volume (StatsD).** Losing 0.1% of metrics datagrams is irrelevant to an aggregate; blocking your application thread because a metrics backend is slow is unacceptable. UDP's fire-and-forget is precisely the desired semantic — and this is a case where the "unreliability" is a *feature* providing isolation from a non-critical dependency.

**QUIC / HTTP/3.** Uses UDP not because it wants unreliability, but because it wants to build reliability *in userspace where it can be evolved*. Concept 13.

**And a case where UDP is straightforwardly wrong:** a database wire protocol, a file transfer, an API call. Anything where losing part of the data means the result is wrong rather than slightly degraded. If you would have to reimplement retransmission, ordering, and congestion control, you should use TCP — because you will implement all three worse than the kernel does.

### Runnable example — UDP, and cataloguing what didn't happen

```python
# udp_vs_tcp.py — same logical exchange over both transports, side by side.
# The interesting output is the list of things UDP DIDN'T do.
import socket
import threading
import time

HOST = "127.0.0.1"
UDP_PORT, TCP_PORT = 9110, 9111
MESSAGES = [b"MSG-1", b"MSG-2", b"MSG-3", b"MSG-4"]


# ---------- UDP ----------
def udp_server(out: list):
    s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)   # SOCK_DGRAM = UDP
    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    s.bind((HOST, UDP_PORT))
    # NOTE: no listen(), no accept(). There is no connection to accept.
    for _ in range(len(MESSAGES)):
        data, addr = s.recvfrom(65535)      # returns (payload, sender address)
        out.append((data, addr))
    s.close()


udp_out: list = []
t = threading.Thread(target=udp_server, args=(udp_out,)); t.start()
time.sleep(0.2)

t0 = time.perf_counter()
u = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
for m in MESSAGES:
    u.sendto(m, (HOST, UDP_PORT))          # no connect() needed — address per send
udp_elapsed = time.perf_counter() - t0
u.close(); t.join()

print("UDP:")
print(f"  4 sendto() calls → {len(udp_out)} recvfrom() results "
      f"(message boundaries PRESERVED)")
for data, addr in udp_out:
    print(f"    {data!r} from {addr}")
print(f"  elapsed: {udp_elapsed * 1000:.3f} ms  (zero round trips of setup)")


# ---------- TCP, for contrast ----------
def tcp_server(out: list):
    lis = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    lis.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    lis.bind((HOST, TCP_PORT)); lis.listen(1)
    conn, _ = lis.accept()
    time.sleep(0.3)                          # let sends accumulate
    while chunk := conn.recv(65536):
        out.append(chunk)
    conn.close(); lis.close()


tcp_out: list = []
t = threading.Thread(target=tcp_server, args=(tcp_out,)); t.start()
time.sleep(0.2)

t0 = time.perf_counter()
c = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
c.connect((HOST, TCP_PORT))                  # ← a full handshake happens HERE
connect_ms = (time.perf_counter() - t0) * 1000
for m in MESSAGES:
    c.send(m)
c.close(); t.join()

print("\nTCP:")
print(f"  connect() (3-way handshake) took {connect_ms:.3f} ms "
      f"— on a 50 ms-RTT path this would be ~50 ms")
print(f"  4 send() calls → {len(tcp_out)} recv() results "
      f"(message boundaries DESTROYED)")
for chunk in tcp_out:
    print(f"    {chunk!r}")

print("""
What UDP did NOT do, in order of how much it matters:
  1. No handshake            → no setup round trip (0 RTT vs 1 RTT)
  2. No ordering guarantee   → datagrams may arrive in any order
  3. No retransmission       → a lost datagram is simply gone, silently
  4. No duplicate detection  → the same datagram may be delivered twice
  5. No flow control         → you can overwhelm the receiver freely
  6. No congestion control   → you can overwhelm the NETWORK freely (see below)
  7. No connection state     → the server holds nothing per client
  8. PRESERVED your message boundaries → the one thing it gave you back
""")
```

Output:

```
UDP:
  4 sendto() calls → 4 recvfrom() results (message boundaries PRESERVED)
    b'MSG-1' from ('127.0.0.1', 61204)
    b'MSG-2' from ('127.0.0.1', 61204)
    b'MSG-3' from ('127.0.0.1', 61204)
    b'MSG-4' from ('127.0.0.1', 61204)
  elapsed: 0.061 ms  (zero round trips of setup)

TCP:
  connect() (3-way handshake) took 0.412 ms — on a 50 ms-RTT path this would be ~50 ms
  4 send() calls → 1 recv() results (message boundaries DESTROYED)
    b'MSG-1MSG-2MSG-3MSG-4'
```

**Why this works, and the two lines to take away:**

- `SOCK_DGRAM` instead of `SOCK_STREAM` is the entire API difference at socket-creation time. Everything else follows.
- **There is no `listen()` and no `accept()`** in the UDP server, because there is nothing to accept. `recvfrom()` returns the sender's address with each datagram, because the socket isn't bound to a peer — one socket serves all clients. That's property 4 above (no per-client state) made visible in the API shape.
- `sendto()` takes the destination *per call*. You can interleave datagrams to a hundred different peers on one socket.
- **4 datagrams in → 4 datagrams out, intact.** Contrast the TCP run: 4 sends → 1 read. That's the framing difference, and it's the one thing UDP hands you that TCP takes away.
- `connect()` on the TCP socket is where the handshake happens, and timing it separately makes visible the cost UDP avoids. On loopback it's 0.4 ms and irrelevant; on a real path it's a full RTT and it's the reason DNS uses UDP.

**Honesty caveat:** on loopback, UDP does not drop, reorder, or duplicate anything, so this example demonstrates the *API and header* differences but cannot demonstrate the failure modes. Do not conclude UDP is reliable in practice. To see real loss and reordering you must impair the path:

```bash
# Linux: 10% loss and 20 ms of reordering jitter on loopback
sudo tc qdisc add dev lo root netem loss 10% delay 20ms 10ms
python udp_vs_tcp.py     # some UDP messages will simply not arrive; TCP's all will
sudo tc qdisc del dev lo root
```

**Run that.** Watching TCP deliver all four messages while UDP delivers two, over the identical impaired path, is the clearest possible demonstration of what the extra 12 header bytes buy.

### What you must build yourself if you choose UDP

If you use UDP for something that *does* need reliability, you inherit all of TCP's problems. Be explicit about which ones you're solving:

| Need | You must implement |
|---|---|
| Reliability | sequence numbers, ACKs, retransmission timers |
| Ordering | sequence numbers + a reorder buffer |
| Duplicate suppression | seen-sequence tracking with a bounded window |
| Flow control | your own window/credit scheme |
| **Congestion control** | **backoff on loss — and this is not optional** |
| Path MTU discovery | probing, or conservatively small datagrams |
| Connection liveness | your own heartbeats and timeouts |

**Congestion control is the ethical obligation, not a nicety.** A UDP application that sends as fast as it can, with no backoff on loss, is exactly the 1986 congestion-collapse behaviour (concept 7) — it will starve every TCP flow sharing its bottleneck, because TCP backs off and you don't. This is *the* reason to be suspicious of "we used UDP for speed." Speed obtained by refusing to back off is speed stolen from your neighbours, and at scale it degrades the network you depend on.

If you find yourself implementing more than two rows of that table, **stop and use QUIC.** It is a well-tested, standardised implementation of all of them, in userspace, over UDP. Rebuilding it is a multi-year project that the industry has already done.

### The IP fragmentation trap, and the practical datagram size limit

A UDP datagram can theoretically be 65,507 bytes of payload. In practice, **do not exceed roughly 1,400 bytes** on the open internet, and the reason is worth understanding rather than memorising.

If your datagram exceeds the path MTU (typically 1500 bytes for Ethernet, less over tunnels and PPPoE), **IP fragments it** into multiple packets (Day 10). Then:

1. **Losing any single fragment destroys the entire datagram.** The receiver cannot reassemble and discards everything it holds. A 4 KB datagram over a 1500-byte MTU becomes 3 fragments; with 1% per-packet loss, the datagram's effective loss rate is roughly 1 − 0.99³ ≈ **3%**. Fragmentation *multiplies* your loss rate by the fragment count.
2. **Many firewalls and NATs drop fragments** outright, because only the first fragment carries the UDP header (and therefore the port numbers) — so a stateless filter can't classify the rest. This produces the maddening symptom: small datagrams work, large ones vanish, with no error anywhere.
3. **Reassembly is a DoS surface**, which is why some middleboxes drop fragments on principle.

Practical guidance:

```
Ethernet MTU:                        1500
  − IPv4 header                        20
  − UDP header                          8
  = safe UDP payload                 1472

Conservative real-world figure:  ~1200 bytes
  (leaves room for tunnels, VPNs, PPPoE, IPv6's 40-byte header, and QUIC's
   own overhead — and note QUIC mandates a minimum path MTU of 1200 for
   exactly this reason)
```

DNS's historic 512-byte limit (Day 12) comes from precisely this concern, and EDNS0's negotiated larger sizes reintroduce the fragmentation problem — which is one of the reasons DNS-over-TCP and DNS-over-TLS matter more now than they used to.

### The amplification problem: UDP's structural security flaw

This is the one security property you must internalise before deploying anything on UDP.

UDP has no handshake, so **there is nothing that verifies the source address.** An attacker sends a small query with a *forged* source address (the victim's), and the server dutifully sends a large response to the victim. The attacker's bandwidth is multiplied by the amplification factor.

```
Attacker (spoofing victim's IP)              Server              Victim
   │── 60-byte query ──────────────────────────►│
   │   (source address: VICTIM)                 │
   │                                            │── 4,000-byte ──►│
   │                                            │   response      │
   
   Amplification factor: ~66×
   1 Gbit/s of attacker traffic → 66 Gbit/s at the victim
```

Measured amplification factors for real protocols (US-CERT TA14-013A):

| Protocol | Amplification |
|---|---|
| DNS | 28–54× |
| NTP (`monlist`) | ~557× |
| memcached | up to ~51,000× |
| CLDAP | ~56–70× |

The memcached figure is not a typo, and it's how the **1.35 Tbit/s attack on GitHub (Feb 2018)** was performed — thousands of memcached servers exposed on UDP port 11211 with no authentication, reflecting a small request into an enormous response. GitHub's own postmortem documents it; the mitigation was Akamai/Prolexic scrubbing, and the industry-wide fix was closing those UDP ports.

**The engineering lessons, and they're general:**

1. **Never expose a UDP service that returns more than it receives, to untrusted networks.** memcached on UDP was a *default-on* feature that had no business being reachable from the internet — and after the attack, memcached shipped with UDP disabled by default. That's the correct fix: change the default, because most operators never change defaults.
2. **Response-rate limiting** is the mitigation where you must serve untrusted clients (DNS servers implement RRL for exactly this).
3. **The root cause is source-address spoofing**, which BCP 38 would prevent, which brings us back to concept 4's unhappy ending: the fix has existed since 2000 and remains unevenly deployed.
4. **A protocol that answers before verifying the asker is an amplifier.** QUIC's design addresses this deliberately: it requires that a server's initial response not substantially exceed the client's initial packet (hence the mandated 1200-byte minimum client packet), and its Retry mechanism forces address validation before expensive work. **That is concept 4's SYN-cookie lesson, applied at design time instead of retrofitted.**

### System design — a real-time telemetry ingestion pipeline

**Problem:** 50,000 IoT devices each report a 200-byte sensor reading every second — 50,000 messages/second, 10 MB/s. Losing a small fraction is acceptable (readings are aggregated into per-minute averages). Devices are on cellular with variable connectivity. Design the ingestion transport.

**Requirements:** handle 50k msg/s on modest hardware; tolerate a few percent loss; don't hold 50,000 concurrent connections; devices are battery-powered so minimise radio-on time.

**Alternatives:**

1. **TCP with a persistent connection per device.**
2. **UDP, fire-and-forget, per reading.**
3. **UDP with batching (10 readings per datagram) plus a sequence number for loss measurement.**

**The decision: (3).**

**The actual reason:** run the numbers on each.

*Option (1):* 50,000 concurrent TCP connections means 50,000 × (send buffer + receive buffer + connection state). At even 20 KB per connection that's **1 GB of kernel memory** just for sockets. Worse for battery: a persistent TCP connection needs keep-alives to survive cellular NAT timeouts, and **every keep-alive wakes the radio**, which is the dominant power cost on a cellular IoT device. TCP's reliability is also buying us nothing here — we explicitly don't need every reading.

*Option (2):* right transport, wrong granularity. 50,000 datagrams/second means 50,000 packets/second of *packet-rate* processing, and per-packet overhead (interrupt, header parse, syscall) dominates when payloads are 200 bytes. Also, 200 bytes in a 28-byte-header packet is 12% overhead.

*Option (3):* batching 10 readings gives 2,000 bytes per datagram — **but that exceeds the safe MTU**, so we batch 6 readings (~1,200 bytes) instead and get 8,333 datagrams/second. Packet rate drops 6×, header overhead drops to ~2%, and the radio transmits in bursts every 6 seconds instead of continuously, which is dramatically better for battery. The sequence number costs 4 bytes per datagram and gives us something valuable: **the receiver can measure the actual loss rate**, so "acceptable loss" becomes a monitored number rather than an assumption.

**The trade-off honestly:** batching adds up to 6 seconds of latency to each reading, and it makes loss *lumpier* — one lost datagram now loses 6 readings instead of 1. For per-minute averages that's fine (you still have 54 of 60 samples); for anomaly detection that needs sub-second reaction it is not. You are trading latency and loss granularity for packet rate and battery life, and if the requirement changed to "alert within 2 seconds of a threshold breach" the batching window would have to shrink or become adaptive (send immediately on an anomalous reading, batch otherwise).

Second honest cost: choosing UDP means you own the amplification question. This ingestion endpoint must not be an amplifier — so it must respond with either nothing or something tiny, and it should rate-limit per source. Also, without a handshake, **anyone can inject forged readings**, so you need application-level authentication (a per-device key and a MAC over the payload) that TCP+TLS would have given you structurally.

**Flip condition:** switch to TCP (or QUIC, or MQTT-over-TLS) the moment any of these becomes true: (a) readings become individually valuable so loss is unacceptable (billing meters, safety sensors), (b) the device count drops enough that connection state is affordable (5,000 devices is 100 MB — fine), or (c) you need bidirectional command-and-control to devices, where a connection's existence is genuinely useful. The last one is subtle and often decisive: **UDP gives you no way to reach a device behind NAT**, because there's no connection holding the NAT mapping open. If you need to send commands *to* devices, you need either a persistent connection or a polling protocol, and at that point TCP/QUIC's connection is a feature.

**Failure modes:** datagrams over the MTU → fragmentation → the loss-rate multiplication described above, presenting as "large batches mysteriously fail." No sequence numbers → loss is invisible, so a 30% loss rate looks identical to a 0.1% one and you find out from wrong dashboards. No per-source rate limiting → one malfunctioning device floods your ingest. No authentication → forged data.

### Case study — the GitHub memcached amplification attack (28 February 2018)

**What happened:** GitHub was hit by what was then the largest recorded DDoS attack — **1.35 Tbit/s** at peak, sustained for roughly 8 minutes with a second spike after. The traffic was UDP reflection off exposed memcached servers on port 11211. GitHub's edge was overwhelmed; they moved traffic to Akamai's Prolexic scrubbing service, and service was restored in under 10 minutes of total impact.

**The mechanism, precisely:** memcached, by default at the time, listened on UDP port 11211 with **no authentication**. An attacker could store a large value and then issue a tiny `get` request with a spoofed source address; the server would send the large value to the spoofed address. The amplification factor reached roughly 51,000× — the highest of any known reflector. Thousands of memcached instances were reachable from the internet because operators had deployed them on cloud hosts without firewalling, assuming (reasonably, but wrongly) that a cache is an internal service.

**The engineering lessons, and there are three distinct ones:**

1. **The vulnerability was a *default*, not a bug.** memcached's UDP listener worked exactly as designed. The failure was that "listen on UDP, no auth" was the out-of-the-box configuration, and most operators never change defaults. The industry fix was memcached 1.5.6 disabling UDP by default — **changing the default was more effective than any amount of documentation**, which is a lesson about how security actually improves.
2. **UDP services must be firewalled by default, not by intention.** The affected servers were not misconfigured in any way an operator would have noticed; they were simply reachable. Any UDP service on a public interface should be treated as an amplifier until proven otherwise.
3. **The protocol-level root cause is unverified source addresses**, and the durable fix is address validation before expensive response — which is exactly what QUIC builds in (concept 13) and exactly what SYN cookies retrofitted onto TCP (concept 4). The same idea, arrived at three times.

*Primary sources:* GitHub Engineering blog, "February 28th DDoS Incident Report" (1 March 2018); Cloudflare's "Memcrashed" analysis; US-CERT TA14-013A for the amplification-factor table. Verify current amplification figures and default configurations before relying on them — both have changed since 2018, mostly for the better.

---

## 13. QUIC and HTTP/3: reliability rebuilt in userspace

**Depth: [WORKING]**

### Intuition

QUIC is the answer to a specific and slightly awkward question: *TCP has four problems we cannot fix. What now?*

The four problems, each of which you now understand from first principles:

1. **Head-of-line blocking across multiplexed streams** (concept 10) — TCP delivers one ordered byte stream, so one loss stalls everything sharing the connection.
2. **Handshake latency** — 1 RTT for TCP plus 1–2 for TLS, before any data.
3. **Connection identity is the 4-tuple** (concept 2) — so changing network (Wi-Fi → cellular) destroys the connection.
4. **TCP is in the kernel and ossified by middleboxes** — deploying a TCP change requires OS upgrades on both ends *and* middleboxes that don't mangle unfamiliar options. TCP Fast Open's patchy deployment is the cautionary tale.

Problem 4 is the one that forced the shape of the answer. You cannot iterate on TCP at internet speed. But you *can* iterate on a userspace protocol shipped inside a browser and a server binary — and to get a userspace protocol past middleboxes, you need a transport they already pass unmolested. That's UDP.

So: **QUIC is a reliable, multiplexed, encrypted transport implemented in userspace on top of UDP.** It uses UDP not for unreliability but as a substrate that middleboxes already forward and that requires no kernel changes to build on. *Primary sources:* RFC 9000 (QUIC transport), RFC 9001 (QUIC-TLS), RFC 9002 (loss detection and congestion control), RFC 9114 (HTTP/3).

### What QUIC changes, mapped to the four problems

**1. Streams are independent — head-of-line blocking is gone (across streams).**

```
QUIC connection
├── stream 1  ──────────  independent ordering & loss recovery
├── stream 2  ──────────
├── stream 3  ──X───────  ← loss here stalls ONLY stream 3
└── stream 4  ──────────

Streams 1, 2, 4 are delivered immediately. Compare concept 10's TCP diagram.
```

Note the honest scope: ordering is still guaranteed *within* a stream, so a loss in stream 3 does block the rest of stream 3. QUIC didn't eliminate head-of-line blocking; it **reduced its granularity from connection to stream**, which is precisely the "apply the guarantee where it belongs" lesson from concept 10.

**2. Handshake merges transport and crypto: 1 RTT, or 0-RTT on resumption.**

```
TCP + TLS 1.3:  SYN → SYN-ACK → ACK  then  ClientHello → ServerHello → ...
                └─── 1 RTT ────┘          └──── 1 RTT ────┘        = 2 RTT

QUIC:           Initial (crypto + transport params) → response      = 1 RTT
QUIC 0-RTT:     Initial + application data, using a cached key      = 0 RTT
```

TLS is not layered *on* QUIC, it's integrated *into* it — the crypto handshake and the transport handshake are the same exchange. On a 100 ms path that's 100–200 ms saved per new connection.

**3. Connection IDs replace the 4-tuple as connection identity — connections survive network changes.** Each QUIC connection has a Connection ID in its header. When your phone moves from Wi-Fi to cellular and its IP changes, the 4-tuple changes but the Connection ID doesn't, so the server recognises the connection and it continues. **This directly answers concept 2's flip condition** — and it's why a video call over QUIC survives a handover that would kill a TCP connection.

**4. Everything except a small header is encrypted, including transport metadata.** ACKs, sequence numbers, and connection-control frames are inside the encrypted payload. Middleboxes cannot read or rewrite them, which means they **cannot ossify QUIC the way they ossified TCP.** This is deliberate: encryption here is as much an evolvability mechanism as a privacy one. It's also why QUIC can change its loss-recovery algorithm in a browser update, which TCP structurally cannot.

### Worked example — connection establishment, counted in round trips

Loading `https://example.com` and fetching one 20 KB resource, RTT 100 ms:

```
HTTP/1.1 over TCP+TLS 1.3:
  t=0    SYN
  t=100  SYN-ACK          (1 RTT: TCP)
  t=100  ACK + ClientHello
  t=200  ServerHello, Finished   (1 RTT: TLS)
  t=200  HTTP GET
  t=300  first response byte
  ───────────────────────────────────  300 ms to first byte

HTTP/3 over QUIC (first visit):
  t=0    Initial (crypto + transport params)
  t=100  Initial response + Handshake
  t=100  HTTP/3 request (on a QUIC stream)
  t=200  first response byte
  ───────────────────────────────────  200 ms to first byte   (−33%)

HTTP/3 over QUIC (0-RTT, returning visitor with a cached ticket):
  t=0    Initial + early data: the HTTP request IS in the first packet
  t=100  first response byte
  ───────────────────────────────────  100 ms to first byte   (−66%)
```

**The 0-RTT caveat, stated because it's a genuine correctness constraint, not a footnote:** 0-RTT data is **replayable**. An attacker who captures the first packet can resend it, and the server will process it again, because there's been no round trip to prove freshness. So 0-RTT is only safe for **idempotent** requests — a `GET` is fine, a `POST /transfer-money` is not. Implementations restrict what may go in early data for exactly this reason. This is the same idempotency constraint that appeared in concept 11's retry discussion, and it's the same shape: **any optimisation that removes a round trip removes a freshness proof, and the application must supply idempotency in its place.**

### Judgment: should you deploy HTTP/3?

**The decision:** enable HTTP/3 on your edge.

**The realistic alternative:** stay on HTTP/2 over TCP, which is universally supported, universally debuggable, and already fast.

**The actual reason to move:** the gains concentrate exactly where your worst-performing users are — mobile clients on lossy networks, high-RTT international paths, and networks with frequent handovers. Those are the users whose experience your p99 reflects, and HTTP/3's three wins (no cross-stream HoL blocking, fewer setup RTTs, connection migration) all target them specifically. On a fast wired connection with negligible loss, HTTP/3 and HTTP/2 are close to indistinguishable — the honest framing is that HTTP/3 is a **tail-latency** improvement, not a throughput one.

**The trade-offs, honestly, and there are several real ones:**

- **UDP is blocked or throttled on some enterprise and hotel networks.** So you must run HTTP/2 in parallel and advertise HTTP/3 via `Alt-Svc`, letting clients discover and fall back. You are now operating two protocol stacks, not one.
- **Higher CPU cost per byte.** TCP benefits from decades of kernel optimisation and NIC hardware offload (TSO, LRO, checksum offload). QUIC runs in userspace with per-packet crypto and, until recently, little hardware assistance. Measured CPU overhead has been substantial (Google and Fastly have published figures in the range of 2× per byte in early deployments, improving with `sendmsg` segmentation offload and kernel GSO support). At CDN scale that is a real hardware bill.
- **Observability is worse.** Your existing TCP tooling — `ss`, `netstat`, connection-state dashboards, most Wireshark analysis, middlebox flow logs — sees UDP packets and nothing else. Debugging requires QUIC-aware tooling (`qlog`, `qvis`, Wireshark's QUIC dissector with keys) and, crucially, **the encryption that prevents ossification also prevents *you* from inspecting your own traffic** without deliberately exporting keys. Teams underestimate this until their first HTTP/3 incident.
- **It's newer.** Fewer engineers can debug it, and library maturity varies by language.

**Flip condition:** skip HTTP/3 if your traffic is server-to-server inside a datacentre (negligible loss and RTT means nothing to gain, and you'd pay the CPU cost for free), or if you're on a platform whose HTTP/3 support is immature, or if your operational tooling can't see it and you're not prepared to invest in tooling that can. And note that **enabling HTTP/3 at a CDN edge is nearly free** — the CDN operates it and absorbs the complexity — while running it on your own origin fleet is a real project. That asymmetry means the correct answer for most teams is "yes at the edge, not yet at the origin."

**Deliberate stop:** we are not opening QUIC's frame types, packet number spaces, loss-detection algorithm, or the details of QPACK header compression. Know that QUIC reimplements everything in concepts 5–7 (its loss recovery and congestion control are specified in RFC 9002 and are recognisably descendants of what you've just learned, with the benefit of hindsight — e.g. it eliminates TCP's retransmission ambiguity by never reusing a packet number). **Treat QUIC's internals as a black box unless you're implementing a QUIC stack**; what you need operationally is the four changes above and the trade-offs.

### Case study — why Google built QUIC (and the measurement that justified it)

**What happened:** Google deployed SPDY (which became HTTP/2) and measured a real improvement — but found the improvement *degraded on lossy networks*, because concentrating all requests onto one TCP connection meant one loss stalled everything (concept 10). They had built a protocol whose central feature was undermined by the transport beneath it.

Rather than propose a TCP change (which would take a decade to deploy through OS vendors and middleboxes), they built QUIC in userspace over UDP, shipped it in Chrome and on Google's servers, and iterated with live A/B measurement across an enormous user base. Published results (Langley et al., "The QUIC Transport Protocol: Design and Internet-Scale Deployment," SIGCOMM 2017) reported meaningful reductions in search latency and in video rebuffer rates, **with the largest gains on the highest-latency and most-lossy connections** — exactly where the theory predicted.

**The engineering lessons, and the second is the more valuable one:**

1. **A layered architecture can pin you.** HTTP/2's multiplexing was correct and its benefit was capped by an assumption in a lower layer that HTTP/2 could not change. When your optimisation is limited by a layer below you, the options are to fix that layer (slow, if you don't control it) or to replace it (expensive, but possible if you control both endpoints).
2. **Deployability is a first-class design constraint, not an afterthought.** QUIC-over-UDP is not the technically cleanest design — a new IP protocol number would be cleaner, and SCTP already existed and solved several of the same problems. QUIC won because it can be deployed *today*, through existing middleboxes, with a software update at each end and no kernel involvement. SCTP was arguably technically superior and is essentially undeployable on the public internet because middleboxes drop it. **The best protocol that cannot be deployed loses to a worse protocol that can.** That is one of the most useful pieces of engineering judgment in this note, and it generalises far beyond networking — to database migrations, API versioning, and library upgrades.

*Primary sources:* Langley et al., SIGCOMM 2017; RFC 9000/9001/9002; RFC 9114. Verify current adoption figures (Cloudflare Radar and W3Techs track HTTP/3 share) — they change quarterly.

---

## 14. Bufferbloat: when big buffers make things worse

**Depth: [AWARE]**

Concept 7 established that TCP fills a queue until it overflows, because loss is its congestion signal. Now notice the perverse consequence: **if a router has a very large buffer, TCP will fill it — and a full buffer is pure added latency for everyone using that link.**

Concretely. A home router with a 1 MB buffer on a 10 Mbit/s uplink:

```
1,000,000 bytes ÷ 1,250,000 bytes/s = 0.8 seconds of queueing delay
```

Your download is running at full speed and utilisation looks perfect — and every other packet crossing that link now waits up to 800 ms behind your bulk data. Your video call stutters, your SSH session becomes unusable, DNS lookups take a second. **The link is fast and the experience is terrible**, and the cause is a buffer someone added to be helpful.

Router vendors added large buffers reasoning "buffers prevent loss, loss is bad, therefore more buffer is better." But loss is TCP's *signal*, so a large buffer doesn't prevent congestion — it **delays the signal** while the queue fills, converting a small amount of loss into a large amount of latency. This is the classic case of an intervention that optimises the wrong metric: throughput went up slightly, latency went up enormously, and the user experience is dominated by latency.

Fixes, all of which attack the "signal too late" problem:

- **AQM (Active Queue Management)** — **CoDel** and **FQ-CoDel** deliberately drop or mark packets *before* the queue is full, so TCP gets its signal while the queue is still short. FQ-CoDel additionally gives each flow its own queue, so your bulk download can't delay your video call at all.
- **ECN (Explicit Congestion Notification)** — routers *mark* packets instead of dropping them, letting the sender slow down without losing data. Congestion signal with no retransmission cost.
- **BBR** (concept 7) — sender-side: model the bottleneck and don't fill the queue in the first place.

**Deliberate stop:** the theory of AQM (CoDel's target/interval parameters, the fair-queueing scheduler internals) is a genuinely deep area and we are not opening it. What you need is the recognition pattern: **if latency rises sharply whenever a link is busy, and utilisation looks healthy, suspect bufferbloat, not bandwidth.** The diagnostic is a `ping` running alongside a bulk transfer — if RTT jumps from 20 ms to 500 ms while the transfer runs, that's bufferbloat, and you can measure it at `waveform.com/tools/bufferbloat` or with `flent`. Treat as a black box unless a project forces you deeper.

---
## Case study — the load-balancer idle-timeout bug (two correct components, one broken system)

This is the case study to internalise, because unlike the SYN flood or the memcached amplification it will happen *to you*, in a system where nothing is misconfigured.

**The setup.** An application connects to a database (or a backend service) through a load balancer. The application uses a connection pool that keeps idle connections indefinitely, because that's the sensible thing to do — connections are expensive (concepts 4, 7, 9). The load balancer closes connections idle for more than 60 seconds, because holding state for dead connections is wasteful and 60 seconds is a sensible default. **Both decisions are correct in isolation.**

**The symptom.** The first query after a quiet period fails with a connection reset. The immediate retry succeeds. The failure rate correlates with *how quiet* the traffic is: heaviest at 4 a.m., after lunch, on weekends, and immediately after a deploy that reduced traffic. Sometimes it clears up when traffic increases and everyone forgets about it.

**What's actually happening, in the vocabulary you now have:**

```
t=0     Pool checks a connection back in. It is now idle.
        Client state:  ESTABLISHED     LB state: tracked     Server state: ESTABLISHED

t=60    LB's idle timer fires. It removes its connection-tracking entry.
        Depending on the LB, it either sends RSTs to both sides, or (very
        commonly for NAT-style LBs and stateful firewalls) silently forgets.
        Client state:  ESTABLISHED     LB state: NOTHING     Server state: ESTABLISHED
                       ^^^^^^^^^^^^^^                        ^^^^^^^^^^^^^^^^
                       both endpoints still believe in a connection that has no path

t=300   Application needs a connection. The pool hands it this one — it looks
        perfectly healthy, because from the client's kernel's point of view it IS
        established. There is nothing to check.

t=300   Client writes the query. Packet reaches the LB. No matching entry.
        → LB sends RST (fast failure), or drops silently (the client then
          retransmits for many minutes — concept 5's black-hole row).

t=300   Application: ConnectionResetError, or a hang. On a query the developer
        knows is trivial. Once. And then it works.
```

**Why it is so hard to diagnose, and this is the instructive part:**

1. **No component logged an error.** The LB did what it was configured to do, at a level it doesn't log. The pool handed out a connection its API reported as healthy. The kernel had no reason to complain.
2. **It is anti-correlated with load**, which inverts every debugging instinct. Engineers look for problems under load; this one *disappears* under load, because busy pools have no idle connections.
3. **The retry succeeds**, so the incident looks transient and gets closed as "network blip."
4. **Concept 3's lesson bites:** there is no way to ask a socket "are you actually still connected?" A socket's `ESTABLISHED` state is one endpoint's *memory*, not a fact about the world. The only way to find out is to send something and see what happens — which is precisely the failed query.

**The fixes, and the ranking matters:**

| Fix | Where | Quality |
|---|---|---|
| Pool `max_idle_time` **shorter** than the LB's idle timeout (e.g. 30 s vs 60 s) | client config | **Best — removes the cause.** *You* close the connection cleanly, on your terms, before anyone else can forget it. |
| Tune TCP keep-alive below the LB timeout (`TCP_KEEPIDLE=30`) | client socket opts | **Good.** Traffic on the connection resets the LB's idle timer, so it never expires. Requires per-platform code (concept 11). |
| Pool validation query before handing out a connection | client code | **Works, costs a round trip per checkout.** Postgres pools call this `test-on-borrow`. |
| Retry once on connection error for idempotent operations | client code | **Necessary belt-and-braces**, not a fix. Hides the symptom; you still pay the latency. |
| Raise the LB's idle timeout to something enormous | LB config | **Worst.** Moves the wall instead of removing it, and now the LB holds state for genuinely dead connections. |

**The engineering lesson, which is the whole reason this case study is here:** two components with individually sensible defaults produced a system-level bug, because **nobody owned the invariant that connects them** — `pool_idle_timeout < network_idle_timeout`. That invariant is not written down in either component's documentation, is not checked by anything, and is invisible in every dashboard.

That's a general and repeating shape. Whenever two independent components each hold a timer about the same shared resource, **the relationship between those timers is a system invariant that someone must own.** Other instances of the identical shape:

- `terminationGracePeriodSeconds` vs (health-check drain + request drain) — concept 9's deploy scenario
- HTTP client timeout vs server-side request timeout (client gives up first → server does work nobody reads; server gives up first → client sees an unexplained reset)
- Session TTL vs token expiry
- Cache TTL vs DNS TTL — which is Day 12's material, and where the same bug reappears with different nouns
- Retry backoff ceiling vs circuit-breaker window

**When you learn a new component, find its timers and write down how they must relate to the timers around it.** That habit prevents more production incidents than any amount of monitoring.

*Verify current:* AWS ALB/CLB idle timeout defaults to 60 s and is configurable; NLB has historically used 350 s. Azure LB defaults to 4 minutes. These change — check the provider docs rather than trusting a note. And critically, **measure your actual path**: there may be a corporate firewall or a NAT gateway with a shorter timeout than the load balancer you know about.

---

## Failure modes and common mistakes: the consolidated list

Grouped by the misconception that causes them, because a fix you understand is one you'll apply next time.

### Misconceptions about what the API means

| Belief | Reality | Consequence |
|---|---|---|
| `send()` succeeding means delivered | It means "copied to the kernel's send buffer" | Lost data on close; false confidence in audit logs |
| `recv(n)` returns `n` bytes | Returns *up to* `n`, at least 1, blocking until then | Truncated messages, corrupted parsing |
| One `send()` = one `recv()` | TCP is a byte stream with no message boundaries | Works in dev, corrupts under load |
| `send()` writes everything you gave it | May write partially; returns the count | Silently truncated writes — use `sendall` |
| `ESTABLISHED` means the peer is reachable | It means *your kernel remembers* a connection | Stale-connection failures (the case study above) |
| Closing a socket flushes to the peer | Data may still be in flight or discarded on RST | Lost final writes |

### Misconceptions about performance

| Belief | Reality |
|---|---|
| Throughput is determined by bandwidth | It's `min(window/RTT, MSS/(RTT·√p), link rate)` — bandwidth is often not the binding term |
| "Slow start" is slow | It's *exponential*. It's slow only at the start |
| Bigger buffers are always better | Beyond BDP they add latency, not throughput (bufferbloat), and cost kernel memory per connection |
| Setting `SO_SNDBUF`/`SO_RCVBUF` is "tuning" | It **disables auto-tuning**, and is usually a pessimisation |
| A little packet loss doesn't matter | 1% loss caps a connection at ~18 Mbit/s on an 80 ms path, regardless of link speed |
| More parallel connections is always faster | Each starts in slow start; you may just be taking others' bandwidth |

### Misconceptions about connection lifecycle

| Belief | Reality |
|---|---|
| `TIME_WAIT` is a bug / waste | It prevents old duplicates from corrupting a new connection on the same 4-tuple |
| `tcp_tw_recycle` fixes `TIME_WAIT` | **Removed from Linux 4.12** — it broke NATed clients |
| `CLOSE_WAIT` piling up is a network problem | It's **your application** not calling `close()` after EOF. Always. |
| TCP keep-alive detects dead peers promptly | Default first probe is after **2 hours** on Linux |
| Keep-alive proves the app is healthy | It's answered by the peer's **kernel**; a wedged app answers fine |
| A port being open means the service works | The kernel completes handshakes with no application involvement |

### The five most expensive mistakes, ranked by how much production pain they cause

1. **No timeouts anywhere.** A blocking network call with no timeout is an unbounded hang. This causes more outages than any other item on this list, because one hung call holds a worker, workers run out, and the service stops serving everything.
2. **Pool idle timeout longer than the network's idle timeout.** The case study above. Presents as random unexplainable resets.
3. **No `max_lifetime` on pooled connections.** Silently defeats load balancing and DNS failover after deploys.
4. **Unbounded retries.** Jacobson's 1986 lesson relearned the hard way — you build a retry storm that keeps your own service down.
5. **Assuming `recv()` boundaries.** Data corruption that appears only at scale.

---

## Interview questions and practice

### Conceptual

1. **Why is the TCP handshake three messages and not two or four?** *(Two directions to establish; step 2's two jobs combine legitimately; the initiator must confirm it received the responder's number.)*
2. **What is `TIME_WAIT` for? Name both reasons.** *(Re-ACK a retransmitted FIN; prevent old duplicates from being accepted by a new connection reusing the 4-tuple. The second is the one people forget and the more serious one.)*
3. **Distinguish flow control from congestion control.** *(Receiver buffer vs network queues; told vs inferred; `rwnd` vs `cwnd`; both apply simultaneously and either can bind.)*
4. **A server has 50,000 established connections on port 443. How?** *(A connection is identified by the 4-tuple, not the port. The port is a rendezvous point.)*
5. **Why can't a client open more than ~28,000 connections to one destination IP:port?** *(Only the source port varies; the ephemeral range bounds it. And `TIME_WAIT` holds tuples for 60 s after close, so the *rate* limit is range ÷ 60.)*
6. **Your `recv()` returns `b""`. What happened?** *(Peer sent FIN — orderly shutdown. You must now `close()` or you'll accumulate `CLOSE_WAIT`.)*
7. **Why did HTTP/2 make head-of-line blocking worse, given that it was designed to fix it?** *(It fixed HTTP-layer HoL and concentrated all streams onto one TCP connection, so transport-layer HoL now stalls every stream instead of one-sixth of them.)*
8. **When is UDP the correct choice, and what do you take responsibility for?** *(One-shot request/response, real-time media, high-volume telemetry, multicast, or building your own transport. You owe: reliability if needed, ordering, dedup, MTU discipline, and — non-negotiably — congestion control.)*
9. **What does a latency histogram with a hard spike at exactly 40 ms suggest?** *(Nagle × delayed ACK; look for a two-write request pattern.)*
10. **Why is `SO_REUSEADDR` dangerous on Windows but routine on Unix?** *(On Unix it permits binding over `TIME_WAIT`; on Windows it permits a second live socket on the same address — port hijacking. Windows' `SO_EXCLUSIVEADDRUSE` is the protection.)*

### Diagnostic scenarios — say what you'd check and why

11. **A 10 Gbit/s link between two datacentres 200 ms apart transfers at 3 MB/s. Link utilisation 0.2%, no errors.** *(BDP: you need 10 Gbit/s × 0.2 s = 250 MB in flight; you have 3 MB/s × 0.2 s = 600 KB. Check window scaling is negotiated (`tcpdump` the SYN), check `tcp_rmem/wmem` maxima, check nobody set `SO_RCVBUF` explicitly. Then check loss with Mathis before concluding.)*
12. **A service accumulates `CLOSE_WAIT` until it hits "too many open files."** *(An application code path receives EOF and doesn't close. Find the error/break branch that skips cleanup. Not a network problem.)*
13. **After every deploy, some pods get almost no traffic for 20 minutes.** *(Pooled connections pinned to old pods; missing `max_lifetime`. Set it to a few minutes.)*
14. **Requests to a backend intermittently fail with connection reset, worst at low traffic.** *(Stale pooled connection vs middlebox idle timeout. Set `max_idle_time` below the shortest idle timeout in the path; consider tuned TCP keep-alive.)*
15. **`connect()` hangs for 2 minutes then fails, to a host you can `ping`.** *(SYN retransmissions with nothing coming back — a firewall dropping SYNs silently, or a SYN-queue overflow. A closed port would RST immediately, so "hang" specifically implicates a drop.)*
16. **Latency is fine when the link is idle and terrible when a backup runs, but utilisation looks healthy.** *(Bufferbloat. Run `ping` alongside the transfer; if RTT jumps 20 ms → 500 ms, the queue is the problem. Fix with FQ-CoDel or move the bulk transfer to a rate limit.)*
17. **A UDP service works for small requests and silently fails for large ones.** *(Datagrams exceeding path MTU getting fragmented, and fragments being dropped by a middlebox. Cap datagrams at ~1200 bytes.)*
18. **`ss -tanl` shows `Recv-Q 512 / Send-Q 512` on a LISTEN row.** *(Accept queue is full — the application isn't calling `accept()` fast enough. Clients will see hung requests, not connection failures.)*

### Design questions

19. Design connection management for a service calling 8 backends at 3,000 req/s. *(Pooling with the three timeouts; do the `TIME_WAIT` arithmetic; justify each timeout value against a specific failure.)*
20. Design a health check that catches a deadlocked application but doesn't cascade when a shared database hiccups. *(Two-tier livez/readyz; keep shared dependencies out of the check the LB acts on.)*
21. You must move 500 TB between continents over a path with 250 ms RTT and 0.5% loss. What do you do? *(Mathis says one connection caps around 8 Mbit/s → months. Options: parallel streams, split-TCP relays, BBR, reduce loss, or — genuinely — ship physical disks. Compute the numbers and compare honestly; AWS Snowball exists because this arithmetic sometimes wins.)*

---

# Topic-wide wrap-up

## Glossary

**4-tuple** — the combination (source IP, source port, destination IP, destination port) that uniquely identifies a TCP connection on a host; the port alone does not.

**ACK (acknowledgement)** — a TCP segment field reporting the next byte the receiver expects; cumulative, so it confirms everything before that point.

**AIMD (Additive Increase, Multiplicative Decrease)** — TCP's congestion-window policy: grow by one segment per round trip, halve on loss. Provably converges to a fair allocation among competing flows.

**AQM (Active Queue Management)** — router techniques such as CoDel and FQ-CoDel that drop or mark packets before a queue is full, giving senders an earlier congestion signal and preventing bufferbloat.

**Backlog** — the `listen()` argument bounding the accept queue: how many completed connections may wait for `accept()`.

**BBR** — a congestion-control algorithm that estimates bottleneck bandwidth and minimum RTT and paces to their product, instead of treating loss as the congestion signal.

**BDP (bandwidth-delay product)** — `bandwidth × RTT`; the number of bytes that must be in flight to keep a path fully utilised. Sets the minimum useful window and buffer size.

**Bufferbloat** — excessive latency caused by oversized router buffers that TCP fills before receiving its loss signal.

**CLOSE_WAIT** — the state of a connection whose peer has sent FIN and whose local application has not yet called `close()`. Accumulation is an application bug.

**Congestion collapse** — a stable state in which a network carries traffic at capacity while delivering almost no useful data, because most traffic is retransmissions. Observed on NSFNET in 1986.

**Congestion control** — the sender's self-imposed limit on in-flight data, inferred from loss and delay, protecting the network's queues. Distinct from flow control.

**Congestion window (`cwnd`)** — the sender's own estimate of how much unacknowledged data the network can carry.

**CUBIC** — the default Linux congestion-control algorithm; grows the window as a cubic function of time since the last loss, making it RTT-independent and usable on high-BDP paths.

**Delayed ACK** — a receiver optimisation that waits up to ~40 ms (Linux) before acknowledging, hoping to piggyback the ACK on outgoing data or cover several segments at once.

**Duplicate ACK** — a repeated ACK for the same sequence number, signalling that data arrived out of order or a segment is missing; three of them trigger fast retransmit.

**Ephemeral port** — a high-numbered port the kernel assigns as the source port for outbound connections; the range's size bounds simultaneous connections to any single destination.

**Fast retransmit** — retransmitting a segment immediately upon receiving three duplicate ACKs, rather than waiting for the retransmission timeout.

**FIN** — the TCP flag meaning "I have no more data to send"; each direction is closed independently.

**Flow control** — the receiver-imposed limit on in-flight data, communicated as the window field, protecting the receiver's buffer. Distinct from congestion control.

**Half-close** — the state after one side has sent FIN and the other has not; that side can still send. Reached via `shutdown(SHUT_WR)`.

**Half-open connection** — a connection that one or both endpoints believe is established but which has no working path, typically because a middlebox discarded its state.

**Head-of-line blocking** — data that has arrived being undeliverable because earlier data is missing; TCP's in-order guarantee applies to the whole connection, so one loss stalls everything multiplexed onto it.

**ISN (initial sequence number)** — the starting sequence number each side chooses in the handshake; randomised cryptographically to prevent blind injection attacks.

**Keep-alive (TCP)** — kernel probes on an idle connection to detect a dead peer. Default first probe after 2 hours on Linux; must be tuned to be useful.

**Keep-alive (HTTP)** — reusing a TCP connection for multiple HTTP requests; the default in HTTP/1.1. Unrelated to TCP keep-alive despite the name.

**Mathis equation** — `throughput ≈ MSS / (RTT × √p)`; the approximate ceiling a single loss-based TCP flow can achieve at loss rate `p`.

**MSL (maximum segment lifetime)** — the assumed longest time a packet can survive in the network; `TIME_WAIT` lasts 2×MSL so that stale packets expire before a 4-tuple is reused.

**MSS (maximum segment size)** — the largest TCP payload a host will accept, announced in the SYN and not renegotiable; typically 1460 bytes on Ethernet.

**Nagle's algorithm** — a sender optimisation that withholds a small segment while earlier small data is unacknowledged, coalescing small writes. Disabled with `TCP_NODELAY`.

**Persist timer** — the timer that makes a sender probe a zero-window receiver, preventing deadlock if a window update is lost.

**Port** — a 16-bit number identifying which application on a host a segment belongs to; a rendezvous point, not a connection identity.

**QUIC** — a reliable, multiplexed, encrypted transport implemented in userspace over UDP; provides per-stream ordering, a 1-RTT (or 0-RTT) handshake, and connection identity independent of the 4-tuple.

**RST (reset)** — a TCP flag that aborts a connection immediately, discarding buffered data; sent for connections that don't exist, refused ports, or explicit aborts. Not acknowledged.

**RTO (retransmission timeout)** — the timer after which unacknowledged data is resent; computed from smoothed RTT and RTT variance, and doubled on each successive failure.

**SACK (selective acknowledgement)** — a TCP option letting the receiver report which non-contiguous byte ranges it holds, so the sender retransmits only what's actually missing.

**Silly window syndrome** — a pathology in which tiny window advertisements cause tiny segments and catastrophic header overhead; prevented by Clark's receiver-side rule and Nagle's sender-side rule.

**Slow start** — the initial congestion-control phase in which `cwnd` doubles every round trip; exponential despite the name.

**`SO_REUSEADDR`** — a socket option allowing `bind()` over a port with connections in `TIME_WAIT` (Unix); on Windows it instead permits a second live socket on the same address.

**`SO_REUSEPORT`** — a socket option (Linux/BSD, not Windows) allowing multiple live sockets to bind one port, with the kernel distributing incoming connections across them.

**Socket** — a file descriptor plus kernel state: a 4-tuple, a send buffer, a receive buffer, a connection state machine, sequence numbers, window and congestion variables, and timers.

**SYN cookie** — a defence against SYN floods in which the server encodes connection state into the ISN it sends, holding no memory until the client's ACK proves reachability.

**SYN flood** — a denial-of-service attack exhausting the SYN queue with half-open connections from spoofed sources.

**SYN queue** — the kernel queue of half-open connections awaiting the final handshake ACK.

**`TIME_WAIT`** — the state entered by the side that closes first, lasting 2×MSL, so that a retransmitted FIN can be answered and stale packets cannot poison a reused 4-tuple.

**Two-army problem** — the impossibility of two parties over a lossy channel both becoming certain they have agreed; the reason TCP's close is engineered around a timer rather than solved.

**UDP** — a transport providing only ports, a length, and a checksum: message boundaries preserved, no handshake, ordering, retransmission, flow control, or congestion control.

**Window (receive window, `rwnd`)** — the receiver's advertised free buffer space, in the TCP header's 16-bit window field, scaled by the negotiated window-scale factor.

**Window scale** — a TCP option negotiated in the SYN that left-shifts every subsequent window advertisement, allowing windows beyond 65535 bytes; required for throughput on high-BDP paths.

**Zero window** — an advertisement of no free receive buffer, halting transmission until a window update or a persist-timer probe frees it.

---

## Cheat sheet

**The two independent limits**
```
bytes in flight ≤ min(rwnd, cwnd)
                       │      └── congestion control: sender's guess, from loss/delay
                       └───────── flow control: receiver's advertisement
```

**The three throughput ceilings — the binding one wins**
```
window / RTT                  ← flow control / buffer sizing
MSS / (RTT × √p)              ← Mathis: loss-based congestion control
link rate                     ← actual bandwidth
```

**BDP quick reference** (bytes needed in flight)
| RTT | 10 Mbit/s | 100 Mbit/s | 1 Gbit/s |
|---|---|---|---|
| 1 ms | 1.25 KB | 12.5 KB | 125 KB |
| 20 ms | 25 KB | 250 KB | 2.5 MB |
| 80 ms | 100 KB | 1 MB | 10 MB |
| 250 ms | 313 KB | 3.1 MB | 31 MB |

**Connection states, and who is at fault**
| State | Meaning | Action |
|---|---|---|
| `SYN_SENT` (many) | Connects not completing | Firewall / dead peer / wrong port |
| `SYN_RECV` (many) | Handshakes not completing | SYN flood, or clients vanishing |
| `ESTABLISHED` | Two kernels agree — proves nothing about the app | — |
| `FIN_WAIT_2` (many) | Peer's app hasn't closed | Peer's bug |
| `CLOSE_WAIT` (many) | **Your** app hasn't closed | **Your bug** — FD leak |
| `TIME_WAIT` (many) | You closed first; normal | Pool connections; make server close first |
| `LISTEN` with `Recv-Q > 0` | Accept queue backing up | App not accepting fast enough |

**Socket options worth knowing**
| Option | Effect | Portability |
|---|---|---|
| `TCP_NODELAY` | Disable Nagle — for interactive request/response | Everywhere |
| `SO_KEEPALIVE` | Enable kernel keep-alive probes | Everywhere |
| `TCP_KEEPIDLE/INTVL/CNT` | Tune keep-alive timing | Linux (macOS differs; Windows uses an ioctl) |
| `SO_REUSEADDR` | Bind over `TIME_WAIT` | Unix; **different semantics on Windows** |
| `SO_REUSEPORT` | Multiple listeners on one port | Linux 3.9+/BSD; **not Windows** |
| `SO_SNDBUF`/`SO_RCVBUF` | Set buffer size — **disables auto-tuning** | Everywhere; avoid unless measured |
| `SO_LINGER(0)` | Close with RST, discarding buffered data | Everywhere; rarely what you want |

**The three pool timeouts**
```
max_connections   bounds backend load and memory
max_idle_time     MUST be < the shortest idle timeout in the network path
max_lifetime      forces rebalancing after deploys / DNS changes
acquire_timeout   converts an invisible queue into a visible error
```

**Diagnostics**
| Task | Windows | Linux |
|---|---|---|
| Connections by state | `Get-NetTCPConnection \| Group-Object State` | `ss -tan \| awk '{print $1}' \| sort \| uniq -c` |
| Per-connection TCP internals | (not available) | `ss -tin` |
| Retransmit counters | `netstat -s -p tcp` | `nstat -az \| grep -i retrans` |
| Packet capture | Wireshark | `tcpdump -i eth0 -n 'tcp port 443'` |
| Global TCP settings | `Get-NetTCPSetting` | `sysctl net.ipv4.tcp_*` |
| Ephemeral port range | `netsh int ipv4 show dynamicport tcp` | `cat /proc/sys/net/ipv4/ip_local_port_range` |
| Impair a path for testing | Clumsy / NEWT | `tc qdisc add dev lo root netem delay 50ms loss 1%` |

**TCP vs UDP decision table**
| Need | Choose |
|---|---|
| Every byte matters, order matters | **TCP** |
| One small request, one small reply, latency critical | **UDP** (DNS) |
| Real-time media where stale data is worthless | **UDP** |
| High-volume telemetry, some loss acceptable | **UDP** batched under MTU |
| Multiplexed streams that shouldn't share loss fate | **QUIC** |
| Must survive an IP address change | **QUIC** |
| You'd have to build reliability + ordering + congestion control | **TCP** (or QUIC) — not hand-rolled UDP |

---

## Build this

Four tasks, each with a runnable definition of done. Task 1 is the study plan's core build; do it first and do it properly.

### Task 1 — a raw TCP client and server, observed on the wire

**Build:**
1. A TCP server with Python `socket` that accepts one connection, receives a length-prefixed message, and echoes it back.
2. A TCP client that connects, sends a message with a 4-byte big-endian length prefix, reads the reply with a `recv_exactly` loop, and closes.
3. Capture the whole exchange (Wireshark on Windows, `tcpdump -i lo -n -S 'tcp port 9200'` on Linux).

**Definition of done — you can point at each of these in your own capture:**
- [ ] The three handshake packets, and you can state the two ISNs and verify each `ack` is the peer's ISN + 1.
- [ ] The options in each SYN: `mss`, `sackOK`, `wscale`. State the effective window on packet 3 (advertised × 2^wscale).
- [ ] Your data segment, with a payload length matching your message + 4 prefix bytes.
- [ ] The four teardown packets (FIN, ACK, FIN, ACK), and you can say which side entered `TIME_WAIT` and why.
- [ ] `ss -tan state time-wait` (or `Get-NetTCPConnection -State TimeWait`) shows your connection, and it disappears roughly 60 s later.

### Task 2 — break it deliberately, three ways

For each, predict what the client will see *before* you run it, then check.

- [ ] **Kill the server process mid-transfer** while streaming a large payload. Expect: `ConnectionResetError` within milliseconds. Find the RST in the capture.
- [ ] **Black-hole the port** (firewall DROP rule) mid-transfer. Expect: no RST, retransmissions with doubling intervals, and no error for many minutes. Count the retransmissions in the capture and note the intervals — you are watching exponential backoff.
- [ ] **Have the server stop calling `recv()`** while the client keeps sending. Expect: the client's `send()` blocks after a few MB. Find the `win=0` advertisement in the capture, then find the zero-window probes.

**Definition of done:** you can explain, in one sentence each, why the first failed in milliseconds and the second took minutes.

### Task 3 — one UDP datagram, and a catalogue of what didn't happen

- [ ] Send a single UDP datagram to a listening `SOCK_DGRAM` socket and receive it.
- [ ] Capture it. **Count the packets: there should be exactly one.** Compare to Task 1's total.
- [ ] Write down, from your own capture, every TCP packet in Task 1 that has no UDP counterpart, and say what each one was for.
- [ ] Send a 3000-byte datagram and observe IP fragmentation in the capture. Then impair the path with 20% loss and observe that the large datagram fails far more often than a 500-byte one. Explain the multiplication.

### Task 4 — measure a real ceiling

- [ ] Impair loopback with `tc netem delay 50ms` (or use a real remote host).
- [ ] Run `window_ceiling.py` from concept 6 and confirm throughput tracks `window / RTT` within a factor of two.
- [ ] Add `loss 1%` and predict the new throughput with the Mathis equation *before* measuring. Compare. Being within an order of magnitude is a pass — the point is that the formula predicts the right regime.

**Stretch:** rerun with `sysctl -w net.ipv4.tcp_congestion_control=bbr` and compare against cubic on the lossy path. Explain the difference in terms of what each algorithm treats as a congestion signal.

---

## Active recall and self-test

Answer from memory. Write the answer down before checking.

1. Name IP's four failure modes and one physical cause of each.
2. Why must the handshake exchange sequence numbers *before* any data? What breaks if it doesn't?
3. Draw the sender's three pointers (`snd_una`, `snd_nxt`) and label "in flight."
4. Why three duplicate ACKs and not one?
5. Write the RTO formula and explain why it includes variance.
6. Compute the BDP for 1 Gbit/s at 100 ms RTT. Now state the throughput a 64 KB window would achieve on that path.
7. Write the Mathis equation. What throughput does 1% loss allow at 80 ms RTT with MSS 1460?
8. Give both reasons `TIME_WAIT` exists. Which one causes silent data corruption if you skip it?
9. A pile of `CLOSE_WAIT` connections: whose bug, and what's the fix?
10. Explain the Nagle × delayed-ACK interaction and give two fixes, saying which is better and why.
11. Why did HTTP/2 make transport-layer head-of-line blocking worse than HTTP/1.1?
12. Give three reasons DNS uses UDP.
13. What's the largest UDP payload you should send on the open internet, and why does exceeding it multiply your loss rate?
14. What are the three pool timeouts, and which specific failure does each prevent?
15. Why is "the port is open" a bad health check? Which concept explains it?

### 60-second teach-back

Explain, out loud, to someone with no networking background:

> **"IP delivers packets on a best-effort basis, which means four specific things can go wrong: packets can be lost, duplicated, reordered, or corrupted. TCP's whole job is to hide all four. It does it by numbering every byte, which requires both sides to first agree on where the numbers start — that's the three-way handshake. Then the receiver constantly reports the highest byte it has received contiguously, and the sender resends anything that isn't acknowledged. On top of that, TCP has two separate speed limits: the receiver says how much buffer space it has, and the sender guesses how much the network can take by watching for loss. All of that reliability has one cost: because TCP promises to deliver bytes in order, one missing packet holds back everything that arrived behind it. When people started multiplexing dozens of web requests onto one TCP connection, that single cost became the dominant problem — which is why HTTP/3 abandoned TCP for QUIC, which rebuilds all of this in userspace over UDP, but keeps ordering per-stream instead of per-connection. UDP, meanwhile, is what's left when you delete all of TCP's machinery: it just adds port numbers and a checksum to IP. That sounds useless until you notice it's exactly right for a single small question-and-answer like DNS, or for live audio where a late packet is worthless anyway."**

If you can deliver that without notes, and then answer "so what breaks when a load balancer times out an idle connection?" — you have this topic.

---

## Spaced-repetition flashcards

| Q | A |
|---|---|
| What identifies a TCP connection? | The 4-tuple: (src IP, src port, dst IP, dst port) — not the port |
| Why 3 packets in the handshake? | Two directions to establish; each side must confirm its number was received; step 2's two jobs combine |
| Does `send()` returning mean delivered? | No — it means copied to the kernel send buffer |
| What does `recv()` returning `b""` mean? | Peer sent FIN; you must `close()` |
| Formula: throughput vs window | `throughput ≈ window / RTT` |
| Formula: throughput vs loss | `throughput ≈ MSS / (RTT × √p)` (Mathis) |
| Is slow start slow? | No — it doubles `cwnd` every RTT (exponential) |
| Linux initial congestion window | 10 segments (~14 KB), RFC 6928 |
| How many dup ACKs trigger fast retransmit? | 3 — fewer would misfire on ordinary reordering |
| Two reasons for `TIME_WAIT` | Re-ACK a retransmitted FIN; stop old duplicates poisoning a reused 4-tuple |
| `TIME_WAIT` duration | 2×MSL — 60 s on Linux, historically 240 s on Windows |
| Piles of `CLOSE_WAIT` mean | Your app isn't calling `close()` after EOF — FD leak |
| Linux default TCP keep-alive idle | 7200 s (2 hours) — useless untuned |
| Does keep-alive prove the app is healthy? | No — the peer's *kernel* answers it |
| Cause of a hard 40 ms latency spike | Nagle × delayed ACK, from a two-write request |
| Max window without scaling | 65535 bytes |
| Safe max UDP payload on the internet | ~1200 bytes (avoid IP fragmentation) |
| Why does fragmentation multiply loss? | Losing any fragment destroys the whole datagram |
| Why does DNS use UDP? | One-shot exchange; a handshake would triple latency; no per-client server state |
| What does QUIC fix that TCP can't? | Per-stream ordering, 1/0-RTT handshake, connection ID survives IP change, encrypted transport metadata resists ossification |
| Congestion control vs flow control | Network queues (inferred) vs receiver buffer (advertised) |
| What causes bufferbloat? | Oversized router buffers delaying TCP's loss signal |
| Which Linux knob was removed for breaking NAT? | `tcp_tw_recycle` (removed in 4.12) |
| Pool timeout that prevents stale-connection resets | `max_idle_time` < the network path's shortest idle timeout |
| Pool timeout that fixes post-deploy imbalance | `max_lifetime` |

---

## Primary sources

Verify version-specific claims against these rather than against this note.

**Core specifications**
- **RFC 9293** — Transport Control Protocol (2022; consolidates and obsoletes RFC 793 and its many updates). The current authoritative TCP spec.
- **RFC 768** — UDP. Three pages; worth reading in full precisely because it's so short.
- **RFC 1122** — Requirements for Internet Hosts. Where delayed ACK, keep-alive, and much host behaviour is specified.

**Reliability and performance**
- **RFC 5681** — TCP Congestion Control (slow start, congestion avoidance, fast retransmit/recovery).
- **RFC 6298** — Computing TCP's Retransmission Timer (the SRTT/RTTVAR formulas).
- **RFC 2018** — TCP Selective Acknowledgement Options. **RFC 6675** — SACK-based loss recovery.
- **RFC 7323** — TCP Extensions for High Performance (window scaling, timestamps, PAWS). Obsoletes RFC 1323.
- **RFC 6928** — Increasing TCP's Initial Window (the 3 → 10 segment change, with Google's measurements).
- **RFC 8312** — CUBIC.
- **RFC 896** — Congestion Control in IP/TCP Internetworks (Nagle's original).

**Security**
- **RFC 4987** — TCP SYN Flooding Attacks and Common Mitigations.
- **RFC 6528** — Defending against Sequence Number Attacks (ISN randomisation).
- **RFC 2827 / BCP 38** — Network Ingress Filtering.
- **US-CERT TA14-013A** — UDP-Based Amplification Attacks (the amplification-factor table).

**QUIC / HTTP/3**
- **RFC 9000** (transport), **RFC 9001** (TLS integration), **RFC 9002** (loss detection & congestion control), **RFC 9114** (HTTP/3).

**Papers**
- Van Jacobson, *Congestion Avoidance and Control*, SIGCOMM 1988 — the 1986 collapse and its fix.
- Chiu & Jain, *Analysis of the Increase and Decrease Algorithms for Congestion Avoidance in Computer Networks*, 1989 — why AIMD converges to fairness.
- Mathis, Semke, Mahdavi, Ott, *The Macroscopic Behavior of the TCP Congestion Avoidance Algorithm*, CCR 1997 — the loss/throughput equation.
- Langley et al., *The QUIC Transport Protocol: Design and Internet-Scale Deployment*, SIGCOMM 2017.
- Cardwell et al., *BBR: Congestion-Based Congestion Control*, ACM Queue 2016.

**Books**
- Stevens, *TCP/IP Illustrated, Volume 1* (2nd ed., Fall & Stevens) — the standard reference; the packet-trace style of this note is borrowed from it.
- Grigorik, *High Performance Browser Networking* — free online; the best treatment of how TCP behaviour shapes web performance.

**Operational references**
- `fasterdata.es.net` — ESnet's data-transfer tuning knowledge base; the practical authority on BDP, buffers, and the "wizard gap."
- GitHub Engineering, *February 28th DDoS Incident Report* (2018) — the memcached amplification postmortem.
- Nginx blog, *Socket Sharding in NGINX Release 1.9.1* — `SO_REUSEPORT` in practice.
- Vincent Bernat, *Coping with the TCP TIME-WAIT state on busy Linux servers* — the definitive practical treatment, including why `tcp_tw_recycle` is gone.

**Fast-drifting facts — verify before relying on any of these**
- OS default buffer sizes, ephemeral port ranges, keep-alive timings, and default congestion-control algorithm (all changed within the last few years on both Linux and Windows).
- Cloud load-balancer idle timeouts (AWS ALB 60 s, NLB 350 s, Azure 4 min — all subject to change and some now configurable).
- HTTP/3 adoption share and QUIC CPU overhead figures (both moving fast).

---

## Layered explanations

**10 seconds.** IP loses, duplicates, reorders, and corrupts packets. TCP numbers every byte and acknowledges them so it can fix all four, at the cost of one missing packet stalling everything behind it. UDP does none of that, which makes it right for anything where a late packet is worthless.

**1 minute.** TCP turns unreliable packet delivery into a reliable ordered byte stream. It opens with a three-way handshake whose real job is agreeing on where each side's byte numbering starts. Then the receiver continuously reports the highest contiguously-received byte, and the sender resends anything unacknowledged — quickly on three duplicate ACKs, slowly on a timeout. Two separate limits bound how fast it sends: the receiver's advertised window (protecting its buffer) and the sender's congestion window (protecting the network's queues, inferred from loss). The whole thing lives as state in two kernels' memory, which is why a connection can be dead in the middle while both endpoints still believe in it. UDP is IP plus ports plus a checksum: no handshake, no ordering, no retransmission, no congestion control — and message boundaries preserved, which TCP throws away.

**5 minutes.** Start with IP's four failure modes — loss (mostly full router queues), duplication, reordering (multiple paths), corruption (IP's checksum covers only the header). Reassembly requires numbering every byte, which requires agreeing on a starting number, which is the handshake. The receiver's cumulative ACK reports the next byte it wants; that single field gives robustness (lost ACKs are harmless) and a limitation (it can't describe holes), which SACK later fixed. Loss is detected two ways: three duplicate ACKs (fast, one RTT) or a timeout (slow, ≥200 ms, computed from smoothed RTT *and* RTT variance). Speed is bounded by `min(rwnd, cwnd)`: `rwnd` is told to you by the receiver, and `cwnd` is guessed from loss because routers can't talk to you. Congestion control exists because in 1986 the internet suffered congestion collapse — throughput fell 1000× as retransmissions crowded out real traffic — and the fix was making every endpoint back off. That gives you two throughput formulas worth memorising: `window/RTT` and Mathis's `MSS/(RTT·√p)`, the second of which explains why 1% loss caps you at ~18 Mbit/s regardless of link speed. Closing is a four-way exchange because each direction closes independently, and the closer waits in `TIME_WAIT` for two reasons: to re-ACK a retransmitted FIN, and to stop stale packets poisoning a new connection on the same 4-tuple. Connections are expensive (a handshake RTT, a cold congestion window, a held 4-tuple), so you pool them — and then discover that a pooled connection can silently die when a middlebox forgets it, which is why the pool's idle timeout must be shorter than the network's. Finally, the cost of in-order delivery: one loss stalls everything on the connection, which became critical when HTTP/2 multiplexed all requests onto one connection, which is why HTTP/3 moved to QUIC — the same reliability machinery, rebuilt in userspace over UDP, with ordering enforced per stream instead of per connection.

**Expert summary.** TCP is a control loop that synthesises a reliable ordered byte stream over a best-effort datagram service, using cumulative acknowledgement as its sole feedback channel. Its design embodies the end-to-end principle: reliability state lives only at the endpoints, which is what allows the network to be stateless and survivable, at the cost of losing local loss recovery (recovered pragmatically at the edges by link-layer ARQ). Its two rate limiters are architecturally distinct — flow control is an explicit credit scheme with a known correspondent, congestion control an inference problem against an unobservable shared resource — and the latter's use of loss as signal makes it both self-correcting and mis-triggered by non-congestive loss, motivating delay- and model-based alternatives (Vegas, BBR). AIMD's convergence to a fair allocation (Chiu–Jain) means fairness is emergent from arithmetic rather than enforced, which is precisely why non-conforming UDP traffic is a tragedy-of-the-commons problem rather than a policy violation. TCP's principal structural limitation is that its ordering guarantee is applied at connection rather than stream granularity, converting independent stream failures into shared fate — a cost that was negligible when connections carried one object and became dominant under HTTP/2's multiplexing, prompting QUIC to re-implement the entire stack in userspace over UDP with per-stream ordering, integrated crypto, connection identity decoupled from the 4-tuple, and encrypted transport metadata as an explicit anti-ossification measure. The recurring meta-lesson across the topic is that guarantees have granularity, and a guarantee applied more coarsely than the application's actual independence structure converts isolated failures into correlated ones — a pattern that recurs in message queues, replication streams, and agent orchestration, always with the same fix: partition so failures don't share fate.
