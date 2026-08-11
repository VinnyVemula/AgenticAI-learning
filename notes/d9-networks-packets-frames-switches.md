# Day 9 — What a Network Is: Packets, Frames, and Switches

> **Framing.** Days 1–8 gave you one machine: a CPU, memory, an OS, processes, file descriptors. Today a second machine appears, and everything you know stops being enough — because between two machines there is a *wire*, and a wire has no operating system to keep things orderly. This note builds the bottom two floors of the internet from nothing: why the network sends *packets* instead of holding *circuits*, how a chunk of your data gets wrapped in envelope after envelope on its way out of your NIC, what a MAC address is and why it isn't an IP address, and what the difference between a switch and a router actually is (it is the single most load-bearing boundary in all of networking).
>
> This is a **foundational systems topic, and mostly it stays that way** — I am not going to pretend that Ethernet frame padding has an agentic angle, because it doesn't. There is exactly one place where an LLM in the loop genuinely changes the engineering, and it's a real one: when you let a model write code and then *run* that code, the network is the layer where "sandboxed" either holds or quietly doesn't (Day 8 gave you process, memory, and FD isolation; network isolation is the piece that was missing). That shows up in one system-design scenario near the end, where it belongs. Everywhere else, this is plumbing — and plumbing is what the rest of Phase 1 is built on.

---

## How to read this note

Concepts, in the order they build, with the depth tier each is developed to:

| # | Concept | Tier |
|---|---|---|
| 1 | What a network is: the shared-medium problem | **[CORE]** |
| 2 | Circuit switching vs packet switching | **[CORE]** |
| 3 | Layering, and why it exists | **[CORE]** |
| 4 | Encapsulation / decapsulation | **[CORE]** |
| 5 | NICs, MAC addresses, the Ethernet frame | **[CORE]** |
| 6 | Hubs → switches; collision vs broadcast domains | **[WORKING]** |
| 7 | Switches vs routers: the layer-2/layer-3 boundary | **[CORE]** |
| 8 | ARP | **[WORKING]** |
| 9 | MTU and the number 1500 | **[WORKING]** |
| 10 | VLANs (802.1Q) | **[AWARE]** |
| 11 | Spanning tree and broadcast storms | **[WORKING]** |
| 12 | Cloud segmentation (VPCs, subnets, security groups) | **[WORKING]** |

**Two black boxes I am deliberately leaving closed today, on purpose, and where they get opened:**

- **IP addressing, subnet math, routing tables, TTL, NAT** → **Day 10**. Today you will see the IP header's *shape* and learn what each field is *for*, because you cannot understand encapsulation or the switch/router boundary without it. You will not learn to carve `10.0.0.0/16` into subnets by hand — that's tomorrow.
- **TCP's mechanics** — the handshake, sequence numbers, retransmission, windows, congestion control → **Day 11**. Today TCP appears as *a header with ports and sequence numbers sitting inside an IP packet*, which is exactly enough to see encapsulation work. Why those numbers matter is Day 11's whole job.

If at any point today you think "but *how* does the router know where to send it?" — good. That question is Day 10, and I'll flag it again when we get there.

---

## 1. What a network actually is — the shared-medium problem

**Depth: [CORE]**

### Intuition: the moment one machine becomes two

On Day 1 you learned the latency pyramid: a register access is ~1 ns, RAM ~100 ns, an SSD ~100 µs, and a network call 1–100 ms. That last row is the one we're now going to live inside. But before we talk about speed, we have to talk about something more basic, which is that **inside one computer, everything is coordinated by a single authority, and between two computers, nothing is.**

Think about what happens when your Python program writes to a variable. The CPU issues a store, the cache and memory controller handle it, and no other program can corrupt it because the OS installed a page table that makes your memory yours (Day 4). When two threads *do* share memory, you saw on Day 3 that you need locks, because sharing without coordination produces races. The OS is the referee. It can be the referee because it sits above both parties and controls the hardware they both use.

Now put a cable between two computers. Who is the referee? Neither OS can see the other's memory. Neither can pause the other. Neither can even *tell* whether the other is switched on. And if you put ten computers on that cable, they can all try to transmit at the same instant, and the electrical signals will physically collide and turn into garbage — and there is no kernel anywhere that can prevent it, because there is no kernel that owns the cable.

So a network is not "a wire between computers." **A network is a shared physical medium plus a set of agreements — protocols — that let mutually distrustful, independently-scheduled machines take turns on it and make sense of what comes back.** Every single thing in this note and the next nine days exists to fill in one of those agreements: how to name a machine, how to know whose turn it is, how to detect corruption, how to find a path, how to notice loss.

### Analogy: a crowded room with no chairperson

Ten people in a room with one microphone. If two people speak at once, nobody hears either of them. There is no chairperson, so the group must agree on rules in advance: *listen before you speak; if you hear someone else start at the same moment as you, both stop and wait a random amount of time before retrying; say who you're talking to at the start of every sentence so everyone else can ignore it.* Those three rules are, almost literally, classic Ethernet (carrier sense, collision detection with random backoff, and destination addressing).

**Where the analogy breaks — and this matters:** in a room, everyone hears everything, so "was I heard?" is answerable by watching faces. On a wire that runs 100 metres through a wall, a sender has *no* feedback channel at all unless one is explicitly built. Worse, modern switched Ethernet is *not* a shared room anymore — each machine has a private, full-duplex link to a switch, so physical collisions basically stopped happening around 2000. The room analogy explains why the *protocol* looks the way it does historically; it badly misdescribes the hardware you actually own today. Keep the analogy for the "why," throw it away for the "what."

### Worked example: two ways this can fail, with numbers

Suppose machine A wants to send 1 MB to machine B over a 1 Gbit/s link.

At 1 Gbit/s = 125 MB/s, the pure *transmission* time for 1 MB is 1 / 125 = **8 ms**. But the link is 200 km long, and signals in fibre travel at roughly 2 × 10⁸ m/s, so **propagation** takes 200,000 / 2×10⁸ = **1 ms** each way. Those are two completely different costs, and confusing them is the single most common beginner error in networking:

- **Bandwidth** (125 MB/s) determines how long it takes to *push the bits out*. Doubling bandwidth halves this.
- **Latency** (1 ms one way) determines how long the first bit takes to *arrive*. Doubling bandwidth changes this **not at all** — it is set by distance and the speed of light.

So if A sends the 1 MB as one continuous blast, B has the whole megabyte after roughly 8 + 1 = 9 ms. But if the protocol requires A to send 1 KB and wait for an acknowledgement before sending the next 1 KB, then each 1 KB costs 0.008 ms of transmission plus 2 ms of round-trip waiting — and 1024 chunks × ~2 ms = **over 2 seconds** for the same megabyte. Same wire, same bandwidth, 230× slower, purely because of how the protocol interleaves sending and waiting.

That number is why TCP has a *window* (Day 11), why HTTP/1.1's one-request-at-a-time-per-connection hurt (Day 14), and why the TLS handshake's extra round trip is worth designing away (Day 14). Hold onto it: **on a network, round trips are the expensive thing, not bytes.**

---

## 2. Circuit switching vs packet switching

**Depth: [CORE]**

### Intuition: the telephone network's beautiful, doomed idea

By the 1950s humanity had already built a colossal, working, global communications network — the telephone system — and it worked on a principle called **circuit switching**. When you dialled a number, a chain of electromechanical switches physically closed a path from your handset, through your local exchange, through trunk exchanges, to the other handset. For the duration of the call, that path — that *circuit* — was **yours**. Reserved. Nobody else could use those wires and those switch contacts even if you sat in silence for ten minutes.

This has genuinely lovely properties. Once the circuit is up, the delay is constant, the bandwidth is guaranteed, bits arrive in the order you sent them, and none go missing. You never need a sequence number, an acknowledgement, or a retransmission, because the path is a dedicated physical thing. Voice is a continuous stream and circuit switching is a near-perfect match for it.

Then people started connecting *computers*, and two problems appeared that circuit switching could not survive.

**Problem one: computer traffic is bursty, so reservation wastes almost everything.** A person at a terminal types a line, waits, reads the response, thinks, types another. The actual data is a trickle punctuated by long silence. If you reserve a circuit for that, you are paying full price for a resource you use maybe 1% of the time.

**Problem two: a reserved path is a single point of failure by construction.** The whole value of the circuit is that it's *this specific* chain of switches. Destroy one link in the middle and the call doesn't degrade — it drops, completely, and has to be rebuilt from scratch.

**Packet switching** inverts both. Instead of reserving a path, you chop your data into small, independently-addressed chunks — **packets** — each carrying its own destination address and enough information to be reassembled. You hand a packet to the network and the network forwards it hop by hop, deciding at *each* hop where to send it next. No reservation exists. The wire between two switches is used by whoever has a packet ready right now, interleaved microsecond by microsecond, which is called **statistical multiplexing**. And because each packet is independently routed, if a link dies mid-transfer, the *next* packet can simply take a different path.

### Analogy: the motorway lane vs the postal system

Circuit switching is being given your own private lane on the motorway for your whole journey: guaranteed speed, no traffic, and completely wasted while your car is parked at a service station. Packet switching is the postal system: you write the address on every envelope, drop them in the box, and each sorting office independently decides which van each envelope goes on next. The envelopes may take different routes, may arrive out of order, and one may be lost — but the vans run full, and if a sorting office burns down, tomorrow's envelopes go via a different one.

**Where the analogy breaks:** the post office never *deliberately destroys* mail when it's busy, but a packet-switched network absolutely does. When a router's output queue for a link is full and another packet arrives, the router **drops it on the floor** — silently, with no notification to anyone. This isn't a bug; it's the design. Congestion control (Day 11) works precisely by *interpreting* those drops as a signal to slow down. Also, unlike the post office, there is no tracking, no receipt, and no undeliverable-return service at this layer. If you want any of that, you build it yourself on top — which is exactly what TCP is.

### Worked example: the utilization arithmetic that decided the argument

Ten users share a link. Each user, when active, needs 1 Mbit/s. Each user is active only 10% of the time (bursty terminal traffic).

**Circuit-switched design.** Each user gets a reserved 1 Mbit/s circuit, so you must provision 10 × 1 = **10 Mbit/s** of link capacity. Average utilization: 10 users × 10% × 1 Mbit/s = 1 Mbit/s out of 10 = **10%**. You bought ten times the capacity you use. Guaranteed performance, terrible economics.

**Packet-switched design.** You provision, say, **2 Mbit/s** total and let users share it statistically. Average load is still 1 Mbit/s, so average utilization is **50%** — five times better. And most of the time it works beautifully, because the probability that many users burst simultaneously is low. Specifically, with each user independently active 10% of the time, the chance that 3 or more of the 10 are active at once (which is when 2 Mbit/s isn't enough) is about 7%.

That 7% is the honest price. Seven percent of the time, this network is *congested*: queues build, latency spikes, and packets get dropped. Circuit switching would never have done that — but circuit switching would have cost five times as much. **Packet switching trades guaranteed performance for enormous efficiency, and then spends the next fifty years of protocol design coping with the consequences of that trade.** Every retransmission timer, every congestion-control algorithm, every "why is the API slow today," every p99 latency dashboard you will ever stare at descends directly from this one decision.

### Visual: the same conversation, two networks

```
CIRCUIT SWITCHED  --  path reserved end to end, for the whole call
                      idle time is paid for and wasted

  A ==========[SW1]==========[SW2]==========[SW3]========== B
     ^^^^^^^^^^ this entire chain belongs to A<->B until hangup ^^^^^^^^^

  time:  |setup (~200ms of signalling)|-------- data + silence --------|teardown|


PACKET SWITCHED  --  nothing reserved; each packet addressed and forwarded
                     independently; links carry everyone's packets interleaved

              [R1]---------[R2]
             /   |  \      /   \
   A -------+    |   \    /     +------- B
             \   |    \  /     /
              [R3]-----[R4]---+

  A's packets:   #1 -> R1 -> R2 -> B
                 #2 -> R1 -> R4 -> B     (different path! link R1-R2 got busy)
                 #3 -> R3 -> R4 -> B     (R1 died; #3 rerouted; #2 was lost)

  time:  |no setup| #1 #2 #3 ... interleaved with C's, D's, E's packets |no teardown|
```

### Under the hood: store-and-forward, and where the delay actually comes from

Here is the mechanism a packet switch executes, and it's worth knowing precisely because it explains a number you'll see in every `traceroute` on Day 10.

A router or switch generally does **store-and-forward**: it receives the *entire* packet into a buffer, checks it, decides which output port it belongs on, and only then starts transmitting it. It does not begin forwarding while still receiving. So each hop adds four distinct delays:

1. **Transmission delay** — pushing the bits onto the outgoing link. `packet_bits / link_rate`. A 1500-byte frame on a 1 Gbit/s link = 12,000 bits / 10⁹ = **12 µs**.
2. **Propagation delay** — the bits travelling down the physical medium. Distance / ~2×10⁸ m/s. Across a 100 m office cable: **0.5 µs**. Across the Atlantic: **~30 ms**. This is the one you can never engineer away.
3. **Processing delay** — reading the header, looking up the destination, verifying the checksum. Sub-microsecond in modern hardware ASICs.
4. **Queueing delay** — waiting in the output buffer behind other packets. **This is the one that varies, and it is where all your unpredictability lives.** It's zero on an idle link and unbounded on a saturated one. When the buffer is full, the packet is dropped instead of queued.

Add up 1–3 and you get a floor you can compute. Item 4 is why your p50 latency is 20 ms and your p99 is 400 ms. When you eventually stare at a latency histogram with a long right tail (Phase 8), queueing delay is your first suspect, and it is a direct consequence of choosing statistical multiplexing over reservation.

**A cut-through variant exists:** some switches start forwarding as soon as they've read the destination address (~first 6 bytes), cutting transmission delay per hop dramatically. The trade-off is that they've forwarded the frame before they've seen its checksum, so they propagate corrupted frames instead of discarding them. In low-latency financial networks (the world Day 1's LMAX case study came from) that trade is worth it; in a general enterprise network it usually isn't.

### The senior-architect view: when does circuit switching still win?

It would be easy to read the above as "circuit switching lost, packets won, the end." That's the junior version. The honest version:

- **The real alternative:** reservation-based networking, which absolutely still exists. Dedicated leased lines and MPLS circuits, AWS Direct Connect / Azure ExpressRoute (a reserved private path to your cloud instead of the public internet), and cellular networks' scheduled radio slots are all reservation systems.
- **Why packet switching won for the internet:** the workload is bursty and heterogeneous, the topology changes constantly, and no one entity owns the whole path — so per-hop independent forwarding is the only thing that scales administratively as well as economically.
- **What was given up:** predictability. Circuit switching gives bounded jitter and no loss for free. Packet networks buy that back expensively and imperfectly, through QoS classes, over-provisioning, and application-layer retries.
- **The flip condition — be concrete:** reach for reservation when *jitter*, not throughput, is your SLO, and when the cost of a bad tail is higher than the cost of idle capacity. A trading firm co-locating in an exchange's data centre buys a dedicated cross-connect, not internet transit. A hospital's imaging network (you'll meet a real one in §11) needs predictable bandwidth for multi-hundred-megabyte studies. A telco's voice core still reserves radio resources. Every one of those is someone deciding "I will pay for capacity I don't always use, because 7% congestion is unacceptable to me."

### Case study ①: ARPANET, and the constraint that produced the internet — plus the myth you should not repeat

This is the day's first required case study, and it needs care, because the popular version of it is **wrong in a specific and instructive way**.

**What actually happened, in order:**

**Paul Baran at RAND (1960–64).** Baran was funded by the U.S. Air Force to answer one question: could you build a communications network that keeps working after a nuclear attack destroys much of it? His answer, published as the eleven-volume *On Distributed Communications* (RAND memoranda RM-3420 onward, 1964), is the founding document of the idea. He proposed a *distributed* mesh — no central switch, every node connected to several others — carrying what he called **"message blocks"**: fixed-size chunks, each independently addressed and independently routed by nodes making local decisions with no global knowledge. He explicitly quantified survivability: how much of the network you could destroy before end-to-end communication failed. The distributed mesh survived losses that a centralized or hierarchical star network did not. ([RAND RM-3420](https://www.rand.org/pubs/research_memoranda/RM3420.html); [RAND's retrospective](https://www.rand.org/pubs/articles/2018/paul-baran-and-the-origins-of-the-internet.html))

**Donald Davies at the UK's National Physical Laboratory (1965–66),** working entirely independently and with no military motivation at all, arrived at the same architecture for the *efficiency* reason from §2's arithmetic — interactive time-sharing wastes a circuit. He also gave it the name we use: **packet**.

**ARPANET (1966–69).** Bob Taylor and Larry Roberts at ARPA built the first large packet-switched network, with the first four nodes live in late 1969. Packets were forwarded by dedicated minicomputers called **IMPs** (Interface Message Processors) — the first routers.

**Here is the myth.** ARPANET is constantly described as having been *built to survive a nuclear war*. It wasn't. Taylor's and Roberts' stated goal was **resource sharing**: expensive, scarce, incompatible research computers should be usable remotely by researchers who couldn't afford their own. Roberts said so directly and repeatedly. The nuclear-survivability framing attached itself to ARPANET afterwards, partly via later DARPA leadership describing the agency's broader mission. ([Internet Society, *Brief History of the Internet*](https://www.internetsociety.org/internet/history-internet/brief-history-internet/); [Wikipedia's ARPANET article documents the misattribution and its source](https://en.wikipedia.org/wiki/ARPANET))

**So why is the case study still exactly right?** Because the *technique* ARPANET adopted came from Baran's survivability work, and survivability came along with it whether or not it was ARPA's goal. This is the engineering lesson, and it's a good one: **a constraint imposed on one design can produce a mechanism whose properties outlive the constraint.** Baran needed "no node is essential," and the only way to get that is per-hop independent forwarding of self-describing chunks. That same mechanism then turned out to be the right answer for bursty traffic, for heterogeneous links, and for a network with no central administration — three problems Baran wasn't solving.

**And the constraint is still visibly stamped on the protocols you'll use tomorrow:**
- **Each packet carries a full destination address.** Redundant if you had a reserved path; essential if any node can vanish. (Day 10)
- **No router knows the whole path.** Each makes a local next-hop decision from its own table, which is why routing can re-converge around a failure without any central authority. (Day 10)
- **Reliability lives in the endpoints, not the network.** This is the part most worth internalizing. Cerf and Kahn's 1974 paper, *A Protocol for Packet Network Intercommunication* (IEEE Transactions on Communications COM-22(5), May 1974, pp. 637–648 — [PDF](https://www.cs.princeton.edu/courses/archive/fall06/cos561/papers/cerf74.pdf)), split their original single protocol into what became **IP** (best-effort, stateless, dumb, forgets everything) and **TCP** (reliable, stateful, smart, lives only at the two ends). If the network can lose nodes at any time, you cannot store your connection state *in* the network. So you don't. You put it in the hosts. That principle — later named the **end-to-end argument** — is why a TCP connection survives a router rebooting, and why the internet could grow for fifty years without anyone upgrading the middle.

**One honest caveat:** this history is genuinely contested in its details and emphasis, and popular accounts (including many textbooks) get the ARPANET motivation wrong. Cite Baran for survivability, Davies for efficiency and the word "packet," Roberts/Taylor for ARPANET's resource-sharing goal, and Cerf/Kahn for the TCP/IP split. That mapping is defensible; "the internet was built to survive nukes" is not.

---

## 3. Layering: why the network is built as a stack, and what that costs

**Depth: [CORE]**

### Intuition: nobody can hold the whole problem in their head at once

Look at everything that has to be true for you to load a web page:

Voltage levels have to encode ones and zeros. Corrupted bits have to be detected. Two machines on the same office cable have to take turns. Machines on *different* continents have to find each other. A path has to be chosen across networks owned by companies that have never met. Lost data has to be re-sent. Data has to be delivered to *the right program* on the destination machine, not just the right machine. Bytes have to be encrypted so an ISP can't read them. And finally, something has to mean "GET the resource at /index.html."

If you tried to write one protocol that did all of that, you'd get a specification nobody could implement and nobody could ever change. Worse, every new physical medium — Wi-Fi, fibre, 5G, satellite — would require rewriting the whole thing, including the part about HTTP.

**Layering is the answer, and it is the same idea as every abstraction you already know.** Each layer solves exactly one problem, exposes a narrow interface, and *trusts its neighbours to have done their jobs*. A layer talks to the layer directly above and below it and knows nothing about any other. When you called `f.write(data)` on Day 5, you did not know or care whether the bytes were headed to an SSD, a RAM disk, or a pipe — the file-descriptor abstraction hid it. Network layering is that, four times over.

The concrete payoff is **independent evolution**. HTTP/1.1 was written in 1997 and still works over Wi-Fi 7 and 400 Gbit/s fibre, neither of which existed then, because HTTP never knew what a wire was. And Wi-Fi could be invented and deployed globally without changing one line of any web server, because the physical and link layers were replaceable underneath an unchanged IP.

### Analogy: an international shipping company

You want a crate delivered from a factory in Osaka to a shop in Lisbon.

- The **shop and factory** agree on what's in the crate and what it means (this is your *application* protocol — HTTP, or "send me index.html").
- The **freight forwarder** guarantees the whole shipment arrives, complete and in order, re-sending anything lost, and delivers it to the right *department* at the destination address (*transport* — TCP; the department number is the **port**).
- The **global logistics network** figures out the route: Osaka → Shanghai → Suez → Rotterdam → Lisbon, deciding one leg at a time (*internet layer* — IP).
- Each **individual leg** is done by whoever owns that segment — a truck company, a shipping line, a rail operator — each with its own paperwork, its own container standards, and no knowledge of any other leg (*link layer* — Ethernet, Wi-Fi, 5G).

The truck driver in Rotterdam does not know or care that this crate came from Osaka or that the shop wants it by Friday. He knows: pick up at the port, deliver to the rail terminal. That is the entire content of "each layer trusts its neighbours."

**Where the analogy breaks — three ways, all of them things you will hit in real work:**

1. **In shipping, the crate is never opened and re-labelled by a leg operator. In networking, middleboxes do exactly that.** A NAT router (Day 10) rewrites the IP *and* TCP headers of packets passing through it — a layer-3 device reaching up into layer 4. A layer-7 load balancer (Day 16) terminates the TCP connection entirely and opens a *new* one to your app, reading and rewriting the HTTP inside. Layer purity is an ideal that real networks violate constantly and deliberately, and knowing *which* layer a given box violates is most of debugging.
2. **The shipping legs don't duplicate each other's work; network layers do.** Ethernet has a CRC checksum, IP has a header checksum, TCP has its own checksum covering the whole segment, and TLS has an authentication tag. Four integrity checks on the same bytes. That's not incompetence — each protects a different scope against a different failure, and no layer is allowed to assume another one exists.
3. **A crate has one obvious "size." A packet's size is negotiated across layers and gets it wrong constantly.** The link layer imposes a maximum frame size, the IP layer has to fragment to fit, and TCP tries to guess the right segment size in advance. This mismatch (§9) is the source of some of the most maddening bugs in networking.

### The two models: seven layers you'll be asked about, four that actually shipped

Here is the part where most material hands you a seven-row table and moves on. That table is not the lesson. The lesson is that **there are two models, they disagree, and the one you'll be quizzed on is not the one that runs the internet.**

The **OSI model** (ISO/IEC 7498-1) was a formal international standard, designed by committee in the late 1970s and 80s, describing seven layers. It came with its own protocol suite, which lost, comprehensively, to TCP/IP. What survived is the *vocabulary* — and it survived so thoroughly that professional networking people say "layer 2," "layer 3," "layer 4 load balancer," and "layer 7 routing" every single day, all of which are OSI numbers.

The **TCP/IP model** is what actually exists. Its normative description for hosts is **RFC 1122**, *Requirements for Internet Hosts — Communication Layers* (October 1989, [rfc-editor.org/rfc/rfc1122](https://www.rfc-editor.org/rfc/rfc1122.html)), and it names **four** layers: application, transport, internet, and link. It has no session layer and no presentation layer, because in practice nothing needed them as separate layers — that work ended up inside applications and libraries.

Now that you know what each is, here's the mapping as a reference table — reading it is not learning it, and the reason it's here rather than at the top of this section:

| OSI # | OSI name | TCP/IP layer (RFC 1122) | What it actually solves | Real examples | Address it uses | Unit of data |
|---|---|---|---|---|---|---|
| 7 | Application | Application | "What do these bytes *mean*?" | HTTP, DNS, SMTP, gRPC | URL / hostname | message |
| 6 | Presentation | *(folded into app)* | encoding, serialization | JSON, TLS-ish, UTF-8 | — | — |
| 5 | Session | *(folded into app)* | dialogue state | (cookies, sessions — your job) | — | — |
| 4 | Transport | Transport | "which *program*, and did it all arrive?" | **TCP**, UDP, QUIC | **port** (16-bit) | segment / datagram |
| 3 | Network | Internet | "how do I reach a machine on *another* network?" | **IP**, ICMP, ARP-adjacent | **IP address** | packet |
| 2 | Data link | Link | "whose turn on *this* wire, and did the bits survive?" | **Ethernet**, Wi-Fi (802.11), PPP | **MAC address** | **frame** |
| 1 | Physical | *(part of link)* | voltages, light, radio, connectors | 1000BASE-T, 802.11ax | — | bit |

Two things to actually memorize out of that table, because they're the ones that make everything else clickable:

- **Layer 2 gets a frame to the next machine on the same physical network. Layer 3 gets a packet to a machine anywhere.** That distinction is §7 and it is the crux of today.
- **Layer 3 addresses a *machine*. Layer 4 addresses a *program on* that machine.** IP gets the packet to `192.168.1.7`; the TCP port `443` is what tells the kernel to hand it to nginx rather than to your SSH daemon. (Day 11 goes into how the kernel does that lookup, via the socket — which, from Day 5, is just a file descriptor.)

**Why "layer 5" is a joke among network engineers:** you will hear people refer to a bug as "a layer-8 problem" (the user) or "layer 5" vaguely. The genuine reason those layers are mushy is that OSI's committee defined them before anyone had built distributed applications, and reality routed around them. It's fine to know 7 numbers and use 4 layers — just don't mistake the numbers for a description of software that exists.

**A note on where layering leaks upward:** the "narrow waist" or **hourglass** shape of the internet is the deepest structural fact here. There are hundreds of application protocols on top and dozens of link technologies underneath, and in the middle, one protocol: IP. Everything is wide at the top and bottom and pinched to a single point in the middle. That's *why* you can invent a new application protocol on a Tuesday and it works everywhere, and also why changing IP itself (IPv6, Day 10) has taken thirty years and is still not finished. The narrow waist is what makes innovation cheap at the edges and nearly impossible in the middle.

---

## 4. Encapsulation: envelope inside envelope

**Depth: [CORE]**

### Intuition: how layers actually hand data to each other

Layering is a nice diagram. **Encapsulation is the mechanism that makes the diagram real**, and it is startlingly simple once you see it: *each layer takes whatever the layer above handed it, treats it as an opaque blob of bytes, and glues its own header on the front.*

That's it. Layer 4 hands layer 3 a chunk of bytes. Layer 3 does not look inside — it does not know or care whether it's TCP, UDP, or something invented next year. It prepends a 20-byte IP header saying "this is going to 203.0.113.10, and by the way the thing inside is protocol number 6." Then it hands the whole thing — its header plus the opaque blob — down to layer 2, which *also* doesn't look inside, prepends a 14-byte Ethernet header saying "next hop is the NIC with MAC 08:00:27:9c:5e:b2, and the thing inside is EtherType 0x0800," and pushes it onto the wire.

The receiving machine does the reverse: **decapsulation**. Layer 2 reads its own 14 bytes, sees EtherType 0x0800, strips its header, and hands the rest up to the IP code. IP reads its 20 bytes, sees protocol 6, strips its header, hands the rest to the TCP code. TCP reads its 20 bytes, sees destination port 80, strips its header, and hands the rest to whichever program has that port open.

The two crucial properties, and they're worth stating flatly:

1. **Every layer's header contains a "what's inside me" field.** EtherType at layer 2, the Protocol field at layer 3, the destination port at layer 4. Without those, the receiver couldn't know which code to hand the payload to. When you're reading a hex dump, these fields are your map.
2. **Every layer's payload is *entirely* opaque to it.** This is what makes the abstraction hold. Ethernet transporting IPv6 required exactly zero changes to Ethernet — a new EtherType value (0x86DD) and nothing else.

### Analogy: envelope inside envelope inside envelope

You write a letter (the HTTP request). You seal it in an envelope addressed to *Alice Chen, Accounts Payable* (the TCP header — the port is the department). You put that envelope inside a bigger one addressed to *Acme Corp, 12 High Street, Lisbon* (the IP header — the machine). You hand that to the internal mail room, which puts it in a courier sack labelled *for the van going to the sorting depot* (the Ethernet frame — the next hop, and *only* the next hop).

At the depot, the sack is opened and thrown away. A *new* sack, labelled for the next van, is made. The inner envelopes are untouched. Eventually at Acme's mail room the outer envelope is opened, and the inner one goes to Alice in Accounts Payable, who opens it and reads the letter.

**Where the analogy breaks — and this is the single most important sentence in this section:** the courier sack — the Ethernet frame — is **rebuilt from scratch at every single hop**, while the inner envelopes survive the whole journey untouched. A frame's life is one link, measured in metres. An IP packet's life is the whole journey, measured in continents. This is not a detail; it *is* the difference between a switch and a router (§7), and if you take only one thing from today, take this.

Also: real envelopes don't have a trailer. Ethernet does — a 4-byte checksum (the FCS) glued to the *end*, not the front. And real mail rooms don't pad short letters to a minimum size; Ethernet does (§5).

### Visual: the same bytes, four times

```
  YOU WRITE:                                            74 bytes
  +---------------------------------------------+
  | GET / HTTP/1.1\r\nHost: example.com\r\n...  |        <- L7 payload
  +---------------------------------------------+

  TCP ADDS 20 BYTES IN FRONT:                           94 bytes
  +----------------+---------------------------------------------+
  | TCP hdr        | GET / HTTP/1.1 ...                          |   <- "segment"
  | sport=52134    |                                             |
  | dport=80  <----+-- which PROGRAM on the far machine          |
  | seq, ack, flags|                                             |
  +----------------+---------------------------------------------+

  IP ADDS 20 BYTES IN FRONT:                            114 bytes
  +-------------+----------------+---------------------------------------------+
  | IP hdr      | TCP hdr        | GET / HTTP/1.1 ...                          | <- "packet"
  | src=.1.42   |                |                                             |
  | dst=203.0.  |                |                                             |
  |    113.10   |                |                                             |
  | proto=6 <---+-- what's inside|                                             |
  | TTL=128     |                |                                             |
  +-------------+----------------+---------------------------------------------+

  ETHERNET ADDS 14 IN FRONT + 4 AT THE END:             128 bytes + 4 FCS
  +----------------+-------------+----------------+-----------------------+------+
  | Ethernet hdr   | IP hdr      | TCP hdr        | GET / HTTP/1.1 ...    | FCS  |  <- "frame"
  | dst=router MAC |             |                |                       | CRC32|
  | src=my MAC     |             |                |                       |      |
  | type=0x0800 <--+-- what's ins|ide             |                       |      |
  +----------------+-------------+----------------+-----------------------+------+
   \_______________/                                                      \_____/
    REBUILT AT EVERY HOP                                    recomputed every hop
                     \___________________________________________________/
                      SURVIVES THE WHOLE JOURNEY UNCHANGED
                      (except TTL, which each router decrements -- Day 10)
```

Notice the overhead arithmetic: to deliver **74 bytes** of actual HTTP, the wire carried **128 bytes** of frame (plus 4 bytes of FCS, plus 7 bytes of preamble, 1 byte of start-of-frame delimiter, and a 12-byte inter-frame gap the link must stay silent for — so **152 byte-times** of link occupancy). That's a **~2× tax on small payloads**, which is precisely why chatty, small-message protocols are inefficient on a network in a way they never are in memory, and part of why batching is such a reliable performance win. (The same insight reappears on Day 15 as "why HTTP header compression was worth inventing," and again in Phase 7 as "why you batch LLM API calls.")

### Runnable example — peel one real frame apart, byte by byte

This is the concept made undeniable: the same 128 bytes, decoded by code you can read, showing four layers nested inside each other with real numbers. It's also directly the day's Build (§16) — the script accepts a hex dump copied straight out of Wireshark.

```python
# peel.py -- decode one Ethernet frame, layer by layer.
# No install needed: standard library only (Python 3.8+).
#   Windows:  python peel.py frame.txt      or      type frame.txt | python peel.py
#   Linux:    python3 peel.py frame.txt     or      cat frame.txt | python3 peel.py
import struct
import sys

BROADCAST = b"\xff\xff\xff\xff\xff\xff"
ETHERTYPES = {0x0800: "IPv4", 0x0806: "ARP", 0x86DD: "IPv6", 0x8100: "802.1Q VLAN tag"}
IP_PROTOS = {1: "ICMP", 6: "TCP", 17: "UDP"}
TCP_FLAGS = [(0x01, "FIN"), (0x02, "SYN"), (0x04, "RST"),
             (0x08, "PSH"), (0x10, "ACK"), (0x20, "URG")]


def mac(b):
    return ":".join(f"{x:02x}" for x in b)


def ipv4(b):
    return ".".join(str(x) for x in b)


def checksum(data):
    """The internet checksum (RFC 1071): 16-bit one's-complement sum, folded,
    then inverted. Run it over a *valid* header and you get 0."""
    if len(data) % 2:
        data += b"\x00"
    total = sum((data[i] << 8) | data[i + 1] for i in range(0, len(data), 2))
    while total >> 16:
        total = (total & 0xFFFF) + (total >> 16)
    return (~total) & 0xFFFF


def peel(frame):
    print(f"WIRE: {len(frame)} bytes captured "
          f"(the NIC already stripped the 7-byte preamble, 1-byte SFD and 4-byte FCS)\n")

    # ---- Layer 2: Ethernet II header. ALWAYS the first 14 bytes: 6 + 6 + 2. ----
    dst, src, etype = struct.unpack("!6s6sH", frame[:14])
    bcast = "  <- BROADCAST (every NIC in this LAN must accept it)" if dst == BROADCAST else ""
    print("L2 ETHERNET II  [bytes 0..13]")
    print(f"  destination MAC : {mac(dst)}{bcast}")
    print(f"  source MAC      : {mac(src)}   (OUI {mac(src[:3])})")
    print(f"  ethertype       : 0x{etype:04x} = {ETHERTYPES.get(etype, 'unknown')}")
    body, offset = frame[14:], 14

    if etype == 0x0806:                      # ---- ARP sits directly on Ethernet ----
        hw, pr, hl, pl, op = struct.unpack("!HHBBH", body[:8])
        sha, spa, tha, tpa = struct.unpack("!6s4s6s4s", body[8:28])
        print(f"\nL2.5 ARP  [bytes {offset}..{offset + 27}]  (28 bytes)")
        print(f"  htype/ptype     : {hw} (Ethernet) / 0x{pr:04x} (IPv4), "
              f"addr sizes {hl}/{pl}")
        print(f"  operation       : {op} = {'request' if op == 1 else 'reply'}")
        print(f"  sender          : {mac(sha)} = {ipv4(spa)}")
        print(f"  target          : {mac(tha)} = {ipv4(tpa)}")
        pad = len(body) - 28
        if pad:
            print(f"  padding         : {pad} zero bytes -> 14+28+{pad} = "
                  f"{len(frame)}, the 802.3 minimum (64 incl. FCS)")
        return

    if etype != 0x0800:
        print("\n(not IPv4 - stopping here)")
        return

    # ---- Layer 3: IPv4. Length is NOT fixed: low nibble of byte 0 = 32-bit words. ----
    ihl = (body[0] & 0x0F) * 4
    ver = body[0] >> 4
    tlen, ident, ff, ttl, proto, ck = struct.unpack("!HHHBBH", body[2:12])
    print(f"\nL3 IPv4  [bytes {offset}..{offset + ihl - 1}]  ({ihl} bytes)")
    print(f"  version / IHL   : {ver} / {ihl} bytes")
    print(f"  total length    : {tlen} bytes (IP header + everything above it)")
    print(f"  identification  : 0x{ident:04x}   flags/frag: 0x{ff:04x}"
          f"{'  DF set' if ff & 0x4000 else ''}")
    print(f"  TTL             : {ttl}   protocol: {proto} = "
          f"{IP_PROTOS.get(proto, '?')}")
    print(f"  header checksum : 0x{ck:04x}  -> recomputed over the header: "
          f"{checksum(body[:ihl])} (0 means valid)")
    print(f"  {ipv4(body[12:16])}  ->  {ipv4(body[16:20])}")
    upper, offset = body[ihl:tlen], offset + ihl

    if proto == 1:                                    # ---- ICMP ----
        t, c, ick, ident, seq = struct.unpack("!BBHHH", upper[:8])
        print(f"\nL3.5 ICMP  [bytes {offset}..{offset + 7}]  (8 bytes)")
        print(f"  type/code       : {t}/{c} = "
              f"{'echo request' if t == 8 else 'echo reply' if t == 0 else '?'}")
        print(f"  id / seq        : {ident} / {seq}")
        print(f"  checksum        : 0x{ick:04x} -> recomputed: {checksum(upper)}")
        print(f"\nPAYLOAD  [bytes {offset + 8}..{offset + len(upper) - 1}]  "
              f"({len(upper) - 8} bytes)")
        print(f"  {upper[8:]!r}")
        return

    if proto == 6:                                    # ---- TCP ----
        sp, dp, seq, ackn, offres, win, tck, urg = struct.unpack("!HHIIHHHH", upper[:20])
        doff = (offres >> 12) * 4
        flags = "|".join(n for bit, n in TCP_FLAGS if offres & bit) or "none"
        print(f"\nL4 TCP  [bytes {offset}..{offset + doff - 1}]  ({doff} bytes)")
        print(f"  ports           : {sp} -> {dp}")
        print(f"  seq / ack       : {seq} / {ackn}")
        print(f"  data offset     : {doff} bytes    flags: {flags}")
        print(f"  window          : {win}   checksum: 0x{tck:04x}")
        pseudo = body[12:20] + struct.pack("!BBH", 0, 6, len(upper))
        print(f"  checksum recomputed WITH the IPv4 pseudo-header: "
              f"{checksum(pseudo + upper)}")
        data = upper[doff:]
        print(f"\nL7 APPLICATION PAYLOAD  [bytes {offset + doff}.."
              f"{offset + len(upper) - 1}]  ({len(data)} bytes)")
        for line in data.split(b"\r\n"):
            print(f"  | {line.decode('ascii', 'replace')}")
        return


def parse_hexdump(text):
    """Accept a Wireshark/tcpdump-style dump: optional offset column, hex byte
    pairs, optional trailing ASCII column. Also accepts one long hex stream."""
    out = bytearray()
    for line in text.splitlines():
        toks = line.replace(":", " ").split()
        if toks and len(toks[0]) > 2 and all(c in "0123456789abcdefABCDEF" for c in toks[0]):
            toks = toks[1:]                       # drop the offset column
        for t in toks:
            if len(t) == 2 and all(c in "0123456789abcdefABCDEF" for c in t):
                out.append(int(t, 16))
            else:
                break                             # hit the ASCII column - next line
    return bytes(out)


if __name__ == "__main__":
    text = sys.stdin.read() if len(sys.argv) < 2 else open(sys.argv[1]).read()
    peel(parse_hexdump(text))
```

**The input.** Save this as `curl.txt`. This is a real, byte-correct frame — a `curl http://example.com/` request leaving a Windows 11 laptop at `192.168.1.42` towards an off-LAN server. (The destination `203.0.113.10` is from `192.0.2.0/24`-style documentation space, [RFC 5737](https://www.rfc-editor.org/rfc/rfc5737.html), so nothing here points at a live host. All four checksums in it are genuinely valid — the script proves it.)

```
0000  08 00 27 9c 5e b2 00 15  5d 3f a1 07 08 00 45 00
0010  00 72 3d 4e 40 00 80 06  bf 5a c0 a8 01 2a cb 00
0020  71 0a cb a6 00 50 1a 2b  3c 4d 5e 6f 7a 8b 50 18
0030  20 00 01 85 00 00 47 45  54 20 2f 20 48 54 54 50
0040  2f 31 2e 31 0d 0a 48 6f  73 74 3a 20 65 78 61 6d
0050  70 6c 65 2e 63 6f 6d 0d  0a 55 73 65 72 2d 41 67
0060  65 6e 74 3a 20 63 75 72  6c 2f 38 2e 39 2e 31 0d
0070  0a 41 63 63 65 70 74 3a  20 2a 2f 2a 0d 0a 0d 0a
```

**The invocation and the real output:**

```console
PS> python peel.py curl.txt
```
```
WIRE: 128 bytes captured (the NIC already stripped the 7-byte preamble, 1-byte SFD and 4-byte FCS)

L2 ETHERNET II  [bytes 0..13]
  destination MAC : 08:00:27:9c:5e:b2
  source MAC      : 00:15:5d:3f:a1:07   (OUI 00:15:5d)
  ethertype       : 0x0800 = IPv4

L3 IPv4  [bytes 14..33]  (20 bytes)
  version / IHL   : 4 / 20 bytes
  total length    : 114 bytes (IP header + everything above it)
  identification  : 0x3d4e   flags/frag: 0x4000  DF set
  TTL             : 128   protocol: 6 = TCP
  header checksum : 0xbf5a  -> recomputed over the header: 0 (0 means valid)
  192.168.1.42  ->  203.0.113.10

L4 TCP  [bytes 34..53]  (20 bytes)
  ports           : 52134 -> 80
  seq / ack       : 439041101 / 1584364171
  data offset     : 20 bytes    flags: PSH|ACK
  window          : 8192   checksum: 0x0185
  checksum recomputed WITH the IPv4 pseudo-header: 0

L7 APPLICATION PAYLOAD  [bytes 54..127]  (74 bytes)
  | GET / HTTP/1.1
  | Host: example.com
  | User-Agent: curl/8.9.1
  | Accept: */*
  |
  |
```

**Why this works, line by line — the load-bearing parts:**

- `struct.unpack("!6s6sH", frame[:14])` — the `!` means **network byte order**, i.e. big-endian: the most significant byte first. Your x86 CPU is little-endian, so every multi-byte field in every internet header must be byte-swapped on read and write. This is not a stylistic choice; RFC 791 fixed it in 1981 and every protocol since has followed. Getting `!` wrong is the classic first bug when hand-parsing packets, and it produces plausible-looking garbage (a port of 20480 instead of 80) rather than a crash. The `6s6sH` says: six raw bytes (destination MAC), six raw bytes (source MAC), one 16-bit unsigned int (EtherType). **The Ethernet header is always exactly these 14 bytes** — no options, no variable length. That fixed size is a deliberate design decision so that switching hardware can find the destination address at a known offset without parsing anything.
- `ihl = (body[0] & 0x0F) * 4` — and here, immediately, is the contrast. The **IP header is *not* fixed length**. Byte 0 packs two 4-bit fields: the high nibble is the version (4), the low nibble is the **IHL** — header length counted in 32-bit words. `0x45` → version 4, IHL 5 → 5 × 4 = **20 bytes**. IHL can legally go up to 15 (60 bytes) when IP options are present, which almost never happens in practice but which your parser must respect anyway. This is why you must never hardcode 20: you'd silently mis-parse everything above it.
- `struct.unpack("!HHHBBH", body[2:12])` pulls the fields you need to know today, and each one exists for a reason worth naming:
  - **total length = 114.** The IP packet's own length, header included. Ethernet already told us the frame was 128 bytes; 128 − 14 = 114. They agree, which is the check the code is implicitly making. Ethernet doesn't put a length in its header at all (in Ethernet II, that slot holds the EtherType), so the layers *must* each carry their own length independently — a small, concrete example of layers not trusting each other's bookkeeping.
  - **flags/frag `0x4000` = the DF (Don't Fragment) bit.** The sender is telling every router on the path: *if this doesn't fit, do not chop it up — throw it away and tell me.* TCP sets this deliberately so it can discover the path's maximum packet size. That's §9, and the failure mode when the "tell me" message gets blocked is one of networking's nastiest bugs.
  - **TTL = 128.** A hop counter. Every router decrements it; at zero the packet is destroyed and an error is sent back. It exists so that a routing loop can't produce immortal packets circulating forever. The initial value is an OS fingerprint: **Windows uses 128, Linux and macOS use 64.** Seeing 128 here tells you this frame came from a Windows box. (Day 10 owns TTL properly, including how `traceroute` weaponizes it.)
  - **protocol = 6.** The "what's inside me" field, exactly as promised in the intuition. 1 = ICMP, 6 = TCP, 17 = UDP. Without it, the receiving kernel wouldn't know which module to hand the payload to.
- `checksum(body[:ihl])` returning **0** is the neatest trick in the file. The internet checksum ([RFC 1071](https://www.rfc-editor.org/rfc/rfc1071.html)) is a 16-bit one's-complement sum, folded and inverted. Because the stored checksum is the *inverse* of the sum of all the other words, summing **everything including the checksum field** and inverting gives zero for a valid header. So a receiver doesn't compare checksums — it just recomputes over the whole thing and asserts the answer is 0. That's cheaper and simpler, and it's why routers can do it in hardware at line rate.
- `pseudo = body[12:20] + struct.pack("!BBH", 0, 6, len(upper))` — **this is a genuine, deliberate layering violation, baked into the standard.** TCP's checksum does not cover only the TCP segment. It also covers a fabricated 12-byte "pseudo-header" containing the source IP, destination IP, protocol number, and TCP length — fields that belong to **layer 3**. TCP is required to reach down and read IP's addresses. Why? So that a segment misdelivered to the wrong host, or with a corrupted destination address, fails its checksum instead of being silently accepted by the wrong machine. It's the right engineering call and it flatly contradicts the clean-layers story from §3, which is exactly why I told you in the analogy-breaks note that layer purity is an ideal, not a fact. (It also makes NAT expensive: rewriting an IP address forces a recomputation of the TCP checksum — Day 10.)
- `data = upper[doff:]` and the final loop — the payload is just **ASCII text**. That's the last and most demystifying observation of the whole exercise. There is no magic in HTTP; it's a string. You will type one by hand over a raw socket on Day 13.

**Honest caveats about this example, so you don't get confused when you do the real capture:**
- **The FCS is missing, and that's normal.** The 4-byte Ethernet checksum is verified and stripped by the NIC hardware before the OS ever sees the frame, so captures almost never contain it. When you compute frame sizes (§5), remember the FCS is on the wire even though it isn't in your capture.
- **`peel.py` handles exactly the four protocol combinations shown here.** It is a teaching tool, not a packet-analysis library — no IPv6, no VLAN tags, no IP options, no TCP options, no fragment reassembly. Wireshark handles all of those and thousands more; the point of writing your own is that after this you know what Wireshark is doing.
- **This does not sniff.** Capturing live traffic needs raw-socket privileges (Linux: `CAP_NET_RAW`/root; Windows: a driver like Npcap). Use Wireshark or `pktmon` to capture (§16) and paste the hex in here. Any tutorial that shows a pure-Python live sniffer is Linux-and-root-only and won't run as-is on Windows.

---

## 5. Layer 2 up close: the NIC, the MAC address, and the Ethernet frame

**Depth: [CORE]**

### Intuition: the layer that only knows about *this* wire

Layer 2's entire job description is one sentence: **get a frame from one device to another device on the same physical network, and tell the receiver whether the bits arrived intact.** That's all. It has no concept of "another network," no concept of a path, no concept of the internet. If you ask an Ethernet frame "how do I get to Google," it has no vocabulary in which to hear the question.

The hardware that does this is the **NIC** — network interface card (on modern machines, a chip on the motherboard, or a virtual device your hypervisor synthesizes). The NIC is the boundary between "software" and "electricity." Above it, the kernel builds byte buffers; below it, there are voltages on copper or pulses of light. And like the storage devices from Day 5, the NIC is presented to your code through the file-descriptor abstraction — a socket is an FD, and `send()` on it eventually results in the NIC's DMA engine pulling bytes out of RAM and clocking them onto the wire.

### The MAC address: layer-2 identity, and why it isn't an IP address

Every NIC has a **MAC address** (Media Access Control address): a **48-bit** number, conventionally written as six hex bytes — `00:15:5d:3f:a1:07`. Its purpose is to answer "which NIC on this wire is this frame for?"

It is **not** a location. This is the entire point and the source of endless confusion, so let's make the distinction sharp with a comparison you can reason from:

| | MAC address | IP address |
|---|---|---|
| Size | 48 bits (6 bytes) | 32 bits IPv4 / 128 bits IPv6 |
| Assigned by | the NIC manufacturer, at the factory | the network you plug into (DHCP, or by hand) |
| Changes when you move | **no** — it's the hardware | **yes** — new network, new address |
| Meaningful scope | one physical/logical LAN | the entire internet |
| Structure | flat: a vendor prefix, then a serial number | hierarchical: network part, then host part |
| Used by | switches | routers |
| Analogy | a person's fingerprint | a person's postal address |

The fingerprint/address analogy is the fastest way to hold this. Your fingerprint identifies *you* uniquely and forever, but nobody can deliver a parcel to a fingerprint. Your postal address changes when you move but is what makes delivery possible, because it's *hierarchical* — country, then city, then street, then number — so a sorting office can make a decision from a prefix without knowing every address in the world.

**Where that analogy breaks:** MAC addresses are not really permanent anymore, and you should know why. Every modern phone and laptop **randomizes** its Wi-Fi MAC address per network, on purpose, because a stable hardware identifier broadcast constantly into the air is a device-tracking mechanism. Virtual machines get synthesized MACs from their hypervisor's vendor block. Cloud VMs get MACs from the cloud provider. And any OS lets you override it in software. So treat a MAC as "the layer-2 identity this interface is currently using," not as "the machine's true name." Security designs that authenticate by MAC address (MAC filtering on Wi-Fi) are for this reason approximately worthless.

**The structure of those 48 bits** is worth one paragraph because it's how you identify what a device *is* from a capture. The first three bytes are the **OUI** (Organizationally Unique Identifier), a block bought from the IEEE Registration Authority by the manufacturer; the last three are assigned by that manufacturer. So in our frame, `00:15:5d` is registered to **Microsoft Corporation** and is what Hyper-V virtual NICs use; `08:00:27` traces to **PCS Systemtechnik GmbH** and is the block Oracle VirtualBox hands out. That's how the decoder's "(OUI 00:15:5d)" line becomes actionable: you look it up. The authoritative list is the IEEE registry itself ([standards-oui.ieee.org/oui/oui.txt](https://standards-oui.ieee.org/oui/oui.txt)) — *verify vendor claims there, because these lists drift as companies are acquired and blocks are reassigned.* Two bits inside the first byte carry meaning too: the lowest bit of the first byte is the **I/G bit** (0 = individual, 1 = group/multicast), and the next bit is the **U/L bit** (0 = globally unique OUI-derived, 1 = locally administered, i.e. made up). The all-ones address `ff:ff:ff:ff:ff:ff` is the **broadcast** address, meaning "every NIC on this LAN must accept this frame" — it has the I/G bit set, as it must.

### The frame itself: what those bytes are and why each one is there

The format is standardized as **IEEE 802.3** (the modern published edition, IEEE Std 802.3-2022, is the normative source; the [IEEE 802.3 working group page](https://www.ieee802.org/3/) is the entry point). Here is a real frame — the ICMP echo request from a Windows `ping 192.168.1.7`, byte-for-byte correct:

```
        preamble + SFD (8 bytes)      generated by the PHY, never in a capture
        |
        v
  [ 55 55 55 55 55 55 55 D5 ]
  +-----------------------------------------------------------------------+
  | dst MAC  | src MAC  | type | ................ payload ............... | FCS |
  |  6 bytes |  6 bytes | 2 B  |          46 .. 1500 bytes               | 4 B |
  +-----------------------------------------------------------------------+
  \_____________ 14-byte header _________/                              \_____/
                                                                    CRC-32 of
                                                                everything above
  then the link must stay SILENT for the inter-frame gap: 96 bit times (12 bytes)

  our actual ping frame, 74 bytes as captured (78 on the wire with FCS):

  0000  08 00 27 4b 1d e9 00 15  5d 3f a1 07 08 00 45 00
  0010  00 3c 1a 2b 00 00 80 01  9d 14 c0 a8 01 2a c0 a8
  0020  01 07 08 00 4d 5a 00 01  00 01 61 62 63 64 65 66
  0030  67 68 69 6a 6b 6c 6d 6e  6f 70 71 72 73 74 75 76
  0040  77 61 62 63 64 65 66 67  68 69
        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ "wabcdefghi" - Windows ping's payload is
                                        the ASCII string
                                        "abcdefghijklmnopqrstuvwabcdefghi" (32 B)
```

Run it through the decoder and it resolves completely:

```console
PS> python peel.py ping.txt
```
```
WIRE: 74 bytes captured (the NIC already stripped the 7-byte preamble, 1-byte SFD and 4-byte FCS)

L2 ETHERNET II  [bytes 0..13]
  destination MAC : 08:00:27:4b:1d:e9
  source MAC      : 00:15:5d:3f:a1:07   (OUI 00:15:5d)
  ethertype       : 0x0800 = IPv4

L3 IPv4  [bytes 14..33]  (20 bytes)
  version / IHL   : 4 / 20 bytes
  total length    : 60 bytes (IP header + everything above it)
  identification  : 0x1a2b   flags/frag: 0x0000
  TTL             : 128   protocol: 1 = ICMP
  header checksum : 0x9d14  -> recomputed over the header: 0 (0 means valid)
  192.168.1.42  ->  192.168.1.7

L3.5 ICMP  [bytes 34..41]  (8 bytes)
  type/code       : 8/0 = echo request
  id / seq        : 1 / 1
  checksum        : 0x4d5a -> recomputed: 0

PAYLOAD  [bytes 42..73]  (32 bytes)
  b'abcdefghijklmnopqrstuvwabcdefghi'
```

Now the field-by-field reasoning, which is where the design decisions live:

**Destination MAC first, not source.** The order looks arbitrary until you think about a switch. A store-and-forward switch could wait for the whole frame, but a *cut-through* switch wants to start forwarding as early as physically possible. Putting the destination in bytes 0–5 means the forwarding decision can be made after **six bytes** have arrived. Source second, because you need it to *learn* (§6) but not to *forward*. This is a hardware-latency optimization frozen into a 1980s frame format, and it's still paying off.

**EtherType (2 bytes).** The "what's inside" field: `0x0800` IPv4, `0x0806` ARP, `0x86DD` IPv6, `0x8100` a VLAN tag (§10). The registry is maintained by the IEEE Registration Authority, and assignments are deliberately scarce — [the IEEE's own EtherType tutorial](https://standards.ieee.org/wp-content/uploads/import/documents/tutorials/ethertype.pdf) explains the constraint. A historical wrinkle worth knowing so old captures don't confuse you: in the original IEEE 802.3 framing, this field was a **length** field, not a type field. The two coexist by a hack — values ≤ 1500 are read as a length, values ≥ 1536 (0x0600) as an EtherType. Everything you'll see in practice is the type interpretation (called "Ethernet II" or "DIX" framing, after DEC/Intel/Xerox).

**Minimum payload 46 bytes, minimum frame 64 bytes — and this is the number people get wrong.** Add it up: 14 (header) + 46 (payload) + 4 (FCS) = **64 bytes**, the 802.3 minimum frame size. If your payload is smaller, the sender must **pad with zeros** to reach it. You saw this happen: the ARP frame in §8 has a 28-byte ARP message, so it's padded with 18 zero bytes to reach 60 captured bytes (= 64 with FCS).

*Why 64?* Because of collision detection on the original shared coaxial cable. The rule was that a sender must still be transmitting when the earliest possible collision signal could get back to it — otherwise it would finish, assume success, and never learn the frame was destroyed. The standard fixed the maximum round-trip time for the largest legal cable segment as the **slot time**, 512 bit times. 512 bits = **64 bytes**. So the minimum frame size is literally the speed of light multiplied by the maximum cable length, rounded into a protocol constant. It is a beautiful example of physics leaking into a data format — and it survives today, thirty years after switched full-duplex Ethernet made collisions extinct, because changing it would break every NIC in the world.

**Maximum payload 1500 bytes, maximum frame 1518.** 14 + 1500 + 4 = **1518**. The 1500 is the **MTU** and it gets its own section (§9), because it is the single most consequential arbitrary number in networking. With one 802.1Q VLAN tag inserted (§10) the frame grows by 4 bytes to a maximum of **1522**.

**FCS (4 bytes, at the end).** A CRC-32 over everything above it. If it doesn't match, the frame is **discarded silently** — no error is sent to anyone, no retransmission is attempted. Layer 2 detects corruption; it does not correct it and does not report it. Whoever cares (TCP) must notice the gap themselves. This is the end-to-end argument again, from §2. It's also why `ifconfig`/`Get-NetAdapterStatistics` showing a rising CRC error count is a *physical* problem — a bad cable, a failing transceiver, electrical noise — and never a software one.

**Preamble (7 bytes of `0x55`) + SFD (1 byte, `0xD5`) + inter-frame gap (96 bit times = 12 bytes).** These are physical-layer scaffolding, not part of the frame. The preamble is an alternating `10101010` pattern that lets the receiver's clock lock onto the sender's bit timing (there is no shared clock — this is how the two ends synchronize), the SFD says "the real frame starts now," and the gap is mandatory silence so receivers can reset between frames. **You will never see these in Wireshark**, but they count against your link's capacity: our 74-byte ping actually occupies 74 + 4 + 8 + 12 = **98 byte-times** on the wire. That's the honest denominator when someone asks "how many packets per second fits in 1 Gbit/s?"

### Case study ②: Metcalfe & Boggs, 1976 — why the frame looks like this

Ethernet was designed at Xerox PARC by Robert Metcalfe and David Boggs, and published as **"Ethernet: Distributed Packet Switching for Local Computer Networks," Communications of the ACM 19(7), July 1976, pp. 395–404** ([ACM DL](https://dl.acm.org/doi/10.1145/360248.360253)). Read the title again: *distributed* packet switching. It's §2's principle applied to a single building.

Its intellectual parent was the **ALOHAnet** at the University of Hawaii, a packet radio network where stations transmitted whenever they had data and simply retried after a random delay if the acknowledgement didn't come. ALOHAnet worked but its efficiency was poor, because stations transmitted blindly into collisions. Metcalfe and Boggs' key addition was **carrier sense**: listen first, and if the cable is busy, wait. Plus collision detection with **exponential backoff** — on a collision, wait a random interval drawn from a window that *doubles* with each successive collision, which makes the network gracefully spread out contending senders under load instead of livelocking.

**The engineering lessons, tied to the bytes above:**
1. **A broadcast medium plus randomized backoff is a legitimate coordination mechanism** when you can't have a central arbiter. There is no lock, no leader, no scheduler — just "listen, transmit, detect, back off randomly." Exponential backoff is now everywhere in your professional life: it is how you should retry a failed HTTP call to an LLM provider, and it came from here.
2. **The design constraint that produced the 64-byte minimum was CSMA/CD, and it outlived CSMA/CD.** Backwards compatibility freezes physics into formats. You will meet this pattern repeatedly (TCP's 16-bit window field, IPv4's 32-bit addresses, the 1500-byte MTU).
3. **Ethernet won on economics, not elegance.** Token Ring (IEEE 802.5) was technically superior in several ways — deterministic access, no collisions, graceful degradation under load — and it lost, badly, because Ethernet was cheaper to build, cheaper to cable, and easier to extend. *This is the flip-condition lesson:* the technically better protocol loses to the one with a better cost curve and a bigger ecosystem, unless the requirement that the better protocol satisfies is a hard one. Token Ring's determinism only mattered where determinism was contractually required — which is why deterministic industrial variants of Ethernet (EtherCAT, TSN) exist today for factory floors, and plain Ethernet is everywhere else.

---

## 6. From hubs to switches, and the two "domains" you must be able to size

**Depth: [WORKING]**

### Intuition: what actually improved

The first Ethernet was a single coaxial cable that every machine physically tapped into. Then came the **hub** (a "multiport repeater"), which let you use star-shaped twisted-pair cabling instead — but a hub is electrically *dumb*: a signal arriving on any port is regenerated out **every other port**. It doesn't read the frame. It doesn't know what a MAC address is. Physically, a hub is still one shared cable, just with tidier wiring.

Two consequences, both bad:
- **Every machine receives every frame** and must inspect the destination MAC to decide whether to discard it. Any machine in promiscuous mode can therefore read all traffic on the LAN. (This is why packet sniffing used to be trivial, and why it mostly isn't now.)
- **All machines contend for the same bandwidth**, so a 10 Mbit/s hub with 20 machines gives everyone a share of 10 Mbit/s, not 10 each. And any two simultaneous transmissions collide.

The **switch** (originally called a "bridge") fixed this by doing something a hub never does: **it reads the frame's destination MAC and sends the frame out only the one port where that MAC lives.** To do that it needs to know which MAC is on which port — and it learns this, automatically, with no configuration, via an algorithm so simple it's almost a trick.

### Under the hood: the three rules of a learning switch

A switch maintains a **MAC address table** (also called a forwarding table, or on Cisco gear a **CAM table**, after the content-addressable memory it's stored in). It's a map: `MAC address -> port number, last-seen timestamp`. Three rules govern everything:

1. **Learn.** For every frame that arrives on port *P*, record `source MAC -> P`. Refresh the timestamp if it's already there. The switch learns from *source* addresses, always — never from destination addresses, because it has no evidence about where a destination is.
2. **Forward or flood.** Look up the *destination* MAC.
   - **Found** → send the frame out that one port only. This is called *forwarding*, and it's the whole value proposition: a conversation between ports 3 and 7 doesn't touch ports 1, 2, 4, 5, 6, 8.
   - **Not found** (an "unknown unicast") → send it out **every port except the one it arrived on**. This is *flooding*, and it's the switch's honest admission of ignorance. It's safe (the frame reaches its destination, which is what matters) and it's self-correcting: the destination will reply, and that reply teaches the switch where it lives.
   - **Broadcast destination** (`ff:ff:ff:ff:ff:ff`) or an unknown multicast → always flood. Not ignorance this time — that's what broadcast *means*.
3. **Age out.** Entries expire after an idle timeout — **300 seconds is the near-universal default** (Cisco's default aging time; verify on your own gear, as vendors differ). Without aging, a laptop unplugged from port 4 and plugged into port 9 would be unreachable forever. Ageing is why a switch heals from topology changes without anyone touching it.

### Worked example: trace a switch learning, frame by frame

Four machines on an 8-port switch. The table starts **empty** — a switch fresh out of the box knows nothing.

```
                       +--------------------------------------+
   A (aa:...:01) ------| p1                                p3 |------ C (cc:...:03)
                       |          8-PORT SWITCH               |
   B (bb:...:02) ------| p2                                p4 |------ D (dd:...:04)
                       +--------------------------------------+
```

| Step | Frame | Table before | Switch's action | Table after |
|---|---|---|---|---|
| 1 | A → C (unicast), arrives p1 | *(empty)* | Learn `aa:01 -> p1`. Destination `cc:03` unknown → **FLOOD out p2, p3, p4**. B and D receive it and discard it (wrong dst MAC). C accepts it. | `aa:01 -> p1` |
| 2 | C → A (the reply), arrives p3 | `aa:01 -> p1` | Learn `cc:03 -> p3`. Destination `aa:01` **known → forward out p1 only.** B and D see nothing. | `aa:01 -> p1`, `cc:03 -> p3` |
| 3 | A → C again, arrives p1 | both known | **Forward out p3 only.** No flooding. Ports p2/p4 are completely free — B↔D could be transmitting at full speed simultaneously. | unchanged |
| 4 | B sends an ARP broadcast, arrives p2 | 2 entries | Learn `bb:02 -> p2`. Destination is `ff:ff:ff:ff:ff:ff` → **FLOOD out p1, p3, p4.** Every machine's NIC accepts it and passes it up to the OS. | 3 entries |
| 5 | *(310 seconds of silence from C)* | 3 entries | `cc:03` ages out and is deleted. | `aa:01`, `bb:02` |
| 6 | A → C, arrives p1 | `cc:03` gone | **FLOOD again.** The switch has forgotten and must re-learn. | re-learns on C's reply |

Three things fall straight out of that table:

- **Step 3 is the whole point.** Because the switch forwards rather than floods, an 8-port gigabit switch can carry up to four simultaneous full-speed conversations — 4 Gbit/s of aggregate traffic — where a hub would have given you 1 Gbit/s shared. That's the "each port is its own collision domain" claim, below, made concrete.
- **Step 4 is the whole problem.** Broadcasts are exempt from all of this cleverness. They go everywhere, always, and every machine's CPU must process them. Switching does nothing to contain broadcasts. This is the seed of §11's disaster.
- **Step 6 explains a real symptom you'll eventually debug:** a mostly-silent server (a backup target, a rarely-queried database replica) ages out of switch tables and every packet to it gets flooded to the whole segment until it speaks again. On a busy segment this is measurable noise, and it's why some designs deliberately keep chatty keepalives running.

### Collision domain vs broadcast domain — the two sizes you must be able to reason about

These two terms sound similar, are constantly confused, and mean almost opposite things. Prose first, then the recap:

A **collision domain** is the set of devices that can physically interfere with each other's transmissions. On a shared cable or a hub, that's every attached device. On a switch, **each port is its own collision domain** — the switch buffers frames rather than letting signals meet — and once links became full-duplex (separate wires for transmit and receive, so a device can send and receive simultaneously), the collision domain shrank to a single link with exactly two endpoints, where a collision is impossible by construction. **Collision domains are, for practical purposes, a solved historical problem.** You should know the term, know why the 64-byte minimum frame exists because of it, and then stop worrying about it.

A **broadcast domain** is the set of devices that will receive a frame sent to `ff:ff:ff:ff:ff:ff`. And here is the crux: **a switch does not divide a broadcast domain.** Connect ten switches together and you have one broadcast domain spanning all of them. Connect a thousand devices and every ARP request, every DHCP discovery, every Windows name-resolution broadcast reaches all thousand NICs and interrupts all thousand CPUs. **Only a router (or a VLAN, which is a router boundary in disguise — §10) divides a broadcast domain.** Broadcast domains are absolutely *not* a solved problem; sizing them is an active design decision, and getting it wrong is how you get §11.

| | Collision domain | Broadcast domain |
|---|---|---|
| Defines | who can physically interfere | who receives a broadcast frame |
| Divided by | a **switch** (per port), or full-duplex | a **router** or a **VLAN** — *never* a plain switch |
| Enlarged by | a hub / shared cable | adding more switches |
| Still a real problem in 2026? | No — full-duplex killed it | **Yes** — and it scales badly |
| Symptom when too big | collisions, late collisions, poor throughput | broadcast flooding, CPU load on every host, storm risk |

### The judgment call: how big should a broadcast domain be?

This is a real decision engineers make, with a real alternative and a real flip condition.

- **The two options.** (a) One big flat layer-2 network — simple, no routing configuration, any device can move anywhere and keep its IP. (b) Many small segments joined by routing — more configuration, but broadcasts and failures are contained.
- **Why segmentation usually wins:** broadcast traffic grows roughly with the number of hosts, and *every* host pays the CPU cost of every broadcast, so cost grows superlinearly in aggregate. More importantly, a layer-2 failure (a loop, a storm, a misbehaving NIC) is contained by a router boundary and is *not* contained by a switch.
- **The trade-off you're accepting:** each boundary needs an IP subnet, a gateway, routing rules, and firewall policy. Mobility gets harder (moving a server across a boundary means changing its IP). More configuration means more ways to be wrong, and layer-3 misconfiguration causes its own outages.
- **The rule of thumb and its flip condition.** The common guidance is a few hundred hosts per broadcast domain — a `/24` (254 usable addresses) is the classic unit, and people get nervous above roughly 500–1000. *Verify against your own measurements rather than trusting the folklore number:* what actually matters is broadcast packets per second per host, which depends entirely on what the hosts are running. **Flip toward larger flat domains** when you genuinely need layer-2 adjacency across many machines — a legacy clustering product that requires it, a live-VM-migration fabric that must preserve IPs, or a storage protocol that expects a flat network. **Flip toward much smaller domains** when the segment carries different trust levels: then the driver isn't broadcast volume at all, it's blast radius, and you'd segment a 30-host network. That second case is exactly §12.

---

## 7. Switches vs routers: the most important boundary in networking

**Depth: [CORE]**

If you internalize one thing today, make it this section. Everything in Phase 1 — routing (Day 10), NAT (Day 10), load balancers (Day 16), cloud VPCs (§12) — is a variation on this boundary.

### Intuition: two different questions

A **switch** answers: *"which of my ports is this MAC address on?"* Its world is one LAN. It reads only the 14-byte Ethernet header and never looks at the IP header at all — as far as a switch is concerned, the IP packet is opaque payload. It operates at **layer 2**.

A **router** answers: *"which of my neighbours is closer to this IP network?"* Its world is the internet. It reads the IP header, consults a routing table, and — this is the mechanical heart of it — **throws the incoming Ethernet frame away and builds a brand-new one** addressed to the next hop. It operates at **layer 3**.

That last sentence is the mechanism. Let's watch it happen.

### Worked example: two pings from the same laptop, traced hop by hop

Your laptop is `192.168.1.42` with MAC `00:15:5d:3f:a1:07`. Your home router is `192.168.1.1` with MAC `08:00:27:9c:5e:b2`. There's a switch between you and the router (in a home router these are the same box, but keep them separate mentally).

**Case A — `ping 192.168.1.7` (a machine on your own LAN).**

Your OS does one subtraction first, and this subtraction is *the* decision. Your interface is configured `192.168.1.42/24`, meaning the first 24 bits are the network part. Compare the network part of your address (`192.168.1`) with the network part of the destination (`192.168.1`). **They match** → the destination is on my own LAN → **I can deliver it myself; no router involved.**

So the destination MAC must be `192.168.1.7`'s own MAC. Which the laptop doesn't know. So it ARPs (§8), gets `08:00:27:4b:1d:e9`, and sends:

```
  dst MAC 08:00:27:4b:1d:e9   <- the TARGET machine's own NIC
  src MAC 00:15:5d:3f:a1:07
  IP 192.168.1.42 -> 192.168.1.7        TTL 128
```

The switch looks up `08:00:27:4b:1d:e9`, finds it on port 3, forwards it out port 3. **The switch never read the IP header. The IP header and the Ethernet header both arrive exactly as sent. TTL is untouched at 128.** One layer-2 hop, zero layer-3 hops.

**Case B — `curl http://203.0.113.10/` (a machine on a different network).**

Same subtraction: my network part is `192.168.1`, the destination's is `203.0.113`. **They don't match** → the destination is *not* on my LAN → **I cannot deliver this myself. I must hand it to my default gateway and let it deal with it.**

Now the critical, non-obvious consequence: the laptop puts **the router's MAC address** in the destination MAC field — while leaving the IP destination as `203.0.113.10`:

```
  dst MAC 08:00:27:9c:5e:b2   <- the ROUTER's NIC. NOT the server's.
  src MAC 00:15:5d:3f:a1:07
  IP 192.168.1.42 -> 203.0.113.10       TTL 128     <- the real destination, untouched
```

This is exactly the frame in §4's runnable example, and now you know why its two addresses point at different machines. **Layer 2 says "next hop." Layer 3 says "final destination." They disagree, and that disagreement is normal and correct.** Beginners find this deeply confusing; it is the single highest-value insight of the day.

Then the router does five things, in order:

1. **Accepts the frame** because the destination MAC matches its own NIC. Strips the 14-byte Ethernet header and **discards it**.
2. **Reads the IP header**, looks up `203.0.113.10` in its routing table, and picks an outgoing interface and a next hop.
3. **Decrements TTL**: 128 → 127. This is a *mutation of the packet*, and it forces step 4.
4. **Recomputes the IP header checksum**, because it just changed a byte the checksum covers. (Note what it does *not* recompute: the TCP checksum. TCP's payload and headers are untouched, so its checksum stays valid — which is precisely why NAT, which *does* rewrite IP addresses, is expensive: the pseudo-header from §4 means the TCP checksum must be recomputed too.)
5. **Builds a completely new Ethernet frame** with its own outgoing NIC's MAC as source and the next hop's MAC as destination — ARPing for it if necessary — and transmits.

```
  ON YOUR LAN:                     INSIDE THE ROUTER:              ON THE NEXT LINK:

  +-----------------+                                            +-----------------+
  | dst 08:00:27:9c |  frame        header thrown away           | dst aa:bb:cc:.. | new frame
  | src 00:15:5d:3f |  #1     --->  +-------------------+  --->  | src 08:00:27:aa | #2
  +-----------------+               | new L2 header made|        +-----------------+
  | IP .1.42 ->     |               | TTL 128 -> 127    |        | IP .1.42 ->     |
  |    203.0.113.10 |  SAME PACKET  | checksum redone   |        |    203.0.113.10 |  SAME PACKET
  | TTL 128         | ------------- +-------------------+ ------ | TTL 127         |
  +-----------------+                                            +-----------------+
  | TCP + HTTP      |  byte-identical, checksum still valid      | TCP + HTTP      |
  +-----------------+                                            +-----------------+

  L2 addresses: CHANGE AT EVERY HOP.   L3 addresses: CONSTANT END TO END (unless NAT - Day 10).
  TTL: DECREMENTS AT EVERY L3 HOP.     Payload: NEVER TOUCHED by the network.
```

**Here is the diagnostic superpower this gives you.** When you eventually stare at a `traceroute` (Day 10) or a Wireshark capture and something is broken, you can now ask two independent questions:

- *Is layer 2 working?* Can this machine reach the next hop at all — is there an ARP entry for the gateway, does the switch have the right port? A failure here is local: bad cable, wrong VLAN, ARP problem.
- *Is layer 3 working?* Does a route to the destination exist, is TTL surviving, is a firewall dropping it? A failure here is remote and the local link is fine.

"Ping the gateway, then ping something beyond it" is the entire opening move of network debugging, and it works because of the boundary described above.

### The comparison table — recapping what the trace already showed

| | **Switch** (layer 2) | **Router** (layer 3) |
|---|---|---|
| Reads | Ethernet header only | Ethernet header **and** IP header |
| Decides using | destination **MAC** | destination **IP network** |
| Table | MAC address table (learned automatically from source MACs) | routing table (configured, or learned via a routing protocol — Day 10) |
| Table size | one entry per device on the LAN | one entry per *network* — this is why hierarchy matters |
| Unknown destination | **floods** out every port | **drops** the packet and sends back an ICMP error |
| Modifies the packet? | no — passes it through unchanged | yes — rewrites the L2 header, decrements TTL, recomputes the IP checksum |
| Broadcast domain | does **not** divide it | **divides** it — this is the containment boundary |
| Loop behaviour | **catastrophic** — frames circulate forever (§11) | self-limiting — TTL kills looping packets |
| Typical latency | microseconds (hardware ASIC) | microseconds to milliseconds (more work per packet) |
| Scale limit | table size and broadcast volume | routing-table size and convergence time |

**The row that deserves to be underlined is "loop behaviour."** An IP packet caught in a routing loop dies after at most 255 hops because of TTL. **An Ethernet frame has no TTL field at all.** There is no hop counter, no expiry, no self-destruct. A frame caught in a layer-2 loop circulates *forever*, and because switches flood broadcasts out every port, one broadcast frame in a looped topology multiplies. That absence — a missing 8-bit field in a 1970s frame format — is the direct cause of §11's four-day hospital outage.

### The judgment call: should this boundary be a switch or a router?

- **The real alternative to segmenting:** just switch everything. It genuinely is simpler, cheaper per port, faster per packet, and requires no addressing design. Plenty of small offices run one flat switched network and are fine.
- **Why you route anyway, at scale:** containment. A router boundary stops broadcasts, stops layer-2 loops, and gives you a place to *enforce policy* — you can filter, log, rate-limit, and audit at a layer-3 boundary. You cannot meaningfully do any of that inside a flat layer-2 segment.
- **What you give up:** cost (routing is more expensive per port), latency (slightly), configuration burden, and layer-2 adjacency.
- **The flip condition, stated concretely:** put a router boundary wherever you would want a *failure* or a *breach* to stop. Not where the org chart has a line, not where the building has a floor — where you want the blast radius to end. If the answer to "what happens if this segment melts down?" is "everything else keeps working," you've put the boundary in the right place. §11 is the story of an organization whose answer to that question turned out to be "the hospital closes."

---

## 8. ARP: the glue between layer 3 and layer 2

**Depth: [WORKING]**

### Intuition: the missing translation

§7 hid a gap. Your OS decided "the destination is on my LAN, so I'll deliver it directly to `192.168.1.7`." But Ethernet has never heard of `192.168.1.7`. To build the frame, the OS needs `192.168.1.7`'s **MAC address**, and nothing it has tells it that.

The mapping cannot be computed. There is no arithmetic relationship between an IP address and a MAC address — one is assigned by the network administrator or DHCP, the other was burned in at a factory. It also cannot be pre-configured, because that would mean hand-maintaining a table of every device's hardware address, which was tried in the 1970s and was miserable.

So it's *discovered*, at runtime, by asking the whole LAN out loud. That protocol is **ARP** — the Address Resolution Protocol, specified in **RFC 826** ("An Ethernet Address Resolution Protocol," David C. Plummer, November 1982 — [rfc-editor.org/rfc/rfc826](https://www.rfc-editor.org/rfc/rfc826.html)). It is one of the shortest and most consequential RFCs ever written, and its opening problem statement is still the clearest one-line summary: *"The 10Mbit Ethernet requires 48-bit addresses on the physical cable, yet most protocol addresses are not 48 bits long, nor do they necessarily have any relationship to the 48-bit Ethernet address."*

### Analogy: shouting in an open-plan office

You have a memo for "the person who handles invoice 4471" but you don't know their name or desk. So you stand up and shout to the whole floor: **"Who handles invoice 4471?"** Everyone hears it. Everyone but one person ignores it. That one person shouts back **"That's me, I'm Dana at desk 12."** You write that on a sticky note so you don't have to shout again, and you throw the note away after a while in case Dana moves desks.

That is ARP, essentially exactly: a broadcast question, a unicast answer, and a cache with a timeout.

**Where the analogy breaks — and it's a security-relevant break.** In an office, if a stranger shouted "That's me, I'm Dana," people would notice. **ARP has no authentication whatsoever.** Any machine on the LAN can answer any ARP request, truthfully or not, and can even send *unsolicited* replies that hosts will happily believe. That's **ARP spoofing**, and it lets an attacker on your LAN insert themselves as the gateway and read or modify all your traffic. RFC 826 was written in 1982 for a trusted research network and has no defence against this by design. This is a large part of why "the LAN is trusted" is a dead assumption and why TLS everywhere (Day 14) and zero-trust networking (§12) exist. It's also why you should never treat "we're on the same private network" as an authentication statement.

### Worked example: the two frames, byte-correct

Laptop `192.168.1.42` (`00:15:5d:3f:a1:07`) wants `192.168.1.7`. This is the plan's own example question — *"who has 192.168.1.7?"*

**Frame 1 — the request, broadcast to the entire LAN.** 60 bytes as captured:

```
0000  ff ff ff ff ff ff 00 15  5d 3f a1 07 08 06 00 01
0010  08 00 06 04 00 01 00 15  5d 3f a1 07 c0 a8 01 2a
0020  00 00 00 00 00 00 c0 a8  01 07 00 00 00 00 00 00
0030  00 00 00 00 00 00 00 00  00 00 00 00
```

Run it through the decoder from §4:

```console
PS> python peel.py arp.txt
```
```
WIRE: 60 bytes captured (the NIC already stripped the 7-byte preamble, 1-byte SFD and 4-byte FCS)

L2 ETHERNET II  [bytes 0..13]
  destination MAC : ff:ff:ff:ff:ff:ff  <- BROADCAST (every NIC in this LAN must accept it)
  source MAC      : 00:15:5d:3f:a1:07   (OUI 00:15:5d)
  ethertype       : 0x0806 = ARP

L2.5 ARP  [bytes 14..41]  (28 bytes)
  htype/ptype     : 1 (Ethernet) / 0x0800 (IPv4), addr sizes 6/4
  operation       : 1 = request
  sender          : 00:15:5d:3f:a1:07 = 192.168.1.42
  target          : 00:00:00:00:00:00 = 192.168.1.7
  padding         : 18 zero bytes -> 14+28+18 = 60, the 802.3 minimum (64 incl. FCS)
```

Read the target line again: **`00:00:00:00:00:00 = 192.168.1.7`.** That is the question, encoded. *"I know the IP. The MAC field is blank. Whoever owns this IP, fill it in and send it back."*

Three things to notice in those bytes:

- **`c0 a8 01 2a` is `192.168.1.42`** — hex `c0`=192, `a8`=168, `01`=1, `2a`=42. Learning to read dotted-quad IPs in hex is a small skill that pays off constantly when reading dumps.
- **The 18 zero bytes of padding** are §5's 64-byte minimum in action. The ARP message is 28 bytes; 14 + 28 = 42, which is below the 60-byte minimum (excluding FCS), so the NIC pads. Those zeros carry no information. Wireshark labels them "Padding" and occasionally, on buggy drivers, they contain leftover memory contents rather than zeros — a historical information-leak class of bug called *Etherleak*.
- **EtherType is `0x0806`, not `0x0800`.** ARP is **not** carried inside IP. It sits *directly* on Ethernet, as a peer of IP. That's why the OSI mapping question "what layer is ARP?" has no clean answer — it serves layer 3 but rides on layer 2 and is usually called "layer 2.5." Don't let an interviewer's confident answer bother you; the honest answer is "it's the glue between them, and it deliberately doesn't fit."

**Frame 2 — the reply, sent as a *unicast* straight back.** Only the asker needs it, so there's no reason to broadcast:

```
0000  00 15 5d 3f a1 07 08 00  27 4b 1d e9 08 06 00 01
0010  08 00 06 04 00 02 08 00  27 4b 1d e9 c0 a8 01 07
0020  00 15 5d 3f a1 07 c0 a8  01 2a 00 00 00 00 00 00
0030  00 00 00 00 00 00 00 00  00 00 00 00
```

Diff it against frame 1 and you can see exactly what changed: the operation byte pair went `00 01` → `00 02` (request → reply), and sender/target swapped, with the previously-blank hardware address now filled in as `08:00:27:4b:1d:e9`. Everything else is the same 28-byte structure — one message format serving both directions, which is why RFC 826 is so short.

**The complete timeline of what `ping 192.168.1.7` actually does on a cold cache** — this is the trace worth being able to recite:

```
  t=0.000  ping asks the kernel to send an ICMP echo to 192.168.1.7
  t=0.000  kernel: is .1.7 in my subnet 192.168.1.0/24?  YES -> deliver directly
  t=0.000  kernel: ARP cache lookup for 192.168.1.7 ... MISS
  t=0.000  the ICMP packet is QUEUED (not sent, not dropped -- held pending resolution)
  t=0.000  ARP REQUEST broadcast --> switch floods it out every port
             every host on the LAN receives it; every host's OS inspects it;
             all but one discard it
  t=0.001  192.168.1.7 sends an ARP REPLY, unicast, straight back
  t=0.001  kernel caches  192.168.1.7 -> 08:00:27:4b:1d:e9  (Windows: ~15-45 s;
             Linux: driven by base_reachable_time_ms, nominally ~30 s, randomized)
  t=0.001  the QUEUED ICMP packet is now framed and sent
  t=0.002  echo reply comes back;  ping prints "time=1ms"
  ---- the next 100 pings skip all of the above: the cache hits ----
```

**That is why the first ping in a sequence is often visibly slower than the rest.** It's not "warming up" in any vague sense — it paid for a full ARP round trip that the others didn't. Same effect, much larger, will explain DNS on Day 12.

### Hands-on: look at your own ARP cache right now

This is worth doing while reading, because it makes the abstraction concrete in about ten seconds.

**Windows (PowerShell) — native form:**
```powershell
Get-NetNeighbor -AddressFamily IPv4 | Where-Object State -ne 'Unreachable' |
    Select-Object ifIndex, IPAddress, LinkLayerAddress, State

# the classic tool still works too, and is what you'll see in older docs:
arp -a

# force a resolution and watch a new entry appear:
ping 192.168.1.1
Get-NetNeighbor -IPAddress 192.168.1.1

# clear it (needs an elevated prompt) and watch it get re-learned:
Remove-NetNeighbor -IPAddress 192.168.1.1 -Confirm:$false
```

**Linux / macOS equivalent:**
```bash
ip neigh show          # Linux, the modern command
arp -a                 # macOS and older Linux; `arp` is deprecated on Linux
sudo ip neigh flush all   # Linux: clear the cache
```

**Reading the output.** Windows `State` values map to the same neighbor-state machine Linux exposes: `Reachable` (confirmed recently), `Stale` (cached but unverified — will be re-probed on next use), `Permanent` (a static entry someone configured), `Incomplete` (an ARP request is outstanding and unanswered — *this is what a dead or wrong-subnet neighbour looks like*), and `Unreachable`. Seeing `Incomplete` for your default gateway is one of the most useful single diagnostics in networking: it means the machine cannot even reach its own router at layer 2, so nothing about layer 3 is worth investigating yet. Check cable, port, VLAN, and IP configuration — not routes.

**An honest platform note:** Windows and Linux differ in how aggressively they age entries, and neither documents a single fixed number, because both use randomized, usage-driven timers to avoid synchronized re-ARP storms across a LAN. Don't memorize a number; know that entries expire in tens of seconds to minutes and that this is deliberately fuzzy. (Windows' behaviour is documented under the neighbor-cache section of the Microsoft networking docs; Linux's knobs are in `/proc/sys/net/ipv4/neigh/*`. *Verify on your own system rather than trusting a blog.*)

### Two ARP variants worth recognizing

**Gratuitous ARP** — an *unsolicited* ARP reply (or a request for one's own address) broadcast to say "I am `192.168.1.50`, and my MAC is *this*, update your caches." Nobody asked. It's used legitimately for two things: detecting duplicate IP addresses when an interface comes up, and — importantly for backend work — **failover**. When a highly-available pair shares a virtual IP and the standby takes over, it must convince the whole LAN that `192.168.1.50` now lives at a *different* MAC, and gratuitous ARP is how. If you've ever seen a failover that took 30 seconds to actually shift traffic, a stale ARP cache was very likely the reason. The same mechanism, abused, is exactly ARP spoofing — legitimate use and attack are literally the same packet.

**Proxy ARP** — a router answering ARP requests on behalf of hosts that aren't on this segment, to fool hosts into thinking a remote machine is local. It was a useful crutch for stitching segments together before subnetting was widely understood. Treat it as **[AWARE]**: recognize the name, know it exists mostly in legacy configurations and some VPN setups, and treat it as a smell in new designs, because it makes the layer-2/layer-3 boundary lie to hosts — which is precisely the clarity §7 spent so long building.

**Where this is going:** IPv6 replaced ARP entirely with **NDP** (Neighbor Discovery Protocol, RFC 4861), which does the same job using ICMPv6 multicast instead of layer-2 broadcast — a cleaner design that fixes the "every host must process every ARP" problem. Day 10 mentions IPv6; the detail is beyond today's scope, and I'm stopping here deliberately.

---

## 9. MTU: the most consequential arbitrary number in networking

**Depth: [WORKING]**

### Intuition: why is there a maximum at all?

Ethernet's payload maxes out at **1500 bytes**. Why cap it? Two reasons, both about fairness and failure:

**A cap bounds how long one sender can monopolize the link.** On the original shared cable, while one machine transmits, nobody else can. A 1500-byte frame on 10 Mbit/s Ethernet takes 1.2 ms; a 64 KB frame would take 52 ms, during which every other machine waits. A cap is a *fairness* mechanism — the same reason an OS scheduler uses time slices instead of running a process to completion (Day 3). The parallel is exact: **a maximum frame size is a preemption quantum for a wire.**

**A cap bounds the cost of an error.** Layer 2 has no partial retransmission — a single flipped bit fails the FCS and the *whole frame* is discarded. Larger frames mean more data lost per error and more work wasted.

So 1500 is a compromise between per-frame overhead (which favours big frames — recall §4's 2× tax on small payloads) and fairness plus error cost (which favour small ones). It was chosen for 10 Mbit/s coax in 1980 and it has never changed, because changing it would break interoperability with every device on earth. **The number is arbitrary in origin and immovable in practice.** That combination is worth sitting with, because it's a recurring pattern in infrastructure.

The general term is **MTU** — Maximum Transmission Unit — the largest layer-3 packet a link will carry. Ethernet's is 1500. Some others differ, which is where the trouble starts.

### Worked example: measure your own path MTU from the command line

`ping` can send a packet with the DF (Don't Fragment) bit set — the `0x4000` flag you saw in §4 — and a payload size you choose. If the packet is too big for a link on the path, that link's router must drop it and reply with an ICMP "Fragmentation Needed" error rather than splitting it. So you can binary-search the largest size that gets through.

**Windows (PowerShell):** `-f` sets DF, `-l` sets the ICMP *payload* size.
```powershell
ping -f -l 1472 8.8.8.8      # 1472 + 8 (ICMP hdr) + 20 (IP hdr) = 1500 -> should work
ping -f -l 1473 8.8.8.8      # 1501 -> too big
```
Real output on a path with a 1500-byte MTU:
```
Reply from 8.8.8.8: bytes=1472 time=14ms TTL=116

Packet needs to be fragmented but DF set.
```
**Linux / macOS equivalent** (note that Linux's `-s` also sets payload size, but DF is `-M do`):
```bash
ping -M do -s 1472 8.8.8.8    # Linux
ping -D -s 1472 8.8.8.8       # macOS
```

**The arithmetic to be able to do in your head:** `1472 + 8 + 20 = 1500`. The 1500 MTU is the IP packet's total size — the IP header counts against it, the Ethernet header does not. So a "1500-byte MTU" means at most **1500 bytes of IP packet**, which is at most **1460 bytes of TCP payload** (1500 − 20 IP − 20 TCP), which becomes the **MSS** (Maximum Segment Size) that TCP advertises during its handshake (Day 11). Those three numbers — 1500 / 1460 / 1472 — are worth memorizing because they show up constantly.

Now the real-world twist: run that same test over a VPN or a cloud overlay network and you'll often find the answer is **1422**, or **1400**, or **1358**. Why? Because a VPN *encapsulates your entire packet inside another packet* — which is §4's mechanism applied recursively — and the outer headers eat into the 1500. A WireGuard tunnel typically leaves ~1420; IPsec less; various cloud overlays land near 1400. Every layer of tunnelling costs you payload.

### Under the hood: fragmentation, PMTUD, and the black hole

Three mechanisms handle a packet that's too big, and they fail in progressively nastier ways.

**1. IPv4 fragmentation (the old way, RFC 791).** If a router receives a packet too big for the next link and DF is *not* set, it chops the packet into fragments, each with its own IP header, using the Identification, Flags, and Fragment Offset fields you saw in §4. Only the *final destination* reassembles them. This is a bad design for reasons that took years to become clear: lose any one fragment and the entire original packet is lost (all that work wasted); reassembly requires the destination to buffer fragments and time them out, which is a denial-of-service surface; and stateless firewalls struggle because only the first fragment carries the TCP ports. **IPv6 removed router fragmentation entirely** — routers must drop and report, never fragment.

**2. Path MTU Discovery (the intended way, RFC 1191).** The sender sets DF on everything, and if some router can't fit the packet, it drops it and sends back **ICMP type 3, code 4** ("Destination Unreachable: Fragmentation Needed"), *including the MTU it can actually handle*. The sender caches that per-destination and sends smaller packets. Elegant, and it's why TCP sets DF by default.

**3. The PMTUD black hole (what actually happens too often).** PMTUD depends entirely on an ICMP error message getting back to the sender. Now consider a firewall administrator who, following advice from 1998, blocks "all ICMP" as a security measure. The oversized packet is dropped; the ICMP error that would have explained it is blocked; **the sender never learns anything.** So it retransmits the same too-large packet, forever.

The resulting symptom is legendarily confusing and worth memorizing as a pattern: **the TCP handshake succeeds, small requests work perfectly, and large transfers hang.** A `curl` of a tiny endpoint returns instantly; a `curl` of a 2 MB response stalls and eventually times out. `ping` works. DNS works. SSH connects and then freezes the moment you `cat` a big file. This is a *classic* interview question and a very real production bug, especially with VPNs, container overlay networks, and cloud tunnels in the path.

**How to fix it, in order of preference:** (a) stop blocking ICMP type 3 code 4 — it is not optional, it is a required part of IP's operation; (b) lower the MTU on the affected interface so packets fit without discovery; (c) **MSS clamping** — have a router rewrite the MSS value in TCP handshake packets down to something that fits, which is what your ISP's router is very likely doing for you right now on a PPPoE line. Note that (c) is another deliberate layering violation: a layer-3 device editing a layer-4 header. §3's analogy-break, appearing for the third time.

### Jumbo frames: the flip condition

Some networks use an MTU of **9000** ("jumbo frames"). The reasoning is §4's overhead tax: at 1500 bytes, headers plus inter-frame gap are ~2–3% of the wire and, more importantly, the *host* must process ~81,000 packets per second to fill a 1 Gbit/s link — and per-packet interrupt and syscall cost is what actually limits throughput on a fast link. Six times the payload per packet means roughly a sixth of the CPU per byte.

- **The alternative:** stay at 1500 and let hardware offloads (TSO/LRO — the NIC segments and reassembles for you, so the OS handles big buffers even though the wire carries 1500-byte frames) recover most of the benefit. This is what almost everyone does, and it's why jumbo frames matter much less than they did in 2005.
- **Why 1500 usually wins:** jumbo frames only work if **every single device in the path agrees**, and a single 1500-byte-MTU device silently ruins it, producing exactly the black hole described above. That's a brittle, all-or-nothing configuration dependency.
- **The honest trade-off:** jumbo frames buy real CPU savings on high-throughput bulk transfer; they cost you a fragile invariant across every switch, NIC, and host in the path.
- **The flip condition, concretely:** use 9000 when you fully control a *closed* high-throughput fabric — a storage network (iSCSI/NFS to a SAN), a database replication VLAN, an HPC or GPU-training interconnect. Never enable it on a path that touches the internet, where you cannot control the far end. Note that most cloud providers support ~9001 within a VPC but drop to 1500 for internet-bound traffic — *verify current values in your provider's docs, as these change.*

---

## 10. VLANs: one physical switch, many logical LANs

**Depth: [AWARE]**

### Intuition and the one thing to take away

§6 established that only a router divides a broadcast domain, and §7 established that you want those divisions where you want blast radius to end. But there's an awkward physical consequence: if segmentation requires separate switches, then dividing your network requires *buying and cabling* separate switches, and moving a machine between segments requires physically re-patching a cable. In a 1990s office with finance on the third floor and engineering on the fourth, that meant two parallel sets of switches and cabling in every room.

**VLANs (Virtual LANs) decouple the logical segment from the physical wiring.** One switch is configured so that ports 1–8 belong to VLAN 10 and ports 9–16 belong to VLAN 20, and the switch then behaves *exactly as if it were two entirely separate switches*: a broadcast on port 3 reaches ports 1–8 and never reaches ports 9–16. Two broadcast domains, one box. To get traffic between them you must route — which is precisely the containment property you wanted, now available without re-cabling.

The mechanism is a **4-byte tag** inserted into the Ethernet frame, standardized as **IEEE 802.1Q** (folded into the current published 802.1Q edition; the [IEEE 802.1 working group page](https://www.ieee802.org/1/) is the entry point — *verify the current edition year, as it is periodically revised*):

```
  UNTAGGED (what a normal host sends, and what §5 showed you):
  +----------+----------+------+----------------+-----+
  | dst MAC  | src MAC  | type | payload        | FCS |     max 1518 bytes
  +----------+----------+------+----------------+-----+

  802.1Q TAGGED (what travels between switches, on a "trunk" port):
  +----------+----------+---------------+------+----------------+-----+
  | dst MAC  | src MAC  | 8100 | TCI    | type | payload        | FCS |  max 1522
  +----------+----------+---------------+------+----------------+-----+
                        \_____ 4 bytes _/
                         TPID   PCP(3) DEI(1) VID(12 bits -> 4094 usable VLANs)
                         0x8100  ^                ^
                                 |                +-- WHICH VLAN this frame is in
                                 +-- priority, for QoS
```

Two consequences worth carrying:

- **Access ports vs trunk ports.** A port facing a host is an *access* port: it carries one VLAN, untagged, and the host has no idea VLANs exist. A port facing another switch is a *trunk*: it carries many VLANs, tagged, so the receiving switch knows which segment each frame belongs to. Your laptop has never seen an 802.1Q tag and doesn't need to.
- **The max frame size grew to 1522** while the MTU stayed at 1500. That's §9's problem waiting to happen: stack a VLAN tag plus a tunnel header and you quietly lose payload space. Historically, gear that couldn't handle the extra 4 bytes produced "baby giant" frame errors.

**The security caveat, which is why this is [AWARE] and not more:** a VLAN is a *configuration*, not a physical wall. Misconfigure the "native VLAN" on a trunk and frames can leak between segments (**VLAN hopping**, via double-tagging or switch-spoofing attacks). A VLAN is a good containment boundary against *accident* and a mediocre one against a *determined attacker on the same switch*. For real isolation between trust levels, you want a routed boundary with a filtering policy — which is exactly what §12's cloud subnets give you.

**I'm stopping here deliberately.** VLAN design (trunking protocols, VTP, private VLANs, MSTP per-VLAN spanning trees) is a whole discipline that matters enormously if you administer physical switches and almost not at all if you work in a cloud, where the provider's software-defined network hides all of it behind subnet and security-group APIs. Know what a VLAN is, know a tag is 4 bytes and the VID is 12 bits, know it's a broadcast-domain boundary and a soft security boundary. That's the working set.

---

## 11. Loops, broadcast storms, and spanning tree

**Depth: [WORKING]**

### Intuition: redundancy is mandatory, and at layer 2 it is also lethal

Everything you know about reliability says: add a second path. One cable is a single point of failure, so run two, so that if one is cut the other carries the traffic. This instinct is correct at layer 3 and at the application layer, and at **layer 2 it will destroy your network** — not degrade it, destroy it.

Here is why, and it comes down to the missing field flagged in §7's comparison table. **An Ethernet frame has no TTL.** An IP packet caught in a loop is killed by its hop counter after at most 255 hops. A frame has no such field, no expiry, and no self-destruct. Combine that with the switch's flooding rule — *broadcasts and unknown unicasts go out every port except the one they arrived on* — and a physical loop turns one frame into an exponentially growing flood that never stops.

### Worked example: trace one broadcast frame into a loop, with numbers

Three switches, cabled in a triangle, which is exactly what you get if two well-meaning people each add "a redundant link":

```
                  +--------+
             +----|  SW-A  |----+
             |    +--------+    |
             |                  |
        +--------+          +--------+
        |  SW-B  |----------|  SW-C  |
        +--------+          +--------+
             |
          host H  (about to send ONE ARP request, 64 bytes)
```

**t = 0.** H sends **one** broadcast frame into SW-B.

**Round 1.** SW-B floods it out all other ports: one copy to SW-A, one copy to SW-C. **2 copies in flight.**

**Round 2.** SW-A receives its copy and floods it out every port except the one it arrived on — so it sends a copy to SW-C. SW-C received its copy from SW-B and floods a copy to SW-A. *Independently*, SW-C also floods back toward SW-B, and SW-A floods back toward SW-B. **4 copies in flight.**

**Round 3.** Each of those 4 arrives somewhere and is flooded out 2 other ports. **8 copies.**

**Round *n*: 2ⁿ copies.** There is no mechanism anywhere in this system that removes a copy. Not one. Every copy is a valid frame with a valid checksum and a broadcast destination, and every switch is following the standard correctly.

**Now the timing, because this is what makes it violent.** A 64-byte frame plus its preamble, SFD, FCS and inter-frame gap occupies ~84 byte-times = 672 bits. On a 1 Gbit/s link that's **0.672 microseconds**, so that link can carry at most 10⁹ / 672 ≈ **1.49 million such frames per second**. Reaching that from a single frame takes 21 doublings (2²¹ ≈ 2.1 million), and each round is on the order of one frame time. **21 × 0.672 µs ≈ 14 microseconds** — call it *tens of microseconds* once you add real propagation and switch processing.

**Tens of microseconds from one ARP request to a completely saturated network.** No human reaction time exists at that scale. And it gets worse than "saturated," for three compounding reasons:

1. **Every host's CPU is now processing millions of broadcast frames per second,** because broadcasts are, by definition, delivered up the stack to the operating system rather than filtered by the NIC. Machines become unresponsive at the *console*, not just on the network — which is what makes remote recovery impossible and turns this into a physical, walk-to-the-rack incident.
2. **The MAC address tables become garbage.** SW-A now sees H's source MAC arriving on *two different ports*, alternating, thousands of times a second. It rewrites the entry constantly — this is called **MAC table thrashing** — so even legitimate unicast traffic starts getting flooded because the table can never settle. The loop poisons unicast forwarding too.
3. **It is self-sustaining.** Even if H is unplugged, the copies already in flight keep multiplying. Pulling the cable to the *host* does nothing; you must break the *loop*.

This is a **broadcast storm**, and internalizing that ~14-microsecond number is what makes the case study below make sense.

### Spanning tree: the fix, and what it costs

The solution, invented by **Radia Perlman** and published as *"An Algorithm for Distributed Computation of a Spanning Tree in an Extended LAN"* (SIGCOMM '85, pp. 44–53 — [ACM DL](https://dl.acm.org/doi/10.1145/319056.319004)), is elegant: **let the switches cooperatively discover the loops and deliberately switch off enough links to leave exactly one path between any two points** — a tree. The redundant links stay physically connected but are put in a *blocking* state, forwarding nothing. If an active link fails, spanning tree recalculates and unblocks a previously-blocked link.

The protocol became **IEEE 802.1D** (STP). Mechanically, switches exchange small frames called **BPDUs** (Bridge Protocol Data Units), elect a **root bridge** (lowest bridge ID wins), and each switch then blocks every port that isn't on its own shortest path to the root.

**Here is the cost, and it's the part that produced a hospital outage.** The original 802.1D was *slow*, deliberately, because it had to be conservative — unblocking a port too early would create the very loop it was preventing. Its default timers are hello = 2 s, max age = 20 s, forward delay = 15 s, and a port transitions blocking → listening → learning → forwarding through two forward-delay periods. **Reconverging after a topology change takes 30 to 50 seconds** — during which affected traffic simply stops. ([Cisco's timer documentation](https://www.cisco.com/c/en/us/support/docs/lan-switching/spanning-tree-protocol/19120-122.html) walks through the arithmetic.)

**And those timers encode an assumption you must know about.** They are derived from a formula that assumes a maximum **bridge diameter of 7** — no more than seven switches between any two endpoints. The IEEE recommendation is explicit about this, and the forward-delay default falls out of it: `((4 × hello) + (3 × diameter)) / 2 = ((4 × 2) + (3 × 7)) / 2 = 14.5`, rounded to 15 seconds. **If your network is deeper than 7 switches, the default timers are too short for information to propagate across it before other switches conclude the topology has changed — and STP can fail to converge at all,** oscillating instead of settling. Remember that number: **7**.

**Rapid Spanning Tree (RSTP)**, originally IEEE 802.1w and folded into 802.1D-2004 (and now living inside the 802.1Q standard — the standalone 802.1D was withdrawn in 2011), replaced the timer-driven state machine with explicit handshakes between neighbouring switches, cutting reconvergence from tens of seconds to typically **under a second**. RSTP is what modern gear runs by default. *Verify what your specific switches are actually running — plenty of production networks are still on classic STP because nobody changed it.*

**The judgment call here is real.** The alternative to spanning tree is *not* "no redundancy" — it's moving redundancy up a layer. Modern data-centre design largely does exactly that: instead of a wide layer-2 domain protected by STP, you build a routed (layer-3) fabric where every link is *active* and equal-cost multipath routing spreads traffic across all of them. Spanning tree's fundamental waste is that blocked links carry **zero** traffic — you pay for capacity you cannot use. **The flip condition:** stay on STP when your topology is small, when you need layer-2 adjacency across switches, and when operational simplicity matters more than link utilization (a branch office). Move to a routed fabric when you have enough links that idling half of them is a real cost, or when convergence time matters more than STP can deliver. Data-centre "leaf-spine" designs, and technologies like MC-LAG, TRILL, and SPB, all exist to escape the "half your links are off" tax.

### Case study ③: CareGroup / Beth Israel Deaconess Medical Center, 13–16 November 2002

This is a real, named, publicly documented incident whose root cause was a layer-2 loop meeting a spanning-tree design that could not cope. It is *the* canonical teaching case for this failure mode, and unusually well documented because the CIO chose to talk about it in detail afterwards.

**The setting.** CareGroup was a Boston health system whose flagship was Beth Israel Deaconess Medical Center, a Harvard teaching hospital. Its CIO, Dr. John Halamka, ran an IT organization that *InformationWeek* had, shortly before, ranked #1 among hospitals for innovation. The clinical systems were genuinely advanced for 2002 — electronic records, electronic prescribing, and a **PACS** (Picture Archive Communication System) carrying large radiology image studies.

**The network underneath was not.** It was a flat, almost entirely layer-2 switched network that had accreted through a hospital merger without ever being redesigned. Halamka's own descriptions are the most quotable engineering summary of technical debt you will find: *"You have a state-of-the-art network — for 1996,"* and *"a network of extension cords to extension cords. It was very fragile."* A core switch installed in 1996 was still the main aggregation point. Crucially, **the PACS segment sat 10 switch hops from the nearest core switch — three more than spanning tree's diameter assumption of 7.**

**What happened.** On Wednesday 13 November 2002, a researcher uploaded several gigabytes of data into a medical file-sharing application, **and it looped**. The traffic saturated the network. Spanning tree, already operating beyond its supported diameter, could not converge — and with the topology unable to settle, the network did not merely slow down; it collapsed, repeatedly, on and off across roughly **four days**.

**The clinical consequences, which is why this is the case study and not a lab exercise.** Clinicians reverted to paper records that hadn't been used in years — physicians hand-wrote prescriptions for the first time in their careers at that hospital. Lab results that normally came back in about 45 minutes took **five hours**. One physician's account: *"Usually I get labs back in less than an hour; they were taking five hours, and here I have a patient who could die. I was scared."* And at **3:50 p.m. Beth Israel Deaconess closed its emergency department**, diverting ambulances elsewhere, and it stayed closed for four hours until 7:50 p.m.

**The fix.** Cisco escalated to its Customer Assurance Program, flying engineers and hardware from San Jose to Boston. The structural remedy was exactly the boundary from §7: they inserted **Cisco 6509 routers** between the core network and the PACS segment — replacing a layer-2 path with a **layer-3** one, which both eliminated the seven-hop spanning-tree constraint on that path and, more importantly, created a containment boundary that a layer-2 storm cannot cross. Afterwards, CareGroup rebuilt the network with redundancy and segmentation as first-class properties.

**Sources:** Scott Berinato, "All Systems Down," *CIO* magazine, February 2003 — [Computerworld's reprint](https://www.computerworld.com/article/1346623/all-systems-down.html) — and [CIO's interview with Halamka](https://www.cio.com/article/2440210/halamka-on-beth-israel-s-health-care-it-disaster.html). Halamka later described the incident repeatedly in public talks and writing, and it became a Harvard teaching case. *Details in secondary retellings vary slightly; the primary accounts above are the ones to trust.*

**The engineering lessons, each tied to a specific mechanism in this note:**

1. **A layer-2 domain is a single fault domain, and it was the *whole hospital*.** The blast radius of one looping application was every clinical system, because §6's rule holds: switches do not divide broadcast domains. The remedy was a router, which does. This is §7's flip condition — *put the boundary where you want failure to stop* — learned the expensive way.
2. **Spanning tree's defaults encode an assumption that nobody had checked.** The 7-hop diameter isn't a soft guideline; it's baked into the timer arithmetic. A network can be inside spec for years, then a single new segment pushes it out of spec, and the failure appears only under load. **Know your invariants and monitor them.** "Number of switch hops from any access segment to the core" is a boring metric that would have caught this.
3. **The trigger was an application bug, and the *cause* was the network's inability to contain it.** Blaming the researcher's file-sharing app would be a total misreading. Applications will always misbehave; the design question is whether one misbehaving application can take down everything. This is the single most transferable idea here, and it is the same principle behind bulkheads, circuit breakers, and per-tenant rate limits that you'll meet in Phase 5 and Phase 8.
4. **Infrastructure debt is invisible until it is catastrophic.** The clinical applications on top were genuinely award-winning. Halamka's own conclusion — *"You can't treat your network like a utility"* … *"We took the plumbing for granted"* — is the whole lesson. Layers below you being boring is not the same as them being safe.
5. **Recovery required physical presence and days, not minutes.** When the network is the failure, your monitoring, your remote access, and your management tools are all *downstream of the thing that's broken*. Plan for out-of-band access before you need it. (This exact compounding — the tool you'd fix it with is behind the thing that broke — is what made Meta's 2021 BGP outage so long, which you'll study on Day 10.)

**An honest note on the evidence base.** Publicly documented, named postmortems of layer-2 loops are **rare**, and that's not because loops are rare — it's a reporting artifact. Layer-2 storms happen inside private enterprise LANs, where there is no public status page, no customer-facing SLA to explain, and every incentive not to publish. The cloud-era outages that *do* get postmortems (AWS, Cloudflare, Meta) are overwhelmingly layer-3-and-above, because cloud providers' internal fabrics are routed rather than flat-switched — partly *because* of lessons like this one. So CareGroup is doing double duty here: it's a real named incident with real published detail, and it's close to the only one at this depth. **I'm not going to invent a second one to fill a slot.** If you want more of this failure class, the honest place to look is vendor troubleshooting documentation (Cisco's STP troubleshooting guides describe the symptom set precisely) and network-operator mailing-list archives (NANOG), not a curated list of famous outages.

---

## 12. The same ideas, renamed: segmentation in the cloud

**Depth: [WORKING]**

### Intuition: you will never touch a switch, and you still need all of the above

Here's the thing that confuses people coming to cloud from a book like this one: in AWS, Azure, or GCP you never configure a switch, never see a MAC address, never think about spanning tree. So why did you just read §5–§11?

Because **the cloud didn't remove those concepts; it renamed them and made them API calls.** Under a cloud VPC is a software-defined network where the provider's hypervisors encapsulate your packets inside *their* packets (§4's mechanism, applied recursively — which is also why cloud MTUs are odd) and route them across their own fabric. The abstractions they expose to you map almost one-to-one onto what you now understand:

| Physical networking (what you just learned) | Cloud equivalent | What actually changed |
|---|---|---|
| a switched LAN / broadcast domain | a **subnet** | the provider suppresses broadcast entirely; ARP is answered by the fabric |
| the router between segments | the VPC's implicit router (a route table) | it always exists; you configure routes, not a device |
| a whole routed site | a **VPC / VNet** | it's an isolated address space you're handed |
| a stateful firewall on a host | a **security group / NSG** | attached to the interface, **stateful** — return traffic is automatic |
| a stateless ACL on a router interface | a **network ACL** | attached to the *subnet*, **stateless** — you must allow both directions |
| "is there a route to the internet?" | an **internet gateway** attached (or not) to a route table | absence of a route *is* the isolation mechanism |
| outbound-only internet access | a **NAT gateway** | one-way by construction |
| a VLAN as a soft boundary | subnet + security group + route table | a policy boundary, enforced in the hypervisor |

And the design principle transfers exactly: **put a boundary where you want failure and compromise to stop** (§7). The cloud makes that cheap — a subnet is a JSON object, not a purchase order — which means there is no excuse for a flat network anymore.

One genuinely new property worth naming: **there is no broadcast.** AWS, Azure, and GCP all suppress broadcast and multicast within a VPC by default. ARP still conceptually happens from your VM's point of view (it has an ARP cache; you can look at it), but the fabric answers on behalf of the destination rather than flooding. §11's storm is therefore structurally impossible in a VPC. That's not a reason to skip §11 — it's a reason to understand *why* cloud networks were designed to make it impossible, and it means the failure modes you must design against in the cloud are different (misconfigured routes and over-permissive security groups, not loops).

---

### System design ①: network segmentation for a 3-tier application

**The problem.** You're deploying a standard web application: an internet-facing load balancer, a fleet of stateless application servers running your FastAPI app (Day 7's skeleton, eventually), and a managed PostgreSQL database. Traffic is a few thousand requests per second. Requirement: **a compromise of any single tier must not automatically grant access to the tier behind it**, and the database must not be reachable from the internet under *any* misconfiguration of a single control.

**Requirements, stated properly before designing anything:**
- Public HTTPS ingress from anywhere.
- App servers must reach the DB, and must reach the internet *outbound* (to call an external API, fetch OS packages).
- The DB must reach nothing outbound, and be reachable only from app servers.
- Two availability zones, because a single AZ is a single failure domain.
- An engineer must be able to operate the app servers without exposing SSH to the internet.

**The alternatives a competent engineer would actually consider:**

**(a) One flat subnet, security groups only.** Everything — LB, app, DB — in one public subnet, isolated purely by security-group rules. This is not a straw man; it's genuinely the fastest thing to build, and security groups are stateful and reasonably expressive. **Why it loses:** it's a single-control design. One over-permissive security group rule — `0.0.0.0/0` on port 5432, added at 11 p.m. during an incident by someone debugging — and your database is on the public internet with no second barrier. The failure is one mistake deep.

**(b) Three-layer subnet design with routing as the primary control.** Public / private / isolated subnets, where the *routing table* is the outer control and security groups are the inner one.

**(c) Fully private, no internet gateway at all**, with all outbound access via private endpoints to provider services. Maximum isolation, and legitimately the right answer for some regulated workloads.

**The decision: (b), with elements of (c) for the data tier.** Because the specific requirement was *"a compromise of any single tier must not automatically grant access to the tier behind it,"* and (b) is the only option that gives **two independent, differently-shaped controls** on the critical path. To reach the DB from the internet you would have to defeat both a routing decision (there is no route from the internet to that subnet) *and* a security-group decision. A single fat-fingered rule cannot do it, because a security group cannot create a route.

```
   Internet
      |
  [ Internet Gateway ]
      |
+-----+-------------------------------------------------------------+
|  VPC  10.0.0.0/16                                                 |
|                                                                   |
|  PUBLIC subnets      10.0.0.0/24 (az-a)   10.0.1.0/24 (az-b)      |
|    route: 0.0.0.0/0 -> IGW           <-- the ONLY subnets with this
|    contains: the load balancer ONLY, and the NAT gateway           |
|    SG(lb):  in  443 from 0.0.0.0/0    out 8000 to SG(app)          |
|      |                                              |             |
|      v                                              v             |
|  PRIVATE subnets     10.0.10.0/24 (az-a)  10.0.11.0/24 (az-b)     |
|    route: 0.0.0.0/0 -> NAT gateway   <-- outbound only, never in   |
|    contains: app servers                                          |
|    SG(app): in  8000 from SG(lb)      out 5432 to SG(db)          |
|                                       out 443 to 0.0.0.0/0        |
|      |                                                            |
|      v                                                            |
|  ISOLATED subnets    10.0.20.0/24 (az-a)  10.0.21.0/24 (az-b)     |
|    route: LOCAL ONLY -- no IGW, no NAT. There is no path out.      |
|    contains: the database                                         |
|    SG(db):  in  5432 from SG(app)     out: NOTHING                |
+-------------------------------------------------------------------+
```

**The load-bearing details, each with its reason:**

- **Security groups reference *other security groups*, not IP ranges.** `SG(db)` allows 5432 from `SG(app)`, not from `10.0.10.0/24`. This matters because it stays correct when you autoscale, re-IP, or add a third AZ — and because it means "being in the private subnet" is not sufficient to reach the DB; you must be an *app server*. A compromised sidecar or a debug container in the same subnet doesn't inherit access.
- **The isolated subnet's route table is the real control.** No internet gateway, no NAT. So even if malware runs on the database host, **it has nowhere to send data.** This is the single most valuable property in the design, and it costs nothing.
- **The NAT gateway lives in the *public* subnet, and app servers route through it.** This is the standard shape and it confuses everyone once: outbound-only internet access requires a device that *has* inbound-capable placement. NAT is one-directional by construction — it can translate a reply back to an outbound flow it remembers, but it has no state for an unsolicited inbound connection, so there is nothing for an attacker to connect to. (NAT mechanics are Day 10.)
- **Operator access is via a provider-managed session service** (AWS SSM Session Manager, Azure Bastion) rather than SSH from the internet or a bastion host with a public IP. The reason is that it removes an internet-reachable port entirely — you're authenticating to a control-plane API, which is auditable and integrates with your identity system, instead of managing SSH keys.
- **Network ACLs, if you use them, are the stateless belt to security groups' braces.** Remember the stateful/stateless distinction from the translation table at the top of §12: an NACL that allows outbound 443 also needs an inbound rule for the **ephemeral port range** (roughly 1024–65535), because it doesn't track connections. Getting this wrong produces "connections work in one direction only," which is a genuinely confusing symptom. Most teams leave NACLs wide open and rely on security groups; the case for using them is defence against a *security-group* misconfiguration, which is exactly the two-independent-controls argument above. *Verify the exact ephemeral range for your OS and provider — it differs.*

**What was given up (be honest):** complexity and cost. Three subnet tiers × 2 AZs = 6 subnets, 3+ route tables, and NAT gateways that are billed hourly *and* per gigabyte processed — often a surprisingly large line item. Debugging is harder: "I can't reach the database" now has five possible causes (route table, security group, NACL, DNS, the app) instead of one. For a hobby project this is over-engineering, and saying so is not weakness.

**The flip condition — when a different design is right:**
- **Toward simpler:** an internal-only tool with no sensitive data and one developer. One private subnet plus security groups. The two-control argument doesn't pay for itself when there's nothing to steal.
- **Toward stricter (option c):** if you handle regulated data (payments, health records), remove the NAT gateway entirely and give app servers outbound access only to *specific* provider endpoints (VPC endpoints / Private Link) and an explicit allowlist of external hosts via a proxy. The flip trigger is a compliance requirement that says "egress must be enumerable" — at that point "outbound to anywhere on 443" stops being acceptable, and you're building the next scenario.
- **Toward a service mesh / mTLS:** when you have dozens of services rather than three tiers, subnet boundaries stop being expressive enough (you can't write "service A may call service B's `/read` but not `/write`" as a route table). Then identity-based authorization at layer 7 becomes the primary control and the network becomes a coarse backstop. Day 14 and Phase 8 cover this; the flip trigger is *number of services*, not amount of traffic.

**Failure modes to design for now, not later:**
- **A single-AZ subnet.** The most common real outage in this shape isn't a breach, it's an AZ failure taking out a subnet you forgot to duplicate. Every tier needs at least two.
- **Subnet CIDR exhaustion.** A `/24` gives 251 usable addresses in AWS (the provider reserves five). Autoscale into that and new instances fail to launch with an error that says nothing about IP addresses. You cannot resize a subnet after creation — size generously up front. (CIDR math is Day 10; today, know that the constraint exists and bites.)
- **The NAT gateway as a hidden single point of failure and a hidden bill.** It's per-AZ; one NAT gateway serving both AZs means an AZ failure takes out egress for both. It also means all your outbound traffic appears from **one IP address**, which is what makes third-party IP allowlisting possible (Day 10's system design) and what makes you look like a single abusive client to a rate limiter.
- **Security-group rule sprawl.** Over months, rules get added and never removed. Without automated review, the design's guarantees quietly erode. Treat security groups as code with review, not as console clicks.

---

### System design ②: network isolation for agent tool execution

**The problem.** This is the one place today where having an LLM in the loop genuinely changes the engineering, and it's the direct continuation of Day 8's sandbox design. You are building a service where an LLM writes Python code and your platform **executes it** — a code-interpreter tool. Day 8 gave you the machine-level layers: a separate process, dropped privileges, memory caps (`cgroups`), CPU and wall-clock timeouts, a read-only filesystem, and no FD inheritance. **Every one of those controls is about the machine. None of them is about the network,** and the network is where the interesting attacks live.

**Why the network specifically is the hard part.** The model's output is untrusted in a way that ordinary application input is not. Your app's user input flows into *code you wrote*; the model's output flows into *code that gets executed*. And the model can be steered by content it reads — a web page, a retrieved document, a tool result — which is **prompt injection**. So the realistic threat isn't "the model is malicious"; it's "an attacker who can put text anywhere the model reads can eventually cause arbitrary code to run in your infrastructure." Given that, ask what code running in your VPC can do *purely with network access*, even with no filesystem and no privileges:

- **Read cloud credentials.** On AWS, `http://169.254.169.254/latest/meta-data/iam/security-credentials/` returns the instance's IAM role credentials — over plain HTTP, with no authentication beyond being on the host. Azure and GCP have equivalents at the same link-local address. **This is the single highest-value target and it is one HTTP GET away.** It is exactly how the 2019 **Capital One** breach escalated: a server-side request forgery against a misconfigured WAF host reached the EC2 metadata endpoint, retrieved that instance's IAM role credentials, and those credentials had enough S3 permissions to exfiltrate data on ~106 million applicants. Disclosed 29 July 2019. ([Krebs on Security's contemporaneous technical writeup](https://krebsonsecurity.com/2019/08/what-we-can-learn-from-the-capital-one-hack/); the criminal complaint is the primary document.) The mechanism — *an untrusted request reaching the metadata endpoint from inside your own infrastructure* — is precisely what agent code execution reproduces, deliberately, every time it runs.
- **Scan and reach your internal services.** Your database, your Redis, your internal APIs — all of which likely trust "anything inside the VPC." §8's lesson applies: *being on the network is not authentication.*
- **Exfiltrate.** Any outbound connection is a data channel. It doesn't need to be HTTP — DNS queries alone are enough to encode and leak data, and DNS is the protocol people most often forget to restrict.
- **Be abused as a proxy.** Crypto mining, spam, or attacking a third party — from your IP address, with your reputation and your bill.

**The alternatives:**

**(a) Restrict at the application layer** — inspect the generated code and block network calls before executing it. **Why it loses, decisively:** this is static analysis of adversarial input, which is undecidable in general and trivially bypassed in practice (`__import__('so'+'cket')`, `getattr`, `eval` of an encoded string, or a shell-out you didn't anticipate). Every allowlist of "dangerous functions" is a list of the bypasses you happened to think of. Do it as defence in depth if you like; never as *the* control.

**(b) A container with no network namespace at all** (`--network none`). Genuinely strong and often exactly right. **Why it's insufficient on its own:** most real agent tools *need* some network — `pip install`, an API the user asked to call, fetching a URL. The moment the answer is "well, just this one thing," you're back to needing a policy, and "no network" gives you no way to express one.

**(c) Egress-restricted subnet with a mandatory filtering proxy.** Isolation as a network-topology property, plus a single enumerable, auditable egress point.

**The decision: (c), layered on top of (b) as the default.** Start every execution with no network. When the tool genuinely needs egress, place the sandbox in a subnet whose *route table* makes the filtering proxy the only possible path out. The reason is the same principle as scenario ①, sharpened: **make the safe behaviour a property of the topology, not of a check that code must remember to perform.** Code can forget a check. Code cannot invent a route.

```
+---------------------------------------------------------------------------+
|  VPC                                                                      |
|                                                                           |
|  SANDBOX SUBNET   10.0.90.0/24    <-- nothing else lives here             |
|    route table:  10.0.0.0/16 -> local ...... REMOVED (no VPC-local route!) |
|                  0.0.0.0/0    -> egress proxy ENI                         |
|                  169.254.169.254 -> blackholed at the host (see below)     |
|                                                                           |
|    +-------------------------------------------------------------+        |
|    |  agent code-exec container (Day 8's sandbox)                |        |
|    |   - unprivileged user, read-only rootfs, no caps            |        |
|    |   - memory cap, CPU cap, 30s wall clock                     |        |
|    |   - IMDS blocked at the host: hop limit 1 + IMDSv2 required |        |
|    |   - DNS -> only the proxy's resolver, nothing else          |        |
|    +----------------------------+--------------------------------+        |
|                                 |  the ONLY route out                     |
|                                 v                                         |
|    +-------------------------------------------------------------+        |
|    |  EGRESS PROXY  (in its own subnet)                          |        |
|    |   - explicit host allowlist ("pypi.org", "api.example.com") |        |
|    |   - TLS SNI / HTTP CONNECT inspection, per-host             |        |
|    |   - request + byte-volume logging for every attempt          |        |
|    |   - rate limits and a byte cap per execution                |        |
|    +----------------------------+--------------------------------+        |
|                                 |                                         |
|  APP / DB SUBNETS               |     SG(db) in 5432 from SG(app) ONLY    |
|   (unreachable from sandbox:    |     -- the sandbox's SG is not SG(app), |
|    no route AND no SG rule)     |        so even a route wouldn't help    |
+---------------------------------+-----------------------------------------+
                                  v
                              Internet (allowlisted hosts only)
```

**The load-bearing controls, each with the specific attack it stops:**

1. **Block the instance metadata service, explicitly and at the host.** This is the highest-value single control and it's the one people forget. Two mechanisms, use both. **Require IMDSv2**, which forces a `PUT` to obtain a session token before any `GET` — defeating the trivial one-shot `GET` that SSRF and naive generated code produce. And keep the **`HttpPutResponseHopLimit` at 1**, which is AWS's default: the token response is returned with an IP TTL of 1, so it dies at the first router — and reaching a container across the Docker bridge counts as that hop. The result is that **containers on the instance cannot use IMDS at all**, which is exactly what you want here. Note the trap: a great deal of container documentation tells you to raise the hop limit to 2 *precisely because* the default breaks container access. For a sandbox, that breakage is the feature — leave it at 1 and give your legitimate workloads credentials by another route (ECS task roles, EKS Pod Identity), never by relaxing this. ([AWS: Use the Instance Metadata Service](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html) — *verify current parameter names and defaults; AWS has been progressively tightening IMDS defaults and this is exactly the kind of fact that drifts.*)
2. **Remove the VPC-local route from the sandbox subnet's route table.** By default, every subnet in a VPC can reach every other subnet — that's the implicit local route. Deleting it means the sandbox cannot even *address* your database, independent of any security-group rule. Two independent controls again: no route *and* no allow rule.
3. **Force all egress through a proxy with a host allowlist.** The point isn't that a proxy is unbypassable — it's that egress becomes **enumerable and logged**. You can answer "what did this execution talk to?" from a log, which you fundamentally cannot do with a NAT gateway. That auditability is worth as much as the blocking.
4. **Restrict DNS to the proxy's resolver.** If sandboxed code can query arbitrary DNS servers, it has a covert channel: encode data in subdomain labels and read it off your own authoritative nameserver's logs. No HTTP required. This is a real, widely-used exfiltration technique and blocking port 53 to anything but your resolver is cheap.
5. **Cap bytes and time per execution, not just CPU.** Day 8 gave you a wall-clock timeout. Add an egress byte cap, because "slowly exfiltrate 10 GB" is a different attack from "spin the CPU," and a CPU limit doesn't touch it.
6. **One sandbox per execution, destroyed afterwards.** Reuse means execution N can leave something behind for execution N+1, including a listener or a poisoned cache. This is a network control as much as a process control: a fresh network namespace per run means no residual connections.

**What was given up:** latency and cost. A cold sandbox per execution costs hundreds of milliseconds to seconds of startup — which is real, since users are already waiting on model tokens. A proxy is another hop, another service to run, and another thing that can go down and break every tool call. And the allowlist is **operational friction by design**: every new legitimate destination is a change request. Teams under delivery pressure will be tempted to add `*` to the allowlist, and that single change silently converts this whole design back into scenario ①'s NAT gateway. **The allowlist must be reviewed like code, or it decays.**

**The flip condition:**
- **Toward stricter — no network at all:** if the tool's job is pure computation (data analysis on a payload you supply, running a test suite), give it `--network none` and pre-stage its dependencies into the image. This is strictly safer *and* faster than the proxy design. Reach for the proxy only when egress is genuinely required. Most code-interpreter workloads fall here, and defaulting to open egress "because someone might need `pip`" is a real and common mistake.
- **Toward stricter still — a different trust boundary entirely:** if the sandbox will run code on behalf of *mutually untrusted tenants*, a shared-kernel container is no longer the right isolation primitive at all, because a kernel escape crosses tenants. Move to a hardware-virtualized microVM (Firecracker, gVisor's syscall interception, or per-tenant VMs). The flip trigger is **multi-tenancy**, not scale.
- **Toward looser:** if the "sandbox" only ever runs code *you* wrote and reviewed, and the model merely chooses *which* function to call with validated arguments (a normal tool-calling agent — Phase 2), then none of this applies and building it is waste. **The entire threat model above is triggered by one specific decision: executing model-authored code.** Be precise about whether you're actually doing that; a lot of "agent security" advice fails to distinguish these two cases and consequently over-engineers the common one.

**Failure modes:**
- **The allowlist grows to `*` under delivery pressure.** The most likely way this design fails is social, not technical. Mitigate by alerting on allowlist changes and reviewing them like production code.
- **The proxy becomes a single point of failure.** Every tool call now depends on it. Run it redundantly and decide *explicitly* whether it fails open or fails closed — and fail **closed**, accepting that a proxy outage breaks tool calls, because failing open silently removes the entire control at exactly the moment nobody is watching.
- **Sandbox escape via a shared resource you forgot is shared.** A mounted volume, a Unix socket, a shared cache, an inherited FD (Day 5), a hypervisor bug. Enumerate what crosses the boundary; the network is one channel, not the only one.
- **The metadata block is silently undone.** Someone re-creates the instance from a template without the hop-limit setting. This should be a policy check in your infrastructure-as-code pipeline, not a hope. (Phase 8 covers policy-as-code.)

---

### System design ③: sizing a broadcast domain for a campus office

A deliberately smaller, distinct problem that exercises §6 and §7 directly rather than the cloud abstractions.

**The problem.** 900 devices across three floors of an office building: laptops on Wi-Fi, VoIP phones, printers, badge readers, HVAC controllers, and a small server room. Design the layer-2/layer-3 boundaries.

**Requirements:** a device on any floor can reach the server room; a compromised badge reader must not be able to reach a laptop or a server; Wi-Fi roaming between floors must not drop calls; and the network must not be one fault domain (§11's lesson).

**The alternatives.** **(a) One flat `/22` broadcast domain** for all 900 devices — simplest, one subnet, roaming trivially works, and it's what the building's original installer would have done. **(b) One VLAN and subnet per floor** — three domains of ~300, matching the physical topology. **(c) One VLAN and subnet per *device class*** — laptops, phones, printers, building systems, servers — regardless of floor.

**The decision: (c), device class, not floor.** The reason is that the stated requirement is about **trust**, not geography: "a compromised badge reader must not reach a laptop." Per-floor segmentation (b) puts the badge reader and the laptop in the *same* domain if they're on the same floor, so it satisfies the broadcast-volume goal while completely failing the security goal. Per-class segmentation puts the boundary where the trust difference actually is. A rough plan: `vlan 10` laptops/Wi-Fi (`/22`, sized for growth and roaming), `vlan 20` VoIP phones (`/24`, tagged for QoS priority — that's the PCP field from §10), `vlan 30` printers (`/25`), `vlan 40` building systems (`/26`, the most restricted), `vlan 50` servers (`/26`).

**The trade-off:** per-class VLANs must be *trunked to every floor's switch*, so the physical topology and the logical topology no longer match — which makes the network harder to draw and harder to reason about during an incident. Option (b) would have been beautifully simple to troubleshoot. You're buying containment with comprehensibility, and that's a genuine cost, not a free win.

**The flip condition.** Flip to per-floor (b) if the tenants are independent organizations on each floor — then the floor *is* the trust boundary and geography and trust coincide. Flip to flat (a) only if the site is small enough that ~250 devices produce negligible broadcast load and there is genuinely one trust level (a 20-person startup in one room). And flip *away* from VLANs entirely toward per-port 802.1X authentication with dynamic VLAN assignment once you have enough device churn that manually mapping ports to VLANs becomes the bottleneck — at that point the boundary should follow the device's *identity*, established at connection time, not the wall socket it happens to be plugged into.

**Failure modes:** the inter-VLAN router becomes the choke point for all cross-class traffic, so size it for aggregate rather than per-VLAN throughput. The `vlan 40` building-systems segment will contain devices that cannot be patched and speak unauthenticated protocols — assume they are compromised and design the ACLs accordingly (this is why that segment gets a `/26` and the tightest rules, not because it has few devices). And per §11, ensure spanning tree is running RSTP with a known root bridge and a monitored hop count from any access switch to the core — the metric that would have saved CareGroup.

---

## 13. In production: how this layer is really operated and gotten wrong

**Depth: [CORE] block, covering §1–§12**

### The practices that actually matter

**Segment for blast radius, then for performance.** Every design decision in §12 came from asking "where do I want failure and compromise to stop?" — not from a performance calculation. Performance segmentation (splitting a busy segment) is a nice side effect. Get the ordering right, because if you segment for performance you'll draw the boundaries in the wrong places.

**Never treat "same network" as authentication.** §8's ARP has no authentication; §10's VLANs are configuration, not walls; §12's security groups are the *only* real control inside a VPC and they're one API call from being wrong. The professional position is **zero trust**: every service authenticates every caller regardless of network position, and the network is a *coarse backstop*, not the security model. Day 14's mTLS and Phase 8's authorization work are where this becomes concrete.

**Make configuration reviewable.** Route tables, security groups, and NACLs are code. A console click leaves no diff, no author, and no reviewer. Everything in §12 should exist as Terraform/Bicep/CloudFormation with a pull request. This isn't process theatre — CareGroup's PACS segment was 10 hops from the core because nobody could *see* the accumulated topology.

**Know your invariants and measure them.** Every failure in this note came from an unchecked invariant: STP's 7-hop diameter, MTU consistency across a path, subnet address-space headroom, the presence of a metadata hop-limit setting. Pick the two or three that would actually hurt you and make them assertions in CI or a monitoring check, not tribal knowledge.

### What to monitor, and what each signal means

| Signal | Where you get it | What a bad value tells you |
|---|---|---|
| **Interface errors / CRC errors** | `Get-NetAdapterStatistics` (Win), `ip -s link` (Linux), switch counters | **Physical.** Bad cable, dying transceiver, EMI. Never a software bug. Rising CRC count = go look at hardware. |
| **Discards / drops (in and out)** | same | Buffer exhaustion — you're congested, or a burst exceeded queue depth. §2's queueing delay made visible. |
| **Broadcast/multicast packets per second per host** | switch counters, host-level capture | Broadcast domain too large (§6), or something is storming. Baseline it so you know what abnormal looks like. |
| **Interface utilization, p95 not mean** | switch/cloud metrics | Means hide bursts, and bursts are what fill queues. A 40% mean with a 95% p95 is a network in trouble. |
| **MAC table size and churn** | switch | Approaching table capacity means unknown-unicast flooding is coming. High churn = a loop or a flapping link (§11). |
| **STP topology change events** | switch logs, SNMP traps | Each one is a reconvergence — traffic stopped somewhere. A steady trickle means a flapping link nobody has noticed. |
| **ICMP type 3 code 4 rate** | firewall/router counters | PMTUD in action. A spike means someone's MTU changed. **Zero, in a network with tunnels, may mean you're blocking them** — §9's black hole. |
| **VPC flow logs** | cloud provider | The cloud replacement for a packet capture: accepted and rejected flows per interface. Your primary evidence for "did the security group block it?" and the audit trail for §12②'s egress. |
| **NAT gateway / egress bytes and connection count** | cloud provider | Cost driver *and* exfiltration signal. An unexplained egress spike from a sandbox subnet is exactly what you want alarms on. |

### Mistakes, ordered from beginner to senior

**Beginner.** Confusing bandwidth with latency and expecting a bigger pipe to fix round-trip cost (§1). Assuming a MAC address is a routable destination. Thinking a switch divides broadcast domains (§6). Building one flat network because it works at 10 hosts. Blocking all ICMP "for security" and creating a PMTUD black hole (§9). Believing a private IP address means a service is safe to leave unauthenticated.

**Intermediate.** Adding a "redundant" layer-2 link without understanding spanning tree (§11). Sizing a subnet with no headroom and hitting CIDR exhaustion under autoscale. Enabling jumbo frames on one device in a path and creating intermittent large-transfer failures. Relying on stateless NACLs without ephemeral-port return rules and getting one-directional connectivity. Treating security groups as a to-do list that only grows.

**Senior.** Getting the *boundary placement* wrong — putting router boundaries where the org chart is rather than where the trust and failure boundaries are (§12③). Building a network whose management plane depends on the network it manages, so a network failure is unrecoverable remotely (CareGroup's four days; Meta's 2021 outage on Day 10). Designing for the steady state and not for **convergence time** after a failure — a network that recovers in 50 seconds has a 50-second outage every time anything changes. Allowing an egress allowlist to decay to `*` because no process guards it.

### Scaling behaviour: what breaks first as you grow

- **Broadcast volume grows with host count, and every host pays for every broadcast.** This is the first wall in a flat network, and it hits somewhere in the high hundreds of hosts (§6). Fix: segment.
- **MAC tables are finite** (tens of thousands of entries on typical gear). Exceed it and the switch starts flooding unknown unicast — a graceful-looking degradation that quietly turns your switched network back into a hub. Fix: segment, or bigger gear.
- **Spanning tree's convergence time does not improve with money.** It's a protocol property. Fix: change protocol (RSTP) or change architecture (routed fabric).
- **Routing table size and convergence** replace the above as your limits once you're routed. That's Day 10's territory.
- **In a cloud**, the walls move: you'll hit **subnet CIDR exhaustion**, **security-group rule-count limits per interface**, **route-table entry limits**, and **NAT gateway port exhaustion** (a single NAT gateway has a finite number of simultaneous connections per destination — a service making tens of thousands of concurrent outbound connections to one API will hit it, and the symptom is intermittent connection failures that look like the remote service's fault). *All of these are documented quotas with current values in the provider's docs — look them up rather than trusting a number from a blog, including this one.*

### Cost

Physical networking cost is switch ports, cabling, and the operational time to configure them. Cloud networking cost is subtler and routinely surprises people, so name the three that bite:

1. **NAT gateways** are billed per hour *and* per gigabyte processed. A chatty service pulling container images through NAT can generate a bill larger than the compute it serves. Mitigation: VPC endpoints for provider services (so traffic never traverses NAT), and image caching.
2. **Cross-AZ traffic is charged per gigabyte, in both directions.** A chatty microservice architecture spread across AZs pays for its own conversations. This creates a genuine tension with the availability requirement that you *must* span AZs — the resolution is usually AZ-affinity for chatty paths plus cross-AZ replication only for state.
3. **Egress to the internet** is the expensive direction (ingress is typically free). This is why CDNs (Day 15) are an economic decision as much as a latency one.

---

## 14. Failure modes and common misconceptions, collected

A recap of things already explained above — cross-referenced rather than re-taught.

**Failure modes, with their signature symptom:**

| Failure | Signature symptom | Section |
|---|---|---|
| Layer-2 loop / broadcast storm | everything dies at once, hosts unresponsive at the console, switch CPU pinned | §11 |
| PMTUD black hole | handshake works, small requests fine, **large transfers hang** | §9 |
| MTU mismatch (jumbo frames half-enabled) | intermittent large-packet loss, works between some hosts | §9 |
| Stale ARP entry after failover | traffic keeps going to the dead node for tens of seconds | §8 |
| ARP spoofing | traffic silently intercepted; gateway MAC changes unexpectedly | §8 |
| MAC table exhaustion / thrashing | unicast being flooded; a switched network performing like a hub | §6, §11 |
| Unknown-unicast flooding to a silent host | steady background noise on a segment; a quiet server "leaks" traffic everywhere | §6 |
| Stateless NACL missing ephemeral-port rule | connections work outbound but replies never arrive | §12① |
| Subnet CIDR exhaustion | autoscaling fails to launch instances, with an unhelpful error | §12① |
| Single-AZ subnet | an AZ event takes out a whole tier | §12① |
| Metadata service reachable from sandboxed code | credential theft, then lateral movement | §12② |
| Egress allowlist decayed to `*` | nothing visibly breaks; the control is simply gone | §12② |
| VLAN hopping via native-VLAN misconfiguration | traffic crossing a boundary that "cannot be crossed" | §10 |

**Misconceptions, stated and corrected:**

- **"A switch and a router are basically the same thing."** No: a switch reads only the 14-byte Ethernet header and forwards unchanged; a router reads the IP header, *destroys* the frame, decrements TTL, and builds a new frame. (§7)
- **"The destination MAC is the destination machine's MAC."** Only if the destination is on your own LAN. Otherwise it's your *gateway's* MAC, while the IP destination is the real target. This is the highest-value insight of the day. (§7)
- **"Layers are cleanly separated."** They aren't, by design: TCP's checksum reads IP's addresses, NAT rewrites layer-4 headers, MSS clamping edits handshakes, L7 load balancers terminate connections. Layer purity is a teaching model. (§3, §4, §9)
- **"Adding a redundant cable makes a LAN more reliable."** At layer 2, without spanning tree, it makes it *catastrophically* less reliable, because frames have no TTL. (§11)
- **"ARP is part of IP."** No — EtherType `0x0806`, riding directly on Ethernet as a peer of IP. (§8)
- **"The internet was designed to survive a nuclear war."** Baran's RAND work was about survivability; ARPANET's stated goal was resource sharing. Both facts matter and the popular conflation is wrong. (§2)
- **"MTU is 1500 bytes of Ethernet frame."** It's 1500 bytes of **IP packet**; the frame is up to 1518 (1522 with a VLAN tag). (§5, §9)
- **"More bandwidth makes my API faster."** Only if you're bandwidth-limited. If you're round-trip-limited, bandwidth changes nothing. (§1)
- **"Blocking ICMP improves security."** Blocking ICMP type 3 code 4 breaks IP's required error signalling and produces the black hole in §9. Rate-limit and filter selectively; don't blanket-drop.
- **"A private IP means it's internal, so it's safe."** ARP is unauthenticated, VLANs are configuration, security groups are one mistake from open, and §12②'s whole threat model lives inside a private network. (§8, §10, §12)
- **"Collisions are why my network is slow."** Full-duplex switching eliminated collisions ~25 years ago. If you see collision counters on a modern link, you have a duplex *mismatch* — a configuration bug — not congestion. (§6)

---

## 15. Interview and practice questions

**Warm-up (answer in one or two sentences):**
1. What is the difference between a MAC address and an IP address, and why do you need both?
2. What are the exact sizes of the Ethernet header, the minimum frame, and the maximum frame — and where does each number come from?
3. Which device divides a broadcast domain, and which does not?
4. Why does an Ethernet frame have no TTL, and what is the consequence?

**Real interview questions, with what the interviewer is actually testing:**

5. **"Walk me through what happens when you `ping` a machine on your LAN, versus a machine on the internet — at the frame level."** *Testing:* whether you understand §7's boundary. The answer they're listening for is that the destination **MAC** differs (target's own MAC vs the gateway's) while the destination **IP** is the same in both cases.
6. **"A user reports that our API works fine for small requests but hangs on large uploads. TCP connects fine. Where do you look?"** *Testing:* §9. Say "PMTUD black hole" and explain the mechanism, then propose the diagnostic (`ping -f -l` binary search) and the three fixes in order.
7. **"Someone added a second cable between two switches for redundancy and the whole office went down. Explain."** *Testing:* §11 — no TTL at layer 2, plus the flooding rule, gives exponential growth; and why spanning tree exists.
8. **"Design the network for a 3-tier app so that a compromised web tier can't reach the database."** *Testing:* §12① — and specifically whether you name **two independent controls** (route table *and* security group) rather than just "a firewall rule."
9. **"You're letting an LLM execute generated Python in your infrastructure. What's the first network control you'd add?"** *Testing:* §12② — block the instance metadata service. If you say "a firewall" without naming IMDS, you haven't thought about it.
10. **"Why did the industry move from spanning-tree layer-2 data centres to routed leaf-spine fabrics?"** *Testing:* §11's flip condition — blocked links carry zero traffic, and convergence is slow; routed fabrics use every link via ECMP.
11. **"What's the difference between a stateful security group and a stateless network ACL, and when does the difference bite you?"** *Testing:* §12① — ephemeral-port return rules, and the argument for having both.
12. **"Why is the first request to a service slower than the rest?"** *Testing:* whether you can enumerate the cold-path costs across layers: ARP resolution (§8), DNS (Day 12), TCP handshake (Day 11), TLS handshake (Day 14), connection-pool warm-up, and JIT/cache warm-up. A great answer lists them by layer.

**Practice reps (do these on paper, 15–20 min each):**
13. Given the hex dump in §5, identify by hand, with byte offsets: the destination MAC, the EtherType, the IP TTL, the IP protocol number, and where the payload starts. Then confirm with `peel.py`.
14. Compute the maximum small-frame packets-per-second a 1 Gbit/s link can carry, including preamble, SFD, FCS, and inter-frame gap. (Hint: §5's 84-byte-times figure for a 64-byte frame.)
15. Sketch a broadcast storm's growth: given 1 Gbit/s links and 64-byte frames, how many doublings to saturation, and how long? (§11)
16. Take the §12① diagram and write out every rule — routes, security groups — needed for the app tier to call an external HTTPS API while the DB can reach nothing. Then find the one rule you'd remove if the compliance requirement changed to "egress must be enumerable."

---

## 16. Build: capture, label, and commit

This is the day's deliverable. Do all four steps; the artefact is the annotated screenshot plus your `peel.py` output committed to your repo.

### Step 1 — capture a `ping` and a `curl`

**Windows 11, option A (recommended): Wireshark.** Install it (`winget install WiresharkFoundation.Wireshark`), which also installs the **Npcap** driver — Windows has no raw packet capture without a driver, so this step is not optional. Then:

1. Start Wireshark, pick your active adapter (Wi-Fi or Ethernet — check `Get-NetAdapter` if unsure).
2. Set the capture filter to cut the noise: `icmp or (tcp port 80)`.
3. In PowerShell: `ping 8.8.8.8` then `curl.exe -v http://neverssl.com/` (use `curl.exe`, because bare `curl` in PowerShell is an alias for `Invoke-WebRequest`; and use a plain-HTTP site so the payload is readable — an HTTPS site would show you TLS, which is Day 14).
4. Stop the capture.

**Windows 11, option B (no install): `pktmon`.** It's built in. Run in an **elevated** PowerShell — the syntax below is from [Microsoft's `pktmon start` reference](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/pktmon-start):

```powershell
pktmon filter remove                        # clear any leftovers
pktmon filter add PingFilter -t ICMP
pktmon filter add WebFilter  -t TCP -p 80
pktmon filter list

# --pkt-size 0 = log the WHOLE packet (default is only 128 bytes)
pktmon start --capture --comp nics --pkt-size 0 --file-name C:\caps\d9.etl

ping 8.8.8.8
curl.exe -v http://neverssl.com/

pktmon stop
pktmon etl2pcap C:\caps\d9.etl --out C:\caps\d9.pcapng    # now open in Wireshark
```
A bonus worth knowing: **`pktmon hex2pkt`** will decode a hex-dumped packet for you — a built-in second opinion on your `peel.py` output.

**Linux / macOS equivalent** (`tcpdump` is native here; there is no native `tcpdump` on Windows, which is the honest reason for the two options above):

```bash
# -i = interface, -n = don't resolve names, -X = hex AND ascii, -s 0 = full packet
sudo tcpdump -i any -nn -X -s 0 'icmp or (tcp port 80)' -w d9.pcap
# in another terminal:
ping -c 3 8.8.8.8
curl -v http://neverssl.com/
# then read it back, or open d9.pcap in Wireshark:
tcpdump -nn -X -r d9.pcap | head -60
```
*(Note that Linux's `ping` defaults to a 56-byte payload, so your frame will be 98 bytes rather than Windows' 74, and TTL will be 64 rather than 128. Both are correct — and noticing the difference is itself the exercise.)*

### Step 2 — label the four layers by hand

Pick the HTTP request packet in Wireshark. The detail pane already shows the nested layers — that nesting *is* §4's encapsulation, drawn by someone else. Your job is to prove you can do it without the tool:

1. Right-click the packet → **Copy → …as a Hex Dump**, paste into `frame.txt`.
2. **Before** running anything, on paper: bracket bytes 0–5 (dst MAC), 6–11 (src MAC), 12–13 (EtherType), 14 onward (IP header — read the low nibble of byte 14 to get its length), then the TCP header, then the payload. Write down the TTL, the protocol number, and the two port numbers.
3. Screenshot Wireshark's detail pane with the four layers expanded, and annotate it (any image editor) with those byte ranges.

### Step 3 — verify with `peel.py`

Save the script from §4 and run it against your own capture:

```powershell
python peel.py frame.txt        # Windows
python3 peel.py frame.txt       # Linux/macOS
```

**Definition of done — all five must hold:**
- [ ] `peel.py` prints all four layers for **your own** captured HTTP request, and the IP checksum line ends in `: 0`.
- [ ] Your hand-annotated byte offsets match `peel.py`'s output exactly.
- [ ] You can point at the ARP request in your capture (there almost certainly is one — filter on `arp`) and say what question it asks and which machine answered.
- [ ] For the `curl` packet, you can state whether the destination MAC is the server's or your gateway's, **and justify it from the subnet comparison** (§7).
- [ ] `ping -f -l 1472` succeeds and `-l 1473` fails on your path — or you can explain the number you actually got and why it's lower than 1472 (VPN? overlay?).

### Step 4 — commit

```powershell
git add notes/d9-networks-packets-frames-switches.md builds/d9/
git commit -m "Day 9: packet capture with layers labelled, peel.py frame decoder"
```
Commit the annotated screenshot, `peel.py`, your `frame.txt`, and a short `README.md` with the four numbers you found (your MTU, your gateway's MAC, your TTL, and the TCP source port). Do **not** commit the raw `.pcap` if you captured anything beyond these tests — a capture of your own traffic contains cookies, tokens, and hostnames. Treat packet captures as secrets.

---

# Topic-wide wrap-up

## Glossary

Every term this note defined, alphabetized. This indexes the note; it doesn't re-teach it.

**802.1Q** — the IEEE standard that adds a 4-byte VLAN tag to an Ethernet frame, letting one physical switch carry several logically separate LANs. (§10)

**802.1D** — the IEEE standard for spanning tree, which prevents layer-2 loops by blocking redundant links; the standalone standard was withdrawn in 2011 and its content now lives in 802.1Q. (§11)

**802.3** — the IEEE standard defining Ethernet: the frame format, the 64-byte minimum and 1518-byte maximum, and the physical layers beneath them. (§5)

**Access port** — a switch port that carries a single VLAN, untagged, and faces a host that knows nothing about VLANs. Contrast trunk port. (§10)

**ARP (Address Resolution Protocol)** — the protocol (RFC 826) that discovers which MAC address owns a given IPv4 address on the local network, by broadcasting the question and caching the unicast answer. (§8)

**ARP spoofing** — answering ARP requests dishonestly to redirect a victim's traffic through the attacker; possible because ARP has no authentication of any kind. (§8)

**Bandwidth** — how many bits per second a link can carry; determines how long it takes to push data out, and has no effect on how long the first bit takes to arrive. Contrast latency. (§1)

**BPDU (Bridge Protocol Data Unit)** — the small frames switches exchange to run spanning tree: electing a root bridge and agreeing which ports to block. (§11)

**Broadcast address** — the all-ones MAC address `ff:ff:ff:ff:ff:ff`, meaning every NIC on this LAN must accept the frame. (§5)

**Broadcast domain** — the set of devices that receive a frame sent to the broadcast address. Divided by a router or a VLAN, never by a plain switch. (§6)

**Broadcast storm** — the exponential multiplication of broadcast frames around a layer-2 loop, saturating links and hosts within microseconds because frames have no TTL. (§11)

**Circuit switching** — reserving a dedicated end-to-end path for the duration of a conversation; gives guaranteed bandwidth and no loss, at the cost of paying for idle capacity and losing the whole path if any link fails. (§2)

**Collision domain** — the set of devices whose transmissions can physically interfere. Reduced to a single link by switching, and made impossible by full duplex; effectively a historical concept. (§6)

**Cut-through switching** — forwarding a frame as soon as its destination address has been read, before the rest arrives; lower latency, but it forwards corrupt frames because it hasn't yet seen the checksum. Contrast store-and-forward. (§2)

**Decapsulation** — the receive-side inverse of encapsulation: each layer reads and strips its own header and hands the remainder up. (§4)

**DF (Don't Fragment) bit** — an IPv4 header flag telling routers to drop an oversized packet and report it rather than fragmenting it; the basis of Path MTU Discovery. (§4, §9)

**Egress-restricted subnet** — a subnet whose route table forces all outbound traffic through a single filtering, logging proxy, making egress enumerable and auditable. (§12②)

**Encapsulation** — the mechanism that makes layering real: each layer treats what it's given as opaque bytes and prepends its own header, including a field naming what's inside. (§4)

**End-to-end argument** — the design principle that reliability and state belong at the communicating endpoints rather than inside the network, because the network's components can fail at any time. (§2)

**Ethernet II / DIX framing** — the framing in universal use, where the 2 bytes after the MAC addresses hold an EtherType rather than a length; values ≤1500 are read as a length, ≥1536 as a type. (§5)

**EtherType** — the 2-byte field in an Ethernet header naming what the payload is: `0x0800` IPv4, `0x0806` ARP, `0x86DD` IPv6, `0x8100` a VLAN tag. (§5)

**FCS (Frame Check Sequence)** — the 4-byte CRC-32 at the end of an Ethernet frame; a mismatch causes the frame to be discarded silently, with no report and no retransmission. (§5)

**Flooding** — a switch's response to a frame whose destination it hasn't learned (or that is a broadcast): send it out every port except the one it arrived on. (§6)

**Fragmentation** — splitting an IP packet too large for the next link into pieces reassembled only at the destination; losing any one piece loses the whole packet. Removed from routers in IPv6. (§9)

**Frame** — the layer-2 unit of data: an Ethernet header, a payload, and an FCS. Its life is one link; it is rebuilt at every hop. (§4, §5)

**Gratuitous ARP** — an unsolicited ARP announcement of one's own address mapping, used legitimately for duplicate-address detection and failover, and illegitimately for spoofing. (§8)

**Hourglass / narrow waist** — the internet's structural shape: many application protocols on top, many link technologies below, and exactly one protocol (IP) in the middle — which is why innovation is cheap at the edges and nearly impossible in the middle. (§3)

**Hub (multiport repeater)** — a dumb device that regenerates any incoming signal out every other port; electrically still one shared cable. Superseded by switches. (§6)

**IHL (Internet Header Length)** — the low nibble of the IPv4 header's first byte, giving the header's length in 32-bit words; 5 (= 20 bytes) normally, up to 15 (= 60 bytes) with options. (§4)

**IMDS (Instance Metadata Service)** — the cloud endpoint at `169.254.169.254` that hands an instance its own IAM credentials over unauthenticated HTTP; the highest-value target for any code executing inside your infrastructure. (§12②)

**Inter-frame gap** — the mandatory 96 bit times (12 bytes) of silence a link must observe between frames, so receivers can reset. (§5)

**Internet checksum** — the 16-bit one's-complement folded sum used by IP, TCP, UDP, and ICMP (RFC 1071); recomputing it over a valid header including its own checksum field yields zero. (§4)

**Jumbo frame** — an Ethernet frame with an MTU around 9000 bytes, used on closed high-throughput fabrics; it requires every device in the path to agree, and silently fails if one doesn't. (§9)

**Latency** — how long the first bit takes to arrive, set by distance and the speed of light; unaffected by bandwidth. Contrast bandwidth. (§1)

**Layering** — organizing network functionality so each layer solves one problem, exposes a narrow interface, and trusts its neighbours — enabling independent evolution of applications and physical media. (§3)

**MAC address** — a 48-bit layer-2 identity assigned by a NIC's manufacturer (first 3 bytes = the OUI); flat, non-routable, and increasingly randomized for privacy. (§5)

**MAC address table (CAM table)** — a switch's learned map of MAC address → port, built from source addresses, refreshed on use, and aged out after idle timeout (commonly 300 s). (§6)

**MAC table thrashing** — a switch repeatedly relearning the same MAC on alternating ports because a loop is delivering copies from multiple directions; it also breaks unicast forwarding. (§11)

**MSS (Maximum Segment Size)** — the largest TCP payload a host will accept, advertised during the handshake; typically MTU − 20 − 20 = 1460 for standard Ethernet. (§9)

**MSS clamping** — a router rewriting the MSS value inside TCP handshake packets so segments fit the path; a deliberate layer violation used to work around PMTUD failures. (§9)

**MTU (Maximum Transmission Unit)** — the largest layer-3 packet a link will carry, 1500 bytes for standard Ethernet; a fairness and error-cost compromise chosen in 1980 and effectively unchangeable. (§9)

**NACL (network ACL)** — a **stateless** packet filter applied at a cloud subnet's boundary; because it doesn't track connections, return traffic needs its own explicit rule. Contrast security group. (§12①)

**NAT gateway** — a cloud device giving a private subnet outbound-only internet access; one-directional by construction, billed per hour and per gigabyte, and the reason all your outbound traffic appears from one IP. (§12①)

**NDP (Neighbor Discovery Protocol)** — IPv6's replacement for ARP, using ICMPv6 multicast instead of layer-2 broadcast. (§8)

**NIC (network interface card)** — the hardware boundary between byte buffers in RAM and signals on a medium; exposed to your code through the file-descriptor abstraction. (§5)

**OSI model** — the seven-layer ISO reference model whose protocol suite lost to TCP/IP but whose *vocabulary* ("layer 2," "layer 7") is what practitioners use daily. (§3)

**OUI (Organizationally Unique Identifier)** — the first 3 bytes of a MAC address, a block registered to a manufacturer with the IEEE; how you identify a device's vendor from a capture. (§5)

**Packet** — the layer-3 unit of data: an IP header plus payload. Unlike a frame, it survives the whole journey, with only TTL and the checksum changing per hop. (§4)

**Packet switching** — chopping data into independently addressed chunks forwarded hop by hop with no reservation; vastly more efficient than circuit switching, at the cost of variable delay and loss. (§2)

**PMTUD (Path MTU Discovery)** — sending with DF set and using the resulting ICMP "fragmentation needed" errors to learn the path's smallest MTU (RFC 1191). (§9)

**PMTUD black hole** — the failure that occurs when those ICMP errors are firewalled: the handshake succeeds, small requests work, and large transfers hang forever. (§9)

**Preamble / SFD** — 7 bytes of `0x55` followed by one `0xD5` that let a receiver's clock lock onto the sender's bit timing; physical-layer scaffolding, never visible in a capture. (§5)

**Propagation delay** — time for bits to travel the physical distance, roughly distance ÷ 2×10⁸ m/s; the component of latency you can never engineer away. (§1, §2)

**Proxy ARP** — a router answering ARP on behalf of hosts on other segments, making a remote machine appear local; a legacy crutch that makes the layer-2/3 boundary lie. (§8)

**Queueing delay** — time a packet spends waiting in an output buffer behind others; zero on an idle link, unbounded when congested, and the source of essentially all latency variance. (§2)

**Root bridge** — the switch elected as the reference point for spanning tree; every other switch blocks ports not on its shortest path to the root. (§11)

**RSTP (Rapid Spanning Tree, originally 802.1w)** — spanning tree with explicit neighbour handshakes instead of timers, cutting reconvergence from tens of seconds to under a second. (§11)

**Security group** — a **stateful** packet filter attached to a cloud network interface; return traffic for an allowed connection is automatic, and rules can reference other security groups rather than IP ranges. (§12①)

**Segment** — the layer-4 unit of data: a TCP header plus payload. (§4)

**Slot time** — 512 bit times, the worst-case collision round trip on legacy Ethernet, which is exactly why the minimum frame is 64 bytes. (§5)

**Spanning tree** — Perlman's distributed algorithm (1985) that finds loops in a switched network and blocks enough links to leave exactly one path between any two points. (§11)

**Statistical multiplexing** — sharing a link among many bursty senders on the assumption they rarely peak together; the economic basis of packet switching, and the reason congestion exists. (§2)

**Store-and-forward** — receiving a whole frame, checking it, and only then transmitting it; adds per-hop delay but never propagates corruption. Contrast cut-through. (§2)

**Subnet (cloud)** — the cloud's rename of a broadcast domain; a range of addresses with an associated route table, where the *absence* of a route is the isolation mechanism. (§12)

**Switch (bridge)** — a layer-2 device that learns which MAC is on which port from source addresses and forwards frames to just that port, flooding only when it doesn't know. (§6, §7)

**TCP pseudo-header** — the fabricated 12-byte block of IP addresses, protocol number, and length that TCP's checksum covers in addition to its own segment; a standardized layering violation that detects misdelivery. (§4)

**Trunk port** — a switch port carrying multiple VLANs with 802.1Q tags, used between switches. Contrast access port. (§10)

**TTL (Time To Live)** — an 8-bit IPv4 hop counter decremented by every router, killing looping packets; 128 initially on Windows, 64 on Linux/macOS. **Ethernet frames have no equivalent.** (§4, §7, §11)

**Unknown unicast** — a frame whose destination MAC isn't in the switch's table, which is therefore flooded; caused by table aging, and why silent hosts generate segment-wide noise. (§6)

**VLAN (Virtual LAN)** — a logical broadcast domain defined by switch configuration rather than physical cabling; a good boundary against accident, a mediocre one against an attacker. (§10)

**VLAN hopping** — crossing a VLAN boundary by exploiting native-VLAN or trunk misconfiguration, usually via double-tagging. (§10)

**VPC (Virtual Private Cloud)** — an isolated cloud address space containing subnets, route tables, and gateways; the cloud's rename of "a routed site you administer." (§12)

**Zero trust** — the position that every service authenticates every caller regardless of network position, because network location is not an identity. (§13)

---

## Cheat sheet

```
THE NUMBERS
  Ethernet header      14 bytes (6 dst + 6 src + 2 ethertype)  -- always, fixed
  Ethernet FCS          4 bytes (CRC-32, at the END)
  min payload          46 bytes  -> min frame  64 bytes  (= 512 bit slot time)
  max payload        1500 bytes  -> max frame 1518 bytes  (1522 with a VLAN tag)
  preamble+SFD          8 bytes;  inter-frame gap 12 bytes (96 bit times)
  IP header            20 bytes normally (IHL=5); up to 60 with options
  TCP header           20 bytes normally (data offset=5); up to 60 with options
  MTU                  1500 = max IP PACKET (not frame).  MSS = 1500-20-20 = 1460
  ping -f -l            1472 = 1500 - 20 (IP) - 8 (ICMP)
  jumbo                9000 (closed fabrics only; all-or-nothing)
  STP defaults         hello 2s, max age 20s, fwd delay 15s -> 30-50s converge
  STP bridge diameter  7  (baked into those timers -- CareGroup exceeded it at 10)
  MAC table aging      ~300 s typical default
  initial TTL          Windows 128, Linux/macOS 64

THE HEADERS, IN ORDER, ON THE WIRE
  [ Ethernet 14B ][ IP 20B ][ TCP 20B ][ ... payload ... ][ FCS 4B ]
    dst MAC 0-5     45 = v4/IHL5        sport/dport
    src MAC 6-11    total length        seq / ack
    type   12-13    TTL, proto          offset+flags, window, checksum
                    src IP / dst IP

  "WHAT'S INSIDE ME" FIELDS -- your map when reading a dump:
    EtherType: 0800 IPv4 | 0806 ARP | 86DD IPv6 | 8100 VLAN
    IP proto:     1 ICMP |    6 TCP |   17 UDP

THE ONE BOUNDARY
  L2 / switch : destination MAC -> a port.   Frame REBUILT every hop.
                Floods what it doesn't know. Does NOT divide broadcast domains.
                NO TTL -> a loop is fatal.
  L3 / router : destination IP  -> a next hop. Packet SURVIVES end to end.
                Drops what it doesn't know.  DIVIDES broadcast domains.
                TTL -> a loop is self-limiting.

  SAME LAN:  dst MAC = the target's own MAC
  OFF LAN:   dst MAC = the GATEWAY's MAC, dst IP = still the real target
             ^^^^^^^^ the single most important fact of Day 9

CLOUD TRANSLATION
  broadcast domain -> subnet      | router       -> route table
  routed site      -> VPC/VNet    | host firewall-> security group (STATEFUL)
  router ACL       -> network ACL (STATELESS: allow return traffic explicitly!)
  no route to IGW  -> isolation   | NAT gateway  -> outbound only

FIRST FIVE DEBUGGING MOVES
  1. ping the default gateway          -> is L2 working at all?
  2. arp -a / Get-NetNeighbor          -> Incomplete for the gateway = L2 problem
  3. ping something beyond it          -> is L3 working?
  4. ping -f -l 1472 <dest>            -> MTU / PMTUD black hole?
  5. capture it                        -> stop guessing
```

## Build this (if you want more than §16)

**A loop simulator, no hardware required.** Write a Python script that models N switches with a given adjacency list, each implementing exactly the three rules from §6 (learn, forward-or-flood, age). Inject one broadcast frame. Count frames in flight per "round" and print the doubling curve. Then add a spanning-tree pass that blocks ports and re-run to show the count stays at N−1.

**Definition of done:** without STP, your frame count grows as 2ⁿ and you can print the round at which it exceeds a 1 Gbit/s link's capacity (using §11's 0.672 µs per 64-byte frame). With STP enabled, the same injection terminates. Commit both outputs side by side. This is the cheapest way to *feel* §11 without breaking a real network — and it's the exercise that makes the CareGroup case study stop being a story and start being arithmetic.

## Active recall & self-test

Answer from memory, out loud, before checking:

1. Why did packet switching beat circuit switching, and what did it give up to win?
2. State the ARPANET myth and the accurate version. Who gets credit for what?
3. Draw the four nested headers of an HTTP request over TCP over IP over Ethernet, and name the "what's inside me" field at each layer.
4. A frame arrives at a router. List, in order, the five things the router does to it.
5. Your laptop is `192.168.1.42/24`. Give the destination MAC for a packet to `192.168.1.9` and for a packet to `142.250.180.14`. Explain the difference in one sentence.
6. Where do 64, 1500, and 1518 come from? Which of them is the MTU?
7. What does a switch do with a frame whose destination MAC it has never seen? What does a router do with a packet whose destination network it has never seen? Why are these opposite?
8. Explain a broadcast storm's growth rate and why the loop can't self-heal. What single missing header field is responsible?
9. Name the symptom triad of a PMTUD black hole, and the three fixes in order of preference.
10. In §12①'s design, name the two independent controls preventing internet→database access, and say why one alone is insufficient.
11. Name the first network control you'd add to a sandbox running LLM-generated code, and the specific attack it stops.
12. What did CareGroup's engineers change to fix the outage, and which single property of the change was the actual fix?

**60-second teach-back.** Explain to someone with no background: *"What happens, physically and layer by layer, when I type a command on my laptop and a server in another country answers?"* Use one analogy, name the frame/packet/segment distinction, and land the point that the outer envelope is rebuilt at every hop while the inner one survives. If you can't do it in 60 seconds without notes, reread §4 and §7 — those two are the load-bearing ones.

## Spaced-repetition flashcards

| Q | A |
|---|---|
| Ethernet header size? | 14 bytes: 6 dst + 6 src + 2 EtherType. Fixed, always. |
| Min / max Ethernet frame? | 64 / 1518 bytes (1522 with a VLAN tag). |
| Why is the minimum 64 bytes? | 512 bit times = the legacy collision slot time — the worst-case round trip on the longest legal cable. |
| MTU 1500 = 1500 bytes of *what*? | The IP **packet** (header included). Not the frame. |
| MSS on standard Ethernet? | 1460 = 1500 − 20 (IP) − 20 (TCP). |
| `ping -f -l` size for a 1500 MTU? | 1472 = 1500 − 20 − 8. |
| Which device divides a broadcast domain? | A router (or a VLAN). **Never** a plain switch. |
| Switch behaviour on an unknown destination MAC? | Flood out every port except the ingress port. |
| Router behaviour on an unknown destination network? | Drop, and send back an ICMP error. |
| Destination MAC when sending off-LAN? | The **default gateway's** MAC; the destination IP stays the real target. |
| What changes in an IP packet at each hop? | TTL (decremented) and the header checksum (recomputed). Nothing else — unless NAT. |
| Why is a layer-2 loop catastrophic but a layer-3 loop isn't? | Frames have no TTL; IP packets do. |
| EtherType for ARP? | `0x0806` — ARP rides directly on Ethernet, not inside IP. |
| STP default reconvergence time? | 30–50 s for classic 802.1D; under 1 s for RSTP. |
| STP's assumed maximum bridge diameter? | 7. It's baked into the default timers. |
| PMTUD black hole symptom? | Handshake fine, small requests fine, **large transfers hang**. |
| Security group vs network ACL? | SG = stateful, on the interface. NACL = stateless, on the subnet — return traffic needs its own rule. |
| Highest-value network control for a code-exec sandbox? | Block the instance metadata service (IMDSv2 + hop limit 1). |
| Initial TTL as an OS fingerprint? | 128 = Windows, 64 = Linux/macOS. |
| Was ARPANET built to survive a nuclear war? | No. Baran's RAND work (1964) was survivability-driven; ARPANET's goal was resource sharing. |

## Primary sources

Verify against these, not against blog posts (including this note).

**Standards and RFCs**
- [RFC 1122 — Requirements for Internet Hosts: Communication Layers](https://www.rfc-editor.org/rfc/rfc1122.html) (Oct 1989) — the normative four-layer TCP/IP model.
- [RFC 791 — Internet Protocol](https://www.rfc-editor.org/rfc/rfc791.html) (Sept 1981) — the IPv4 header, field by field, including fragmentation.
- [RFC 826 — An Ethernet Address Resolution Protocol](https://www.rfc-editor.org/rfc/rfc826.html) (Plummer, Nov 1982) — ARP. Short enough to read in full; do.
- [RFC 9293 — Transmission Control Protocol](https://www.rfc-editor.org/rfc/rfc9293.html) (Aug 2022) — the current TCP spec, which obsoletes RFC 793 and consolidates decades of piecemeal updates. **Use this, not 793.** (Day 11.)
- [RFC 1071 — Computing the Internet Checksum](https://www.rfc-editor.org/rfc/rfc1071.html) — the algorithm in `peel.py`.
- [RFC 1191 — Path MTU Discovery](https://www.rfc-editor.org/rfc/rfc1191.html) — and RFC 4821 for the packetization-layer variant that survives ICMP blocking.
- [RFC 5737 — IPv4 Address Blocks Reserved for Documentation](https://www.rfc-editor.org/rfc/rfc5737.html) — why `203.0.113.10` is safe to use in examples.
- [IEEE 802.3 working group](https://www.ieee802.org/3/) — Ethernet; the published standard is the authority for frame format and timing constants.
- [IEEE 802.1 working group](https://www.ieee802.org/1/) — bridging, VLANs (802.1Q), spanning tree. *Note that standalone 802.1D was withdrawn in 2011; its content lives in 802.1Q now.*
- [IEEE Registration Authority — EtherType tutorial](https://standards.ieee.org/wp-content/uploads/import/documents/tutorials/ethertype.pdf) and the [OUI registry](https://standards-oui.ieee.org/oui/oui.txt) — look up vendors from MAC prefixes here rather than trusting a lookup site.

**Papers**
- Paul Baran, *On Distributed Communications: I. Introduction to Distributed Communications Networks*, RAND RM-3420, 1964 — [rand.org](https://www.rand.org/pubs/research_memoranda/RM3420.html). The survivability argument, quantified.
- Vinton Cerf & Robert Kahn, *A Protocol for Packet Network Intercommunication*, IEEE Transactions on Communications COM-22(5), May 1974, pp. 637–648 — [PDF](https://www.cs.princeton.edu/courses/archive/fall06/cos561/papers/cerf74.pdf). The paper that became TCP/IP.
- Robert Metcalfe & David Boggs, *Ethernet: Distributed Packet Switching for Local Computer Networks*, CACM 19(7), July 1976, pp. 395–404 — [ACM DL](https://dl.acm.org/doi/10.1145/360248.360253).
- Radia Perlman, *An Algorithm for Distributed Computation of a Spanning Tree in an Extended LAN*, SIGCOMM '85, pp. 44–53 — [ACM DL](https://dl.acm.org/doi/10.1145/319056.319004).
- [Internet Society, *A Brief History of the Internet*](https://www.internetsociety.org/internet/history-internet/brief-history-internet/) — co-authored by the people who built it; the reference for the resource-sharing goal.

**Incident and operational documentation**
- Scott Berinato, *All Systems Down*, CIO Magazine, Feb 2003 — [Computerworld reprint](https://www.computerworld.com/article/1346623/all-systems-down.html) — and [CIO's Halamka interview](https://www.cio.com/article/2440210/halamka-on-beth-israel-s-health-care-it-disaster.html). The CareGroup outage.
- [Krebs on Security, *What We Can Learn from the Capital One Hack*](https://krebsonsecurity.com/2019/08/what-we-can-learn-from-the-capital-one-hack/) — the metadata-service escalation path.
- [Cisco: Understand and Tune Spanning Tree Protocol Timers](https://www.cisco.com/c/en/us/support/docs/lan-switching/spanning-tree-protocol/19120-122.html) — where the 7-hop diameter arithmetic comes from.
- [Microsoft: pktmon syntax](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon-syntax) and [pktmon start](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/pktmon-start) — the Windows capture commands in §16.
- [AWS: Use the Instance Metadata Service](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html) — IMDSv2 and the hop limit. *Cloud docs drift fastest of anything cited here; re-check before relying on a default.*

**Books, for the phase**
- Ilya Grigorik, *High Performance Browser Networking* (free at [hpbn.co](https://hpbn.co/)) — the best free treatment of latency vs bandwidth and everything above it.
- Julia Evans' networking zines ([wizardzines.com](https://wizardzines.com/)) — the fastest route to intuition for tcpdump and packet reading.

## Layered explanations

**10 seconds.** A network chops your data into small addressed chunks and wraps each one in layers of envelopes. The outer envelope only reaches the next machine and is thrown away and rebuilt at every hop; the inner one carries the real destination the whole way. Switches read the outer one; routers read the inner one.

**1 minute.** Circuit switching reserved a whole path per conversation, which is great for voice and ruinous for bursty computer traffic — so the internet uses packet switching: independent addressed chunks, no reservation, forwarded hop by hop, which is roughly five times more efficient and pays for it with variable delay and packet loss. Because that network can lose any node at any time, reliability was pushed out of the network into the endpoints — that's why TCP lives in your OS and not in the routers. Data is built up by *encapsulation*: your HTTP text goes inside a TCP segment (which names the destination *program* via a port), inside an IP packet (which names the destination *machine*), inside an Ethernet frame (which names only the *next hop*). A switch reads the frame's MAC address and forwards it within one LAN, learning as it goes. A router reads the IP address, throws the frame away, builds a new one, and decrements the TTL. That boundary is where broadcast domains end, where failure is contained, and where you can enforce policy — which is why every cloud network diagram is a set of subnets with route tables between them.

**5 minutes.** Everything above, plus the mechanisms. Ethernet's frame is a fixed 14-byte header (destination first, so cut-through switches can decide after six bytes), a payload of 46–1500 bytes, and a 4-byte CRC that causes silent discard on mismatch — layer 2 detects corruption and never reports or repairs it, which is the end-to-end argument in one design decision. The 64-byte minimum is the collision slot time (512 bit times) frozen into a format thirty years after collisions stopped happening; the 1500-byte maximum is a 1980 fairness compromise that is now unchangeable, and it propagates upward as TCP's 1460-byte MSS and downward into every VPN's reduced payload. A switch learns MAC-to-port from source addresses, floods what it doesn't know, and ages entries out after ~300 seconds; it does *not* divide broadcast domains, so broadcast volume and layer-2 fault domains grow with host count until you insert a router. Because Ethernet frames have no TTL, a physical loop turns one broadcast frame into 2ⁿ copies and saturates a gigabit link in roughly 14 microseconds — which is why spanning tree exists, why its default timers assume a 7-switch diameter, and why a Boston teaching hospital closed its emergency department for four hours in November 2002 when its PACS segment sat 10 hops from the core. ARP glues layer 3 to layer 2 by broadcasting "who has this IP" and caching the unicast answer, with no authentication of any kind — which is why "same network" is not an identity and why zero trust replaced perimeter thinking. MTU mismatches produce the black hole where handshakes succeed and large transfers hang forever, because someone blocked the ICMP message that was supposed to explain the problem. In the cloud all of this is renamed: broadcast domains become subnets, routers become route tables, and isolation becomes the *absence* of a route — which is what lets you build a three-tier design where reaching the database from the internet requires defeating two independent controls, and an egress-restricted sandbox where LLM-generated code cannot read your instance credentials or talk to anything not on an allowlist.

**Expert summary.** Layer 2 is a single fault domain with no hop counter, so its failure modes are unbounded and its scaling limit is broadcast volume; layer 3 is a hop-limited, drop-on-unknown, per-hop-independent forwarding plane whose scaling limits are table size and convergence time. Every architectural decision in this note is a choice about where to put the transition between those two regimes, and the correct criterion is blast radius, not topology or org chart. The internet's hourglass — arbitrary applications above, arbitrary media below, exactly one protocol at the waist — is what makes edge innovation free and waist evolution nearly impossible; IPv6's thirty-year deployment is the empirical proof. Encapsulation is the mechanism that makes layering real, and it is systematically violated wherever performance or correctness demands it (TCP's pseudo-header, NAT, MSS clamping, L7 termination), so "which layer is lying to me" is the operative debugging question rather than "which layer is broken." Finally: the properties this design bought — survive node loss, keep no per-flow state in the middle, put reliability at the endpoints — came from Baran's 1964 survivability constraint and Cerf and Kahn's 1974 split, and they are the same properties that make the network an untrustworthy substrate for anything that assumes the network is a security boundary. Which is precisely the assumption you must not make when you decide to execute a language model's output inside it.
