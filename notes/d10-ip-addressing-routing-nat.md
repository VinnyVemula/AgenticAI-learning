# Day 10 — IP: Addressing, Routing, TTL, and NAT

> **Framing.** Day 9 got a frame from one NIC to another NIC on the *same* wire. That is a solved problem, and it is also a nearly useless one: the machine you actually want to talk to is almost never on your wire. Day 10 is the layer that makes "not on my wire" a non-problem — a single global naming scheme for interfaces (IP addresses), a rule for splitting that scheme into administrative chunks (CIDR), a forwarding algorithm so decentralised that no participant knows the full path (hop-by-hop forwarding + longest-prefix match), a suicide timer that stops mistakes from becoming permanent (TTL), a translation hack that let a 32-bit address space survive an 8-billion-device internet (NAT), and the trust-based gossip protocol that glues 79,000 independent networks together (BGP).
>
> This topic is almost entirely backend/systems, and I am going to teach it that way. There is exactly one place where an agentic system changes the engineering, and it is not a stretch: **an agent that calls third-party LLM APIs from inside a private network inherits a NAT egress design, and a *stable* egress IP is the thing that makes provider-side IP allowlisting possible at all.** That belongs inside the NAT section, where the mechanism lives, and you will find it there. Everywhere else, this note stays silent about agents, because there is nothing honest to say.

---

## 0. What Day 9 already gave you, and where Day 10 picks up

Read this section once; it is the map, and it exists so that you never wonder "wasn't this already covered?"

`notes/d9-networks-packets-frames-switches.md` established:

- **Packet switching and the four delays** (transmission, propagation, queuing, processing). Day 10 will keep referring to these — every millisecond you will read in a `tracert` output is a sum of those four, and if you can't name them, go back.
- **Layering and the narrow waist.** Day 9 explained *why* IP is the hourglass's neck. Day 10 explains *what is actually in* that neck.
- **Encapsulation.** An IP packet travels inside an Ethernet frame's payload. Day 10 lives one envelope in.
- **MAC addresses, switches, and ARP.** Layer-2 identity, layer-2 forwarding, and the address-resolution step. Day 10 needs ARP only as a handoff: "the routing table told me the next hop is `192.168.29.1`; ARP turns that into a MAC." Day 9 owns the mechanism.
- **The L2/L3 boundary** — what makes a router a router. Day 10 opens the box Day 9 labelled.
- **MTU, fragmentation, PMTUD, and the PMTUD black hole.** Day 9 taught this from the *link's* point of view: links have a maximum frame size, oversize packets must be split or rejected, and PMTUD discovers the smallest MTU on a path. **Section 7 of this note does not re-teach any of that.** It adds only the IP-header-level machinery Day 9 named but did not open: the Identification / DF / MF / Fragment Offset fields worked through with real arithmetic, who splits and who reassembles, the reassembly timer, why IPv6 deleted router fragmentation, and fragmentation as a firewall-evasion surface. Section 7 says this again at the top so you can't miss it.
- **VPC / subnet / security group / NACL basics** and a 3-tier segmentation design. That was the *trust-boundary* reasoning: which tier may talk to which. Day 10's Build is the *arithmetic* of the same picture: the actual CIDR blocks, the actual host counts, the cloud-reserved addresses. Deliberately complementary — I will not re-argue security groups here.

Two forward handoffs, so you know these black boxes are shut **on purpose**:

- **Day 11 owns TCP and UDP.** This note needs ports (as opaque 16-bit numbers) for NAT, and it needs "UDP is connectionless" for traceroute. Handshakes, sequence numbers, congestion control, sockets: Day 11.
- **Day 12 owns DNS.** This note needs "DNS turns a name into an IP, and if DNS is unreachable nothing works" for the Meta case study, and "DNS can hand out different answers to different clients" for multi-region failover. The resolution chain, record types, and DNS TTL vs IP TTL: Day 12.

---

## 1. Why IP exists at all: the internetworking problem

**Depth: [CORE]**

### Intuition — the problem before the solution

By 1973 there were several working packet networks and they had nothing in common. ARPANET spoke one protocol over leased telephone lines. Packet Radio Net spoke another over radio, with a different maximum packet size and much worse loss. SATNET went over satellite, with a quarter-second propagation delay baked into physics. Each network had its own addressing scheme, its own packet format, its own idea of how big a packet may be, its own reliability guarantees.

Now suppose you want a host on ARPANET to talk to a host on Packet Radio Net. There are two obvious plans.

**Plan A: make everyone speak the same protocol.** Rip out the radio network's addressing and packet format and replace it with ARPANET's. This is technically clean and politically impossible — the radio network exists precisely *because* radio needs different engineering, and the two networks are owned by different people who did not agree to be standardised. Every future network technology would also have to be retrofitted. This plan dies on contact with reality.

**Plan B: invent one more layer.** Leave every network exactly as it is. Define a new, *universal* packet format and a new, *universal* address space that mean the same thing everywhere. Then, at the boundary between two networks, put a box whose only job is: unwrap the local envelope, read the universal address, decide which network to send it into next, wrap it in *that* network's envelope, send it. That box is a **router** (originally, and in RFCs still, a "gateway"). The universal packet is an **IP datagram**. The universal address is an **IP address**.

That is the whole idea, and it is Cerf and Kahn's 1974 paper, later split into RFC 791 (IP) and RFC 793 (TCP). The reason it worked is that it demanded almost nothing of the underlying networks. To carry IP, a network technology needs to be able to do exactly one thing: take a blob of bytes and try to deliver it to a neighbour. Not reliably. Not in order. Not without duplication. Just *try*.

**Why so weak a requirement?** Because every guarantee IP demanded from the layer below would be a guarantee some future network technology couldn't provide, and that technology would then be excluded from the internet. Asking for almost nothing is what made "the internet runs over anything, including carrier pigeons" a joke that is technically true (RFC 1149, and yes, it was actually implemented once).

### The IP service model, stated precisely

IP promises you three things and refuses four.

It promises: (1) a **global address space**, so an address means the same thing to every router on earth; (2) a **common packet format**, so every router can parse the header without knowing anything about the sender; (3) **best-effort delivery**, meaning it will make a genuine attempt.

It refuses: (1) **reliability** — packets may be silently dropped, and a router that runs out of buffer *will* drop yours without telling anyone; (2) **ordering** — packet 5 may arrive before packet 3, because they can take different paths; (3) **duplicate suppression** — a retransmission somewhere below can deliver the same packet twice; (4) **connection state** — IP keeps *no* memory of you between packets. Each datagram is judged entirely on its own header.

That last refusal deserves a beat, because it is the one people find hardest to internalise. When you `curl https://example.com`, you feel like you opened a channel. You did not. You handed the network a few dozen individually-addressed, individually-routed, stateless envelopes, and TCP (Day 11) built the *illusion* of a channel on top by numbering them and asking for the missing ones again.

### Analogy — the postal system, and exactly where it breaks

Think of IP as the international postal system, and each packet as a postcard.

You write a destination address in a format every post office understands. You drop it in a box. No single postal worker knows the full route from your street to a house in Osaka. Your local sorting office knows only "anything not local goes to the regional hub." The regional hub knows "anything for Asia goes to the international facility." Nobody plans the trip; each stop makes one local decision and passes the postcard on. Postcards are cheap, unacknowledged, and occasionally lost. Two postcards mailed together may arrive days apart. If you need certainty, you number your postcards and ask your friend to tell you which numbers arrived — that is TCP, layered on top by *you*, not provided by the post.

**Where the analogy breaks — three ways, and each one matters.**

1. **The post office reads a street address hierarchically and semantically ("Japan" → "Osaka" → street). Routers do not read semantics; they do arithmetic on bits.** A router does not know that `202.232.2.180` is in Japan. It knows that `202.232.2.180` matches an entry in a table, and that entry has a next hop. The hierarchy in IP is *only* the prefix hierarchy, and it is only as geographic as whoever assigned the addresses chose to make it. This is why you cannot reliably geolocate an IP from the number itself.
2. **The post office won't destroy your postcard for taking too long. IP will.** Every IP packet carries a hop counter (TTL, §5) that guarantees a mis-routed packet dies rather than circulating forever. There is no postal equivalent, and it exists because a routing mistake in a decentralised system is not an "if."
3. **The post office charges you and therefore tracks your item. IP is free at the point of forwarding and tracks nothing.** A router forwards your packet and immediately forgets it existed. This is why network debugging is hard: there is no receipt, no per-packet log, nobody to ask. Sections 5 and 6 exist because we had to *invent* a way to interrogate a system that keeps no records.

### Worked example — one packet, four networks, four different envelopes

Here is the "any network to any network" claim made concrete. This is a real, verified trace from the machine this note was written on (Windows 11, Wi-Fi, Mumbai/India, captured 2026-08-11) to a web server in Japan. I am showing only the *envelope changes*; the full hop list is in §6.

```
Your laptop                                         www.iij.ad.jp
192.168.29.239                                      202.232.2.180
     |                                                    ^
     |  [ Wi-Fi frame | IP: 192.168.29.239 -> 202.232.2.180 | TCP ... ]
     v                                                    |
 home router (192.168.29.1)  <-- rewrites the IP SOURCE (NAT, §9)
     |
     |  [ DOCSIS/PON frame | IP: 49.43.x.x -> 202.232.2.180 | TCP ... ]
     v
 ISP access network (hops 2-7: 10.50.136.1, 172.31.2.18, 192.168.59.122, ...)
     |
     |  [ MPLS-labelled frame | IP: 49.43.x.x -> 202.232.2.180 | TCP ... ]
     v
 ISP core -> Singapore (hop 12: sng-b7-link.ip.twelve99.net)
     |
     |  [ 100G Ethernet frame | IP: 49.43.x.x -> 202.232.2.180 | TCP ... ]
     v
 IIJ backbone: Singapore -> Tokyo -> Osaka (hops 15-18)
     |
     v
 www.iij.ad.jp
```

Read the constant and the variables. **The outer envelope changed at least four times** — Wi-Fi, the ISP's access technology, MPLS-labelled Ethernet in the core, long-haul Ethernet across the Pacific. Different framing, different MACs at every single hop, different maximum sizes, different physics. **The IP header's destination address never changed once.** That invariance is the entire product IP sells. (The *source* address did change exactly once, at the home router — that is NAT, and §9 is about why.)

### Under the hood — the IPv4 header, and reading it as an engineering document

Day 9 showed you this header in its decoder. Here it is again not to re-teach the layout, but because Day 10 needs three of its fields in detail (TTL in §5, Identification/Flags/Fragment Offset in §7) and because the *choice* of fields tells you what the designers thought was expensive.

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version|  IHL  |Type of Service|          Total Length         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Identification        |Flags|      Fragment Offset     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Time to Live |    Protocol   |         Header Checksum       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Source Address                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Destination Address                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options (0-40 bytes)         |   Padding   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

Field widths, verbatim from RFC 791 §3.1: Version 4 bits, IHL 4 bits, Type of Service 8, Total Length 16, Identification 16, Flags 3, Fragment Offset 13, Time to Live 8, Protocol 8, Header Checksum 16, Source Address 32, Destination Address 32.

Three observations that are worth more than the layout itself.

**The whole mandatory header is 20 bytes.** Twenty. On a link that can carry 1500 bytes, that is 1.3% overhead. This is not an accident; it is the design constraint. Every field that isn't there was left out because a router has to parse this header for *every packet it ever sees*, and in 1981 that was expensive. (Today it is expensive again, for a different reason: at 100 Gbps a router gets about 6.7 nanoseconds per minimum-size packet.)

**`Total Length` is 16 bits, so the theoretical maximum IP datagram is 65,535 bytes.** You will essentially never see one, because the link below almost always caps you at 1500 (§7).

**There is a `Header Checksum`, and it covers only the header — not the payload.** IP will happily deliver a datagram whose *data* has been corrupted, because catching that is somebody else's job (TCP's checksum, Day 11). And because TTL changes at every hop, the header checksum must be **recomputed at every hop**, which was a real per-packet cost. IPv6 deleted the header checksum entirely for exactly this reason (§10).

**Where I am deliberately stopping:** I am not opening `Type of Service` / DSCP (QoS marking) or IP `Options` (source routing, record route). Options are essentially dead on the public internet — most routers drop or slow-path packets carrying them, which is why the "record route" option never replaced traceroute. Treat both as black boxes unless a project forces you deeper.

---

## 2. What an IP address actually is

**Depth: [CORE]**

### Intuition — 32 bits with a dot problem

An IPv4 address is a 32-bit unsigned integer. That is the entire truth. `192.168.29.239` is not "four numbers"; it is one number, 3232243695, which humans refuse to read, so we chop the 32 bits into four 8-bit groups, write each group in decimal, and separate them with dots. This is called **dotted-decimal** or **dotted-quad** notation and it is a *display format*, exactly like writing a phone number with dashes.

Internalising "it is one integer" fixes three-quarters of the confusion beginners have with subnetting, because it makes the next fact obvious: since it is one number, ranges of addresses are just ranges of integers, and "is this address in that network?" is just "is this integer in that range?"

Let me do the conversion by hand once, fully, because you should be able to do it without a tool.

**Worked example — `198.51.100.77` to binary and back.** (`198.51.100.0/24` is RFC 5737 documentation space, reserved precisely so examples like this can never collide with a real network.)

Take each octet and express it in the 8 place-values 128, 64, 32, 16, 8, 4, 2, 1. The method is greedy: go left to right, and at each place-value ask "does it fit in what's left?"

```
198 : 128 fits (remainder 70) -> 1
       64 fits (remainder  6) -> 1
       32 no                  -> 0
       16 no                  -> 0
        8 no                  -> 0
        4 fits (remainder  2) -> 1
        2 fits (remainder  0) -> 1
        1 no                  -> 0      => 11000110
 51 : 128 no, 64 no, 32 fits (19), 16 fits (3), 8 no, 4 no, 2 fits (1), 1 fits (0)
                                        => 00110011
100 : 64 fits (36), 32 fits (4), 4 fits (0)
                                        => 01100100
 77 : 64 fits (13), 8 fits (5), 4 fits (1), 1 fits (0)
                                        => 01001101

198.51.100.77 = 11000110 00110011 01100100 01001101
```

Verified by computation (`ipaddress` + a binary formatter, output in §3's runnable block): `11000110.00110011.01100100.01001101`. Correct.

Going the other way is easier: sum the place-values of the 1 bits. `01001101` = 64+8+4+1 = 77.

**Practical shortcut worth memorising.** The eight place-values 128/64/32/16/8/4/2/1 and their running totals from the left — 128, 192, 224, 240, 248, 252, 254, 255 — are the only eight non-zero octet values a subnet mask can ever contain. Learn that list and mask arithmetic becomes recall instead of computation. (§3 explains why.)

### The misconception that costs people real debugging hours

**An IP address does not identify a machine. It identifies a *network interface* — and more precisely, an interface's attachment to a particular network.**

This is not pedantry, and here is the proof, from this machine's actual configuration:

```
Wireless LAN adapter Wi-Fi:
   IPv4 Address. . . . . . . . . . . : 192.168.29.239
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 192.168.29.1

Ethernet adapter vEthernet (Default Switch):     <- Hyper-V virtual switch
   IPv4 Address. . . . . . . . . . . : 172.18.96.1
   Subnet Mask . . . . . . . . . . . : 255.255.240.0
   Default Gateway . . . . . . . . . :              <- none; it isn't a path anywhere
```

One laptop. Two IPv4 addresses, in two unrelated networks, plus `127.0.0.1` on the loopback interface, plus (if I started a container) a Docker bridge address. "The IP of this machine" is a question with no single answer. A router with 40 ports has 40 IP addresses. A cloud load balancer has one address per availability zone.

Three consequences you will meet in production:

- **Binding matters.** A server that binds to `192.168.29.239:8000` is unreachable from a container on `172.18.96.0/20`, even though both are "on this machine." Binding to `0.0.0.0` means "every address on every interface," which is why tutorials tell you to do it and why security reviews tell you not to.
- **Multi-homed hosts pick a source address.** When your machine sends a packet, the *routing table* decides which interface it leaves by, and the source address is then that interface's address. You do not choose it; the route does. This is why a VPN can silently change the source IP your outbound API calls appear from.
- **Which is why the answer to "what is my IP?" depends entirely on who is asking.** From inside: `192.168.29.239`. From the internet: `49.43.x.x` (measured live; redacted here). Two different, both-correct answers, and §9 is the reason.

### The two-part structure: network bits and host bits

Here is the design decision that makes the whole internet scalable, and it is worth stating as a decision with a rejected alternative, because it is the single most important trade-off in this note.

**The alternative that lost: flat addressing.** Give every interface a globally unique number with no internal structure — exactly like MAC addresses (Day 9). It is simple and it needs no coordination beyond uniqueness. Why did it lose? Because a router's job is to map "destination address" → "next hop," and with flat addressing that map has *one entry per interface on earth*. Day 9's learning switch does this and it works fine for 200 devices in an office. At internet scale you would need billions of table entries, updated continuously, in every router. It is not "slow"; it is arithmetically impossible.

**The decision: split the address into a prefix that says *which network* and a suffix that says *which interface within that network*.** Now a router in Tokyo does not need an entry for your laptop. It needs one entry for your ISP's whole address block. Everything inside that block shares one table row. That collapse — from "one row per host" to "one row per network" — is the only reason the routing table is ~1.07 million entries (measured: 1,071,851 active IPv4 BGP entries, AS65000 report, 11 Aug 2026) instead of ~20 billion.

**The trade-off, stated honestly.** Flat addressing gives you *portability for free*: your address is yours, and it means the same thing wherever you plug in. Hierarchical addressing does not. Your IP address encodes *where you are attached*, so moving networks means changing address. That is a genuine loss and we pay for it constantly: it is why mobile IP is hard, why your laptop's IP changes when you move from Wi-Fi to tethering (breaking every open TCP connection), and why NAT (§9) and dynamic DNS exist. We accepted it because the alternative doesn't scale at all.

**The flip condition — where flat addressing is still the right answer.** Inside a single broadcast domain, where the table is small and portability is worth more than aggregation, flat wins, and that is exactly what Ethernet does with MACs (Day 9 §5). It also reappears at internet scale in a limited form: a "portable" /24 that an organisation carries between providers is deliberately un-aggregatable — the owner is choosing to burn a global routing-table slot to buy portability. Roughly 461,596 of the routes in the March 2026 CIDR Report are more-specific routes that could in principle be aggregated away; a large slice of that is exactly this trade being made on purpose.

### The dead system you still see fossils of: classful addressing

Original IPv4 (RFC 791) hard-coded *where* the network/host split falls, using the leading bits:

| Class | Leading bits | Prefix length | Networks | Hosts per network |
|---|---|---|---|---|
| A | `0` | /8 | 128 | 16,777,214 |
| B | `10` | /16 | 16,384 | 65,534 |
| C | `110` | /24 | 2,097,152 | 254 |
| D | `1110` | (multicast) | — | — |
| E | `1111` | (reserved) | — | — |

You could not choose your prefix length; the first bits of your address chose it for you. RFC 4632 §1 names the resulting crisis precisely: "Class C, with a maximum of 254 host addresses, is too small, whereas Class B, with a maximum of 65534 host addresses, is too large for most organizations." An organisation with 400 hosts had exactly two options: take a Class B and waste 65,000 addresses, or take two Class Cs and now consume two routing-table slots and manage two networks. Almost everyone took the Class B. By 1992 the IETF's ROAD (Routing and Addressing) working group projected Class B exhaustion within about two years — while less than 4% of the address space was actually *in use on a host*.

Classful addressing is dead. It was replaced by CIDR in 1993 (RFC 1338, later RFC 1519, current: **RFC 4632**, 2006, which is explicit that "the deployment of CIDR" eliminated classes). But its fossils are everywhere and you must recognise them:

- **"Class C" as slang for /24.** Still said constantly. Now it's just wrong — a /24 is a /24.
- **Default masks in old tooling.** Some ancient config tools still guess /8 for `10.x.x.x` because of Class A.
- **RFC 1918's own wording** describes `172.16.0.0/12` as "16 contiguous class B networks" — the RFC predates the vocabulary change.
- **Default routes in the routing table shaped by class.** Occasionally you'll see a `10.0.0.0/8` route somewhere nobody intended.

**Verify before relying on it:** if you find a tool that *behaves* classfully in 2026, that is a bug, not a feature.

### Case study — how `1.1.1.0/24` became the world's most polluted address block

**What happened.** `1.0.0.0/8` sat unallocated for decades, and because `1.1.1.1` is trivially typeable, it became the address that every lazy default, every test config, every misconfigured captive portal, and every piece of enterprise networking equipment pointed at. APNIC was eventually allocated `1.0.0.0/8` and found that `1.1.1.1` was receiving enormous volumes of unsolicited garbage traffic — enough that the address was effectively unusable for any normal purpose. In 2018 APNIC partnered with Cloudflare: Cloudflare would announce the prefix on its anycast network (§12), absorb the junk traffic at scale, run a public DNS resolver on it, and give APNIC research data about the pollution.

**The engineering lesson, tied to this section.** Addresses are not interchangeable integers. An address's *history* is part of its properties, because the internet is full of long-lived configuration nobody will ever revisit. A "clean" allocation on paper can be operationally radioactive. This is also the practical argument for RFC 5737 (`192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24` reserved for documentation) and RFC 3849 (`2001:db8::/32` for IPv6 docs): if every tutorial used real addresses, the real owners of those addresses would inherit the world's typos. It's why every example in this note uses documentation space or a real incident's real prefixes, and nothing else.

**Primary source:** Cloudflare's announcement and APNIC's research framing — https://blog.cloudflare.com/announcing-1111/ and https://blog.apnic.net/2018/04/02/apnic-labs-enters-agreement-with-cloudflare-on-1-1-1-0-24-and-1-0-0-0-24-research/ (verify current — the research arrangement's details may have changed).

*(An honest gap: I have only one strong named case study for "what an address is," rather than the two a [CORE] concept nominally owes, and the second-best candidate — the AWS public-IPv4 charge — is really an exhaustion story, so it lives in §8 where it belongs rather than being duplicated here. The CIDR/classful history above is real, named, sourced, and functions as the failure case: a documented design failure with a documented fix.)*

---

## 3. CIDR: computing the split by hand

**Depth: [CORE]**

### Intuition — the one idea

Classful addressing failed because the network/host split was fixed. **CIDR (Classless Inter-Domain Routing) makes the split a parameter you state explicitly.** You write the address, then a slash, then how many leading bits are the network part. RFC 4632 §3.1 defines the notation as "a 4-octet quantity, just like a traditional IPv4 address or network number, followed by the '/' (slash) character, followed by a decimal value between 0 and 32 that describes the number of significant bits."

`198.51.100.0/24` means: the first 24 bits identify the network; the remaining 8 bits identify an interface within it. `198.51.100.0/26` means: 26 network bits, 6 host bits — a quarter of the size. Same base address, different meaning, because the prefix length is part of the statement.

Everything else in this section is consequences of that one idea.

### Analogy — postcodes, and where it breaks

A CIDR prefix is a postcode prefix. `SW1A` narrows you to a district; `SW1A 1AA` is one building. Shortening the prefix widens the area; lengthening it narrows it. A sorting office reasoning about `SW1A` doesn't care which building — one rule covers the whole district. That is aggregation, and it is exactly why routers can be small.

**Where it breaks — and this one is important enough that missing it will make subnetting feel like magic.** Postcodes are *arbitrary strings* and you can subdivide them however you like: `SW1A 1`, `SW1A 1A`, any granularity, any boundary. **CIDR blocks are powers of two, aligned to powers of two.** You cannot have a network of 100 addresses. You cannot start a /24 at `198.51.100.50`. Every block's size is 2^(32−prefixlen), and its starting address must be an exact multiple of its size. This constraint is not bureaucratic pedantry; it is what makes the router's job a single bitwise AND instead of a range comparison, and it is the source of every "why can't I just..." frustration in subnet planning.

Second break: postcodes are geographic by construction. CIDR blocks are geographic only by convention, and often not at all — a single /24 can be announced simultaneously from six continents (§12, anycast).

### The mechanism: the mask, and the AND

The prefix length is usually also written as a **subnet mask**: a 32-bit number with the network bits set to 1 and the host bits set to 0. `/24` is `11111111 11111111 11111111 00000000` = `255.255.255.0`. `/26` is `11111111 11111111 11111111 11000000` = `255.255.255.192`.

Now recall the running-total list from §2: 128, 192, 224, 240, 248, 252, 254, 255. Those are the only possible non-zero, non-full octet values in a mask, and they appear in exactly that order as the prefix lengthens through an octet, because a mask's 1-bits are always contiguous and left-aligned. `/25` → `.128`, `/26` → `.192`, `/27` → `.224`, `/28` → `.240`, `/29` → `.248`, `/30` → `.252`, `/31` → `.254`, `/32` → `.255`. Memorise the list; you have memorised mask arithmetic.

**The operation a router performs** is a bitwise AND of the address with the mask. Where the mask bit is 1, the address bit survives; where it is 0, the result is 0. What survives is the **network address**. Two addresses are in the same network exactly when `addr AND mask` gives the same result for both.

**Worked example — is `198.51.100.77` inside `198.51.100.64/27`?** Do it fully in binary.

```
address  198.51.100.77   11000110.00110011.01100100.01001101
mask     /27             11111111.11111111.11111111.11100000
                         --------------------------- AND ---
network                  11000110.00110011.01100100.01000000  = 198.51.100.64
```

The AND gives `198.51.100.64`, which is the network we asked about. **Yes, it's inside.**

Now get the boundaries. Set all host bits to 0 → network address. Set all host bits to 1 → broadcast address.

```
network   11000110.00110011.01100100.010 00000  = 198.51.100.64
broadcast 11000110.00110011.01100100.010 11111  = 198.51.100.95
                                        ^^^ ^^^^^
                                network | host (5 bits)
```

So `198.51.100.64/27` spans `198.51.100.64` through `198.51.100.95`. That is 32 addresses (2^5), of which the first is the network address and the last is the broadcast address, leaving **30 assignable host addresses**.

**The shortcut that makes this fast without binary.** The block size is 2^(32−prefix). For a /27 that is 32. Blocks of size 32 in the last octet start at 0, 32, 64, 96, 128, 160, 192, 224. `77` falls between 64 and 95, so the network is `.64` and the broadcast is `.64 + 32 − 1 = .95`. Two subtractions, no binary. Use binary to *understand*; use the block-size shortcut to *work*.

All of the above verified by computation:

### Runnable example — verify every subnet calculation you will ever hand-write

Do not trust your arithmetic, and do not trust mine. Python's standard library ships the checker.

```python
# no install needed - ipaddress is in the stdlib (Python 3.3+)
import ipaddress as ip

def explain(cidr, reserved=2):
    """reserved=2 for classic IP (network+broadcast); reserved=5 for AWS/Azure."""
    n = ip.ip_network(cidr, strict=True)
    tobin = lambda a: ".".join(f"{int(o):08b}" for o in str(a).split("."))
    print(f"{str(n):<20} mask={n.netmask}  hostbits={32-n.prefixlen}")
    print(f"  range      {n.network_address} - {n.broadcast_address}")
    print(f"  total      {n.num_addresses}   usable {max(n.num_addresses-reserved,0)}")
    print(f"  addr  bits {tobin(n.network_address)}")
    print(f"  mask  bits {tobin(n.netmask)}")

explain("198.51.100.64/27")
print("198.51.100.77 inside? ",
      ip.ip_address("198.51.100.77") in ip.ip_network("198.51.100.64/27"))

# split a block into equal children
for s in ip.ip_network("203.0.113.0/24").subnets(new_prefix=26):
    print(s, s.network_address, "-", s.broadcast_address)

# does one block sit inside another?  (this is the YouTube hijack, section 11)
print("208.65.153.0/24 inside 208.65.152.0/22? ",
      ip.ip_network("208.65.153.0/24").subnet_of(ip.ip_network("208.65.152.0/22")))
```

Real output (Python 3.12, executed while writing this note):

```
# -> 198.51.100.64/27     mask=255.255.255.224  hostbits=5
# ->   range      198.51.100.64 - 198.51.100.95
# ->   total      32   usable 30
# ->   addr  bits 11000110.00110011.01100100.01000000
# ->   mask  bits 11111111.11111111.11111111.11100000
# -> 198.51.100.77 inside?  True
# -> 203.0.113.0/26   203.0.113.0 - 203.0.113.63
# -> 203.0.113.64/26  203.0.113.64 - 203.0.113.127
# -> 203.0.113.128/26 203.0.113.128 - 203.0.113.191
# -> 203.0.113.192/26 203.0.113.192 - 203.0.113.255
# -> 208.65.153.0/24 inside 208.65.152.0/22?  True
```

**Why this works, line by line.** `ip_network(cidr, strict=True)` is the important one: `strict=True` **raises** if you pass an address with host bits set, e.g. `ip_network("198.51.100.77/27")` → `ValueError: 198.51.100.77/27 has host bits set`. That error is a feature — it is the library refusing the "unaligned block" mistake the analogy-break warned about, and it will catch a real class of Terraform/config bugs before you deploy them. `n.netmask` derives the mask from the prefix length, so you never type a mask by hand. `n.num_addresses` is 2^(32−prefixlen) — total addresses, *before* any reservation, which is why `explain` takes `reserved` as a parameter instead of hard-coding 2 (clouds reserve 5; see below). `.subnets(new_prefix=26)` performs the split-into-equal-children operation and guarantees alignment. `.subnet_of()` answers containment between two *blocks* rather than address-in-block, which is precisely the question longest-prefix match asks (§4) and precisely the mechanism the 2008 YouTube hijack abused (§11).

**Honesty note:** `usable = total − 2` is the classic-IP answer and it is wrong in two directions. On a /31 point-to-point link, RFC 3021 explicitly permits using *both* addresses, so a /31 has 2 usable, not 0. And in every major cloud the answer is `total − 5` or `total − 4`. The next subsection is that gap.

### The prefix-length table (recap — everything above, compressed)

Now that the reasoning is established, here is the lookup table. Every number in it was generated by the script above, not recalled.

| Prefix | Mask | Host bits | Total addrs | Usable (classic, −2) | Usable (AWS/Azure, −5) | Blocks per /16 |
|---|---|---|---|---|---|---|
| /16 | 255.255.0.0 | 16 | 65,536 | 65,534 | 65,531 | 1 |
| /17 | 255.255.128.0 | 15 | 32,768 | 32,766 | 32,763 | 2 |
| /18 | 255.255.192.0 | 14 | 16,384 | 16,382 | 16,379 | 4 |
| /19 | 255.255.224.0 | 13 | 8,192 | 8,190 | 8,187 | 8 |
| /20 | 255.255.240.0 | 12 | 4,096 | 4,094 | 4,091 | 16 |
| /21 | 255.255.248.0 | 11 | 2,048 | 2,046 | 2,043 | 32 |
| /22 | 255.255.252.0 | 10 | 1,024 | 1,022 | 1,019 | 64 |
| /23 | 255.255.254.0 | 9 | 512 | 510 | 507 | 128 |
| /24 | 255.255.255.0 | 8 | 256 | 254 | 251 | 256 |
| /25 | 255.255.255.128 | 7 | 128 | 126 | 123 | 512 |
| /26 | 255.255.255.192 | 6 | 64 | 62 | 59 | 1,024 |
| /27 | 255.255.255.224 | 5 | 32 | 30 | 27 | 2,048 |
| /28 | 255.255.255.240 | 4 | 16 | 14 | 11 | 4,096 |
| /29 | 255.255.255.248 | 3 | 8 | 6 | 3 | 8,192 |
| /30 | 255.255.255.252 | 2 | 4 | 2 | 0 | 16,384 |
| /31 | 255.255.255.254 | 1 | 2 | 0 (2 per RFC 3021) | 0 | 32,768 |
| /32 | 255.255.255.255 | 0 | 1 | 1 (a single host) | 0 | 65,536 |

Two rows to stare at. **/29 in a cloud gives you three usable addresses.** Three. That is why "I'll just use a /29 for the small subnet" is a mistake people make exactly once. And **/30 in a cloud gives you zero** — which is why AWS, Azure, and GCP all refuse to create a subnet smaller than /28 (AWS) or /29 (Azure, GCP).

### The cloud reservation gap [WORKING]

Classic IP burns 2 addresses per subnet: the all-zeros network address and the all-ones broadcast address. Cloud providers burn more, because their virtual network is a software-defined overlay that needs addresses for its own plumbing — a virtual router, a DNS resolver, spare capacity for future features. They take those out of *your* subnet.

- **AWS** reserves **5** per subnet: the first four and the last one. For `192.168.0.0/24`: `.0` network, `.1` VPC router, `.2` DNS (the Route 53 Resolver, i.e. the "VPC+2" address), `.3` reserved for future use, `.255` broadcast — and AWS notes it doesn't actually *support* broadcast but reserves the address anyway. Minimum subnet /28, maximum /16. Source: https://docs.aws.amazon.com/vpc/latest/userguide/subnet-sizing.html
- **Azure** reserves **5**, same shape: "Azure reserves the first four addresses and the last address, for a total of five IP addresses within each subnet." For `192.168.1.0/24`: `.0` network, `.1` default gateway, `.2` and `.3` mapped to Azure DNS, `.255` broadcast. Smallest supported IPv4 subnet **/29**, largest **/2**; IPv6 subnets must be exactly /64. Source: https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-faq
- **GCP** reserves **4** on the primary IPv4 range: first (network), second (default gateway), second-to-last (reserved for future use), last (broadcast). So `10.1.2.0/24` gives you `10.1.2.2` through `10.1.2.253`. Longest permitted primary-range mask **/29**. Notably, **secondary** IPv4 ranges (used for GKE pods/services) have *no* reservations — every address is usable. Source: https://cloud.google.com/vpc/docs/subnets

**Verify current** — these are exactly the kind of provider-specific numbers that change, and the DNS-address convention in particular is the sort of thing that gets an extra reserved address added to it.

**Why this is not trivia.** Kubernetes assigns an IP per pod. On EKS with the VPC CNI, pods get real VPC addresses out of the subnet. A team sizes an app subnet at /24 (251 usable on AWS), runs 4 nodes with 30 pods each = 120 pod IPs + node IPs + ENIs for load balancers and endpoints, then autoscales to 8 nodes and the cluster stops scheduling with `failed to assign an IP address to container`. The subnet did not "fill up with servers"; it filled up with pods. Size cloud app subnets by *pod* count, not instance count, and put a factor of 4 on top for rolling deployments (which need double capacity briefly) and future growth. Note that GCP's zero-reservation secondary ranges exist precisely because Google hit this problem first.

### Special-purpose address ranges you must recognise on sight

**Depth: [WORKING]**

These come from the IANA IPv4 Special-Purpose Address Registry, which is the authoritative list; the RFC attributions below are the ones IANA itself cites. Registry: https://www.iana.org/assignments/iana-ipv4-special-registry/

The reason to know these is diagnostic. When you see an address in a log or a traceroute, its *range* tells you immediately what kind of thing you are looking at, and that often solves the problem before you've read anything else.

| Block | Name / purpose | RFC (per IANA) | What seeing it tells you |
|---|---|---|---|
| `0.0.0.0/8` | "this network"; `0.0.0.0` = unspecified / "all addresses" | RFC 791 | As a bind address: listening on every interface. As a route: the default route. |
| `10.0.0.0/8` | Private | RFC 1918 | Someone's internal network. 16,777,216 addresses. |
| `100.64.0.0/10` | **Shared Address Space** (CGNAT) | RFC 6598 | You are behind a carrier-grade NAT. ~4.19M addresses. |
| `127.0.0.0/8` | Loopback | RFC 1122 | Never leaves the host. A whole /8 for it — a famously extravagant 16.7M addresses. |
| `169.254.0.0/16` | Link-local (APIPA) | RFC 3927 | **DHCP failed.** The host self-assigned. Also: `169.254.169.254` is the cloud metadata service. |
| `172.16.0.0/12` | Private | RFC 1918 | Internal. Note it is `172.16`–`172.31`, **not** all of `172.x`. |
| `192.0.2.0/24` | Documentation (TEST-NET-1) | RFC 5737 | Someone pasted an example into production config. |
| `192.168.0.0/16` | Private | RFC 1918 | Almost certainly a home/small-office LAN. |
| `198.18.0.0/15` | Benchmarking | RFC 2544 | Test equipment. |
| `198.51.100.0/24` | Documentation (TEST-NET-2) | RFC 5737 | As above. |
| `203.0.113.0/24` | Documentation (TEST-NET-3) | RFC 5737 | As above. |
| `224.0.0.0/4` | Multicast | RFC 1112 (class D) | Group traffic — mDNS, IGMP, routing protocols. |
| `240.0.0.0/4` | Reserved (former class E) | RFC 1112 | 268M addresses nobody can use. Periodically proposed for reclamation. |
| `255.255.255.255/32` | Limited broadcast | RFC 8190, RFC 919 | "Everyone on this link." |

Two of these earn extra prose because they are the ones that actually save you time.

**`169.254.x.x` is a diagnosis, not an address.** If a host has one, DHCP did not answer and the host fell back to picking a random link-local address (RFC 3927). Windows calls this APIPA. Seeing it in a bug report means: check the DHCP server, the VLAN assignment, or the cable — not the application. And the single most operationally significant address on the internet lives in this block: **`169.254.169.254`**, the cloud instance-metadata service on AWS, Azure, and GCP. It is unrouted, link-local, unauthenticated by design, and it hands out IAM credentials to anything that can issue an HTTP GET. Day 9 covered why an agent sandbox must block it; I'm not repeating that argument, only noting here *why the address is in this range* — link-local means "cannot be reached from off-link," which is the only thing protecting it.

**`100.64.0.0/10` means your assumptions about "one customer, one IP" are void.** RFC 6598 created it because ISPs ran out of public addresses for customers and needed private-ish space that would *not* collide with the RFC 1918 space customers use behind their own routers. That collision is the whole reason a new block was needed: "Unless at least one of the conditions above is true, the Service Provider cannot safely use [RFC1918] address space." The RFC's operational rules are strict — "Packets with Shared Address Space source or destination addresses MUST NOT be forwarded across Service Provider boundaries," and providers must filter route advertisements for it. §9 covers the consequences.

**Worked example — read this real routing table and diagnose the machine.** From the machine this note was written on (`Get-NetRoute`, 2026-08-11, edited to remove duplicate multicast rows):

```
ifIndex DestinationPrefix    NextHop         Protocol
     11 0.0.0.0/0            192.168.29.1    NetMgmt     <- default route, learned by DHCP
      1 127.0.0.0/8          0.0.0.0         Local       <- loopback net
      1 127.0.0.1/32         0.0.0.0         Local
      1 127.255.255.255/32   0.0.0.0         Local       <- loopback broadcast
     36 172.18.96.0/20       0.0.0.0         Local       <- Hyper-V virtual switch
     36 172.18.96.1/32       0.0.0.0         Local
     36 172.18.111.255/32    0.0.0.0         Local       <- .111.255 = broadcast of /20
     11 192.168.29.0/24      0.0.0.0         Local       <- the Wi-Fi LAN
     11 192.168.29.239/32    0.0.0.0         Local       <- this host
     11 192.168.29.255/32    0.0.0.0         Local       <- LAN broadcast
     11 224.0.0.0/4          0.0.0.0         Local       <- multicast
     11 255.255.255.255/32   0.0.0.0         Local       <- limited broadcast
```

Everything you need is readable from ranges and prefix lengths. `192.168.29.0/24` and `172.18.96.0/20` are both RFC 1918 → **this machine has no public IP; it is behind NAT** (§9). Two private networks on one host → **multi-homed** (§2's point about addresses belonging to interfaces). `0.0.0.0/0` with a next hop → a default route exists, so off-LAN traffic has somewhere to go. And check the arithmetic on the Hyper-V row: a /20 starting at `172.18.96.0` has 12 host bits, so it spans `172.18.96.0`–`172.18.111.255`, which is exactly the broadcast address Windows installed. The table is internally consistent, and now you can verify that by hand.

---

## 4. Hop-by-hop forwarding and the routing table

**Depth: [CORE]**

### Intuition — nobody knows the way

Here is the fact that most surprises people, and it is worth sitting with: **no router on the internet knows the path from you to your destination.** Not one. There is no component anywhere that holds the route.

Each router knows only one thing per destination: *which neighbour to hand the packet to next*. It hands it over and forgets. The next router asks the same question and does the same thing. The path exists only as an emergent consequence of a few dozen independent local decisions, and it is not written down anywhere — which is precisely why `traceroute` had to be invented (§6): the only way to learn the path is to make the routers confess it one hop at a time.

Why build it this way? Because the alternative requires global knowledge, and global knowledge in a system with 79,231 independently-administered participants (measured, AS65000 report, 11 Aug 2026) is unobtainable. To compute a full path you would need an accurate, current, consistent view of every link on earth — including links inside networks whose owners consider their topology a trade secret. Hop-by-hop forwarding needs each router to know only about its own neighbours. Then a routing protocol (§11) gossips reachability around, and paths emerge.

**The trade-off, honestly.** You give up two real things. First, **you cannot control your packet's path.** You get whatever the routers collectively decide, which may be absurd — §6 shows a real trace from India to Oregon that goes *east* via Singapore and Tokyo across the Pacific. Second, **the path can change mid-connection**, without warning, and can be different in each direction. Asymmetric routing is normal, not pathological, and it defeats a whole class of debugging: you can measure the path *out* and have no way to see the path *back*.

**The flip condition — where the alternative wins.** Inside a network you control, global knowledge *is* obtainable, and then explicit path control becomes worth the cost. That is exactly what MPLS traffic engineering and (more recently) segment routing do: a source computes a full path and encodes it in the packet, because a single operator's own topology is knowable. Google's B4 SDN WAN goes further — a central controller computes forwarding for the whole network. So the answer flips on administrative scope: one owner → central path computation can win; many owners → hop-by-hop is the only thing that works.

### Analogy — the relay of strangers, and where it breaks

You are lost in a foreign city with a letter for an address across town. You don't ask for directions; you hand the letter to the nearest person and say "get this closer." They don't know the address either, but they know the general direction of the city centre, so they walk a block and hand it to someone else. That person knows the district. That person hands it to a local, who knows the street. Nobody ever knew the route. The letter still arrives.

**Where the analogy breaks — three ways.**

1. **Your relay of strangers is guessing from vague knowledge; routers are consulting a table with a mathematically exact matching rule.** A router does not have a "general sense of direction." It has entries, and one deterministic tie-break rule (longest prefix match, below). Given the same table and the same packet, a router's decision is entirely reproducible.
2. **A stranger who doesn't know what to do can ask, or refuse. A router cannot.** It applies the table and forwards, or it has no matching entry and drops the packet (usually sending back ICMP Destination Unreachable). There is no "hmm, let me find out." A wrong table entry produces confident, silent, high-speed wrongness — which is how the incidents in §11 blackholed traffic for millions of users.
3. **People will not hand the letter in a circle forever; they'd notice.** Routers absolutely will. Two routers can each believe the other is closer to the destination and pass a packet back and forth at line rate until something stops them. Nothing in hop-by-hop forwarding detects this. §5 is the thing that stops it.

### The routing table, field by field

A routing table (properly, the **forwarding table** or FIB — see "control plane vs data plane" below) is a list of rules. Each rule is at minimum:

- **Destination prefix** — a CIDR block, e.g. `192.168.29.0/24` or `0.0.0.0/0`.
- **Next hop** — the IP address of the neighbouring router to hand matching packets to. If the destination network is attached *directly* to this router, there is no next hop; the convention is to write `0.0.0.0`, or `on-link`, or leave it blank.
- **Outgoing interface** — which physical or virtual port to send it out of. (Windows shows `ifIndex`; Linux shows `dev eth0`.)
- **Metric** — a cost used to break ties when two entries have the *same* prefix length. Lower wins.
- **Protocol / source** — how the entry got there: statically configured, learned from DHCP, learned from OSPF, learned from BGP.

Look again at the real table in §3. `0.0.0.0/0 → 192.168.29.1` is the **default route**, also called the gateway of last resort. A /0 has zero network bits, so *every* address matches it. It is the "I have no idea, ask upstream" rule, and on your laptop it is essentially the entire internet. `192.168.29.0/24 → 0.0.0.0` (on-link) says "this network is on my Wi-Fi segment; don't route, just ARP for the host directly."

That distinction — **directly connected vs via a next hop** — is where Day 9 hands off. If the destination is on-link, the host ARPs for the *destination's* MAC and frames the packet to it. If the destination is off-link, the host ARPs for the **next hop's** MAC and frames the packet to *the router*, while the IP destination in the header stays the final destination. This is the single most clarifying fact about the L2/L3 split: the **Ethernet destination changes at every hop; the IP destination never does**. Day 9 §7 proved it with a packet decode; I'm only naming the connection.

### Control plane vs data plane — two tables, not one

I have been saying "routing table" as though there is one. There are two, and the distinction explains a class of confusing behaviour.

The **control plane** is the slow, smart part: it runs the routing protocols, talks to neighbours, applies policy, and decides what the best path to each prefix is. Its output is the **RIB** (Routing Information Base) — *every* route the router has learned, including all the ones it rejected. A router with three BGP neighbours offering paths to the same prefix holds all three in the RIB, along with the reasons one won.

The **data plane** is the fast, dumb part: for each arriving packet, look up the destination and forward. Its table is the **FIB** (Forwarding Information Base) — only the *winning* route per prefix, stripped down to exactly what forwarding needs (prefix, next hop, outgoing interface, rewritten link-layer header), and laid out in whatever structure the lookup hardware wants (a trie, or TCAM — see "Under the hood" below).

**Why the split matters practically.** Three things follow from it.

First, **the FIB is what actually moves your packets, and it can differ from what you expect the RIB to have chosen.** When a vendor's `show ip route` prints the RIB and `show ip fib` prints the FIB, and they disagree, you have found a real bug — and it is a class of bug that has caused outages, because the router *reports* a correct route while *forwarding* by a stale one.

Second, **the FIB is the resource with the hard physical limit.** §4's 512k-day case study below is entirely a FIB-capacity story: the RIB lived in ordinary DRAM and was fine; the FIB lived in TCAM and ran out. When you read "active BGP entries (FIB)" in the routing-table statistics quoted in §2, that parenthetical is telling you which table is being counted, and it is the one that matters for capacity.

Third, **control-plane work is expensive and is deliberately deprioritised**, which is exactly why §6's traceroute output is full of missing and slow hops. Generating an ICMP Time Exceeded is a control-plane task. Forwarding your actual traffic is a data-plane task. A busy router will happily forward at 100 Gbps while taking 200 ms to answer a traceroute probe, or refusing to answer at all. **A router that looks slow in traceroute is usually a router that is correctly refusing to prioritise your diagnostics over other people's traffic.** Nothing else in §6 makes sense without this.

Day 9 §12's cloud-networking material has a direct analogue: in a VPC, the control plane is the provider's SDN controller distributing your route tables and security groups, and the data plane is the hypervisor-level enforcement. That is why a security-group change takes effect in seconds rather than instantly, and why "I changed the route table, why is traffic still going the old way?" has a propagation answer.

### Longest prefix match — the rule that makes it all work

**Depth: [CORE]** — this is the load-bearing algorithm of the internet, and it is also the mechanism every incident in §11 abuses.

A packet's destination often matches *several* table entries at once. `203.0.113.130` matches `0.0.0.0/0`, and it matches `203.0.113.0/24`, and it matches `203.0.113.128/25`. Which rule applies?

**The rule: the match with the longest prefix wins.** Most specific route, always, regardless of metric, regardless of which routing protocol supplied it, regardless of anything else. Metrics only break ties *between entries of equal prefix length*. This is normative for routers in RFC 1812 §5.2.4.3.

The reasoning is straightforward once stated: a longer prefix is a more precise claim about a smaller set of addresses, and a more precise claim comes from something closer to the truth. `0.0.0.0/0` says "I know nothing, try upstream." `203.0.113.128/25` says "I know specifically about these 128 addresses." Trusting the specific over the vague is right almost always — and §11's case studies are the "almost."

**Worked example — trace four packets through one table.** Verified by computation:

```
Routing table:
  0.0.0.0/0            -> next hop A   (default)
  203.0.113.0/24       -> next hop B
  203.0.113.128/25     -> next hop C
  203.0.113.130/31     -> next hop D

Packet to 203.0.113.130
  matches 0.0.0.0/0 (len 0), 203.0.113.0/24 (24), 203.0.113.128/25 (25), 203.0.113.130/31 (31)
  longest = /31  -> next hop D

Packet to 203.0.113.200
  matches /0, /24, /25          (not the /31: that covers only .130 and .131)
  longest = /25  -> next hop C

Packet to 203.0.113.5
  matches /0, /24               (not the /25: .5 < .128)
  longest = /24  -> next hop B

Packet to 198.51.100.9
  matches /0 only
  longest = /0   -> next hop A   (default route)
```

Check the /31 by hand to be sure you believe it: `203.0.113.130/31` has 1 host bit, so it covers exactly two addresses, `.130` and `.131`. `.200` is not one of them.

**The full forwarding decision, as pseudocode:**

```
on packet arrival:
  1. verify IP header checksum; if bad, DROP silently
  2. if TTL <= 1: DROP, send ICMP Time Exceeded (type 11 code 0) to source   # section 5
  3. decrement TTL by 1
  4. candidates = every FIB entry whose prefix contains dest_addr
  5. if candidates empty: DROP, send ICMP Destination Unreachable (type 3)
  6. best = the candidate with the longest prefix
       (tie -> lowest metric -> then ECMP hash across equal paths)
  7. if next_hop is on-link: resolve dest_addr via ARP/NDP
     else:                   resolve next_hop  via ARP/NDP
  8. if packet_size > outgoing_interface_MTU:                                # section 7
       if DF set: DROP, send ICMP type 3 code 4 (frag needed)
       else:      fragment
  9. recompute header checksum (TTL changed), re-frame for the outgoing link, transmit
 10. forget everything about this packet
```

Step 10 is not a joke. It is why the network is fast and why it is opaque.

Two details in that pseudocode that matter operationally. **Step 6's ECMP** (Equal-Cost Multi-Path): when several equal-length, equal-metric paths exist, the router hashes some subset of the packet header — typically source IP, destination IP, protocol, source port, destination port, the "5-tuple" — and picks a path from the hash. Hashing rather than round-robin is deliberate: every packet of one TCP connection hashes identically, so it takes one consistent path and doesn't get reordered. This is also why traceroute can show you *different* paths on consecutive probes (§6) and why "Paris traceroute" exists.

**Step 2's `TTL <= 1` rather than `TTL == 0`** is the correct formulation and a classic interview trip-wire. A router receiving a packet with TTL=1 cannot forward it, because forwarding requires decrementing to 0 and a TTL-0 packet must not be transmitted. So the packet dies *at* that router, and that router sends the ICMP error. Which is exactly what §6's traceroute exploits.

### Runnable example — the real forwarding decision, on your own machine

This one earns a code block because the answer is genuinely non-obvious and the tool does longest-prefix match *for you*, letting you check your reasoning against the kernel's.

```powershell
# Windows PowerShell 5.1+ — no install needed.
# Ask the OS: for this destination, which route wins and what source address will I use?
Find-NetRoute -RemoteIPAddress 8.8.8.8      | Format-List IPAddress, InterfaceIndex, DestinationPrefix, NextHop
Find-NetRoute -RemoteIPAddress 192.168.29.7 | Format-List IPAddress, InterfaceIndex, DestinationPrefix, NextHop
Find-NetRoute -RemoteIPAddress 172.18.96.50 | Format-List IPAddress, InterfaceIndex, DestinationPrefix, NextHop
```

```
# Linux / macOS equivalents:
#   ip route get 8.8.8.8
#   ip route get 192.168.29.7
#   route -n get 8.8.8.8          # macOS
```

**Why this works, and what to look for.** `Find-NetRoute` runs the kernel's actual route-lookup path rather than printing the table, so it applies longest-prefix match, metric tie-breaks, and source-address selection exactly as a real packet would. Two things to read in the output. The `DestinationPrefix` tells you *which rule won* — for `8.8.8.8` on this machine it is `0.0.0.0/0` (nothing more specific exists), for `192.168.29.7` it is `192.168.29.0/24` (on-link, so no next hop), and for `172.18.96.50` it is `172.18.96.0/20` (the Hyper-V switch). The `IPAddress` field tells you the **source address the kernel would stamp on the packet** — and this is the field that answers "why do my outbound requests appear to come from the wrong IP?" The route chose the interface; the interface chose the source. Change the routing table (connect a VPN, which typically installs a more-specific route or a competing default with a better metric) and the source address changes silently underneath your application. Run these three before and after connecting a VPN; the diff *is* the lesson.

### Under the hood — how a router does this in nanoseconds

The pseudocode above looks like a linear scan of a million entries. At 100 Gbps with 64-byte packets a router has roughly 6.7 nanoseconds per packet. A linear scan is not happening. Two mechanisms, and they map onto a real cost/flexibility decision.

**Software forwarding: the trie.** Store prefixes in a binary tree where depth = bit position. Walk the destination address bit by bit from the most significant end; each 1 or 0 chooses a child; remember the deepest node that was a real prefix. When you fall off the tree, that remembered node is the longest match. This is O(32) for IPv4 — bounded, table-size-independent. Real implementations compress it heavily: Linux uses **LPC-trie** (level-compressed path-compressed trie) for IPv4 in `net/ipv4/fib_trie.c`, which collapses long single-child chains and turns dense subtrees into multi-bit array lookups. This is what runs on your laptop and in software routers like VPP and DPDK-based forwarders.

**Hardware forwarding: TCAM.** Ternary Content-Addressable Memory is a chip that compares an input against *every* stored entry simultaneously, in one clock cycle. "Ternary" means each stored bit can be 0, 1, or **don't-care** — and don't-care is exactly what a prefix's host bits are. Store `203.0.113.128/25` as `11001011 00000000 01110001 1XXXXXXX` and TCAM matches it in one cycle. Entries are ordered so that the longest prefix is found first. TCAM is why a hardware router forwards at line rate.

And TCAM is also why **routing table size is a hard physical limit, not a soft one**, which produced one of networking's best-known incidents:

### Case study — "512k day," 12 August 2014

**What happened.** Many widely-deployed Cisco routers (notably the 6500/7600 with default configuration) allocated TCAM for a maximum of 512,000 IPv4 routes. On 12 August 2014 the global BGP table crossed 512,000 prefixes. Routers that had not been reconfigured began failing to install new routes — some dropped routes, some crashed, some fell back to slow-path software forwarding and collapsed under load. Verizon was reported to have briefly de-aggregated a large number of prefixes that day, pushing the count over the line slightly early. Widespread instability followed for hours at multiple providers, including reported problems at eBay, LastPass, and various ISPs.

**The engineering lesson, tied directly to this section.** The global routing table is a **shared, unbounded, adversarially-growing resource that every router on earth must fit in fixed silicon.** Nobody owns it. Anyone can add to it — every time an operator announces a more-specific prefix instead of an aggregate (see the Verizon/DQE incident in §11 for the same mechanism causing a different disaster), the table everyone must hold gets bigger. This is a textbook tragedy of the commons implemented in hardware, and it recurs: the table crossed 768,000 in 2019 and caused a smaller repeat, and it is 1,071,851 as of 11 August 2026 with 79,231 ASes contributing. The mitigations are all unglamorous: raise the TCAM partition *before* you need to, filter more-specifics you don't need, aggregate your own announcements, and monitor `bgp table size` as a capacity metric rather than a curiosity.

**Primary sources:** the historical table-size data is at https://bgp.potaroo.net/as2.0/bgp-active.html (Geoff Huston's AS65000 report; the graph shows the 2014 inflection directly), and Cisco's own guidance on TCAM partitioning for the affected platforms — https://www.cisco.com/c/en/us/support/docs/switches/catalyst-6500-series-switches/117712-problemsolution-cat6500-00.html. **Verify current:** the specific platform limits are long obsolete; the *pattern* is not.

### System design ① — multi-region failover at the network level: what actually moves the traffic?

*(This scenario exercises forwarding/routing and anycast together. §12 defines anycast; read this after it if the term is new, or read it now and let §12 fill in the mechanism — I have deliberately put it here because the *decision* is a routing decision.)*

**The problem.** You run an API in `us-east-1` and `eu-west-1`. Both regions are live and can serve any request. When a region fails — not a single instance, the whole region, or the network path to it — traffic must move to the other one. Requirements: recovery under 60 seconds for the median client; no client-side changes; correct behaviour when the failure is a *network* failure rather than a server failure; must work for both browser clients and server-to-server API clients including SDKs that cache DNS badly.

**The question that matters.** "What actually moves the traffic?" — because *something* has to change a client's mind about where to send packets, and there are exactly three places that decision can live: in the client, in DNS, or in the routing table.

**Alternative 1: client-side failover.** Ship a client library with both regional hostnames; on failure, retry the other. This is the *fastest* possible failover (no propagation delay at all, sub-second) and the most precise (the client knows its own request failed). It loses because of the "no client-side changes" requirement and because you do not control every client — browsers, third-party integrations, and that one team's five-year-old Python script will never be updated. **Flip condition:** if you *do* control all clients (a mobile app you ship, an internal service mesh), client-side failover is genuinely the best answer and you should prefer it. This is why service meshes do retries and outlier ejection at the sidecar rather than relying on DNS.

**Alternative 2: DNS-based failover.** One hostname, `api.example.com`. A health-checking service probes both regions; DNS returns only healthy regions' addresses. Traffic moves when resolvers re-query.

**Alternative 3: anycast.** Announce *the same* IP prefix via BGP from edge locations in both regions. Every client sends to the same address; the routing system delivers each packet to whichever announcement is topologically nearest. To fail a region out, **withdraw its BGP announcement** — and every router on the internet re-converges to the remaining announcement within seconds.

**The decision: anycast for the front door, DNS-based failover as the coarse-grained layer behind it.** The reason is the specific constraint "must work when the failure is a network failure." Consider DNS-only: your health checker in `us-east-1` says the region is fine (the servers *are* fine), but the transit path from Europe to `us-east-1` is broken. DNS keeps handing out the dead address, because DNS health checks measure *the server*, not *the client's path to it*. Anycast has no such blind spot, because the thing that steers traffic **is** the routing system — if the path is broken, BGP itself re-converges and the traffic moves without anyone deciding anything.

**The trade-off, honestly.** Anycast costs a great deal that DNS does not. You need your own address space (an ARIN/RIPE allocation, which is now expensive and slow to get — §8), your own AS number, BGP sessions with transit providers in every location, and the operational competence to not repeat §11. You lose fine-grained control: you cannot say "send 10% to region B," because you are not choosing — the internet is. And RFC 4786 §4.4 is explicit about the sharpest limitation: anycast suits "short-lived, stateless transactions" and faces real problems with "long-lived TCP connections where mid-session route changes can redirect packets to different nodes." A BGP re-convergence mid-upload sends your packets to a machine that has never heard of your TCP connection, and it replies RST. DNS-based failover, by contrast, costs almost nothing, works with a hosted zone and a credit card, and gives you weighted/geo/latency policies for free.

**The flip condition.** Use DNS-only when: you don't own address space; your traffic is HTTP through a CDN or cloud load balancer that already does the anycast for you (this is the common case — CloudFront, Cloudflare, and AWS Global Accelerator are anycast, so you rent it rather than build it); or your RTO tolerance is minutes rather than seconds. Use anycast when you are the infrastructure: DNS resolvers, CDN edges, DDoS scrubbing. The clean summary: **anycast moves traffic by changing routing; DNS moves traffic by changing answers. Routing is faster and sees network failures, but is coarse and stateless-friendly. Answers are cheap and flexible, but are cached by things you don't control and are blind to path failures.**

**Failure modes to design against.**

- **DNS TTL is a lie.** You set 60 seconds. Some resolvers, some JVMs (`networkaddress.cache.ttl` defaulted to *forever* in old JVMs), and some SDKs will cache far longer. Your 60-second RTO is 60 seconds *for well-behaved clients*, and a long tail behind that. Day 12 owns the mechanics; the design consequence is: never let DNS be your *only* failover layer, and always make the old region return clean errors rather than hanging, so client retries can work.
- **Health-check flapping.** A marginal region oscillates in and out, and every flap breaks in-flight connections. Require N consecutive failures to fail out and M consecutive successes to fail back in, with M > N (asymmetric hysteresis — slow to trust, quick to distrust).
- **The health check passes but the service is broken.** "Port 443 is open" is not "the API works." Check a real dependency-touching endpoint. Day 11's system design covers the TCP-vs-app-level health check distinction.
- **You failed over and now the surviving region falls over.** Capacity, not networking. Each region must be able to carry the whole load, or you must shed load on failover. Meta's own postmortem flagged the mirror image of this: bringing everything back at once risked "a new round of crashes due to a surge in traffic."
- **Split brain on the data layer.** Moving traffic is the easy half. If both regions accept writes, the network solved nothing and created a data problem. Out of scope here, but never out of scope in reality.

### System design ② — routing for hybrid cloud, and the asymmetric-return-path trap

**The problem.** A company runs an on-prem datacentre (`10.10.0.0/16`) and an AWS VPC (`10.0.0.0/16`), connected by both a Direct Connect (dedicated fibre, low latency, expensive, 10 Gbps) and a Site-to-Site VPN over the internet (cheap, higher latency, ~1.25 Gbps) as backup. A new team stands up an application in the VPC that must reach an on-prem database. It works from some subnets and not others, and when it does work, throughput is one-tenth of expectations. Design the routing, and explain the symptom.

**Requirements.** Prefer Direct Connect; fail over to VPN automatically; on-prem must reach VPC and vice versa; no asymmetry.

**The alternatives for how routes get into each side's table.**

*Static routes.* Hand-write `10.0.0.0/16 → Direct Connect` on-prem and `10.10.0.0/16 → Virtual Private Gateway` in the VPC. Dead simple, completely predictable, and it does not fail over — a static route is up as long as the *interface* is up, and a Direct Connect can be logically down while the port is physically up. You would need a separate mechanism to withdraw it.

*BGP over both links.* Both sides run BGP; each advertises its own prefixes; both learn the other's dynamically. AWS's Direct Connect and Site-to-Site VPN both speak BGP for exactly this reason.

**The decision: BGP on both, with path preference — and this is where the bug lives.** Advertise from AWS with `AS_PATH` prepending on the VPN so the Direct Connect path looks shorter; on-prem, prefer the Direct Connect via local preference. AWS's documented route-selection order for a VPC route table is: **longest prefix match first**, then, among equal-length prefixes, static routes over dynamic, then Direct Connect BGP over VPN BGP.

Now the bug. The team's on-prem network team, wanting to be helpful, advertised `10.10.0.0/16` over Direct Connect but *also* advertised a more-specific `10.10.5.0/24` (the database subnet) over the **VPN**, because that's the link they were testing on. Longest prefix match does not care about your intent or your path preference. Every packet from the VPC to the database took the VPN. Return traffic from the database, following the on-prem table, took the Direct Connect. **Asymmetric routing** — out over the slow link, back over the fast one.

Why that produces the exact reported symptoms: throughput is capped by the slow direction (TCP's data path is the slow one), and any **stateful** middlebox on either path — a firewall, a NAT device, an IPS — sees only one direction of the conversation, never the handshake plus the reply, and drops the flow as invalid. That is why it "works from some subnets and not others": only the subnets whose traffic happens to traverse a stateful device break.

**The trade-off of the fix.** The fix is to advertise consistently: the same prefixes over both links, differing only in *preference*, never in *specificity*. That costs you the ability to do fine-grained per-subnet traffic steering with more-specifics — a real capability you are giving up. **Flip condition:** if you genuinely need per-subnet path steering (e.g. bulk backup traffic must use the cheap link while transactional traffic uses the expensive one), you can use more-specifics deliberately — but then you must engineer symmetry explicitly on *both* sides and remove or make stateless every middlebox in the path. In practice most organisations should use separate connections or QoS for that, not asymmetric prefixes.

**Failure modes.**

- **Overlapping CIDRs.** If on-prem and VPC both used `10.0.0.0/16`, no routing can fix it — a router cannot distinguish "my 10.0.1.5" from "their 10.0.1.5." This is the single most common and most expensive VPC-planning mistake, and it is why the Build below starts with an IP-space allocation registry. The only remedies are re-addressing (weeks of work) or double NAT (permanent operational pain).
- **Route limits.** A VPC route table has a quota on propagated routes (50 by default, raisable to 1000 for Direct Connect gateway associations — **verify current**). An on-prem network that advertises 400 un-aggregated prefixes will silently blow it, and the failure looks like "some destinations unreachable."
- **Prefix leaking.** On-prem advertises a default route into the VPC. Now VPC internet traffic goes on-prem. Sometimes intentional (centralised egress inspection); usually a surprise.
- **Direct Connect flaps but BGP holds.** BGP's default hold timer is 90 seconds — that's 90 seconds of blackholing. Use BFD (Bidirectional Forwarding Detection) for sub-second detection.

*(Cross-reference: Day 9 §12 covered which tiers may talk to which — the security-group and NACL layer. This scenario is deliberately the layer underneath: whether a packet has a *path* at all. A correct security group over a broken route is still a timeout.)*

---

## 5. TTL: the eight bits that keep mistakes from becoming permanent

**Depth: [CORE]**

### Intuition — what happens in a world without it

§4 established that routers make independent local decisions from tables they were told about by other routers. Now ask: what happens when two of those tables disagree?

Router X believes the way to `203.0.113.5` is through Y. Router Y believes it is through X. A packet arrives at X. X forwards to Y. Y forwards to X. X forwards to Y. At 100 Gbps, on a 10-microsecond link, that packet crosses the link roughly 100,000 times per second, **forever**. And it is not one packet — every packet destined for that prefix joins the loop. Within a second the link is carrying a self-sustaining, ever-growing mass of packets that will never be delivered and will never leave, consuming the entire link capacity and starving all legitimate traffic. The loop does not just fail to deliver those packets; it destroys the link for everyone.

Nothing in §4's algorithm detects this. A router has no memory of packets it has seen (step 10: "forget everything"), so it cannot notice it has seen this one before. Neither router is malfunctioning; each is faithfully applying its table. And routing-table disagreement is not an exotic failure — it happens routinely and *by design* during **convergence**: when a link goes down, the routers around it learn at slightly different times, and for a few hundred milliseconds to a few seconds, their views are inconsistent. Transient loops during convergence are normal.

So the internet needs a mechanism that (a) requires no per-packet state anywhere, (b) requires no coordination between routers, and (c) guarantees that any packet, however badly mis-routed, eventually stops existing.

**The mechanism is one byte in the header: a counter that every router decrements, and a rule that at zero the packet is destroyed.** That is TTL. It is the cheapest possible solution to the problem, and it is one of the most elegant pieces of engineering in the whole stack: eight bits, one subtraction per hop, and an entire class of catastrophic failure becomes self-limiting.

### Analogy — the parcel with a punch-card, and where it breaks

Imagine a parcel with a card stapled to it reading "**30**". Every depot that handles it must cross out the number and write one less. Any depot that receives a parcel reading "1" must not forward it — it destroys the parcel and sends a note back to the sender saying "this expired here, at my depot." A parcel caught in a loop between two depots therefore survives at most 30 handoffs before it is destroyed. No depot needs to remember any parcel, and no two depots need to agree on anything.

**Where the analogy breaks — three ways, and all three are load-bearing.**

1. **The depot sends a note to the *sender*, and that note is the whole reason traceroute exists.** In the analogy it's a courtesy. In IP it is the *only* channel through which the internet reports its own topology, because §4 established that nothing records the path. §6 is built entirely on harvesting these notes.
2. **The name is a lie, and RFC 791 wrote the lie into the spec.** RFC 791 §3.2 says TTL "indicates the maximum time the datagram is allowed to remain in the internet system" and is measured in **seconds** — routers were expected to decrement by the number of seconds they held the packet, or by 1 at minimum. The spec's own hedge is the sentence that survived: "every module that processes a datagram must decrease the TTL by at least one even if it process the datagram in less than a second." In practice, no router has ever measured queuing time for this purpose, so *every* implementation decrements by exactly 1 and TTL is purely a **hop count**. IPv6 stopped pretending and renamed the field **Hop Limit** (RFC 8200 §3: "8-bit unsigned integer. Decremented by 1 by each node that forwards the packet"). You must know both the honest behaviour and the misleading name, because the name appears in every tool's output.
3. **A parcel destroyed at a depot is gone but the *shipment* isn't; the sender re-sends. IP does not re-send.** TTL expiry is a silent-to-the-application packet loss. Nothing at the IP layer retries. Recovery, if any, is TCP's job (Day 11) — and the ICMP note that comes back is *informational*, not a retransmission trigger. Which means a persistent routing loop presents to your application as a plain timeout, with the *real* cause visible only if someone thinks to look for ICMP Time Exceeded.

### Under the hood — the precise semantics

Three rules, and getting them exactly right is what separates understanding from vibes.

**Rule 1: the field is 8 bits, so the maximum value is 255.** Not 256 — the range is 0–255. RFC 791's framing gives a maximum datagram lifetime of about 4.25 minutes (255 seconds), which is the origin of TCP's `2*MSL` TIME_WAIT reasoning (Day 11).

**Rule 2: the sender chooses the initial value; the OS picks it.** Common defaults, which are stable enough to be useful and non-standard enough to be untrustworthy:

| Initial TTL | Typical origin |
|---|---|
| 64 | Linux, macOS, Android, most BSDs, most network appliances |
| 128 | Windows |
| 255 | Cisco IOS and many router-originated packets; also required for some protocols (see GTSM below) |
| 1 | Deliberately link-local — will not survive one hop |

**Rule 3: the decrement-and-die rule, stated exactly.** A router *about to forward* a packet decrements TTL. If the result would be 0, it must **not** transmit the packet; it discards it and sends **ICMP Time Exceeded, type 11, code 0** ("time to live exceeded in transit", RFC 792) back to the source address. Practically: a packet arriving with **TTL = 1 dies at that router**. A packet arriving with TTL = 2 is forwarded with TTL = 1 and dies at the *next* router.

The ICMP error is not empty. RFC 792 specifies that an ICMP error carries back the **original packet's IP header plus the first 64 bits of its payload**, "used by the host to match the message to the appropriate process." Those 64 bits are enough to contain TCP or UDP source and destination ports, which is how your OS knows *which* of your connections the error refers to — and, as §6 shows, how traceroute matches a reply to the probe that caused it.

One subtlety worth internalising: **the ICMP Time Exceeded message is a brand-new packet, generated by the router, with the router's own source address and its own fresh TTL.** It travels back along whatever return path *its* destination (you) has, which may be completely different from the forward path. This is why traceroute measures a round trip through two possibly-different paths, and it is the root of half the confusions in §6.

### Runnable example — watch TTL kill a packet, one hop at a time, by hand

This is the demonstration that makes TTL concrete, and it needs no special tooling. `ping` lets you set the initial TTL directly, so you can execute traceroute's core trick manually and watch a different router die each time.

```powershell
# Windows PowerShell or cmd — ping is built in. -i sets initial TTL, -n sets probe count.
ping -i 1 -n 1 8.8.8.8
ping -i 2 -n 1 8.8.8.8
ping -i 3 -n 1 8.8.8.8
```

```bash
# Linux / macOS equivalent (note: -t on Linux, -m on macOS, and -c not -n):
#   ping -t 1 -c 1 8.8.8.8      # Linux
#   ping -m 1 -c 1 8.8.8.8      # macOS
```

**Real captured output** (this machine, 2026-08-11, Mumbai, Reliance Jio):

```
=== TTL=1 ===
Pinging 8.8.8.8 with 32 bytes of data:
Reply from 192.168.29.1: TTL expired in transit.

=== TTL=2 ===
Pinging 8.8.8.8 with 32 bytes of data:
Reply from 10.50.136.1: TTL expired in transit.

=== TTL=3 ===
Pinging 8.8.8.8 with 32 bytes of data:
Reply from 172.31.2.18: TTL expired in transit.
```

**Why this works, line by line.**

`ping -i 1` builds an ICMP Echo Request addressed to `8.8.8.8` with the IP header's TTL field set to **1**. The first router to handle it is my home gateway, `192.168.29.1`. It applies Rule 3: decrementing 1 gives 0, so it must not forward. It discards the Echo Request and generates an **ICMP Time Exceeded (type 11, code 0)** whose source address is its own. Windows prints that as "Reply from 192.168.29.1: TTL expired in transit." Note carefully what did **not** happen: `8.8.8.8` never saw this packet and sent nothing. The reply is from a router that killed my packet, and it identified itself in the act of doing so.

`ping -i 2` sets TTL=2. The home gateway decrements it to 1 and forwards. The next router, `10.50.136.1`, receives TTL=1, must not forward, and reports itself. `ping -i 3` reaches `172.31.2.18` by the same logic.

**Now check this against §4's real routing table and §6's traceroute, because the cross-check is the point.** `192.168.29.1` is exactly the next hop in the `0.0.0.0/0` route on this machine — the default gateway, confirmed independently. And the sequence `192.168.29.1, 10.50.136.1, 172.31.2.18` is precisely hops 1, 2, 3 of every `tracert` in §6. Three independent tools agree, because they are all reading the same mechanism.

**One thing to notice that teaches something extra:** `10.50.136.1` and `172.31.2.18` are RFC 1918 *private* addresses (§3), and they belong to my ISP, not to me. My ISP has numbered its internal transit links out of private space — which is legal, common, and tells you something real: the ISP is short of public addresses, and I am several hops of ISP infrastructure away from any publicly-routable address. The first public address in the path doesn't appear until hop 10. That is the shape of a network operating under IPv4 exhaustion (§8) and it is visible for free in a ping.

### TTL as a fingerprint — and a worked example of the heuristic being wrong

Because initial TTLs cluster on 64/128/255, you can *estimate* how many hops away a host is from the TTL value in its reply: take the arriving TTL, round **up** to the nearest common initial value, subtract.

Here is the measurement, and it does not come out the way the folklore promises. From the same machine, same minute:

```
ping -f -l 1472 -n 2 8.8.8.8
# -> Reply from 8.8.8.8: bytes=1472 time=20ms TTL=111
```

Arriving TTL is **111**. Round up to the nearest of {64, 128, 255} → 128. Inferred hop count: 128 − 111 = **17 hops**.

And the actual measurement:

```
tracert -h 30 -w 1200 -d 8.8.8.8
# ->  ... 13    20 ms    21 ms    22 ms  8.8.8.8
# -> Trace complete.
```

**Thirteen hops.** The heuristic said 17. It is wrong by four, and I am reporting it wrong rather than quietly picking a nicer example, because *why* it's wrong is the actual lesson. Three candidate explanations, none of which I can distinguish from outside:

1. **The return path is longer than the forward path.** TTL decrements on the way *back*, so the reply's TTL measures the return hop count, and §4 established that asymmetric routing is normal. Traceroute measures the forward path. There is no contradiction; they are measuring two different things.
2. **The forward path has hidden hops.** Notice that hops 8 and 9 timed out in every trace from this machine. Routers inside an MPLS tunnel commonly do not decrement TTL visibly or do not generate ICMP, so the *real* forward hop count can exceed the number of lines traceroute prints.
3. **Google's responder may not use a standard initial TTL.** 111 is above 64, so the initial value must be at least 112; 128 is the natural guess but nothing enforces it, and some load balancers and appliances rewrite TTL.

**The takeaway is the calibration, not the number.** TTL-based hop estimation is a rough, one-sided, easily-defeated heuristic. It is genuinely useful for one thing — spotting when a "single host" is actually several different machines behind a load balancer, because replies arriving with different TTLs cannot have come from the same box. It is not useful as a measurement. (It is also a mild information leak: TTL 128 suggests Windows, 64 suggests Linux, which is why `nmap -O` uses it as one signal among many.)

### Two real uses of TTL beyond loop prevention

**Depth: [AWARE]** — know these exist and why; treat the details as a black box unless a project forces you deeper.

**TTL=1 as a scope limiter.** Set the initial TTL to 1 and your packet is guaranteed not to leave the local link. This is how link-local protocols enforce their scope — mDNS/Bonjour, some multicast discovery, and various routing-protocol hellos rely on it. It is a *scoping* mechanism built out of a loop-prevention mechanism, which is a nice example of a primitive being reused for something its designers didn't intend.

**TTL=255 as an authentication primitive — GTSM (RFC 5082).** This one inverts the usual logic and is worth understanding because it is genuinely clever. Suppose a router wants to accept BGP connections *only* from its directly-attached neighbour, and reject anything from further away — a real concern, since BGP session hijacking is an attack (§11). The trick: require the neighbour to send with TTL = **255**, and configure the receiver to reject any such packet arriving with TTL **< 254**. Since TTL can only ever *decrease* in transit and cannot be increased, a packet arriving with 255 or 254 provably traversed at most one hop. An attacker three hops away cannot forge this, because they cannot make TTL go up. This is called the Generalized TTL Security Mechanism (GTSM), and it is the standard hardening for eBGP sessions. It is a lovely demonstration that a monotonically-decreasing counter is, incidentally, a distance proof.

*(Honest gap: [CORE] concepts nominally owe two named case studies including a failure, and **TTL does not have one** — there is no famous "TTL outage," precisely because TTL works and has never needed changing since 1981. Rather than manufacture one, I'll note what is real: the incidents that TTL contains are the routing-loop halves of the §11 and §4 case studies, where mis-routed traffic was blackholed rather than melting the links it looped on, and the named engineering artifact built on TTL is GTSM (RFC 5082), in production on eBGP sessions worldwide. Absence of a postmortem here is evidence about the design, not a gap in the research.)*

### System design ③ — detecting and ending a routing loop in production

**The problem.** Your monitoring shows that one specific internal service, `payments-api`, has become unreachable from one specific subnet, `10.0.32.0/19`. Other subnets reach it fine. The service is healthy. There is no packet loss on any interface graph. Design the diagnosis, and design the guardrails that would have caught it automatically.

**Requirements.** Distinguish a routing loop from the four things it looks like — a firewall drop, a blackhole (route to nowhere), an MTU black hole, and an application hang. Diagnose without root access to the routers. Then prevent recurrence.

**The alternatives for diagnosis.**

*Read every routing table.* The definitive answer, and often unobtainable — the loop may involve a device owned by another team, a cloud transit gateway whose table you can only partially see, or a VPN appliance nobody has credentials for. It is also slow at 3 a.m.

*Infer from ICMP.* Run a traceroute from inside the affected subnet. A routing loop has an unmistakable signature that none of the other four failure modes produce: **the same small set of addresses repeats, cycling, until the TTL budget runs out.**

```
  8   12 ms   11 ms   12 ms  10.0.0.9
  9   13 ms   12 ms   13 ms  10.0.0.17
 10   24 ms   23 ms   24 ms  10.0.0.9      <- seen before
 11   25 ms   24 ms   25 ms  10.0.0.17     <- and again
 12   36 ms   35 ms   36 ms  10.0.0.9
 ...
 30   ...                    10.0.0.17
```

Read three things off that. The repetition names the two devices whose tables disagree, so you know exactly whose configuration to look at. The **latency climbing linearly** — roughly +12 ms per hop pair — is the loop latency accumulating, and it confirms the packets are genuinely traversing the link repeatedly rather than the tool being confused. And the trace terminating at the hop cap rather than at the destination confirms nothing is delivering.

Contrast the signatures, because this is the actual diagnostic value: a **blackhole** shows a normal path then `* * *` forever with no repetition and no rising latency. A **firewall drop** typically shows the path completing to the firewall and then `!X` (administratively prohibited, ICMP type 3 code 13) or silence at exactly one consistent hop. An **MTU black hole** (Day 9 §9) lets small packets and traceroute through perfectly and only kills large ones — so traceroute looks *clean*, which is the tell. An **application hang** shows a complete traceroute and a successful TCP connect, with the failure only at the HTTP layer.

**The decision: ICMP inference first, table inspection second, aimed by what ICMP told you.** The reason is time-to-diagnosis under the specific constraint "no root access to the routers": traceroute from the affected subnet costs 30 seconds and requires nothing, and it converts "something is wrong somewhere" into "these two devices disagree about this prefix." **Trade-off:** ICMP inference can be defeated — a device that doesn't generate ICMP Time Exceeded (deliberately, or because of rate-limiting) makes the loop invisible, and you will see `* * *` and misdiagnose it as a blackhole. **Flip condition:** if traceroute shows a clean `* * *` and you have any reason to suspect a loop anyway, go straight to interface counters — a loop shows as *inbound and outbound packet rates on one link far exceeding the offered traffic*, which is a signature ICMP suppression cannot hide.

**The guardrails, which are the more valuable half of the answer.**

- **Alert on ICMP Time Exceeded volume, not just on loss.** Every routing loop generates a flood of type-11 messages back toward the sources. Most shops do not monitor this at all. It is a nearly-zero-false-positive early-warning signal, and in cloud environments it is available: VPC Flow Logs record ICMP, and you can alert on a rate change.
- **Alert on link utilisation that exceeds plausible offered load.** A loop manufactures traffic. If a link is at 90% and your applications account for 5%, you have found it.
- **Continuous synthetic path monitoring.** Run traceroute-style probes on a schedule between every pair of subnets and diff the paths. A path that suddenly grows or starts repeating is a page. This is what commercial tools (ThousandEyes, Catchpoint) sell, and a cron job with `mtr --report` covers 80% of it.
- **Prevent the class, not the instance:** loops in a single administrative domain almost always come from mixing static routes with a dynamic protocol, or from redistributing routes between two protocols without filtering. The structural fix is to have exactly one source of truth per prefix. In cloud terms: do not hand-add static routes to a route table that also receives propagated BGP routes for overlapping space — that is §4's asymmetric-return-path trap with a worse ending.

---

## 6. traceroute: interrogating a network that keeps no records

**Depth: [WORKING]**

### Intuition — turning a safety mechanism into an instrument

§4 established that the path is written down nowhere. §5 established that a router which kills a packet *identifies itself* in an ICMP error. Put those together and there is a way to extract the path from a system designed not to store it:

**Send a packet with TTL=1. The first router kills it and reports its own address. Send TTL=2. The second router reports. Send TTL=3. And so on, until a probe survives all the way to the destination.** Collect the reporters in order and you have the forward path. Time each round trip and you have latency to each point along it.

Van Jacobson wrote this in 1987 and it is one of the great pieces of tooling in computing: it required no new protocol, no cooperation from anyone, no changes to any router. It simply noticed that an existing safety mechanism was, viewed sideways, a topology-discovery service. You already ran it by hand in §5's `ping -i` demo — traceroute is just that loop automated with three probes per TTL and reverse-DNS lookups on the results.

### The probe-protocol problem, which is why the same command shows different things on different OSes

Here is a detail that looks like trivia and is not: **the TTL trick works with any kind of packet, and different implementations chose differently.** This changes what you see, because firewalls filter by protocol and port.

| Implementation | Default probe | How it detects "arrived" |
|---|---|---|
| **Windows `tracert`** | **ICMP Echo Request** (type 8) | Destination replies with Echo Reply (type 0) |
| **Classic Unix `traceroute`** (Linux, BSD, macOS) | **UDP** to high ports, starting at **33434**, incrementing per probe | Destination replies ICMP Port Unreachable (type 3, **code 3**) |
| `traceroute -I` | ICMP Echo (like Windows) | Echo Reply |
| `traceroute -T` / `tcptraceroute` | **TCP SYN** to a chosen port (usually 80 or 443) | Destination replies SYN/ACK or RST |
| `mtr` | ICMP by default, `-u` for UDP, `-T` for TCP | as above, but continuously, with loss statistics |

Note the elegance of the classic Unix choice: it deliberately sends UDP to a port in the 33434–33534 range precisely *because* nothing is listening there, so the destination is guaranteed to answer "port unreachable" — which is how traceroute knows it arrived rather than just timed out. The incrementing port also lets it match replies to probes (remember from §5 that the ICMP error carries the first 64 bits of the original payload, which includes the UDP ports).

**Why this matters operationally, not academically.** A firewall configured to permit ICMP Echo but drop unsolicited UDP will let `tracert` from Windows complete while `traceroute` from Linux dies at the firewall. The reverse is also common: many networks rate-limit or block ICMP Echo (as a DDoS-hygiene reflex) while passing UDP. **So "traceroute fails" is not a fact about the network — it is a fact about the network *and* your probe protocol.** When a trace dies, the first move is to try a different probe type:

```powershell
# Windows: ICMP is the only built-in option for tracert.
tracert -d -h 20 example.com
# For TCP-based tracing on Windows, use the PowerShell test or install nmap/WinMTR:
Test-NetConnection -TraceRoute -Hops 20 -ComputerName 1.1.1.1
```

```bash
# Linux / macOS: change the probe protocol when a trace dies.
traceroute       -n -m 20 example.com     # default: UDP 33434+
traceroute -I    -n -m 20 example.com     # ICMP Echo, like Windows
traceroute -T -p 443 -n -m 20 example.com # TCP SYN to 443 - usually gets furthest
mtr -r -c 10 example.com                  # continuous, with per-hop loss %
```

**The one that almost always gets furthest is `-T -p 443`,** because a TCP SYN to 443 looks exactly like the beginning of a legitimate HTTPS connection, which almost no firewall blocks. If you can only remember one flag, remember that one. (It needs root/administrator, because it crafts raw packets.)

**Honesty note about your Windows environment.** Windows has no `traceroute`; the command is `tracert`, and it has no option to change probe protocol — it is ICMP Echo, always. `Test-NetConnection -TraceRoute` is the PowerShell-native form and also uses ICMP; it returns a clean list of addresses with no timings, which is convenient for scripting and useless for latency work. Verified on this machine:

```powershell
Test-NetConnection -TraceRoute -Hops 12 -ComputerName 1.1.1.1 |
    Select-Object -ExpandProperty TraceRoute
```

```
# -> 192.168.29.1
# -> 10.50.136.1
# -> 172.31.2.18
# -> 192.168.59.120
# -> 172.26.74.71
# -> 172.26.75.130
# -> 49.44.66.194
# -> 49.44.66.195
# -> 1.1.1.1
```

If you want Unix-style probe control on Windows, install `nmap` (which brings `nping`) or WinMTR, or run `traceroute` inside WSL2 — but note that WSL2's NAT adds a hop and can distort the first two lines. Git Bash does **not** provide `traceroute`; it forwards to the Windows binaries, so `traceroute` in Git Bash is "command not found" while `tracert` works.

### Real captures — three continents, and what each one teaches

These are genuine captures from the machine this note was written on: **Windows 11, Wi-Fi, Reliance Jio, Mumbai region, 11 August 2026**, with `tracert -h N -w 1200`. They are real, not illustrative. **Your own output will differ completely** — different ISP, different peering, different CDN mapping — and that is the point of the Build: you will run these and read *yours*.

**Capture A — Europe. `speedtest.tele2.net` (Sweden), 23 hops, ~205 ms.**

```
Tracing route to speedtest.tele2.net [90.130.70.73] over a maximum of 24 hops:

  1     1 ms     1 ms     1 ms  reliance.reliance [192.168.29.1]      <- my home router
  2     8 ms     5 ms     4 ms  10.50.136.1                           <- ISP access, RFC1918
  3     9 ms     9 ms    13 ms  172.31.2.18                           <- ISP, RFC1918
  4     8 ms    11 ms    10 ms  192.168.59.126                        <- ISP, RFC1918
  5    10 ms    11 ms     9 ms  172.26.74.71
  6     9 ms    12 ms    10 ms  172.26.75.130
  7    11 ms    10 ms    15 ms  192.168.60.232                        <- still private!
  8     *        *        *     Request timed out.                    <- hop hidden
  9     *        *        *     Request timed out.                    <- hop hidden
 10    31 ms    31 ms    31 ms  103.198.140.176                       <- first public address
 11   159 ms   175 ms   159 ms  103.198.140.45                        <- +128ms: the ocean
 12     *        *        *     Request timed out.
 13   208 ms   175 ms   140 ms  ...ccr91.lhr01.atlas.cogentco.com     <- lhr = London Heathrow
 14     *      152 ms   151 ms  be3464.ccr81.lon05.atlas.cogentco.com <- lon = London
 15   150 ms   150 ms   150 ms  be3530.ccr41.ams03.atlas.cogentco.com <- ams = Amsterdam
 16   162 ms   162 ms   163 ms  be2815.ccr41.ham01.atlas.cogentco.com <- ham = Hamburg
 17   178 ms   176 ms   178 ms  be2496.rcr21.cph01.atlas.cogentco.com <- cph = Copenhagen
 18   172 ms   172 ms   223 ms  be3090.rcr71.mmx01.atlas.cogentco.com <- mmx = Malmo
 19   156 ms   158 ms   156 ms  149.6.48.42
 20   168 ms   167 ms   168 ms  brn-core-1.bundle-ether26.tele2.net
 21   168 ms   167 ms   167 ms  bck3-core-1.bundle-ether1.tele2.net
 22   176 ms   176 ms   167 ms  gimp-peer-2.ae1-unit0.tele2.net
 23   205 ms   214 ms   208 ms  d90-130-70-73.cust.tele2.se [90.130.70.73]

Trace complete.
```

What to read here. **Hops 1–7 are all RFC 1918 private addresses** — seven hops of ISP infrastructure numbered out of private space (§3, §8). **Hop 10→11 jumps from 31 ms to 159 ms**: +128 ms in one hop is not a slow router, it is *distance*. Light in fibre travels about 200,000 km/s, so 128 ms of one-way... no — 128 ms of *round trip* is ~64 ms one way, ~12,800 km of fibre. Mumbai to London is ~7,200 km great-circle, and fibre routes are never great-circle. The arithmetic confirms this hop crossed an ocean. **This is Day 1's latency pyramid and Day 9's propagation delay showing up as a number you can measure.** And notice: it is one *hop*, not one router being slow. You cannot fix propagation delay with better hardware.

**Hops 13–18 read out a geography lesson from reverse-DNS**: London → Amsterdam → Hamburg → Copenhagen → Malmö. Operators name router interfaces with airport/city codes, and learning to read them (`lhr`, `ams`, `fra`, `sin`, `nrt`, `iad`, `sjc`) turns a list of IPs into a map. **Verify current** — naming conventions are conventions, and a `lhr01` name occasionally survives a physical relocation.

**Capture B — Asia. `www.iij.ad.jp` (Japan), 19 hops, ~138 ms.**

```
Tracing route to www.iij.ad.jp [202.232.2.180] over a maximum of 24 hops:

  1     2 ms     2 ms     2 ms  reliance.reliance [192.168.29.1]
  ... (hops 2-9 as before: private ISP hops, then two timeouts)
 10    21 ms    23 ms    21 ms  103.198.140.64
 11    53 ms    55 ms    53 ms  103.198.140.89                        <- +32ms: Mumbai -> Singapore
 12    54 ms    54 ms    53 ms  sng-b7-link.ip.twelve99.net           <- sng = Singapore (Telia)
 13     *        *       53 ms  sng-b5-link.ip.twelve99.net           <- 2 of 3 probes lost
 14    54 ms    53 ms    53 ms  202.232.1.69
 15    52 ms    51 ms    52 ms  sgp001bb00.IIJ.Net                    <- handoff to IIJ, Singapore
 16   120 ms   118 ms   125 ms  tky009bb10.IIJ.Net                    <- tky = Tokyo (+66ms)
 17   139 ms   138 ms   138 ms  osk011bb00.IIJ.Net                    <- osk = Osaka (+19ms)
 18   131 ms   130 ms   129 ms  osk008agr02.iij.net                   <- LOWER than hop 17
 19   138 ms   138 ms   137 ms  www.iij.ad.jp [202.232.2.180]

Trace complete.
```

Read hop 13: **two of three probes timed out, and the third answered in 53 ms.** The router is fine and the path is fine. This is ICMP **rate-limiting** — routers cap how many ICMP errors per second they will generate, because generating them costs CPU on the control plane, and a router's job is forwarding, not answering questions. Under load, or when many people traceroute through the same box, some probes get no reply. If you see partial loss at a *middle* hop and full success at hops beyond it, the loss is in ICMP generation, not in forwarding.

Read hops 17→18: **139 ms then 131 ms.** Latency went *down* as we got further away. This is the single most misread feature of traceroute output, and §"How to read your own" below explains it properly.

**Capture C — North America. `www.mit.edu`... which never left Asia.**

```
Tracing route to e9566.dscb.akamaiedge.net [23.47.244.83] over a maximum of 24 hops:

  1     1 ms     2 ms     2 ms  reliance.reliance [192.168.29.1]
  ... (hops 2-9: private ISP hops, then two timeouts)
 10    22 ms    20 ms    21 ms  49.44.187.5
 11    21 ms    21 ms    21 ms  a23-47-244-83.deploy.static.akamaitechnologies.com [23.47.244.83]

Trace complete.
```

**This is the most instructive capture of the three, precisely because it failed at the assigned task.** I asked for a trace to a university in Massachusetts. I got 11 hops and 21 ms. Mumbai to Boston is ~12,500 km; the *speed of light in fibre* alone makes a round trip about 125 ms. 21 ms is physically impossible for a trans-Pacific or trans-Atlantic path. So the packet did not go to Massachusetts.

Look at the first line: `www.mit.edu` resolved to `e9566.dscb.akamaiedge.net [23.47.244.83]`, and hop 11's reverse DNS is `deploy.static.akamaitechnologies.com`. MIT's website is served from Akamai's CDN, and Akamai's DNS handed my resolver the address of an edge server *in or near India*. I traced to a cache 21 ms away that happens to serve MIT's content.

Three lessons, all real, none manufacturable:

1. **You cannot traceroute to a CDN-hosted site and learn anything about where its origin is.** By design. The whole product is "the answer to `where is this?` is `near you`." Day 15 owns CDNs; Day 12 owns the DNS half.
2. **Always read the first line of traceroute output.** It shows what the hostname resolved to. Half of all traceroute misinterpretation is skipping that line. The resolved name told me the answer before hop 1.
3. **To trace to a specific *place*, pick a target that isn't behind a CDN or anycast** — a university's non-web service, an ISP's own router, a research network, a speedtest server. This is exactly why I used `speedtest.tele2.net` and `www.iij.ad.jp` for A and B: both are unicast, both are operated by the network operator itself.

**Capture D (bonus) — the trace that shows BGP paths are not geography. `route-views.routeviews.org` (University of Oregon).**

```
 10    21 ms    21 ms    25 ms  103.198.140.64
 11    99 ms    98 ms    98 ms  103.198.140.145
 12    98 ms    99 ms    97 ms  103.198.140.145        <- SAME address twice
 13     *       79 ms    80 ms  ix-...-singapore.as6453.net  <- 80ms: LOWER than hop 11-12
 14     *        *        *     Request timed out.
 15     *        *        *     Request timed out.
 16    97 ms     *        *     if-...-singapore.as6453.net  <- replied after two dead hops
 17   299 ms   273 ms   274 ms  ae-72.a02.sngpsi07.sg.bb.gin.ntt.net   <- Singapore (NTT)
 18   277 ms   277 ms   276 ms  ae-2.r25.sngpsi07.sg.bb.gin.ntt.net
 19   271 ms   284 ms   270 ms  ae-13.r35.tokyjp05.jp.bb.gin.ntt.net   <- Tokyo
 20     *      283 ms     *     ae-1.r34.tokyjp05.jp.bb.gin.ntt.net
 21   274 ms   276 ms   275 ms  ae-4.r27.snjsca04.us.bb.gin.ntt.net    <- San Jose, CA
 22   283 ms   282 ms   282 ms  ae-23.a04.snjsca04.us.bb.gin.ntt.net
 23   385 ms   380 ms   384 ms  ce-0-2-1.a04.snjsca04.us.ce.gin.ntt.net <- 385ms, then...
 24   293 ms   291 ms   291 ms  208.56.232.50                           <- ...293ms. Down 92ms.
 25   264 ms   265 ms   265 ms  bend-valt-pe-01.net.linkoregon.org      <- Bend, Oregon
 26   264 ms   263 ms   264 ms  bend-valt-pe-02.net.linkoregon.org
```

**The path goes east.** Mumbai → Singapore → Tokyo → San Jose → Oregon. Westward (Mumbai → Europe → Atlantic → US East → US West) is not obviously longer in kilometres, but BGP does not choose by kilometres — it chooses by AS-path length and business policy (§11). My ISP's relationship with NTT routes Pacific-ward, so that is where my packets go, regardless of geography.

This capture also contains four of the five reading-hazards in one screen, which is why I kept it. **Hops 11 and 12 show the same IP address twice.** That is not a loop (a loop repeats and latency climbs — see §5's system design ③; here latency is flat and the path progresses). It is one router replying for two different TTL values, which happens with MPLS tunnels, with routers that use a single loopback as the source for all ICMP, and with certain load-balanced paths. **Hops 14 and 15 are fully dead but hop 16 answers** — proof that `* * *` does not mean "the path stops here." **Hop 23 is 385 ms and hop 24 is 293 ms** — a 92 ms *decrease*. And **the trace never reached `128.223.51.103`**; it ran out of my `-h 26` budget while still inside Oregon's regional network. Everything I set out to measure was measured; nothing about it looked tidy. Real network data is like this.

### How to read your own traceroute output — the five things people get wrong

This is the part to internalise, because you will read hundreds of these and the failure modes are always the same five.

**1. `* * *` does **not** mean the path is broken.** It means "no ICMP Time Exceeded came back from the router at this TTL, within the timeout." Reasons, roughly in order of frequency: the router is configured not to generate ICMP errors (common in security-conscious networks and MPLS cores); the router rate-limited this particular probe; the *reply* was dropped or filtered on the way back; the router is inside an MPLS tunnel and doesn't decrement visibly. Capture D's hops 14–15 are the proof: dead hops, live path. **The only `* * *` that means something is broken is one that continues to the end of the trace *and* the destination never answers.**

**2. Latency can go down as hops go up, and that is not a paradox.** Capture B hop 17→18 (139→131 ms) and Capture D hop 23→24 (385→293 ms) both show it. Three real causes: (a) **hop RTT is not that hop's latency** — see point 3; (b) **ICMP generation is a control-plane task and is deprioritised**, so a busy router adds tens of milliseconds of *its own processing* before answering, while forwarding your real traffic at line rate — that router looks slow and isn't; (c) **the return path differs per hop**, since each router's ICMP reply follows its own route back to you.

**3. A hop's number is the round-trip time to *that router*, not the latency of that *link*.** People subtract consecutive hops to get per-link latency. That is usually wrong, for the reasons in point 2 — you'd be subtracting two measurements taken over two different return paths with two different amounts of control-plane delay. Cumulative RTT to a hop is roughly meaningful; the difference between two hops is noise plus signal in unknown proportion. Trust *big* jumps (like Capture A's +128 ms, which is an ocean and cannot be anything else) and distrust small ones.

**4. You are seeing the forward path only, and only for these probes.** The return path is invisible and frequently different. Worse, ECMP (§4) means consecutive probes at the *same* TTL can traverse different routers, so a traceroute can interleave two paths and show you a chimera. The fix, if you care, is **Paris traceroute** / `traceroute --paris`, which holds the ECMP hash inputs constant across probes so all probes follow one path. `mtr` is also more informative than a single traceroute, because it keeps probing and reports per-hop loss percentages over time — and per-hop loss that *does not persist to later hops* is ICMP rate-limiting, while loss that persists to the end is real.

**5. Reverse DNS is a hint, not a fact.** `lhr01` almost certainly means London, `sin`/`sng` Singapore, `nrt` Tokyo Narita. But PTR records are set by humans, go stale, and survive relocations. Treat them as strong evidence combined with latency arithmetic (if the RTT is 21 ms, it is not in another hemisphere, whatever the name says), and never as proof on their own.

**A sixth, for cloud work:** inside a cloud VPC, traceroute is often *deliberately* useless. AWS's network does not generate ICMP Time Exceeded for its internal fabric, so a trace between two EC2 instances typically shows the source, `* * *`, and the destination. That is not a broken network; it is a hidden one. Use VPC Reachability Analyzer and Flow Logs instead — they are the cloud-native replacement for exactly this diagnostic gap, and the reason they exist is that §4's "forget everything about this packet" was made even more thorough by SDN overlays.

### The Build — trace three continents and read every hop

**Do this.** Run all three, on your own machine, and save the output.

```powershell
# Pick targets that are NOT behind a CDN, or you will get Capture C's lesson instead.
tracert -h 30 -w 1500 speedtest.tele2.net       # Europe  (Sweden)
tracert -h 30 -w 1500 www.iij.ad.jp             # Asia    (Japan)
tracert -h 30 -w 1500 route-views.routeviews.org  # N. America (Oregon)

# Then deliberately trace something CDN-hosted and compare:
tracert -h 30 -w 1500 www.mit.edu

# And watch TTL work by hand, matching hops 1-3 of the above:
1..4 | ForEach-Object { Write-Host "--- TTL=$_ ---"; ping -i $_ -n 1 8.8.8.8 }
```

```bash
# Linux / macOS: the -T form gets furthest through firewalls.
sudo traceroute -T -p 443 -n -m 30 speedtest.tele2.net
mtr -r -c 20 www.iij.ad.jp
```

**Definition of done.** For each trace, in writing: (1) identify which hops are your LAN, which are your ISP, and which are transit — using the RFC 1918 table from §3 to spot the private ones; (2) find the hop where latency jumps by more than 50 ms and convert it to a distance using ~200,000 km/s in fibre and remembering RTT is *two* ways; (3) find at least one `* * *` and one non-monotonic latency and write down which of the causes in points 1 and 2 above you think applies, and why; (4) read the first line of each output and say whether you reached the place you aimed at or a CDN edge; (5) confirm that `ping -i 1/2/3` returns the same first three addresses as `tracert`, and explain in one sentence why it must.

---

## 7. IP fragmentation: the header fields, and why IPv6 deleted the feature

**Depth: [WORKING]**

> **Scope, stated up front so you know this is a build-on and not a repeat.** `notes/d9-networks-packets-frames-switches.md` §9 already taught: what an MTU is and why links have one; that an oversize packet must be split or rejected; Path MTU Discovery as the mechanism for finding the smallest MTU on a path; the **PMTUD black hole** (a middlebox drops the ICMP "fragmentation needed" message, so the sender never learns and large packets vanish silently while small ones work); and jumbo frames. **None of that is repeated here.** This section adds only what lives *inside the IP header* and was named-but-not-opened on Day 9: the four fields that make fragmentation work (Identification, DF, MF, Fragment Offset), worked with real arithmetic; who splits and who reassembles, and why those are different parties; the reassembly timer; why IPv6 removed router fragmentation entirely and made PMTUD mandatory; and fragmentation as a security surface. If Day 9's material isn't fresh, re-read it first — this section assumes it.

### Intuition — the problem the four fields solve

Day 9 gave you the situation: a router needs to forward a 4,000-byte datagram onto a link whose MTU is 1,500, and the datagram's DF bit is clear so splitting is permitted. Fine — split it into three pieces. But now the receiving *host* gets three separate IP datagrams and must reconstruct one. To do that it needs to answer four questions, and each question is exactly one header field:

- **"Which original datagram do these pieces belong to?"** Two fragments arriving a millisecond apart may come from entirely different datagrams, possibly from different senders, possibly interleaved. → **Identification** (16 bits).
- **"Was I allowed to split this at all?"** → **DF, Don't Fragment** (1 bit in Flags).
- **"Is this the last piece, or are more coming?"** Because there is no length field for the *original*, only for each fragment. → **MF, More Fragments** (1 bit in Flags).
- **"Where in the original does this piece go?"** Fragments can arrive out of order (§1: IP does not promise ordering), so a piece must carry its own position. → **Fragment Offset** (13 bits).

Note what is *not* there: no "fragment 2 of 3," no total count. The receiver discovers the total only by receiving the fragment with MF=0 and reading its offset and length. That design choice — position-based rather than index-based — is what allows fragments to be re-fragmented again further downstream by a router that knows nothing about the original split. A router facing an even smaller MTU can split a fragment into smaller fragments, adjusting offsets, without any coordination. Index-based numbering could not do that.

### Analogy — the shredded manuscript, and where it breaks

You mail a 300-page manuscript, but the postal service caps envelopes at 100 pages. So you split it into three envelopes. On each you write a **job number** (so the recipient doesn't merge it with a different manuscript), the **page number this envelope starts at** (so they can order the envelopes however they arrive), and a checkbox "**more envelopes follow**" — ticked on the first two, blank on the third, so the recipient knows when they have it all. The recipient assembles nothing until every page from 1 to the last is present.

**Where the analogy breaks — three ways, and each is a real consequence.**

1. **The recipient can see a page is missing and ask you to resend that page. IP cannot.** If one fragment is lost, the receiver has an incomplete set and **must discard everything it has collected** — there is no mechanism to request one fragment. TCP will then retransmit the *entire* original segment, which will be fragmented again from scratch. **Losing one fragment costs you the whole datagram.** So a 3-fragment datagram on a link with 1% packet loss has roughly a 3% chance of total loss. Fragmentation *multiplies* your effective loss rate by the fragment count, and this is the single most important practical fact in this section.
2. **The postal recipient will hold envelopes indefinitely. A host will not.** Holding partial fragments consumes memory, and an attacker can exploit that (below), so there is a hard timer. RFC 791 §3.2 recommends "an initial timer setting of 15 seconds"; Linux defaults to 30 seconds (`net.ipv4.ipfrag_time`); RFC 8200 mandates 60 seconds for IPv6. When the timer fires, everything collected is dropped and the receiver sends **ICMP Time Exceeded type 11 code 1** — "fragment reassembly time exceeded," the *other* code of the same ICMP type you met in §5.
3. **You, the sender, did the splitting in the analogy. In IPv4, a *router in the middle* does it — and that router isn't the one who reassembles.** This asymmetry is the design's original sin. The router that splits gains nothing and pays CPU; the destination host pays memory and reassembly cost; and the *sender*, who is the only party that could have avoided the whole thing by sending smaller packets, is never told. IPv6 fixed exactly this (below).

### Worked example — split a 4,000-byte payload over an MTU of 1,500, field by field

Real arithmetic, verified by computation. A host sends a 4,020-byte datagram: 20 bytes of IP header plus 4,000 bytes of payload. A router must forward it onto a link with **MTU 1,500** and DF is clear.

**Step 1 — how much payload fits per fragment?** The MTU covers the whole IP datagram, header included. Each fragment needs its own 20-byte IP header. So the maximum payload per fragment is 1500 − 20 = **1,480 bytes**.

**Step 2 — the multiple-of-8 rule.** Fragment Offset is 13 bits, but it must be able to express a position anywhere inside a datagram up to 65,535 bytes long — and 13 bits only counts to 8,191. The resolution: **the offset is measured in units of 8 bytes** (RFC 791: "the fragment offset is measured in units of 8 octets (64 bits)"). 8,191 × 8 = 65,528, which covers the address space. The consequence is a hard constraint: **every fragment except the last must carry a payload that is a multiple of 8 bytes**, otherwise the next fragment's offset would not be expressible.

Here 1,480 ÷ 8 = 185 exactly, so no rounding is needed. (If the MTU were 1,492 — common on PPPoE links — the limit would be 1,472, and 1,472 ÷ 8 = 184, also exact. But an MTU of 1,499 would give 1,479, which is *not* a multiple of 8, so the router would use 1,472 and waste 7 bytes per fragment. This is why odd MTUs are quietly inefficient.)

**Step 3 — the three fragments.** Computed, not recalled:

```
Original: IP header (20) + payload 4000 bytes = Total Length 4020
          Identification = 0x4A3B (chosen by the sender), DF=0, MF=0, Frag Offset=0

+------------+-------------+--------+------------+----+----+---------------+
| Fragment   | payload     | Total  | Frag       | DF | MF | Identification|
|            | bytes       | Length | Offset     |    |    |               |
|            | (of orig.)  | field  | field      |    |    |               |
+------------+-------------+--------+------------+----+----+---------------+
| 1          | 0 .. 1479   | 1500   |   0        | 0  | 1  | 0x4A3B        |
| 2          | 1480 .. 2959| 1500   | 185        | 0  | 1  | 0x4A3B        |
| 3          | 2960 .. 3999| 1060   | 370        | 0  | 0  | 0x4A3B        |
+------------+-------------+--------+------------+----+----+---------------+

Check the offsets:  1480 / 8 = 185      2960 / 8 = 370       both exact.
Check the lengths:  1480 + 1480 + 1040 = 4000                all payload accounted for.
Check fragment 3:   1040 + 20 = 1060 = its Total Length       and 1060 < 1500, fine.
Check MF:           set on 1 and 2, clear on 3                so 3 terminates the set.
Check Identification: identical on all three                  so they group correctly.
```

**Step 4 — what the receiver does.** It sees a fragment with Identification `0x4A3B` and MF=1, so it allocates a reassembly buffer keyed on the four-tuple **(source address, destination address, protocol, Identification)** — all four, because Identification is only 16 bits and only unique per sender-destination-protocol pair. It writes each arriving fragment's payload at byte position `offset × 8`. When it has received the fragment with MF=0 (which tells it the total length: 2960 + 1040 = 4000) *and* every byte from 0 to 3999 is present, it reassembles and hands the 4,000-byte payload up to TCP or UDP. If the 30-second timer fires first, it frees the buffer and sends ICMP type 11 code 1.

**Note the crucial asymmetry one more time, because it is the thing to remember: fragmentation happens at any router along the path; reassembly happens only at the final destination.** No intermediate router reassembles — it would need to hold all fragments, which are not guaranteed to take the same path (§4's ECMP), and it would need per-datagram state, which §4 step 10 forbids. This is also why a firewall that wants to inspect a full TCP header can be defeated by fragmentation (below).

### Runnable example — measure your own path MTU, and watch fragmentation actually fail

The DF bit gives you a direct probe: set DF, send progressively larger packets, and the largest one that gets through tells you the path MTU. `ping` exposes both controls.

```powershell
# Windows: -f sets the DF (Don't Fragment) bit, -l sets the ICMP payload size.
ping -f -l 1472 -n 2 8.8.8.8    # 1472 + 8 (ICMP hdr) + 20 (IP hdr) = 1500 exactly
ping -f -l 1473 -n 2 8.8.8.8    # 1501 - one byte too big
ping    -l 1473 -n 2 8.8.8.8    # same size, DF clear - fragmentation permitted
```

```bash
# Linux:  -M do  sets DF and refuses to fragment locally;  -s sets payload size.
#   ping -M do -s 1472 -c 2 8.8.8.8
#   ping -M do -s 1473 -c 2 8.8.8.8
#   ping       -s 1473 -c 2 8.8.8.8
# macOS:  ping -D -s 1472 -c 2 8.8.8.8
```

**Real captured output** (this machine, 2026-08-11):

```
=== 1472, DF set ===
Pinging 8.8.8.8 with 1472 bytes of data:
Reply from 8.8.8.8: bytes=1472 time=20ms TTL=111
Reply from 8.8.8.8: bytes=1472 time=22ms TTL=111
    Packets: Sent = 2, Received = 2, Lost = 0 (0% loss)

=== 1473, DF set ===
Pinging 8.8.8.8 with 1473 bytes of data:
Packet needs to be fragmented but DF set.
Packet needs to be fragmented but DF set.
    Packets: Sent = 2, Received = 0, Lost = 2 (100% loss)

=== 1473, DF clear (fragmentation allowed) ===
Pinging 8.8.8.8 with 1473 bytes of data:
Request timed out.
Request timed out.
    Packets: Sent = 2, Received = 0, Lost = 2 (100% loss)
```

**Why this works, line by line — and the third result is the interesting one.**

The `-l 1472` case: the ICMP payload is 1,472 bytes, the ICMP header is 8 bytes, the IP header is 20 bytes, total **1,500** — exactly the Ethernet MTU. It fits. Two replies, no loss. So the path MTU from here to `8.8.8.8` is at least 1,500, which also means the path contains no PPPoE or tunnel link with a smaller MTU. (Verified arithmetic: 1472 + 8 + 20 = 1500.)

The `-l 1473` case with `-f`: total 1,501 bytes, one over. DF is set, so no router is permitted to split it. Windows prints "Packet needs to be fragmented but DF set." **That message is Windows rendering ICMP Destination Unreachable, type 3, code 4** — the exact ICMP message Day 9 identified as the engine of PMTUD, and the one whose loss causes the PMTUD black hole. Here it arrived, which is why we get a clear error instead of a mystery. Notice that this is *also* how you'd discover a smaller path MTU: bisect on `-l` until the largest success, add 28, and that's your path MTU.

**The `-l 1473` case with DF clear is the honest surprise, and I am keeping it in rather than picking a tidier example.** Fragmentation was permitted. The packet *should* have been split into two fragments and delivered. It timed out — twice. Something on the path dropped either the fragmented request or the fragmented reply. I cannot tell which from here, and I am not going to guess. What I *can* say is that this is an extremely common outcome and it is the empirical case for the whole modern consensus on fragmentation: **fragmented IPv4 traffic is unreliable on the public internet in practice, regardless of what the RFCs permit.** Reasons this happens constantly: load balancers and ECMP hash on the 5-tuple, but only the *first* fragment carries the TCP/UDP ports — so non-first fragments hash differently and can be sent to a different backend that has no reassembly context; firewalls and DDoS scrubbers frequently drop all non-initial fragments as a blunt security policy (see the attacks below); and stateful middleboxes often can't be bothered to reassemble. A working `ping -f -l 1472` and a failing `ping -l 1473` on the same path, back to back, is the measurement that tells you: **design to never fragment.** Which is exactly what IPv6 concluded.

### Why IPv6 deleted router fragmentation

RFC 8200 makes three changes that together retire the whole mess, and the reasoning is worth following because it is a model of how to remove a feature.

**Change 1: routers must never fragment.** RFC 8200 §5: "unlike IPv4, fragmentation in IPv6 is performed only by source nodes, not by routers along a packet's delivery path." A router facing a too-big packet drops it and sends **ICMPv6 Packet Too Big (type 2)**. There is no DF bit in IPv6 because DF is now permanently, implicitly set.

*Why?* Because §7's whole analysis showed the costs land on the wrong parties. The router pays CPU, the destination pays memory, the sender — the only one who could fix it — learns nothing. Making the sender the only fragmenter aligns cost with control.

**Change 2: the fragmentation fields moved out of the header into an optional extension header.** IPv4 spends 32 bits of its 160-bit mandatory header (Identification, Flags, Fragment Offset) on fields that are unused by the overwhelming majority of packets. IPv6's fixed 40-byte header has none of them; a source that genuinely needs to fragment adds a **Fragment extension header** (which carries a 32-bit Identification — four times IPv4's, reducing collision risk — plus offset and an M flag). The common case pays zero bytes for a feature it doesn't use. This is the same instinct that removed the header checksum: get per-packet cost out of the fast path.

**Change 3: a hard MTU floor, and PMTUD becomes the expected mechanism.** RFC 8200 §5: "IPv6 requires that every link in the Internet have an MTU of 1280 octets or greater. This is known as the IPv6 minimum link MTU." So a source can *always* safely send 1,280 bytes with no discovery at all. Above that, "it is strongly recommended that IPv6 nodes implement Path MTU Discovery [RFC 8201], in order to discover and take advantage of path MTUs greater than 1280 octets."

**And here is the honest catch, which Day 9's PMTUD-black-hole section should make you suspicious of immediately.** IPv6 made PMTUD *load-bearing* rather than optional — which means it also made ICMPv6 load-bearing. A network that filters ICMPv6 Packet Too Big does not merely lose a nicety; it creates a black hole that no fallback can rescue, because IPv6 has no fragmentation-in-the-middle to fall back to. This is why **"never block ICMPv6" is a genuine hard rule** where "block all ICMP" was merely a bad habit in IPv4. RFC 4890 exists specifically to tell firewall operators which ICMPv6 types they must not filter. If you take one operational rule from this section: in IPv4, blocking ICMP type 3 code 4 breaks large packets; in IPv6, blocking ICMPv6 type 2 breaks the protocol.

**IPv6 reassembly timeout is 60 seconds**, mandated rather than recommended: "If insufficient fragments are received to complete reassembly of a packet within 60 seconds of the reception of the first-arriving fragment of that packet, reassembly of that packet must be abandoned."

### Fragmentation as a security surface

**Depth: [WORKING]** — you need to recognise these, not implement defences from scratch.

Reassembly is the one place in IP where a receiver must hold state about multiple packets and *combine* them. Anywhere a system must combine untrusted inputs before inspecting them is an attack surface, and fragmentation has produced a long line of real vulnerabilities.

**Overlapping-fragment attacks (firewall evasion).** The offsets in a fragment set are attacker-controlled and nothing requires them not to overlap. So an attacker sends fragment 1 containing a TCP header with destination port 80 (which the firewall permits) and fragment 2 with an offset that *overlaps* fragment 1's TCP header, rewriting the port to 22. The firewall, inspecting fragments independently, sees port 80 and permits. The destination host reassembles and — depending on whether its OS prefers the first or the last data for an overlapping byte range, which **differs between operating systems** — may end up with a connection to port 22. This class is old (Ptacek & Newsham's 1998 paper "Insertion, Evasion, and Denial of Service: Eluding Network Intrusion Detection" is the canonical treatment) and the RFCs eventually responded: **RFC 5722 forbids overlapping fragments in IPv6 outright** — a receiver must discard the entire packet — and **RFC 8200 §4.5 restates that requirement as part of the core IPv6 spec.**

**Tiny-fragment attacks.** Send a first fragment so small that it does not contain the complete TCP header — for example, 8 bytes of payload, holding only the source and destination ports but not the flags. A firewall rule keyed on "TCP SYN" cannot match, because the SYN bit is in the *second* fragment. **RFC 1858** documents this and the mitigation (drop first fragments carrying less than a full transport header). **RFC 3128** corrects an error in RFC 1858's proposed check — worth knowing as an example of the security-advice pipeline itself needing patches.

**Resource-exhaustion attacks.** Send many first-fragments that never complete. Each one causes the target to allocate a reassembly buffer and start a 30–60 second timer. This is cheap for the attacker and expensive for the target. Every modern stack caps total reassembly memory (Linux: `net.ipv4.ipfrag_high_thresh`) and drops the oldest incomplete sets under pressure — meaning a flood of junk fragments can evict *legitimate* in-progress reassemblies. And the historic **teardrop** attack (1997) sent deliberately malformed overlapping fragments with negative resulting lengths, crashing unpatched Windows and Linux kernels outright; a variant resurfaced against Windows in 2009 and again in the SMB context, which is the general lesson: reassembly code is fiddly integer arithmetic on attacker-controlled offsets, and that is exactly the shape of code that keeps having bugs.

**The practical operational conclusion**, which is now near-universal in production networks: **engineer so that fragmentation never happens, and then drop fragments.** Concretely — set MSS clamping on tunnels so TCP negotiates a segment size that fits (AWS NAT gateways do this automatically: "NAT gateways enforce Maximum Segment Size (MSS) clamping for all packets", per RFC 879); keep application datagrams under the safe floor for UDP-based protocols (this is why DNS-over-UDP historically capped at 512 bytes and why EDNS0 buffer sizes are commonly set to 1232 = 1280 IPv6 minimum − 40 IPv6 header − 8 UDP header, a number worth recognising); use TCP or QUIC where you'd otherwise send large datagrams; and monitor fragment counters rather than assuming zero.

---

## 8. IPv4 exhaustion: the arithmetic, the dates, and the market

**Depth: [WORKING]**

### The arithmetic, done plainly

An IPv4 address is 32 bits, so there are exactly **2^32 = 4,294,967,296** possible addresses. That is the entire, permanent, unextendable supply.

Now subtract what can never be assigned to a host on the internet. `10.0.0.0/8`, `172.16.0.0/12`, and `192.168.0.0/16` are private (16,777,216 + 1,048,576 + 65,536 addresses). `127.0.0.0/8` is loopback — a full 16,777,216 addresses so that one host can talk to itself. `224.0.0.0/4` is multicast (268,435,456). `240.0.0.0/4` is the old Class E, reserved and unusable (another 268,435,456). Plus link-local, documentation, benchmarking, and shared address space. Roughly **588 million addresses — about 13.7% of the space — are structurally unavailable** before anyone gets one.

So the real supply is around 3.7 billion, against a world with roughly 8 billion people, most of whom own multiple connected devices, in front of a datacentre industry that wants a public address per load balancer. The shortfall is not marginal; it is a factor of several.

Compare IPv6: **2^128 = 340,282,366,920,938,463,463,374,607,431,768,211,456**. The ratio of IPv6 to IPv4 addresses is 2^96 ≈ **7.9 × 10^28**. Numbers that size stop meaning anything, so here is the calibration that does: IPv6's standard subnet is a **/64**, which contains 2^64 ≈ 1.8 × 10^19 addresses — more than four billion times the *entire* IPv4 internet, in one subnet, on one LAN. And the current global unicast allocation policy hands out `2000::/3`, one-eighth of the space, containing 2^61 /64 subnets. The design goal was explicitly to never do this again.

### The dates, and why "exhaustion" happened five times

Exhaustion was not one event, because IPv4 was distributed in a hierarchy. IANA held the global pool and allocated /8 blocks to five Regional Internet Registries (RIRs), which allocate to ISPs and large organisations, which assign to customers.

- **3 February 2011** — **IANA's free pool depleted.** The last five /8s were distributed one to each RIR under a pre-agreed global policy. This is the date IPv4 "ran out" at the top of the hierarchy.
- **15 April 2011** — **APNIC** (Asia-Pacific) exhausted first, by a wide margin, and moved to a final-/8 policy handing out a single /22 (1,024 addresses) per member. Asia's growth was fastest and its historical allocations smallest.
- **14 September 2012** — **RIPE NCC** (Europe/Middle East/Central Asia) reached its final /8.
- **10 June 2014** — **LACNIC** (Latin America/Caribbean).
- **24 September 2015** — **ARIN** (North America) exhausted its free pool and activated a waiting list.
- **AFRINIC** (Africa) was last, exhausting around 2019–2020, and has since been subject to prolonged governance disputes. **Verify current** — AFRINIC's situation has been in flux and I would not state a date confidently.

Primary sources: https://www.nro.net/ipv4-free-pool-depleted/ (the Number Resource Organization's announcement of the 2011 IANA depletion) and each RIR's own statistics pages; the JPNIC summary at https://www.nic.ad.jp/en/ip/ipv4pool/ collates the per-RIR dates.

Notice what did **not** happen on any of those dates: the internet did not stop working, and no existing address was taken away. Exhaustion of a *free pool* means no new allocations. Everything already allocated kept working. This is why the event felt anticlimactic to almost everyone, and why the *consequences* arrived slowly and are still arriving.

### The consequences you will actually meet

**Consequence 1: NAT became mandatory rather than optional.** This is §9, and it is the big one. If there are not enough addresses for every device, devices must share, and sharing requires translation.

**Consequence 2: IPv4 addresses became a traded asset with a market price.** Since new allocations stopped, the only way to get IPv4 space is to buy it from someone who has it, through RIR-sanctioned transfer processes. Prices for a /24 have run in the tens of dollars per address for years. **Verify current before quoting any figure** — this is a live market and published broker indices move.

**Consequence 3: cloud providers started charging for public IPv4, which changed architecture.** On **1 February 2024** AWS began charging **$0.005 per public IPv4 address per hour** for *all* public IPv4 addresses — attached or not, in every region, across EC2, RDS, EKS nodes, internet-facing ELBs, NAT gateways, and Global Accelerator. That is about **$3.65/month or $43.80/year per address**. AWS's stated reason was precisely the scarcity above. **Verify current** — this rate and its scope have been adjusted before.

The architectural effect was immediate and is the reason this belongs in a note about design and not just history. A fleet of 200 instances each with a public IP went from free to $730/month for nothing but addresses. The correct response is not to pay it: put instances in private subnets and share **one** NAT gateway's address for egress (§9), use VPC endpoints so traffic to AWS services never needs a public address at all, and consider dual-stack or IPv6-only subnets with an egress-only internet gateway. In other words, a pricing change made the *good* network design also the *cheap* one — which is unusually convenient, and is why "why does my laptop/instance not have a public IP?" is now a question with an economic answer as well as a technical one.

**Consequence 4: your ISP probably does not give you a public address either.** §5's `ping -i` capture and §6's Capture A both showed seven consecutive RFC 1918 hops inside my ISP's network before any public address appeared. That is what an address-starved carrier network looks like from the inside, and it leads directly to carrier-grade NAT (§9).

### Case study — the AWS public IPv4 charge as a forcing function

**What happened.** AWS announced in July 2023 that from 1 February 2024 all public IPv4 addresses would carry a $0.005/hour charge, and simultaneously shipped tooling to find and eliminate them: Public IP Insights in VPC IP Address Manager (IPAM) to inventory every public IPv4 in an account, plus expanded IPv6 support across services and the "Bring Your Own IP" path for organisations that already own space.

**The engineering lesson.** For twenty years the cost of a public IPv4 address was hidden inside the cost of a service, so nobody optimised for it, and "just give it a public IP" became a default in tutorials, Terraform modules, and quickstarts. Making the cost explicit and per-hour converted a diffuse global scarcity into a line item on a specific team's bill — and behaviour changed within one billing cycle in a way that twenty years of RFCs and IPv6 advocacy had not achieved. This is a general pattern worth carrying beyond networking: **a shared resource nobody is charged for will be consumed until it is gone; attributing the cost is often a more effective control than exhortation.** (§4's 512k-day is the same tragedy of the commons without a pricing mechanism, which is why it recurred.)

**Primary sources:** https://aws.amazon.com/blogs/aws/new-aws-public-ipv4-address-charge-public-ip-insights/ and the current rates at https://aws.amazon.com/vpc/pricing/. **Verify current** — pricing pages are the fastest-drifting facts in this note.

*(A second named case study for this section would be the RIR transfer market or AFRINIC's governance disputes; both are real but I have not verified either to the standard the rest of this note holds, so I am declining to write them up rather than half-sourcing them. The exhaustion dates above are sourced; the market's current numbers are not, and I've said so.)*

---

## 9. NAT: how one address serves a thousand devices

**Depth: [CORE]** — and I am tiering it CORE rather than WORKING deliberately, because NAT causes more confused backend debugging than any other single mechanism in this note. "Why does the API see a different IP than my server has?" "Why can't the webhook reach my laptop?" "Why did we hit a connection limit at exactly 55,000?" All NAT, all in this section.

### Intuition — the trick, in one paragraph

You have one public IPv4 address and forty devices that all want to talk to the internet. §8 says you cannot have forty public addresses. So put a box between them and the internet, give every device a private address (RFC 1918), and have the box **rewrite the source address of every outbound packet to its own public address**, remembering who it did that for so it can rewrite the *reply* back to the right device.

The rewriting is the easy part. The remembering is where all the interesting engineering is, and where all the problems come from — because §1 established that IP is stateless and §4 step 10 said routers forget everything, and NAT is a box that must do the exact opposite. **NAT is a stateful device pretending to be part of a stateless network, and every one of NAT's pathologies traces back to that contradiction.**

### The problem with the naive version, and how ports fix it

Try the naive design first, because seeing it fail is what makes the real design obvious.

**Naive design (this is "Basic NAT", RFC 3022):** keep a table mapping private address ↔ public address, one-to-one. Device `192.168.1.10` gets rewritten to public `203.0.113.5`; device `192.168.1.11` gets `203.0.113.6`. This works perfectly and solves nothing — you still need one public address per simultaneously-active device. Basic NAT is genuinely useful for a different purpose (renumbering a network without touching hosts), but it does not multiply addresses.

**The real design: also rewrite the port.** Recall that a TCP or UDP header contains a 16-bit source port and a 16-bit destination port (Day 11 owns these; here they are just numbers that let a host distinguish multiple conversations). The NAT box rewrites **both** the source address *and* the source port, and keys its table on the combination. Now forty devices can share one address, because they are distinguished by port instead.

This is called **NAPT** (Network Address/Port Translator), or **PAT** (Port Address Translation) in Cisco's vocabulary, or **IP masquerading** in Linux's, or **source NAT / SNAT** when you're describing the direction. RFC 4787 notes NAPT is "by far the most commonly deployed NAT device." **When anyone says "NAT" in 2026, they mean NAPT.** Basic NAT is a footnote.

### Analogy — the office switchboard, and where it breaks

A company has one public phone number and 400 employees. Nobody dials an employee directly. Calls go to the switchboard, and the switchboard connects them.

Now the outbound direction, which is the analogy that actually matters. An employee at extension 3417 calls a supplier. The supplier's phone shows the *company's* number, not the extension — the extension is not a globally meaningful thing. The switchboard writes down "the call currently on outside line 7 belongs to extension 3417." When the supplier calls back on line 7, the switchboard consults its note and rings 3417.

That note is the **NAT translation table**. The "outside line number" is the rewritten source port. And notice: the supplier fundamentally cannot initiate a call to extension 3417, because it has no way to name it. It can only ever reply on a line the extension opened.

**Where the analogy breaks — four ways, and each one is a real production problem.**

1. **The switchboard operator remembers a call until it ends, because they can hear it end. NAT cannot hear anything.** There is no reliable signal that a UDP conversation is over, and even for TCP the FIN may be lost. So NAT uses **timeouts** — a mapping is deleted after N seconds of silence. RFC 4787 REQ-5: "A NAT UDP mapping timer MUST NOT expire in less than two minutes," with a recommended default of five minutes or more; TCP mappings typically last much longer (RFC 5382 recommends at least 2 hours 4 minutes for established connections). **This is the mechanism behind an entire class of production bug**: a long-idle database connection or WebSocket has its NAT mapping silently reaped, and the next packet in either direction is dropped by a NAT that no longer knows what it is. The application sees a hang, then a reset. TCP keepalives and application-level pings exist substantially to keep NAT mappings warm. (Day 11's ELB idle-timeout case study is the same shape of bug with a different box doing the forgetting.)
2. **A supplier can look up the company in a directory and ask the switchboard to be transferred. There is no directory of NAT'd hosts, and no transfer.** Inbound connections to a NAT'd host are impossible without pre-arrangement. Hence port forwarding, UPnP, STUN/TURN, and every "hole punching" technique in existence.
3. **The switchboard doesn't rewrite the contents of your conversation. NAT sometimes must.** If a protocol carries IP addresses *inside its payload* — classic FTP's PORT command, SIP, some peer-to-peer protocols — then rewriting only the header produces a packet whose header says one thing and whose body says another. NAT boxes grew **Application Layer Gateways (ALGs)** that parse and rewrite payloads. ALGs are a notorious source of subtle breakage, and they are also structurally impossible once traffic is encrypted, which is one quiet reason universal TLS improved the internet's architecture: it made this layering violation infeasible and forced protocols to stop embedding addresses.
4. **The switchboard has 400 extensions and maybe 30 outside lines, and if all 30 are busy the 31st employee simply cannot call out.** NAT has the same hard ceiling — the port space — and it is the thing that bites at scale. §"Port exhaustion" below.

### The translation table, worked concretely

Here is what actually happens, with real numbers. Setup: a home network `192.168.29.0/24` behind a router whose public address is `203.0.113.7` (documentation space, RFC 5737). Two laptops and a phone each open a connection to `93.184.216.34:443`.

**Outbound. What each device sends, and what leaves the router:**

```
                  as sent by the host                    as it leaves the router
device            src IP:port        -> dst IP:port      src IP:port        -> dst IP:port
--------------  ------------------      ---------------  ------------------    ---------------
laptop-A        192.168.29.10:51000 -> 93.184.216.34:443  203.0.113.7:40001 -> 93.184.216.34:443
laptop-B        192.168.29.11:51000 -> 93.184.216.34:443  203.0.113.7:40002 -> 93.184.216.34:443
phone           192.168.29.12:33210 -> 93.184.216.34:443  203.0.113.7:40003 -> 93.184.216.34:443
```

Look at laptop-A and laptop-B: **both chose source port 51000.** Independently, legitimately — a host's port choice is local to that host, and two hosts have no reason to coordinate. This collision is precisely why NAT must rewrite the port and cannot simply preserve it. The router allocated `40001` and `40002` from its own pool, and now the two conversations are distinguishable.

**The table the router holds:**

```
+---------------------+-------------------+--------------------+-------+---------+
| inside (priv:port)  | outside (pub:port)| destination        | proto | idle    |
+---------------------+-------------------+--------------------+-------+---------+
| 192.168.29.10:51000 | 203.0.113.7:40001 | 93.184.216.34:443  | TCP   |   3s    |
| 192.168.29.11:51000 | 203.0.113.7:40002 | 93.184.216.34:443  | TCP   |   1s    |
| 192.168.29.12:33210 | 203.0.113.7:40003 | 93.184.216.34:443  | TCP   |  12s    |
+---------------------+-------------------+--------------------+-------+---------+
```

**Inbound. The reply arrives and gets un-rewritten:**

```
arrives at router:  93.184.216.34:443 -> 203.0.113.7:40002
router looks up destination port 40002 in the table  ->  192.168.29.11:51000
router rewrites destination:  93.184.216.34:443 -> 192.168.29.11:51000
forwards onto the LAN
```

Two things to notice, because they are the whole of NAT's behaviour.

**The router recomputes checksums.** It changed the IP source address, so the IP header checksum must be recomputed (§1). But the TCP and UDP checksums are computed over a "pseudo-header" that *includes the IP addresses*, so those must be recomputed too. **A NAT device must therefore understand and modify the transport layer** — a textbook layering violation, and the reason NAT only works for TCP, UDP, and ICMP. AWS's own documentation states the constraint flatly: "A NAT gateway supports the following protocols: TCP, UDP, and ICMP." Try to run SCTP or a novel transport protocol through NAT and it simply will not pass.

**ICMP has no ports, so NAT cheats.** How does a NAT box demultiplex a ping reply? It uses the ICMP Echo **Identifier** field (16 bits, in the Echo header) as if it were a port, rewriting it the same way. And for ICMP *errors* — the Time Exceeded messages that make §6's traceroute work — the NAT must reach *inside the ICMP payload*, which per RFC 792 contains the original packet's IP header plus 64 bits (§5), find the embedded address and port, and rewrite those too. This is genuinely fiddly, and it is why traceroute through a poorly-implemented NAT sometimes shows nonsense: the NAT failed to rewrite the embedded copy correctly.

### NAT mapping and filtering behaviours — why "NAT type" appears in video games

**Depth: [WORKING]**

Two independent design choices determine how permissive a NAT is, and RFC 4787 names them precisely. They matter because they decide whether peer-to-peer connections are possible at all.

**Mapping behaviour** answers: when host X:x sends to *two different* destinations, does it get the *same* external port both times?

- **Endpoint-Independent Mapping** — the NAT "reuses the port mapping for subsequent packets sent from the same internal IP address and port (X:x) to any external IP address and port." One external port per internal socket, whoever you're talking to.
- **Address-Dependent Mapping** — a new external port per distinct destination *address*.
- **Address and Port-Dependent Mapping** — a new external port per distinct destination address *and* port.

**Filtering behaviour** answers: once a mapping exists, who is allowed to send inbound through it?

- **Endpoint-Independent Filtering** — anyone. Most permissive.
- **Address-Dependent Filtering** — only hosts you have already sent to.
- **Address and Port-Dependent Filtering** — only the exact address:port you sent to. Most restrictive.

**RFC 4787's REQ-1 requires Endpoint-Independent Mapping**, and the reason is entirely about making peer-to-peer possible. If your external port is the same regardless of destination, then you can ask a helper server ("what address:port do you see me as?" — that is **STUN**) and *tell your peer that address*, and your peer's packets to it will arrive. That is **NAT hole punching**. With Address-and-Port-Dependent Mapping, the port you learned from the STUN server is useless for talking to your peer, because talking to a different destination gets a different port — so no hole punching, and you must relay everything through a third-party server (**TURN**), which costs bandwidth and adds latency.

**This is why your games console reports a "NAT Type."** Sony's PlayStation Network labels the situation Type 1 (no NAT / direct), Type 2 (NAT with a workable mapping behaviour — hole punching succeeds), Type 3 (restrictive — relay required or multiplayer features degrade). It is a consumer-facing rendering of RFC 4787's taxonomy, and it is the reason "open your NAT" is a phrase millions of people have said without knowing they were discussing an IETF BCP. The same taxonomy governs whether a WebRTC video call connects peer-to-peer or has to burn a TURN relay — which is a direct cost line for anyone running video infrastructure.

**Hairpinning** (RFC 4787's term) is the third behaviour worth naming: can two hosts *inside* the NAT reach each other using each other's *external* address:port? A NAT that supports hairpinning "relay[s] traffic between them through the NAT device." Many do not, which produces the classic bewildering symptom: your service is reachable from the internet and from the same machine, but **not** from another machine on your own LAN — because that machine's packet to the public address goes to the router, which has no rule for turning it around.

### CGNAT: when your ISP NATs you too

**Depth: [WORKING]**

§8 established that ISPs ran out of public addresses for customers. So they NAT their customers — a second layer of NAT above the one in your home router. This is **Carrier-Grade NAT (CGNAT)**, also called Large Scale NAT (LSN) or NAT444 (because a packet crosses IPv4-to-IPv4 translation twice).

RFC 6598 created `100.64.0.0/10` ("Shared Address Space", ~4.19 million addresses) specifically for the intermediate hop, and the reason a *new* block was needed is worth understanding: the ISP cannot use RFC 1918 space, because the customer is already using RFC 1918 space behind their own router, and if the ISP hands the customer's router a `192.168.x.x` WAN address that collides with the customer's LAN, routing breaks in ways nobody can debug. RFC 6598's rules are strict: "Packets with Shared Address Space source or destination addresses MUST NOT be forwarded across Service Provider boundaries," providers must filter route advertisements for it, and it must not appear in external DNS.

**The consequences, and these are real things you will hit as a backend engineer:**

- **Thousands of unrelated customers share one public IP.** So **IP-based rate limiting punishes bystanders.** Rate-limit "10 requests/minute per IP" and you have rate-limited an entire apartment block. **IP-based blocking is collateral damage by construction** — this is why Wikipedia's blocks of certain mobile carrier ranges affect enormous numbers of innocent users, and why Cloudflare and similar services had to move from IP reputation toward behavioural and cryptographic signals. If you are designing abuse controls, "per IP" is a much weaker identity than it looks.
- **Geolocation degrades.** The CGNAT egress point may be in a different city or country from the subscriber.
- **Inbound is impossible, permanently.** No port forwarding, because you don't control the carrier's NAT. Self-hosting from a CGNAT connection requires a relay or tunnel (Tailscale, Cloudflare Tunnel, ngrok — all of which exist substantially because of this).
- **Logging and attribution require port numbers.** "IP address 100.x.y.z did this at time T" identifies nobody. Law-enforcement and abuse handling under CGNAT require the source *port* and a precise timestamp, plus the carrier retaining port-allocation logs — which is a large data-retention burden and a policy fight in several jurisdictions.

**Am I behind CGNAT? Two checks.** First, look at your router's WAN address: if it is in `100.64.0.0/10`, or in RFC 1918 space, you are. Second, compare your router's WAN address with what the internet sees:

```powershell
# What does the internet think my address is?
(Invoke-RestMethod "https://api.ipify.org?format=json").ip

# What does my machine think it is?
Get-NetIPAddress -AddressFamily IPv4 -PrefixOrigin Dhcp,Manual |
    Select-Object IPAddress, InterfaceAlias, PrefixLength
```

Measured on this machine, 2026-08-11: local address `192.168.29.239/24`, public egress `49.43.x.x` (redacted). **Two different addresses, and the local one is RFC 1918 — so there is at least one NAT between me and the internet.** The `tracert` captures in §6 add the rest of the picture: seven consecutive hops of RFC 1918 addresses inside the ISP before the first public address at hop 10. Whether that specific ISP is doing CGNAT on this connection or merely numbering its transit links privately, I cannot prove from outside, and I am not going to claim it — but the shape is exactly what an address-starved carrier looks like, and the diagnostic technique is the transferable part.

### Port exhaustion — the ceiling, with the real arithmetic

**Depth: [CORE]** within NAT, because this is the failure mode that hits production backends and looks like something else entirely.

A NAT's external port field is 16 bits: 65,536 values, of which the usable ephemeral range is conventionally 1024–65535 = **64,512 ports**. So one public address supports at most ~64,512 simultaneous *mappings*.

But — and this is the part people get wrong in both directions — a mapping is keyed on the **full 5-tuple** (protocol, external address, external port, destination address, destination port). So the same external port can be reused for a *different* destination. The limit is therefore not "64,512 connections total"; it is closer to "64,512 connections **to any single destination address:port**." A well-implemented NAT can hold millions of mappings overall while still being capped per-destination.

AWS documents its own number exactly, and it is the one to remember because it is the one you will hit: **"Each IPv4 address can support up to 55,000 simultaneous connections to each unique destination. A unique destination is identified by a unique combination of destination IP address, the destination port, and protocol (TCP/UDP/ICMP)."** (Not 64,512 — AWS reserves headroom. AWS also notes "NAT gateways use ports 1024–65535.") You can raise it by attaching more addresses: "up to 8 IPv4 addresses to your NAT gateways (1 primary IPv4 address and 7 secondary IPv4 addresses)" — so 8 × 55,000 = **440,000** simultaneous connections per unique destination, with a default limit of 2 Elastic IPs per public NAT gateway that requires a quota increase to exceed. Source: https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-basics.html — **verify current.**

**Why this is the sneakiest capacity limit in cloud networking.** Read the condition again: *per unique destination*. It does not bite when your fleet talks to many different services. It bites hard when your entire fleet hammers **one** endpoint — which is exactly the shape of modern backends:

- 500 pods all calling `api.stripe.com:443`
- a fleet polling one third-party API
- everything writing to one external log/metrics ingest endpoint
- **an agent fleet all calling `api.anthropic.com:443` or `api.openai.com:443`**

That last one is not a contrived example, and it is the one place in this note where having an LLM in the loop changes the numbers rather than just the vocabulary. An agent doing a tool-use loop makes *many* sequential API calls per task rather than one, and those calls are long-lived (streaming responses hold a connection open for the duration of generation, which for a reasoning model can be tens of seconds). So an agent fleet's concurrent-connection count to a *single* destination is far higher, per unit of work, than a conventional request/response service's — and every one of those connections consumes one NAT port mapping against the 55,000 ceiling for `api.anthropic.com:443`. The mitigations are ordinary and effective: **reuse connections** (an HTTP client with keep-alive and a bounded pool turns 1,000 sequential calls into a handful of persistent mappings — this is the single highest-leverage fix, and it is Day 11's connection-pooling design), attach more addresses to the NAT gateway, or split the fleet across per-AZ NAT gateways so the mappings distribute.

**The symptom, and why it is misdiagnosed.** Port exhaustion presents as **intermittent connection timeouts to one specific external service, at high concurrency, with no error anywhere in your application logs and no problem reaching other services.** Teams reliably blame the third-party provider. On AWS the actual evidence is the CloudWatch metric **`ErrorPortAllocation`** on the NAT gateway (non-zero means the NAT could not allocate a source port) and `PacketsDropCount`. **If you take one alarm from this section: alarm on `ErrorPortAllocation > 0`.** It is a metric almost nobody enables and it converts a multi-day misdiagnosis into a five-minute fix.

### What NAT breaks, honestly

NAT is a hack that saved the internet and degraded it. Both halves are true and you should be able to argue both.

**It broke the end-to-end principle.** The original architecture said the network moves packets and endpoints implement semantics. A NAT is a middlebox that must understand transport-layer semantics to function, so new transport protocols cannot be deployed on the internet without NAT vendors' cooperation. This is a substantial part of why SCTP never happened, and why **QUIC was deliberately built on UDP** — not because UDP is a good foundation, but because UDP is the only thing besides TCP that reliably traverses the installed base of NATs. Day 11 and Day 14 cover QUIC; the NAT-shaped hole in the design space is why it looks the way it does.

**It broke inbound connectivity, which reshaped the consumer internet.** Every host behind NAT is client-only by default. The peer-to-peer internet of the 1990s became the client-server, cloud-mediated internet partly for this reason. Your smart doorbell talks to a vendor's cloud rather than to your phone directly because neither can accept an inbound connection.

**It broke address-based identity**, per the CGNAT discussion: "one IP, one user" is false, and any security or abuse control built on that assumption is built on sand.

**It broke logging and forensics** without port-level records.

**And it did not, contrary to persistent belief, make you secure.** NAT provides a *side effect* that resembles a firewall: unsolicited inbound packets have no mapping and get dropped. But that is a consequence of the translation table, not a security policy — there is no rule set, no logging, no egress control, no protection against anything initiated from inside (which is every modern attack), and NAT traversal techniques exist specifically to punch through it. **"We're behind NAT" is not a security control.** Use a real firewall with explicit rules; Day 9 §12 covered the security-group/NACL layer, and that is where the actual policy belongs.

**The trade-off, stated as a decision.** The alternative to NAT was **deploying IPv6 in the 1990s**, giving every device a globally routable address and preserving end-to-end connectivity. That alternative lost for a completely non-technical reason: NAT could be deployed unilaterally, by one network operator, in an afternoon, with no cooperation from anyone else; IPv6 requires *both* endpoints, every network between them, and every application to support it. **NAT's incremental deployability beat IPv6's technical superiority for thirty years,** and understanding why is more useful than any fact in this note: in a system with no central authority, a worse solution that one party can deploy alone will beat a better solution that requires everyone to move together. **The flip condition** — the thing that finally makes IPv6 win — is when enough of the world has IPv6 that the *marginal* deployment becomes unilateral too (you can turn on IPv6 and it just works, because your peers already did). §10's measured numbers say we crossed that threshold roughly in 2026.

### Cloud NAT: the three options and the money

**Depth: [WORKING]**

In a cloud VPC, instances in **private subnets** have no route to an internet gateway and therefore no inbound reachability — which is the point (Day 9 §12 argued the trust-boundary case; I'm not repeating it). But they still need *outbound* access: OS package updates, container image pulls, calls to third-party APIs, telemetry egress. Three mechanisms, and choosing between them is a real decision.

**Option 1 — managed NAT gateway.** AWS NAT Gateway / Azure NAT Gateway / GCP Cloud NAT. A provider-run, horizontally-scaled NAPT service.

AWS's documented characteristics: created in one AZ and "implemented with redundancy in that zone"; **"supports 5 Gbps of bandwidth and automatically scales up to 100 Gbps"**; **"can process one million packets per second and automatically scales up to 10 million packets per second"** and beyond that "will drop packets"; supports TCP, UDP, ICMP only; **cannot have a security group attached** (you control traffic with the instances' own security groups and the subnet's NACL); supports MTU up to 8500 but "the MTU setting for your EC2 instances should not exceed 1500 bytes" for internet-bound traffic through it, and it does MSS clamping and honours PMTUD via ICMP FRAG_NEEDED / ICMPv6 Packet Too Big (§7).

**Option 2 — a NAT instance.** An EC2 instance you run yourself with source/destination checking disabled and `iptables -t nat -A POSTROUTING -j MASQUERADE`. This is what everyone did before managed NAT existed.

**Option 3 — egress-only internet gateway (IPv6 only).** For IPv6, there is no address scarcity, so there is nothing to translate. An egress-only internet gateway is a *stateful filter*, not a translator: it permits outbound IPv6 and the return traffic, and blocks unsolicited inbound. AWS's docs present it as exactly this: "Another option for enabling outbound-only internet communication over IPv6 is using an egress-only internet gateway."

**The decision, with the money made explicit.** AWS NAT Gateway in `us-east-1` (**verify current at https://aws.amazon.com/vpc/pricing/**): roughly **$0.045 per gateway-hour** plus **$0.045 per GB processed**, and the data-processing charge is *in addition to* normal data-transfer-out charges (~$0.09/GB to the internet). Do the arithmetic, because it surprises people:

```
One NAT gateway, running always:
  0.045 $/hr x 730 hr/month                    =  $32.85 / month
Three AZs (the resilient design):
  3 x $32.85                                   =  $98.55 / month  before any traffic

Traffic: 10 TB/month egress through the NAT gateway
  NAT data processing:  10,000 GB x $0.045     = $450.00
  Data transfer out:    10,000 GB x $0.09      = $900.00
                                                 ---------
  monthly total (3 AZ)                           $1,448.55
```

That $450 is the NAT gateway's *processing* fee alone — money spent purely on address translation. It is one of the most common large surprise line items in an AWS bill, and it is why "NAT gateway cost" is a whole genre of blog post. Two structural mitigations, both of which are also good architecture: **VPC endpoints** (PrivateLink / gateway endpoints) route traffic to AWS services like S3, DynamoDB, ECR, and Secrets Manager over the AWS network *without touching the NAT gateway at all* — a container fleet pulling images from ECR through a NAT gateway is paying $0.045/GB for something a gateway endpoint does for free; and **cross-AZ discipline** — a private subnet in AZ-a routing to a NAT gateway in AZ-b pays a cross-AZ data-transfer charge (~$0.01/GB each way, ~$0.02/GB round trip) *on top of* everything else, which is the real reason to put one NAT gateway per AZ rather than share one.

**When a NAT instance is still right, honestly.** It is cheaper at low volume — a `t4g.nano` is a few dollars a month with no per-GB processing fee — so for a dev environment pushing 50 GB/month, a NAT instance costs single-digit dollars against the NAT gateway's ~$35. **Flip condition:** the moment the environment matters, the NAT gateway wins, and not on price. A NAT instance is a single point of failure you must patch, monitor, size, and replace; it caps out at one instance's network bandwidth; failover requires you to script route-table manipulation; and its conntrack table is a fixed-size kernel resource that silently drops connections when full. You are trading roughly $30/month for an on-call obligation. Take the managed service for anything production, and use the instance only for cost-sensitive non-production. **Second flip condition:** if you need something managed NAT cannot do — traffic inspection, protocol support beyond TCP/UDP/ICMP, or a custom egress proxy — you need an instance or a dedicated appliance anyway, and then the comparison isn't about NAT.

**And the IPv6 option changes the shape entirely.** An IPv6-only subnet with an egress-only internet gateway has **no NAT, no per-GB processing charge, no port exhaustion, and no public-IPv4 hourly charge** (§8). That is a genuinely better architecture on every axis that this section has discussed. Why isn't everyone doing it? Because the destination must also speak IPv6, and a great many third-party APIs still do not — so you end up needing DNS64/NAT64 (AWS NAT gateways do NAT64: "For IPv6 traffic, NAT gateway performs NAT64," used with DNS64 on the Route 53 Resolver) and you are back to paying for a NAT gateway for the IPv4-only destinations. **The flip condition is your dependency list**: audit whether your actual outbound destinations have AAAA records. If they do, IPv6-only egress is strictly better. If your payment provider doesn't, you keep the NAT gateway.

### System design ④ — egress for private subnets, and why your outbound calls appear from one IP

*(Day 9 §12's second scenario covered the **security isolation** side of egress-restricted networking — allowlists, DNS exfiltration, IMDS credential theft. That reasoning is not repeated. This scenario is deliberately the **addressing and cost** side: which address the packet appears to come from, whether that address is stable, what it costs, and what breaks at scale.)*

**The problem.** A platform runs ~300 containers across 3 AZs in private subnets. They call four third-party APIs: a payment provider that **requires you to register your egress IP addresses in an allowlist** and rejects traffic from anywhere else; an LLM provider whose enterprise tier offers optional IP allowlisting; a shipping API with per-IP rate limits; and an internal partner behind their own firewall. Design egress. Requirements: a **stable, small, enumerable set** of source addresses; survive an AZ failure; support 20 TB/month; handle traffic growth to 2,000 containers without redesign.

**The core question, and why it has a non-obvious answer.** "Why do all my outbound calls appear to come from one IP?" Because of source NAT: every container's private address is rewritten to the NAT gateway's Elastic IP. From the payment provider's perspective, all 300 containers *are* one host. That is normally an annoyance — it destroys per-container attribution at the network layer — but here it is precisely the feature the requirement depends on. **The provider's allowlist can only work if your egress address is stable and enumerable, and NAT is what makes it so.** If each container had its own public IP, you would have 300 addresses that change on every deploy, and no allowlist could ever be maintained.

**Alternative 1 — public subnets with per-instance public IPs.** Simplest routing. Fails all three ways: 300 ephemeral addresses cannot be allowlisted; every instance becomes inbound-reachable (Day 9's argument); and at $0.005/hr each you are paying ~$1,095/month for addresses alone (§8).

**Alternative 2 — one NAT gateway for the whole VPC.** One stable IP, minimum cost, easiest allowlist. Fails the AZ-failure requirement outright: AWS is explicit that "if the NAT gateway's Availability Zone is down, resources in the other Availability Zones lose internet access." You have deliberately built a single-AZ dependency into a multi-AZ system. It also concentrates all 20 TB through one gateway and all port mappings against one address's 55,000-per-destination ceiling.

**Alternative 3 — one NAT gateway per AZ, each with its own Elastic IP.** Three stable IPs. Each AZ's private subnets route to their own AZ's gateway.

**Alternative 4 — a centralised egress VPC.** All spoke VPCs route `0.0.0.0/0` through a Transit Gateway to one shared egress VPC containing the NAT gateways (and optionally an inspection appliance). One allowlist for the entire organisation, however many VPCs it grows to.

**The decision: alternative 3 for a single-VPC platform; alternative 4 the moment there is a second VPC.** The reason is the specific constraint "a stable, small, enumerable set of source addresses." Three Elastic IPs, one per AZ, allocated once and never released, is exactly that: enumerable (three entries in the provider's allowlist), stable (an Elastic IP is yours until you release it), and AZ-independent. And the per-AZ topology satisfies the failure requirement while also avoiding cross-AZ data-transfer charges on 20 TB — which at ~$0.02/GB round trip would be ~$400/month of pure waste for the privilege of a worse failure domain.

**The trade-off, honestly.** Three gateways cost 3× the hourly fee (~$98.55/month vs ~$32.85) and give you three allowlist entries to keep synchronised with four providers — an operational coupling that will eventually bite you, because the day you add a fourth AZ, or migrate a region, someone must remember to update four external allowlists before the deploy, and nothing in your CI will catch it if they don't. That coupling is a real cost of the design, and the mitigation is to treat egress IPs as a declared, version-controlled interface with an alarm on drift, not as an incidental property of your infrastructure. **The genuine alternative you're giving up** is Alternative 4's single allowlist — which is why the flip condition is so close: at two VPCs, the centralised egress VPC's operational simplicity already beats its added Transit Gateway cost (~$0.05/hr per attachment plus ~$0.02/GB processing) and its extra hop of latency.

**Where the LLM-provider requirement lands, and why it's the same problem.** Provider-side IP allowlisting is one of the few controls that limits the damage from a leaked API key: a stolen key is useless from an address the provider will not accept. Setting that up requires exactly what this design produces — a stable, enumerable egress identity. So an agent platform that runs tool execution in a private subnet inherits this design whether or not anyone planned it, and the practical consequence is worth stating plainly: **if you want provider-side IP allowlisting for your LLM API keys, your NAT egress addresses become a security boundary, and "someone added an AZ" becomes a security-relevant change.** (Note the honest limit of the control: it constrains *where* a key can be used, not *what* it can do, and it is useless against an attacker who has compromised something inside the same VPC — their traffic egresses from the same allowlisted address. It is one layer, not a solution.) Day 9's egress-allowlist reasoning covers the complementary question of restricting *which destinations* the sandbox may reach.

**Failure modes.**

- **Port exhaustion against one destination.** 2,000 containers hammering `api.stripe.com:443` will approach the 55,000-per-unique-destination ceiling. Mitigations, in order of leverage: HTTP keep-alive with a bounded connection pool (turns thousands of sequential calls into a handful of mappings); attach secondary IPv4 addresses to the NAT gateway (up to 8, so 440,000); shard across AZs. **Alarm on `ErrorPortAllocation`.**
- **NAT gateway bandwidth/PPS ceiling.** 5 Gbps scaling to 100 Gbps, 1 Mpps scaling to 10 Mpps, then drops. The PPS limit bites before the bandwidth limit for small-packet workloads.
- **Elastic IP accidentally released.** The address is gone, a new one is different, and every external allowlist is now wrong — an outage caused by an infrastructure change with no code change. Protect Elastic IPs with resource policies and treat them as stateful.
- **The allowlist and reality drift.** Add an AZ, add a gateway, forget the allowlist entry; the new AZ's traffic is rejected while the other two work. Presents as intermittent, AZ-correlated failures. Automate it or alarm on it.
- **Cost blowout via the wrong path.** Container image pulls from ECR, S3 reads, and CloudWatch writes going through the NAT gateway at $0.045/GB when a VPC endpoint would be free. Audit NAT gateway `BytesOutToDestination` against expected *third-party* volume; the gap is money you are lighting on fire.

### System design ⑤ — a fleet outgrows NAT: designing egress for 50,000 concurrent outbound connections

**The problem.** A webhook-delivery service must deliver events to customer endpoints: 50,000 concurrent outbound HTTPS connections, to ~20,000 *distinct* customer domains, sustained. Some customers require you to publish the source IPs you deliver from. Design egress. This scenario is deliberately the mirror image of ④: there, one destination and many sources; here, many destinations.

**Requirements.** 50k concurrent connections; publishable, stable source addresses; per-customer delivery must not be blocked by another customer's slowness; must survive an AZ loss.

**Does NAT even work here?** Do the arithmetic first, because the answer is counterintuitive and it is the whole point of the scenario. The AWS ceiling is 55,000 simultaneous connections **per unique destination**, where a destination is (IP, port, protocol). With 20,000 distinct customer domains, the average connections per destination is 50,000 / 20,000 = **2.5**. That is nowhere near 55,000. So a *single* NAT gateway address handles this comfortably, and the naive fear ("50,000 connections > 55,000 limit? we're close to the edge!") misreads the limit as a global one. **This is exactly why the "per unique destination" qualifier is worth memorising:** ④'s workload is dangerous at 2,000 containers and ⑤'s is safe at 50,000 connections, and the difference is entirely destination cardinality.

The real ceilings here are elsewhere: packets per second (1 Mpps baseline, 10 Mpps scaled) and bandwidth (5 → 100 Gbps). Webhook payloads are small, so PPS is the binding constraint, and 50k connections doing modest request rates sits well inside it.

**Alternative 1 — NAT gateways per AZ (i.e. ④'s answer).** Works, per the arithmetic. Three publishable IPs. Costs the per-GB processing fee on all traffic.

**Alternative 2 — a fleet of egress proxies on EC2/Fargate with Elastic IPs.** More control (you can pin specific customers to specific source IPs, do per-customer connection limits, and log at the application layer), no per-GB NAT processing fee, and you can attach many addresses. Costs you an entire service to operate.

**Alternative 3 — IPv6-first delivery with NAT64 fallback.** No port exhaustion at all for IPv6-capable customers.

**The decision: NAT gateways per AZ, plus per-customer connection limits enforced in the application, and publish the three Elastic IPs.** The reason is that the arithmetic above shows NAT is not the constraint, so the extra machinery of Alternative 2 buys almost nothing against the specific requirements — and the requirement it appears to satisfy ("per-customer delivery must not be blocked by another customer") is not actually a network requirement at all. A slow customer holding connections open is a *concurrency* problem, and the correct fix is a per-customer connection cap and a bounded worker pool in the delivery service (Day 11's connection-management design; Day 49's queueing). Solving it with egress proxies would be solving an application problem with network infrastructure.

**The trade-off.** You give up per-customer source-IP attribution — every customer sees the same three addresses, so if one customer complains about traffic volume you cannot point at a source address to distinguish it. You also pay the $0.045/GB processing fee. **Flip condition:** Alternative 2 becomes right when a *contractual* requirement forces per-customer or per-tenant source addresses (some financial and healthcare integrations do demand a dedicated source IP per tenant), or when your traffic volume makes the per-GB NAT fee exceed the cost of running the proxy fleet — which for 20 TB/month is $900/month of processing, roughly the cost of a small, redundant proxy fleet, so the crossover is genuinely near.

**Failure modes.**

- **One customer's endpoint hangs and holds 10,000 connections open.** Both a NAT-mapping consumer and a worker-pool consumer. Per-customer caps and aggressive timeouts, not more network capacity.
- **A customer's DNS resolves to a shared CDN address.** Now 5,000 "distinct" customers collapse onto one destination IP:port, and the per-destination arithmetic that made this design safe silently stops holding. **This is the sneaky one**, because nothing about your configuration changed — a customer's DNS did. Monitor `ErrorPortAllocation` regardless of how comfortable the arithmetic looked, because destination cardinality is not under your control.
- **Elastic IP reputation.** You are delivering from three addresses to 20,000 endpoints; if one customer's WAF decides those addresses are abusive, every customer's deliveries suffer. This is CGNAT's collateral-damage problem, now with you as the carrier — a genuine argument for Alternative 2 that has nothing to do with capacity.
- **AZ failure with a shared gateway.** Covered in ④; the same trap.

---

## 10. IPv6: the actual fix, and why it took thirty years

**Depth: [WORKING]**

### Intuition — what "more addresses" changes and what it doesn't

IPv6 is not "IPv4 with bigger addresses," although the bigger addresses are the headline. It is a redesign that took the thirty years of operational experience above and removed the things that hurt. §7 already covered the fragmentation changes; §1 mentioned the deleted header checksum. Here is the whole shape.

The address is **128 bits** instead of 32 (§8 did the arithmetic). But the more consequential design decision is the *allocation policy*: an end site gets a **/48** or **/56**, and every LAN gets a **/64** — meaning the standard subnet has 2^64 addresses, more than four billion times the entire IPv4 internet. This is deliberate profligacy. It means **you never subnet for size again.** Every subnet is a /64, always, regardless of whether it holds 3 hosts or 30,000. All of §3's careful arithmetic about how many hosts fit in a /27 simply ceases to be a question you ask. (RFC 4291 §2.5.1 requires /64 for subnets using the standard interface-identifier format, which is why deviating from /64 breaks SLAAC below.)

That single change removes an entire category of work and an entire category of outage. No more "the subnet filled up with pods."

### The notation, and reading an IPv6 address without fear

128 bits written as eight groups of four hex digits, colon-separated. Then two compression rules make it tolerable:

1. **Leading zeros in a group may be omitted.** `0db8` → `db8`, `0000` → `0`.
2. **One run of consecutive all-zero groups may be replaced by `::`** — and only one, because two would be ambiguous.

```
full:        2001:0db8:0000:0000:0000:ff00:0042:8329
rule 1:      2001:db8:0:0:0:ff00:42:8329
rule 2:      2001:db8::ff00:42:8329

loopback:    0000:0000:0000:0000:0000:0000:0000:0001  ->  ::1     (IPv4's 127.0.0.1)
unspecified: 0000:0000:0000:0000:0000:0000:0000:0000  ->  ::      (IPv4's 0.0.0.0)
```

Prefixes work identically to CIDR: `2001:db8::/32` is a 32-bit prefix. `2001:db8:abcd:0012::/64` is a subnet. The `/` notation carried over unchanged, which is one of the reasons §3's mental model transfers wholesale.

Ranges to recognise on sight, the IPv6 counterpart of §3's table:

| Prefix | Meaning | Note |
|---|---|---|
| `2000::/3` | Global unicast — the routable internet | One-eighth of the space; all current allocations come from here |
| `fe80::/10` | Link-local | **Mandatory** — every IPv6 interface always has one; used by NDP |
| `fc00::/7` (in practice `fd00::/8`) | Unique Local Addresses (ULA), RFC 4193 | The rough analogue of RFC 1918 |
| `ff00::/8` | Multicast | IPv6 has **no broadcast** — multicast replaced it entirely |
| `::1/128` | Loopback | One address, not a whole /8 |
| `2001:db8::/32` | **Documentation**, RFC 3849 | Use this in examples; the IPv6 counterpart of RFC 5737 |
| `64:ff9b::/96` | NAT64 well-known prefix, RFC 6052 | An IPv4 address embedded in the low 32 bits |

**Two structural differences from IPv4 worth knowing.** There is **no broadcast address** — everything that was broadcast is now multicast to a specific group, which is why ARP was replaced by **NDP** (Neighbor Discovery Protocol, RFC 4861, using ICMPv6 rather than a separate ethertype). Day 9 forward-referenced NDP; the connection is simply that NDP does ARP's job with multicast and ICMPv6 instead of broadcast. And **every interface has multiple addresses normally** — at minimum a link-local `fe80::` plus one or more global addresses — which makes §2's "an address identifies an interface, not a machine" not merely true but unavoidable.

### SLAAC: the feature that quietly makes IPv6 easier to operate

**Stateless Address Autoconfiguration** (RFC 4862) lets a host configure its own global address with no DHCP server. The router multicasts a Router Advertisement containing the /64 prefix; the host generates the low 64 bits itself and combines them. Because the subnet is always a /64, there is always exactly 64 bits of room for the host part, and the host can pick them without coordination.

Originally the low 64 bits were derived from the MAC address (EUI-64), which was a privacy disaster — your MAC address, and therefore your device, was trackable across every network you joined. The fix is **RFC 8981 temporary addresses** (formerly "privacy extensions"): hosts generate random interface identifiers and rotate them, typically daily. This is now the default on Windows, macOS, iOS, Android, and most Linux distributions.

**The operational consequence you will actually meet:** an IPv6 host's address changes regularly and it has several simultaneously. Any system that logs, allowlists, or rate-limits by exact IPv6 address is broken by design. **The correct unit for IPv6 allowlisting and rate-limiting is the /64** (one subscriber LAN), and often the /56 or /48 (one site). Getting this wrong is the single most common IPv6 mistake in application code: a rate limiter keyed on the full 128-bit address is trivially evaded by a host that has 2^64 addresses available, and a WAF blocking single IPv6 addresses is doing nothing at all.

### The transition mechanisms, and why "just turn it on" isn't a plan

IPv6 is **not** backward compatible with IPv4. An IPv6-only host cannot talk to an IPv4-only host without translation. There is no clever bit; the address spaces are disjoint. This is the fact that determined the last thirty years.

**Dual-stack** is the mainstream answer: run both protocols on every host and network, use IPv6 when both ends support it, IPv4 otherwise. It works, and it costs you *two* of everything — two routing tables, two firewall rule sets, two sets of monitoring, two classes of bug. A firewall correctly locked down on IPv4 and wide open on IPv6 is a real and common failure, because the IPv6 rules were an afterthought. **If you take one security rule from this section: audit your IPv6 rules separately, and assume they are wrong until you have tested them.**

**Happy Eyeballs** (RFC 8305) is the client-side algorithm that makes dual-stack tolerable in practice: try IPv6 and IPv4 connections nearly in parallel, with a small head start for IPv6, and use whichever completes first. This is why broken IPv6 on a network doesn't make websites unreachable for users — the browser silently falls back in a few hundred milliseconds. It also means **broken IPv6 hides**, which is why IPv6 problems are often discovered late and by accident.

**NAT64 + DNS64** lets an IPv6-only client reach an IPv4-only server. DNS64 synthesises an AAAA record for an IPv4-only name by embedding the IPv4 address in a prefix (typically `64:ff9b::/96`); the client sends IPv6 to that synthetic address; a NAT64 translator rewrites it into IPv4 with source NAT. This is §9's egress design again, one layer up — and it is what AWS NAT gateways do when you route IPv6 through them. **464XLAT** (RFC 6877) extends the trick for applications that hard-code IPv4 literals and is what most mobile carriers actually run: a great many phones are IPv6-only on the wire, with 464XLAT making the IPv4 internet appear to work.

### The measured state of adoption, and the answer to "why so slow?"

**Google's public measurement of native IPv6 access crossed 50% for the first time on 28 March 2026, reaching 50.10%, up from 46.33% a year earlier** — eighteen years after Google began measuring in 2008. Source: https://www.google.com/intl/en/ipv6/statistics.html (live data; **verify current** — this is a moving number and the whole point is that it moves).

Eighteen years to half. Why? §9 already gave the core answer and it is worth restating as the general principle: **NAT could be deployed by one operator unilaterally; IPv6 requires everyone.** A network operator who deployed IPv6 in 2005 got approximately nothing for it, because almost nothing they wanted to reach spoke IPv6 — the benefit was entirely external to the party paying the cost. NAT's benefit was entirely internal. In a system with no central authority, that asymmetry decides outcomes, and no amount of technical superiority overrides it.

What finally moved the number was the *cost of IPv4 becoming visible and local*: mobile carriers with tens of millions of subscribers could not obtain enough IPv4 addresses at any price and went IPv6-first with 464XLAT out of necessity; large content networks turned it on because it is now cheaper than buying addresses; and cloud providers began charging for IPv4 (§8), making the internal cost of *not* deploying IPv6 explicit for the first time. That is the flip condition from §9 actually flipping, in public, with a measurable date attached.

*(Per the tier table, [WORKING] concepts owe one worked scenario and one named case study. The worked scenario is §9's egress design, whose IPv6-only variant is analysed there rather than duplicated here; the named case is Google's eighteen-year measurement series above, which is a real, named, primary-sourced dataset rather than an anecdote. I have not manufactured a second.)*

---

## 11. BGP: the trust-based gossip that holds the internet together

**Depth: [WORKING]**

> **A note on scope.** The study plan says "BGP in one paragraph." I am deliberately exceeding that, and here is the justification: **both of Day 10's required case studies are BGP incidents.** The Meta 2021 outage and the 2008 YouTube hijack are incomprehensible — not just less vivid, actually incomprehensible — without knowing what an AS is, what an UPDATE and a withdrawal do, and how longest-prefix match interacts with route propagation. One paragraph would let me *name* BGP; it would not let me *teach* the two things the day exists to teach. So this section runs at [WORKING] depth: you will finish able to explain both incidents and to reason about routing security, but I am not going to open route reflectors, confederations, or the full path-attribute set, and I will say where I stop.

### Intuition — a second, different routing problem

§4 explained hop-by-hop forwarding: each router consults a table. It did not explain **how the table gets filled**, and that turns out to be two completely different problems depending on whether you are inside one organisation or between many.

**Inside one organisation** — a campus, a datacentre, an ISP's own backbone — you control everything, you want the shortest path, and "shortest" is a well-defined engineering question. Protocols for this are called **interior gateway protocols**: OSPF and IS-IS (each router floods link-state information and everyone computes Dijkstra over a shared map) and EIGRP. They optimise a metric.

**Between organisations**, shortest path is not merely hard, it is the **wrong objective**. Consider: your ISP has a cheap peering link to a competitor and an expensive transit link to a tier-1 provider. The shortest path to some destination might be over the expensive link. Your ISP does not want that. It wants the *cheapest* path that meets its obligations — and it will not tell you its costs, its contracts, or its internal topology, because those are trade secrets and competitive information.

So interdomain routing needs a protocol that (a) lets each network apply **arbitrary local policy** without justifying it, (b) reveals as little internal structure as possible, (c) still converges to loop-free paths, and (d) scales to 79,231 independent participants. **BGP is that protocol,** and it is the *only* one — there is genuinely no alternative in production anywhere on the public internet, which is worth saying plainly per the decision-making discipline: this is one of the rare cases where there is no real competing option, and knowing that is itself useful. The relevant decisions in BGP are all about *policy configuration and safety mechanisms*, not about protocol choice.

Current specification: **RFC 4271** (BGP-4, 2006).

### The vocabulary, built from the ground up

**Autonomous System (AS).** One network under a single administrative and routing policy — an ISP, a large enterprise, a university, a cloud provider, a CDN. Each gets a globally unique **AS number** (originally 16-bit, now 32-bit). Real ones you have already seen in this note: **AS32934** is Meta/Facebook, **AS36561** was YouTube in 2008, **AS17557** is Pakistan Telecom, **AS3491** is PCCW Global, **AS701** is Verizon, **AS13335** is Cloudflare, **AS6453** and **AS2914** (NTT) appeared in §6's Capture D reverse-DNS names. Measured count: **79,231** ASes advertising routes as of 11 August 2026.

**What BGP actually exchanges.** Not topology. Not metrics. **Reachability with a path.** An announcement means: *"I, AS X, can reach prefix P, and the sequence of ASes to get there is [X, Y, Z]."* That sequence is the **AS_PATH** attribute — RFC 4271 defines it as identifying "the autonomous systems through which routing information carried in this UPDATE message has passed."

**AS_PATH does two jobs, and the second one is the clever part.** It is the primary metric (shorter path preferred, all else equal), *and* it is the loop-prevention mechanism: **a BGP speaker that receives an announcement whose AS_PATH already contains its own AS number discards it.** RFC 4271 puts it as enabling construction of an AS connectivity graph from which "routing loops may be pruned." Note the contrast with §5: TTL is a *last-resort* loop killer that operates on packets after the fact; AS_PATH prevents *routing-table* loops from forming at the AS level in the first place. Both exist because each catches what the other misses — AS_PATH cannot prevent a loop inside a single AS or between routers whose tables disagree transiently, and TTL cannot prevent a bad route from propagating.

**The four message types** (RFC 4271): **OPEN** establishes a session and negotiates parameters; **UPDATE** carries announcements and withdrawals; **NOTIFICATION** reports an error and closes the session; **KEEPALIVE** proves liveness.

**BGP runs over TCP, port 179.** This is a genuine design decision with a stated reason: RFC 4271 notes that using TCP eliminates "the need to implement explicit fragmentation, retransmission, acknowledgement, and sequencing." A routing protocol that reuses TCP rather than reimplementing reliability. The consequence is that a BGP session is a long-lived TCP connection between two specific routers, which is why §5's GTSM (TTL=255) works as a hardening measure, and why BGP session security is a TCP security problem (MD5 authentication, now TCP-AO).

**eBGP vs iBGP.** RFC 4271: external BGP connects peers "in different autonomous systems," internal BGP connects peers "within the same AS." The crucial behavioural difference: "When advertising routes externally, speakers prepend their own AS number to the AS_PATH; internally, the AS_PATH remains unmodified." So AS_PATH counts *AS hops*, not router hops — a path across a continent-spanning ISP counts as one.

**Withdrawal.** An UPDATE has a "Withdrawn Routes" field identifying routes "no longer available for use." **This is the mechanism at the heart of the Meta outage.** Withdrawing a prefix does not say "this is broken" or "try elsewhere"; it says *"I no longer reach this."* If nobody else announces it, the prefix ceases to exist from the internet's point of view — packets to it are dropped at the first router with no matching route. There is no error, no notification to the owner, no fallback. **The prefix simply stops being a place.**

### Analogy — the rumour network of shopkeepers, and where it breaks

Imagine a network of shopkeepers, each of whom tells their neighbours what goods they can source and through whom: *"I can get olive oil, via Maria, via Giorgio."* Each shopkeeper hears many such claims, picks one per product according to their own private reasons (Maria is family; Giorgio overcharges; the route through Anna is shortest), and repeats their chosen claim onward with their own name added to the front. Nobody publishes their reasons. Nobody verifies anyone's claims. If a shopkeeper announces "I can get olive oil directly," their neighbours generally believe them and route customers accordingly.

**Where the analogy breaks — three ways, and they are precisely the two case studies plus the fix.**

1. **A shopkeeper who lies gets found out when a customer shows up and there's no oil. In BGP, the customer's packets just vanish, and the victim usually finds out from Twitter.** There is no feedback path from "traffic was blackholed" to "someone announced a route they shouldn't have." This is the entire content of the 2008 YouTube incident.
2. **Shopkeepers weigh a specific claim against a vague one using judgement. Routers use longest-prefix match, mechanically and unconditionally.** A more-specific announcement wins **regardless of who made it or how implausible it is** (§4). A liar who announces a *more specific* prefix beats the legitimate owner automatically, everywhere, instantly. This is the mechanism of both the YouTube hijack and the 2019 Verizon/DQE leak.
3. **A shopkeeper who says "I no longer stock olive oil" causes customers to look elsewhere. A network that withdraws a prefix causes traffic to that prefix to be discarded, including — critically — the traffic the network itself needs in order to fix the problem.** Withdrawal is not graceful degradation; it is deletion. That is the Meta outage.

### BGP's path selection, and why "shortest" is the wrong word

When a router hears several announcements for the same prefix, it picks one *best* path. The order is roughly standardised (with vendor variation) and the ordering itself is the interesting part:

```
0. LONGEST PREFIX MATCH happens first and is not part of this contest at all.
   A /24 and a /22 are different destinations. The /24 always wins for addresses
   inside it. Everything below applies only among announcements of the SAME prefix.

1. Highest LOCAL_PREF        <- purely local policy. "prefer my peering link."
2. Shortest AS_PATH          <- the only thing resembling a distance metric
3. Lowest ORIGIN type        <- IGP < EGP < INCOMPLETE
4. Lowest MED                <- a hint from a neighbour about which of ITS entry points
5. eBGP preferred over iBGP  <- prefer externally-learned
6. Lowest IGP metric to next hop  <- "hot potato": hand it off ASAP
7. tie-breakers (oldest route, lowest router ID, ...)  <- stability over optimality
```

**Step 1 comes before step 2, and that is the whole story of interdomain routing.** LOCAL_PREF is *pure local policy* with no technical meaning — an operator sets it to encode business relationships, and it outranks every distance-like consideration. This is why §6's Capture D sent my packets from Mumbai eastward across the Pacific to reach Oregon: not because that is short, but because that is what my ISP's commercial relationships prefer. **BGP is a business-policy protocol wearing a routing protocol's clothes.** Any mental model of "the internet finds the fastest path" is simply wrong, and it explains a category of latency problem that no amount of application optimisation can fix.

**Step 0 deserves one more emphasis because it is the load-bearing fact for both case studies:** prefix length is not a tie-breaker, it is a *pre*-filter. A more-specific announcement does not compete with a less-specific one; it wins by definition.

**Where I am deliberately stopping.** I am not opening: route reflectors and confederations (which solve iBGP's full-mesh scaling problem inside large ASes), the full path-attribute list, communities (the tagging mechanism operators use to signal policy to each other, including blackhole communities), BGP add-path, or MP-BGP (the multiprotocol extension that carries IPv6 and VPN routes). If you operate a network, all of those become [CORE] for you and RFC 4271 plus a good operators' text is the next step. For a backend engineer, the material above is the level at which BGP explains your outages.

### Case study ① — Meta, 4 October 2021: a network that deleted itself

This is the most instructive outage in modern internet history for a backend engineer, because the *interesting* failure was not the trigger. It was everything that happened afterwards.

**The trigger, from Meta's own postmortem.** During routine work, "a command was issued with the intention to assess the availability of global backbone capacity, which unintentionally took down all the connections in our backbone network, effectively disconnecting Facebook data centers globally." There was a safeguard designed to prevent exactly this — an audit tool that should have rejected the command — and "a bug in that audit tool prevented it from properly stopping the command."

**The cascade, which is the part to study.** Meta's smaller points of presence answer DNS queries and advertise their nameserver addresses via BGP. Meta had built a health-check into that design: **"our DNS servers disable those BGP advertisements if they themselves can not speak to our data centers."** The intent is sound and even admirable — if a DNS server is cut off from the authoritative data, it should stop attracting queries it cannot answer correctly, so that traffic goes to a healthy site instead.

But the backbone failure cut off **every** DNS server from **every** data centre simultaneously. So every DNS server independently and correctly concluded it was unhealthy, and every one of them **withdrew its BGP advertisements**. Meta's own words: they "declare themselves unhealthy and withdraw those BGP advertisements."

Now recall what withdrawal means (§11's vocabulary section): the prefixes containing Meta's authoritative nameservers **ceased to exist** from the internet's point of view. Cloudflare's contemporaneous analysis names two of the withdrawn prefixes, `185.89.218.0/23` and `129.134.30.0/23`, and their timeline is precise: routing changes from Facebook peaked at **15:40 UTC**, Cloudflare opened an internal incident about DNS failures at **15:51**, and by **15:58 Facebook had stopped announcing routes to its DNS prefixes.** Every resolver on earth then returned SERVFAIL for `facebook.com`, `instagram.com`, and `whatsapp.com` — not "no such host," but "I could not reach any nameserver." Cloudflare's `1.1.1.1` resolver saw roughly **30× normal query volume** as clients and applications retried aggressively.

**Why this was a six-hour outage and not a twenty-minute one.** The servers were running. The data was intact. The fix was a configuration change. And it took from 15:40 to **21:20 UTC** (DNS availability restored) and **21:28 UTC** (Facebook reconnected to the global internet) — about **5 hours 48 minutes**, universally reported as roughly six hours. The reason is that the failure had destroyed the tools needed to fix it. Meta: **"the total loss of DNS broke many of the internal tools we'd normally use to investigate and resolve outages."** Engineers could not reach the routers remotely, because reaching them required DNS, which required the network that was down. Meta had to send people physically to data centres — and their own postmortem explains why even that was slow: "these facilities are designed with high levels of physical and system security in mind. They're hard to get into," and once inside, "the hardware and routers are designed to be difficult to modify even when you have physical access to them," requiring "extra time to activate the secure access protocols needed to get people onsite and able to work on the servers."

**Two corrections I have to make, because verification changed what I would otherwise have written.**

*Correction 1 — to the study plan's framing, and to what I would have said from memory.* The plan says "a maintenance command withdrew Facebook's BGP routes." **That is not what happened**, and the difference is the entire lesson. The command took down the **backbone**. Meta's **DNS servers then withdrew their own BGP routes, deliberately and correctly, because their health-check logic told them to.** No human and no command withdrew those routes. **A safety mechanism, working exactly as designed, converted a network partition into a global disappearance.** If you flatten this into "a command withdrew the routes," you learn "be careful with commands," which is nearly worthless. The real lesson is much sharper: *a health check that withdraws capacity is a correct local decision that can be a catastrophic global one when the condition it detects is correlated across every instance.* Any withdraw-on-unhealthy mechanism needs a floor — never withdraw the last N, never withdraw more than X% within a window, and require an independent signal before withdrawing everything. Note that this is the same class of bug as a load balancer removing every backend because all health checks failed simultaneously due to a shared dependency, which is why AWS ELB and similar services have a "fail open" behaviour that serves traffic to *all* targets when *none* are healthy — the deliberate choice that no-information is better than no-capacity.

*Correction 2 — on the badge readers.* The plan says "staff badges/tools failed." The **tools** part is confirmed directly by Meta's postmortem, quoted above. The **badge** part is *not* in Meta's postmortem — Meta says only that the facilities are "hard to get into" and the hardware hard to modify. The widely-repeated claim that badge readers failed and engineers cut into server cages comes from press reporting (notably *The New York Times*, sourcing an employee) and from secondary accounts, not from Meta. **It is plausible and consistent with what Meta wrote, but it is not primary-sourced, and I am labelling it as reporting rather than fact.** I'd rather hand you a slightly less dramatic story you can defend in an interview than a vivid one that a careful interviewer can puncture.

**A third detail worth extracting, because it is a design lesson people skip.** Meta could not simply turn everything back on. Their postmortem: "we knew that flipping our services back on all at once could potentially cause a new round of crashes due to a surge in traffic," and the data centres had shown power draw "dips in power usage in the range of tens of megawatts" — so "suddenly reversing such a dip could put everything from electrical systems to caches at risk." They had rehearsed this: their "storm" drills, which deliberately take a service or region offline to practise recovery, are what let them stage the restart. **The recovery from a total outage is itself a capacity problem, and the thundering herd of a cold start can cause the second outage.** This connects directly to §4's system design ① (each region must carry the whole load) and to Day 15's cache-stampede material.

**Engineering lessons, tied to this note's mechanisms.**

1. **BGP withdrawal is deletion, not degradation.** A withdrawn prefix is not slow or degraded; it is gone.
2. **Correlated health checks are a single point of failure wearing a resilience costume.** Add floors and rate limits to any automatic withdrawal.
3. **Your control plane must not depend on your data plane.** Out-of-band management — a separate network, separate DNS, separate credentials, serial console access, addresses reachable without your own infrastructure — is not paranoia, it is the difference between 20 minutes and 6 hours. Most organisations discover they don't have it during the incident that needs it.
4. **Shared infrastructure means correlated failure across things that "aren't related."** Meta's own note flags the physical-security systems' role in slowing recovery; the general form is that the internal tools, the auth system, the chat used to coordinate, and the production network were all one dependency graph.
5. **Day 12 will revisit this from the DNS side**, and the compounding is the point: BGP withdrawal made the *DNS* unreachable, and DNS unreachability made *everything* unreachable. Two layers, one incident.

**Primary sources.** Meta's engineering postmortem: https://engineering.fb.com/2021/10/05/networking-traffic/outage-details/ (and the shorter same-day note at https://engineering.fb.com/2021/10/04/networking-traffic/outage/). Cloudflare's contemporaneous BGP and DNS analysis, with the timeline and prefixes: https://blog.cloudflare.com/october-2021-facebook-outage/. All quotations above are from these two sources.

### Case study ② — Pakistan Telecom vs YouTube, 24 February 2008: two hours, one /24

**What happened, verified against RIPE NCC's Routing Information Service analysis.**

The Pakistani government ordered ISPs to block YouTube. Pakistan Telecom (**AS17557**) implemented the block the way many operators do: it created a route for YouTube's address space pointing into a null interface, so that its own customers' traffic to YouTube would be discarded locally. This is standard, legitimate, and entirely internal.

The mistake was that this internal route **leaked into eBGP.** At **18:47 UTC on 24 February 2008**, AS17557 began announcing **`208.65.153.0/24`** to its upstream provider, **PCCW Global (AS3491)**. PCCW accepted it and propagated it to the rest of the internet.

**Now apply §4's longest-prefix match, and watch a local block become a global outage.** YouTube (**AS36561**) announced **`208.65.152.0/22`**. Pakistan Telecom announced `208.65.153.0/24`. Verified by computation: `208.65.152.0/22` spans `208.65.152.0`–`208.65.155.255` (1,024 addresses), and `208.65.153.0/24` spans `208.65.153.0`–`208.65.153.255` (256 addresses) — and `.subnet_of()` confirms **the /24 sits entirely inside the /22.**

So every router on the internet now held two entries covering `208.65.153.x`: YouTube's /22 and Pakistan Telecom's /24. **Longest prefix match is unconditional.** It does not weigh plausibility, it does not consider that AS36561 is YouTube's actual AS, it does not care that the /24 came from a national telco with no relationship to YouTube. The /24 is more specific, so **the /24 wins, everywhere, instantly.** Traffic destined for that quarter of YouTube's address space — from every continent — was routed to Pakistan Telecom, which dropped it into the null interface it had built for its own customers.

**The counterattack, which is a masterclass in how to fight a hijack.** RIPE NCC's timeline:

```
18:47 UTC  AS17557 announces 208.65.153.0/24; AS3491 propagates it globally.
           YouTube's own 208.65.152.0/22 loses to it by longest-prefix match.
20:07 UTC  YouTube begins announcing 208.65.153.0/24 itself -- matching the
           hijacker's specificity so that normal BGP selection applies again,
           and YouTube wins wherever its AS_PATH is shorter.
20:18 UTC  YouTube announces 208.65.153.0/25 and 208.65.153.128/25 -- two /25s,
           MORE specific than the hijacked /24, which therefore beat it
           unconditionally by the same rule that had beaten YouTube.
20:51 UTC  The hijacked /24 appears with an extra AS17557 prepended to its
           AS_PATH, lengthening it and making YouTube's route more attractive
           in the step-2 comparison.
21:01 UTC  AS3491 (PCCW) withdraws all prefixes originated by AS17557 --
           disconnecting its customer entirely. The hijack ends.
```

Total duration: **approximately two hours.** RIPE's RIS data shows the AS_PATH shift at one collector (AS3333) from `3333 6320 3549 3491 17557` back to `3333 3356 3549 36561` as control returned.

Read the 20:18 move carefully, because it is the deepest lesson in this section: **YouTube defended itself by using the attack.** It announced *more specific* prefixes than the hijacker, because the only reliable way to beat longest-prefix match is to be longer. The consequence is uncomfortable — **the only defence available to a victim is to make the global routing table bigger**, which is §4's 512k-day tragedy of the commons in action. And there is a hard ceiling: you cannot go longer than /24, because essentially every operator filters announcements more specific than /24 to keep the table bounded. So **a /24 hijack of a /24 has no more-specific defence at all.** You are reduced to phoning people.

**And note the fix at 21:01: PCCW withdrew everything from its customer.** The incident ended because a *human at an upstream provider* pulled the plug. Not a protocol, not automation. Two hours of a global service being blackholed, ended by a phone call and a config change.

**Engineering lessons.**

1. **BGP has no authentication of origin.** Any AS could, in 2008, announce any prefix, and its upstream's willingness to propagate was the only control. This is not a bug in an implementation; it was the protocol's design assumption — mutual trust among a small number of cooperating research networks — surviving into a commercial internet with 79,000 participants.
2. **Longest-prefix match is a security property, and it is the wrong one.** The mechanism that makes routing efficient makes hijacking trivially effective. Both are the same line of code.
3. **The real control was PCCW's filter, and it didn't exist.** An upstream provider should accept from a customer only the prefixes that customer is authorised to originate. That filter, applied at AS3491, would have stopped this at the first hop and cost nothing.
4. **Accidents look exactly like attacks.** Pakistan Telecom was not attacking YouTube; it was blocking it locally and leaked. The most effective BGP hijacks of the following decade were, similarly, mostly accidents — and the security posture that defends against one defends against the other.

**Primary sources.** RIPE NCC's RIS case study, which is the authoritative reconstruction: https://www.ripe.net/about-us/news/youtube-hijacking-a-ripe-ncc-ris-case-study/. Renesys's contemporaneous blog analysis (Renesys became Dyn, then Oracle; the original is archived, e.g. https://crysp.uwaterloo.ca/courses/cs458/F08-lectures/local/www.renesys.com/blog/2008/02/pakistan_hijacks_youtube_1.shtml.html — **verify current**, the canonical URL has moved several times). A NANOG-archived operational thread from the day itself provides the raw operator reaction.

### The third incident, which shows the mechanism without the malice: Verizon and a BGP optimiser, 24 June 2019

I am including a third case briefly because it isolates the *mechanism* from the *intent* better than either required case study, and it is thoroughly documented.

**What happened.** A small Pennsylvania ISP, **DQE Communications (AS33154)**, ran a commercial "BGP optimiser" (a Noction appliance). Such a product improves routing by **de-aggregating** received prefixes — splitting them into "more-specifics" so it can steer traffic per sub-block. Cloudflare's `104.20.0.0/20`, for example, became `104.20.0.0/21` and `104.20.8.0/21`. These more-specifics are meant to stay strictly internal.

They leaked. DQE announced them to its customer **Allegheny Technologies (AS396531)**, which announced them to **Verizon (AS701)**, which — without a filter that would have caught it — propagated them to the global internet. By longest-prefix match, the /21s beat the legitimate /20 everywhere, and traffic for large parts of Cloudflare, Amazon, and Linode was drawn into a small ISP's network that could not possibly carry it. **Cloudflare measured "a loss, at the worst of the incident, of about 15% of our global traffic."** The incident began around **10:30 UTC**.

**Why this one is pedagogically valuable.** No hijack was intended by anyone. A *routing-optimisation product* did its job, a *customer* passed it along, and a *tier-1 provider* failed to filter its customer's announcements. Three parties, no malice, global outage. It demonstrates that §4's longest-prefix match plus §11's absent origin validation is a *structural* hazard: you do not need an attacker, only a normal chain of businesses without filters. It is also the clearest possible argument that the fix has to be **filtering at every customer-provider boundary**, not vigilance.

**Primary source:** https://blog.cloudflare.com/how-verizon-and-a-bgp-optimizer-knocked-large-parts-of-the-internet-offline-today/

### The fixes, and their honestly-measured state

Given three incidents with the same shape, what has actually been deployed?

**Prefix filtering (the oldest and still the most important).** A provider accepts from each customer only the prefixes that customer is authorised to originate, built from **IRR** (Internet Routing Registry) records. This would have stopped all three incidents. It is unglamorous, it requires the provider to maintain per-customer filters, and its absence at AS3491 in 2008 and AS701 in 2019 is why both incidents were global rather than local.

**RPKI + Route Origin Validation (the cryptographic fix).** Address holders publish a signed **ROA** (Route Origin Authorisation) stating "AS X is authorised to originate prefix P, up to maximum length L." Routers fetch and validate these, classifying each announcement as Valid, Invalid, or NotFound, and **drop Invalids**. RPKI would have made Pakistan Telecom's announcement Invalid (wrong origin AS) and DQE's more-specifics Invalid (exceeding the ROA's maximum length — note that `maxLength` is the field that specifically defends against de-aggregation leaks).

**Its measured state, and the honest gap.** As of **29 June 2026**, ROA coverage reached a record **67.43% of announced prefixes — 1,065,730 of 1,580,470 routes** covered by a signed ROA. That is real progress. But **signing is not validating**, and the second number is the sobering one: research published in 2026 found only about **12.3% of ASes achieve full ROV protection on their routes, while 36.2% do not validate at all.** So two-thirds of prefixes carry a signature that most networks are not checking. Also worth knowing when you read RPKI alerts: roughly **96.9% of RPKI-invalid prefixes stem from misconfiguration rather than attack** — so an "Invalid" is far more likely to be someone's stale ROA than a hijack. **Verify current** — these are actively-moving figures; the live sources are the NIST RPKI Monitor (https://rpki-monitor.antd.nist.gov/) and Cloudflare's https://isbgpsafeyet.com/.

**What RPKI does *not* fix, which is the part people overstate.** ROV validates the **origin** AS only — it says "AS X may announce P." It does **not** validate the rest of the AS_PATH. So an attacker who announces a legitimate origin with a forged path in front of it produces a *Valid* announcement. Path validation is **BGPsec** (RFC 8205), which requires cryptographic signing of each AS hop, is computationally expensive, requires near-universal adoption to help at all, and is essentially undeployed. **This is the honest state of internet routing security: origin validation is two-thirds signed and one-eighth enforced, and path validation does not exist in practice.**

**MANRS** (Mutually Agreed Norms for Routing Security) is the industry initiative that packages the operational baseline — filtering, anti-spoofing, coordination contacts, publishing routing data. It is a commitment programme rather than a technology, which tells you something true: the remaining problem is mostly organisational.

---

## 12. Anycast: one address in many places at once

**Depth: [WORKING]**

### Intuition — using BGP's ambiguity on purpose

Every incident in §11 came from two networks announcing overlapping address space and routers picking one. Anycast is that exact situation, arranged deliberately, by one owner.

**You announce the same prefix, via BGP, from many locations simultaneously.** Every router on the internet then has several equally-valid paths to that prefix and picks the topologically nearest by its own normal path selection. Clients in Europe reach the European site; clients in Asia reach the Asian site. **Every client uses the identical destination IP address, and the routing system does the steering.**

RFC 4786 (BCP 126) defines it as making "a Service Address available in multiple, discrete, autonomous locations, such that datagrams sent are routed to one of several available locations," and notes "the routing system decides which node is used for each request, based on the topological design of the routing system."

Nothing new is required. No protocol extension, no client support, no DNS trickery. It is BGP being used as a load balancer, and the only reason it works is that BGP already had to tolerate multiple announcements of the same prefix.

### Analogy — the emergency number, and where it breaks

Dial 999 or 911 or 112 anywhere in the country and you reach *your local* dispatch centre. One number, hundreds of centres, and the phone network routes you to the nearest. You never learn which one you got and you don't care. If a centre goes offline, calls route to the next one.

**Where it breaks — three ways, and they define anycast's whole applicability envelope.**

1. **A phone call, once connected, stays connected to that centre. An anycast "connection" has no such guarantee.** Each *packet* is routed independently (§1: IP is stateless), so if BGP re-converges mid-conversation, your next packet arrives at a **different site with no knowledge of your session**. For a single UDP query this is invisible. For a 40-second TCP upload it is a reset. RFC 4786 is explicit that anycast suits "short-lived, stateless transactions" and warns about "long-lived TCP connections where mid-session route changes can redirect packets to different nodes."
2. **You can ask the operator which centre answered. With anycast, "which site am I hitting?" is genuinely hard to answer, and it differs per client.** RFC 4786 notes monitoring becomes much harder because "observed availability changes according to the location of the client within the network." You cannot test anycast from one vantage point; you need probes distributed globally, which is why RIPE Atlas exists and why every serious anycast operator runs distributed monitoring.
3. **"Nearest" means nearest *by the phone network's routing*, which may not be nearest geographically — and with anycast it definitely isn't.** BGP picks by AS_PATH length and LOCAL_PREF (§11), not by distance or latency. §6's Capture D is the proof: my packets to Oregon went east across the Pacific. An anycast site 300 km away can lose to one 8,000 km away if the AS paths favour it. This is why anycast operators spend most of their effort on *peering*, not on adding sites: an unpeered site attracts no traffic, and a badly-peered one attracts the wrong traffic.

### Worked example — anycast measured, right here

`1.1.1.1` is anycast (Cloudflare, AS13335, announced from 300+ locations). Here is the measurement from §6, with the arithmetic that proves it:

```
tracert -h 8 -w 1500 -d 1.1.1.1        # captured 2026-08-11, Mumbai
  1     1 ms     1 ms     1 ms  192.168.29.1
  2     6 ms     5 ms     6 ms  10.50.136.1
  ...
  7    11 ms    11 ms    11 ms  49.44.66.194
  8    11 ms    11 ms    11 ms  49.44.66.195
Trace complete.        <- 8 hops, 11 ms
```

Eight hops and 11 ms to an address that is simultaneously "in" hundreds of cities. Compare with §6's Capture A: 23 hops and 205 ms to a *unicast* address in Sweden. Same laptop, same minute, same ISP. **The difference is not that Cloudflare has faster routers; it is that there is a Cloudflare site inside or adjacent to my ISP's network, and its BGP announcement won.** The 11 ms is the measurement that proves the packet never left the region — 205 ms is what leaving the region costs from here.

That 11 ms figure is also the reason anycast is the backbone of the modern internet's front door. **DNS root servers** are the canonical deployment: there are 13 root server *identities* (`a.root-servers.net` through `m.root-servers.net`) but well over 1,900 actual *instances* worldwide, because each identity is anycast. This is how a system with a hard 13-name limit (originally driven by the 512-byte DNS-over-UDP constraint from §7) scales to serve the planet. Day 12 owns the DNS side; the anycast half is here.

### Where anycast is right, and where it is wrong

**Right for:** DNS resolvers and authoritative DNS (single-packet UDP transactions — the perfect fit); CDN edges and HTTP front doors (short connections, and a re-convergence mid-request costs one retry); DDoS absorption (RFC 4786 lists "DDoS mitigation by localizing attack damage to single nodes" — a 2 Tbps attack against an anycast prefix is automatically divided across every site by the routing system, and *only the sites near the botnet* absorb it, which is why anycast is the primary architectural defence against volumetric DDoS); and NTP.

**Wrong for, or requiring care:** long-lived stateful connections (large uploads, WebSockets, database protocols, long-running SSE streams); anything requiring session affinity without an application-layer mechanism; and any service where you need to deliberately shift a *fraction* of traffic, because BGP gives you no fractional control.

**The mitigation that makes anycast work for TCP anyway**, and it is what every CDN does: **anycast only the edge, and terminate TCP/TLS there.** The client's TCP connection is short (a single HTTPS request), so the probability of re-convergence during its life is tiny. Behind the edge, the connection to the origin is a *separate*, unicast, long-lived, pooled connection. So anycast steers the first hop and unicast carries the state. This is why "anycast" and "CDN" are so entangled in practice, and it is Day 15's material.

**How you actually get anycast without owning address space,** which matters because most readers never will: you rent it. AWS Global Accelerator hands you two static anycast IPs fronting your regional endpoints. CloudFront, Cloudflare, and Fastly are anycast by construction. Azure Front Door and GCP's global load balancer likewise. **The decision for most teams is not "build anycast vs DNS failover" but "rent anycast vs DNS failover,"** and rented anycast removes essentially all of the cost objections raised in §4's system design ① while keeping the benefits — which is precisely why it is the default answer for HTTP front doors today. Build your own only if you *are* the infrastructure.

*(§4's system design ① is the full four-move decision analysis for anycast vs DNS failover vs client-side failover. It is there rather than here because the decision is a routing decision and it needed §4's forwarding model; I am not repeating it.)*

---

## 13. The Build, part two — carve `10.0.0.0/16` on paper

*(Day 9's Build and its first system design covered the **trust boundaries**: which tier may talk to which, security groups, NACLs. This is deliberately the other half — the **arithmetic**. Every number below was generated by computation, not recall.)*

### The requirements, stated before any arithmetic

This is the step people skip, and skipping it is why VPCs get re-addressed. Before choosing a single prefix:

1. **How many AZs now, and how many ever?** Two now, three eventually. Design for three, deploy two.
2. **How many tiers?** Three: public (load balancers, NAT gateways), app (private, compute), data (private, isolated).
3. **How many addresses per tier, per AZ, at peak?** Public: a handful of ENIs — an ALB wants at least 8 free addresses per subnet and scales, a NAT gateway needs 1. Call it under 30, and size generously anyway because addresses are free inside a /16. App: this is the one that matters. If it is EKS with the VPC CNI, **every pod gets a VPC address** (§3). Assume 200 nodes × 30 pods = 6,000, plus ENIs, plus double it for rolling deploys → target **8,000+**. Data: RDS/Aurora instances and endpoints, a few dozen.
4. **What must this VPC never collide with?** On-prem `10.10.0.0/16`, the other two VPCs, any partner network you may peer with, and — this catches people — **Docker's default bridge `172.17.0.0/16`** and Kubernetes' default service CIDR. A documented, widely-hit conflict is Docker's `172.17.0.0/16` (inside RFC 1918's `172.16.0.0/12`) colliding with corporate VPNs that also allocate from `172.16.0.0/12`; the symptom is that containers cannot reach corporate resources, or the VPN breaks when Docker starts. The fix is to set `"bip"` or `"default-address-pools"` in `/etc/docker/daemon.json`. **Choose your VPC space so it cannot collide with your own tooling's defaults.**
5. **Reserve for growth.** Half the /16, unallocated, from day one.

### The plan

```
VPC                    10.0.0.0/16        10.0.0.0   - 10.0.255.255    65,536 addrs

  IN USE               10.0.0.0/17        10.0.0.0   - 10.0.127.255    32,768
  RESERVED (growth)    10.0.128.0/17      10.0.128.0 - 10.0.255.255    32,768

  Inside 10.0.0.0/17 -- tier-major layout:

    public tier        10.0.0.0/20        10.0.0.0   - 10.0.15.255      4,096
      pub-az-a         10.0.0.0/24        10.0.0.0   - 10.0.0.255         256   ->  251 usable
      pub-az-b         10.0.1.0/24        10.0.1.0   - 10.0.1.255         256   ->  251 usable
      (pub-az-c        10.0.2.0/24        reserved for the third AZ)
      free inside tier 10.0.3.0/24, 10.0.4.0/22, 10.0.8.0/21

    data tier          10.0.16.0/20       10.0.16.0  - 10.0.31.255      4,096
      db-az-a          10.0.16.0/24       10.0.16.0  - 10.0.16.255        256   ->  251 usable
      db-az-b          10.0.17.0/24       10.0.17.0  - 10.0.17.255        256   ->  251 usable
      (db-az-c         10.0.18.0/24       reserved)
      free inside tier 10.0.19.0/24, 10.0.20.0/22, 10.0.24.0/21

    app tier
      app-az-a         10.0.32.0/19       10.0.32.0  - 10.0.63.255      8,192   -> 8,187 usable
      app-az-b         10.0.64.0/19       10.0.64.0  - 10.0.95.255      8,192   -> 8,187 usable
      (app-az-c        10.0.96.0/19       reserved for the third AZ)
```

**Verification, by computation.** No overlaps between any pair of the allocated blocks; the six blocks `10.0.0.0/20`, `10.0.16.0/20`, `10.0.32.0/19`, `10.0.64.0/19`, `10.0.96.0/19`, `10.0.128.0/17` sum to exactly **65,536** addresses and `ipaddress.collapse_addresses()` collapses them back to precisely **`10.0.0.0/16`** — which is the proof that the carve is complete, contiguous, and gapless. Run it yourself:

```python
import ipaddress as ip
plan = ["10.0.0.0/20","10.0.16.0/20","10.0.32.0/19",
        "10.0.64.0/19","10.0.96.0/19","10.0.128.0/17"]
nets = [ip.ip_network(c) for c in plan]
assert not any(a.overlaps(b) for i,a in enumerate(nets) for b in nets[i+1:])
print(list(ip.collapse_addresses(nets)))   # -> [IPv4Network('10.0.0.0/16')]
print(sum(n.num_addresses for n in nets))  # -> 65536
```

Check one leaf by hand to make sure you believe it. `10.0.32.0/19`: 32 − 19 = 13 host bits, so 2^13 = 8,192 addresses. `10.0.32.0` + 8,192 − 1: the third octet advances by 8192/256 = 32, so the last address is `10.0.63.255`. Correct. Minus AWS's 5 → **8,187 usable**, which comfortably clears the 8,000 pod target.

### The decisions inside this plan, each with its rejected alternative

**Decision 1: /16 for the VPC.** *Alternative:* a /20, which is plenty of addresses for the current workload and leaves the rest of `10.0.0.0/8` free for other VPCs. *Reason /16 won:* addresses inside a VPC cost nothing, the VPC CIDR **cannot be shrunk** after creation (you can add secondary CIDRs, but that fragments the space and complicates route tables and peering), and running out is a re-addressing project measured in weeks. *Trade-off:* you burn 1/256th of `10.0.0.0/8` per VPC, so an organisation with 300 VPCs cannot give every one a /16 out of a single /8. *Flip condition:* if you are running many VPCs at scale, assign from a central IPAM with per-VPC /20s or /18s sized to actual need — this is exactly what AWS VPC IPAM exists to manage, and at that point the discipline of central allocation beats the convenience of an oversized default.

**Decision 2: tier-major (group by tier, then split by AZ) rather than AZ-major.** *Alternative:* give each AZ a contiguous /19 and subdivide it into tiers inside — `10.0.0.0/19` is all of AZ-a, `10.0.32.0/19` is all of AZ-b. *Reason tier-major won here:* route tables, NACLs, and firewall rules are almost always written **per tier** ("the data tier accepts from the app tier"), and tier-major makes each tier a single aggregatable prefix — so on-prem needs one route for "all AWS databases" rather than three. *Trade-off, honestly:* AZ-major makes each AZ a single aggregate instead, which is better if your dominant concern is per-AZ operations — draining an AZ, per-AZ cost attribution, or a Transit Gateway design that routes per-AZ. *Flip condition:* if your firewall rules are AZ-scoped more often than tier-scoped, or you need to advertise "everything in AZ-a" as one prefix to a partner, invert it. Both are defensible; **what is not defensible is having no scheme at all and allocating the next free block each time**, which is what actually happens and why VPCs end up unsummarisable.

**Decision 3: /19 for app subnets, /24 for public and data.** *Alternative:* uniform /20s everywhere. *Reason:* the tiers have genuinely different address appetites — pod-per-address makes the app tier two orders of magnitude hungrier than a tier holding three load balancers. Uniform sizing either starves the app tier or wastes enormous space on the public tier. *Trade-off:* non-uniform sizes make the plan harder to hold in your head and easier to get wrong. *Flip condition:* if you are **not** running pod-per-VPC-address networking — plain EC2, ECS with `bridge` networking, or a CNI with an overlay such as Calico in VXLAN mode — the app tier's requirement collapses to instance count, and uniform /24s become the better, simpler choice. **This decision is downstream of your container networking model, and that is the thing to check first.**

**Decision 4: reserve half the /16, untouched.** *Alternative:* allocate the whole /16 now, neatly. *Reason:* the things you cannot predict — a third AZ, a new tier for a cache or a message broker, a secondary range for a service mesh, an acquisition to be peered — all need contiguous space, and contiguous space cannot be manufactured later. *Trade-off:* you are "wasting" 32,768 addresses. *Flip condition:* essentially none inside a single private VPC, since the addresses are free. It flips only under central IPAM at organisation scale (Decision 1's flip), where a /17 you never use is a /17 another team needed.

### Definition of done

Written down, on paper or in a file:

1. The table above, reproduced from scratch, **with usable counts computed by you** — and verified against the Python snippet. If your numbers differ from the script's, the script is right.
2. For each leaf subnet: network address, broadcast address, first usable, last usable, mask in dotted-decimal. (`10.0.16.0/24` → network `10.0.16.0`, AWS-reserved `.0–.3` and `.255`, first usable `10.0.16.4`, last usable `10.0.16.254`, mask `255.255.255.0`, 251 usable.)
3. A one-line answer to: *"can this VPC ever be peered with a VPC using `10.0.0.0/8`?"* (No. `10.0.0.0/16` is inside `10.0.0.0/8`, so the ranges overlap and VPC peering requires non-overlapping CIDRs. Verify with `.subnet_of()`.)
4. Which subnets get a route to an internet gateway, which get a route to a NAT gateway, and which get neither — plus **how many NAT gateways** and **why** (§9 design ④).
5. The growth story: where does the third AZ go, and where does a new "cache tier" go, without renumbering anything?
6. One sentence naming what you would have to change if the app tier moved from EKS with the VPC CNI to an overlay CNI.

---

## 14. In production

**Depth: covers the [CORE] concepts' production surface. Day 9 §13 covered L2/switching operations and the cloud-segmentation security posture; this is the L3 layer.**

### What good looks like

**Treat IP space as a managed, version-controlled resource with an owner.** The single highest-value practice in this entire note is having one authoritative registry of every CIDR your organisation has allocated — VPCs, on-prem, VPN pools, container networks, partner ranges — before anyone allocates anything. AWS VPC IPAM, Azure's equivalent, Netbox, or (genuinely fine for a small org) one reviewed YAML file in git. The failure it prevents is CIDR collision, which is the most expensive networking mistake available to you: it cannot be fixed by configuration, only by re-addressing, and it surfaces months later when someone tries to peer two VPCs or connect a new datacentre.

**Size subnets by the addressing model, not by instance count.** §3's pod-exhaustion trap. Ask "what consumes an address here?" before choosing a prefix.

**One NAT gateway per AZ, with allocated-and-protected Elastic IPs, and their addresses declared as a public interface.** §9 design ④.

**Never let the control plane depend on the data plane.** §11's Meta lesson. Out-of-band management access, DNS that does not depend on your DNS, credentials that do not depend on your auth service, and at least one person who has tested the serial-console path in the last year.

**Prefer IPv6 where your dependencies allow it, and audit IPv6 firewall rules separately.** §10.

**Put floors on anything automatic that removes capacity.** Health checks that withdraw routes, deregister targets, or scale in must never be able to remove everything. "Never fewer than N" and "never more than X% per window" are two lines of config that convert a Meta-shaped outage into a degradation.

### The mistakes, beginner → senior

**Beginner.** Confusing the subnet mask with the prefix length, or writing `/255.255.255.0`. Believing an IP identifies a machine (§2). Thinking `0.0.0.0` means "no address" rather than "all addresses." Not knowing that private addresses exist and being confused that `192.168.x.x` appears everywhere. Assuming `ping` failing means the host is down (ICMP is routinely blocked — §6). Treating a `169.254.x.x` address as a real address instead of as a DHCP failure (§3).

**Intermediate.** Sizing a cloud subnet from instance count and getting caught by pod-per-address (§3). Forgetting the 5 reserved addresses and creating a /29 with three usable slots. Believing "we're behind NAT so we're secure" (§9). Believing DNS TTL is honoured (§4 design ①). Reading traceroute's `* * *` as a broken path and per-hop latency deltas as link latency (§6). Blocking all ICMP and creating a PMTUD black hole — and in IPv6, breaking the protocol outright (§7). Rate-limiting per IPv6 address instead of per /64 (§10).

**Senior — the mistakes that cause the actual outages.** Correlated health checks with no floor (§11). No out-of-band access. Asymmetric routing from inconsistent prefix advertisement, then a day debugging a stateful firewall (§4 design ②). Not monitoring `ErrorPortAllocation`, and misdiagnosing NAT port exhaustion as a third-party provider problem for three days (§9). Not monitoring BGP table size as a capacity metric (§4's 512k day). Anycasting a long-lived stateful protocol (§12). Allowing overlapping CIDRs into the estate because there was no IPAM. Publishing egress IPs to partners without treating them as a change-controlled interface (§9 design ④).

### What to monitor, and the alerts almost nobody has

Routine: interface errors and discards; per-subnet address utilisation (**alert at 80% — this is the one that prevents the pod-exhaustion outage**); NAT gateway `BytesOutToDestination`, `ActiveConnectionCount`, `PacketsDropCount`; route-table entry counts against quota; VPC Flow Logs with `REJECT` records as a security and misconfiguration signal.

The four that are rare and high-value:

1. **`ErrorPortAllocation > 0` on every NAT gateway.** Converts a multi-day misdiagnosis into a five-minute fix (§9).
2. **ICMP Time Exceeded volume.** A near-zero-false-positive early warning for routing loops (§5 design ③).
3. **BGP prefix count and RPKI validity for your own prefixes.** If you own address space, monitor whether the internet sees the announcements you intend, from an external vantage point (RIPE RIS, BGPStream, or a commercial hijack-detection service). Meta's outage and every hijack in §11 were visible from outside long before they were understood from inside.
4. **Egress IP drift.** Assert that your observed public egress address matches the declared, allowlisted set. A synthetic check calling an "what is my IP" endpoint from inside each subnet, compared against a version-controlled list, catches an entire class of silent breakage (§9 design ④).

### Scaling behaviour and cost, in one place

**What scales fine:** the forwarding path (O(32) prefix lookup regardless of table size — §4); IP address space in IPv6; anycast (adding a site is a BGP announcement).

**What has a hard ceiling you must plan for:** NAT ports — 55,000 per address **per unique destination**, 8 addresses max (§9); NAT gateway PPS — 1 Mpps baseline, 10 Mpps scaled, then drops; router FIB size (§4's 512k day); subnet size, which is immutable after creation; VPC CIDR, which cannot shrink; route-table entries per table.

**The money, summarised:** public IPv4 addresses ~$0.005/hr each (~$3.65/month) on AWS since Feb 2024; NAT gateway ~$0.045/hr plus ~$0.045/GB processed, *plus* ~$0.09/GB data transfer out, *plus* ~$0.02/GB round trip for cross-AZ; VPC endpoints remove the NAT processing fee for AWS-service traffic entirely; IPv6 with an egress-only internet gateway has no per-GB or per-address charge at all. **Verify all of these at https://aws.amazon.com/vpc/pricing/ — pricing is the fastest-drifting content in this note.**

### Failure modes and recovery, as a table you can use at 3 a.m.

*(Recap of mechanisms already taught above — this table is for use, not for learning from.)*

| Symptom | Likely cause | Confirm it | Fix |
|---|---|---|---|
| Some destinations unreachable, others fine | Missing/wrong route; overlapping CIDR | `Find-NetRoute` / `ip route get`; check for overlaps | Fix the route; if CIDRs overlap, re-address |
| Traceroute shows repeating addresses, latency climbing | Routing loop | Repetition + linear latency growth (§5 ③) | Fix the disagreeing table; one source of truth per prefix |
| Small requests work, large ones hang | MTU / PMTUD black hole | `ping -f -l 1472` vs `-l 1473` (§7) | MSS clamping; permit ICMP type 3 code 4 |
| Intermittent timeouts to **one** external service at high concurrency | NAT port exhaustion | `ErrorPortAllocation` (§9) | Connection pooling first; then more NAT IPs |
| Idle connection dies, next query fails | NAT/LB mapping timeout (§9) | Works after reconnect; fails after idle period | TCP keepalive below the timeout; pool validation |
| Host has `169.254.x.x` | DHCP failed | Read the address (§3) | DHCP server, VLAN, or cable |
| Whole service vanishes globally, servers healthy | BGP withdrawal / hijack (§11) | External vantage point: RIPE RIS, BGPStream | Re-announce; contact upstream; out-of-band access |
| Works from internet and from localhost, not from the LAN | No NAT hairpinning (§9) | Test from a third host on the same LAN | Split-horizon DNS, or a hairpin-capable NAT |
| IPv4 fine, IPv6 broken or wide open | Separate, untested IPv6 rules (§10) | Test both stacks independently | Audit IPv6 rules as a separate exercise |
| Latency 3× expected, path looks absurd | BGP policy, not distance (§11) | traceroute + reverse-DNS geography | Different transit/peering, or a CDN/anycast front door |

---

## 15. Common misconceptions, corrected

Each of these is something a competent-sounding person will tell you. Each is wrong, and the section that fixes it is named.

- **"An IP address identifies a computer."** No — an interface's attachment to a network. One machine has many. §2.
- **"TTL is a time."** No — it is a hop count, and always has been in practice, despite RFC 791's name and units. §5.
- **"`* * *` in traceroute means the path is broken there."** No — it means no ICMP came back. Live paths routinely contain dead hops. §6.
- **"The difference between two traceroute hops is that link's latency."** No. Different return paths and control-plane delay make small deltas meaningless. §6.
- **"Traceroute shows the path my traffic takes."** It shows *a* forward path for *these probes*. The return path is invisible and often different; ECMP can interleave two paths into one nonsensical listing. §4, §6.
- **"NAT is a firewall."** No. It drops unsolicited inbound as a side effect of having no mapping. No rules, no logging, no egress control, and traversal techniques exist to defeat it. §9.
- **"We're out of IPv4 addresses so the internet will break."** The free *pools* ran out; nothing already allocated stopped working. The consequences were NAT, a resale market, and cloud charges. §8.
- **"IPv6 is IPv4 with more addresses."** It also deletes router fragmentation and the header checksum, replaces broadcast with multicast, replaces ARP with NDP, and makes PMTUD mandatory. §7, §10.
- **"A /24 has 256 usable addresses."** 254 classically; **251** on AWS and Azure; 252 on GCP. §3.
- **"BGP finds the shortest path."** BGP applies local policy first, and LOCAL_PREF outranks AS_PATH length. It is a business-policy protocol. §11.
- **"RPKI fixes BGP hijacking."** It validates the *origin* only, ~67% of prefixes are signed, and only ~12% of ASes fully enforce. Path validation (BGPsec) is essentially undeployed. §11.
- **"The Facebook outage happened because someone withdrew their BGP routes."** A command took down the *backbone*; the DNS servers then withdrew their own routes because a health check told them to. The distinction is the entire lesson. §11.
- **"Anycast is load balancing."** It is *proximity* routing with no fractional control, and it is hazardous for long-lived stateful connections. §12.
- **"Blocking ICMP makes me safer."** In IPv4 it breaks PMTUD and diagnostics; in IPv6 it breaks the protocol. §7, §10.
- **"Fragmentation works, it's in the RFC."** Permitted, yes; reliable on the public internet, no — as measured in §7's own capture. Design to never fragment.
- **"One IP, one user, so I can rate-limit per IP."** CGNAT puts thousands of unrelated subscribers behind one address; IPv6 gives one host 2^64 addresses. Rate-limit per /64 for IPv6 and expect collateral damage for IPv4. §9, §10.

---

## 16. Interview questions, with what a good answer contains

**"Walk me through what happens when you type an IP address into curl."** They want layered reasoning. Route lookup by longest prefix match → source-address selection → ARP for next hop (or destination if on-link) → framing → each router decrements TTL, recomputes checksum, re-frames → NAT rewrite at the boundary → return path possibly different. Naming *which* layer does *what* is the signal. (Day 12–14 extend this to DNS, TCP, and TLS; the full trace is Day 16's canonical exercise.)

**"You have `10.0.0.0/16`. Carve it for a 3-tier app across 3 AZs."** They want to hear you *ask about the addressing model* before you pick numbers — pods vs instances is the question that separates candidates. Then correct arithmetic, the 5 reserved addresses named unprompted, and growth reserved. §13.

**"Is `198.51.100.77` in `198.51.100.64/27`? Show your work."** Do the AND, or do the block-size shortcut, and state the range. Say 30 usable classically and 27 on AWS. §3.

**"What is TTL for, and what happens at zero?"** Loop prevention because hop-by-hop forwarding cannot detect loops. At zero the router discards and sends ICMP Time Exceeded type 11 code 0. Say "TTL=1 dies at *this* router" — the `<= 1` formulation. Bonus: it's a hop count despite the name, and IPv6 renamed it Hop Limit. §5.

**"How does traceroute work, and why do some hops show `* * *`?"** Increment TTL, harvest ICMP Time Exceeded. `* * *` = no ICMP returned: rate-limiting, policy, MPLS, or the reply filtered. Bonus: Windows uses ICMP Echo, Unix uses UDP 33434+, so a trace can succeed on one OS and fail on another. §6.

**"Why does my server see one IP for a thousand users?"** NAT/CGNAT, and therefore IP is a weak identity for rate limiting or blocking. §9.

**"What's the biggest number of concurrent connections one NAT address supports?"** The good answer starts by asking *to how many distinct destinations*, because the limit is per unique (destination IP, port, protocol) — 55,000 on AWS — not global. §9.

**"Explain the Facebook 2021 outage."** Backbone command → DNS servers' health checks withdrew their own BGP routes → prefixes ceased to exist → global SERVFAIL → internal tools depended on the same DNS → physical access required → ~6 hours. The insight to voice: **the safety mechanism caused it**. §11.

**"How would you fail over between regions?"** Name all three layers (client, DNS, routing), pick anycast for network-failure visibility, name the RFC 4786 stateful-connection limitation, and give the DNS-TTL-is-a-lie caveat. §4 ①, §12.

**"Why did HTTP/3 choose UDP?"** Partly head-of-line blocking (Day 11), and partly that NAT and middleboxes made deploying a *new* transport protocol impossible — UDP is the only other thing that traverses the installed base. §9.

**"Design egress for private subnets calling a partner API that requires an IP allowlist."** NAT gateway per AZ, Elastic IPs allocated and protected, the addresses treated as a version-controlled interface with drift alarms, VPC endpoints for AWS-service traffic, `ErrorPortAllocation` alarmed, and the cost arithmetic. §9 ④.

---
