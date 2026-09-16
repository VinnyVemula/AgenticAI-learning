# Day 14 — HTTPS/TLS and How HTTP Evolved (1.1 → 2 → 3)

> **Framing.** Days 9–12 got a request's bytes from your machine to the right address and the right name. Day 11 ended with a sharp, unresolved cost: a TCP handshake burns one full round trip before a single byte of your data moves, and it flagged that adding TLS would cost "2–3 RTTs" without saying why. Day 13 is teaching what an HTTP request and response actually contain. This day answers the question that sits between those two: **once the bytes can travel and you know what shape they're in, how do you stop anyone standing on the path from reading them, changing them, or pretending to be the other end?** And once that's answered, a second, related question falls out of it almost for free: given that every connection now pays a security tax, how has HTTP itself been redesigned, twice, specifically to get more use out of the connection you already paid for?
>
> This is foundational transport/security material with one genuine, narrow connection to agentic systems, and it's mechanical rather than thematic: an agent that calls a tool or an LLM API over HTTPS is trusting the same certificate chain a browser trusts, with none of a human's judgment about a browser warning — if that chain is misconfigured, an agent doesn't pause and squint at a padlock icon, it either fails closed (safe, annoying) or, if someone disabled certificate verification to "fix" a local error, it fails wide open to anyone who can position themselves on the path. We'll flag that once, precisely, where it belongs, and otherwise this note stays exactly what it is: the backend and security layer every HTTPS call — human or agentic — is built on.

---

## Roadmap

TLS is usually taught as an alphabet soup of acronyms and a diagram of a handshake, which produces readers who can recite "ClientHello, ServerHello, Finished" and cannot tell you why any single message in that exchange has to exist. We're going to build it the same way Day 12 built DNS: as a chain of forced design decisions, because every strange thing about TLS — and about why HTTP itself has been redesigned twice — is the consequence of one of them.

```
The problem: HTTP puts every byte on the wire in the clear.
                            │
              Anyone on the path can read it, change it,
                  or pretend to be one of the endpoints.
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
  EAVESDROPPING         TAMPERING          IMPERSONATION
   (read it)            (change it)        (pretend to be it)
        │                   │                   │
        └───────────────────┴───────────────────┘
                            │
         Need: confidentiality + integrity + authentication
                            │
           How do two strangers agree on a secret, over
             a channel the attacker can also read?
                            │
        ┌───────────────────┴───────────────────┐
        │                                       │
  SYMMETRIC CRYPTO                      ASYMMETRIC CRYPTO
  one shared key, very fast,            two related keys, slow,
  but: how do you SHARE the key?        but solves key distribution
        │                                       │
        └────────────────  HYBRID  ─────────────┘
              use the slow asymmetric machinery ONCE
              to agree a secret, then the fast symmetric
                    cipher for the whole session
                            │
          A public key alone proves nothing about WHO
             holds it — anyone can generate a keypair
                            │
                      CERTIFICATES
         a trusted third party signs "this key really
          belongs to this name"  →  A CHAIN OF TRUST
                            │
       Package negotiation + key exchange + identity check
             into ONE protocol, before any app data flows
                            │
                    THE TLS HANDSHAKE
                  (and it costs round trips)
                            │
        ┌───────────────────┴───────────────────┐
        │                                       │
   TLS 1.2 — 2 RTT                        TLS 1.3 — 1 RTT
   negotiate, THEN key-exchange           negotiate AND key-exchange
   (two separate round trips)             in the same flight
                            │
        Meanwhile HTTP itself hit a hard ceiling (Day 11,
        concept 10): ONE request in flight per TCP connection
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
     HTTP/1.1            HTTP/2              HTTP/3
   1 request per        many streams,       same streams,
   connection            1 connection,       QUIC transport,
   (Day 11 owns          HPACK header        TLS folded IN,
    this failure          compression         QPACK compression
    mode)                (Day 11's TCP-HOL   (Day 11 concept 13's
                          penalty returns —   payoff, completed
                          Day 11 concept 10)   here)
```

Concepts and tiers:

| # | Concept | Tier |
|---|---|---|
| 1 | The threat model: eavesdropping, tampering, impersonation | **[CORE]** |
| 2 | Symmetric vs. asymmetric crypto, and the hybrid key exchange | **[WORKING]** |
| 3 | Certificates and the chain of trust | **[CORE]** |
| 4 | The TLS handshake: messages, round trips, and what it costs | **[CORE]** |
| 5 | SNI: how one IP address serves a thousand certificates | **[WORKING]** |
| 6 | HTTP/1.1: persistent connections and the pipelining dead end | **[WORKING]** |
| 7 | HTTP/2: binary framing, multiplexed streams, and HPACK | **[CORE]** |
| 8 | HTTP/3: QPACK, stream mapping, and TLS folded into QUIC | **[CORE]** |

Tooling note, continuing Days 11–12's convention: you're on Windows, so every command below has a Windows-native path (PowerShell, or `openssl` via WSL2/git-bash/`winget install OpenSSL`). **Install OpenSSL** if you haven't — WSL2 (`sudo apt install openssl`) or git-bash both ship it, and it's the one tool whose raw output you'll see reproduced in essentially every TLS debugging guide you'll ever read, the same way `dig` was for DNS.

A cross-reference key before we start, so the pointers below read cleanly: **Day 11** already covered *why* HTTP/3 abandoned TCP for QUIC/UDP, including QUIC's four fixes to TCP's problems, the 1-RTT/0-RTT arithmetic for the *transport* handshake, and the case study of why Google built it — we will not re-derive any of that here, only cross-reference it by concept number and build the parts it didn't cover (header compression, frame mechanics, and exactly how TLS's own messages, which concept 4 below builds from scratch, ride inside QUIC). **Day 12** already covered the SVCB/HTTPS DNS record type that carries ALPN and Encrypted Client Hello configuration — we'll call back to it exactly once, in concept 5.

---

## 1. The threat model: eavesdropping, tampering, impersonation

**Depth: [CORE]**

### Intuition

Days 9–12 built a stack that reliably gets bytes from your process to the right machine on the internet, addressed by IP (Day 10), found by name via DNS (Day 12), and delivered in order by TCP (Day 11). Nowhere in any of that machinery is there a promise that the bytes are private, a promise that they arrive unmodified, or a promise that the machine on the other end is who you think it is. Plain HTTP inherits exactly the guarantees TCP gives it — ordered, complete delivery of *whatever bytes you handed it* — and not one guarantee more. If you write `GET /login HTTP/1.1` and a password in a header, TCP will faithfully deliver those literal bytes to whoever is listening on the far end of that 4-tuple, and it will do so whether or not that party is trustworthy, and whether or not anyone along the eleven-plus router hops in between (Day 10's traceroute) chose to make a copy.

That is not a bug in TCP. TCP was never asked to solve this problem — Day 11 built it to solve *reliable, ordered delivery over an unreliable network*, a completely different problem from *private, authenticated delivery over a hostile network*. The two problems look similar because both involve packets crossing wires, but the required properties are disjoint, and bolting privacy onto TCP itself (rather than as a layer TCP merely carries) is one of the real alternatives this section will come back to. For now, the point to sit with is simpler and more uncomfortable: **every plain-HTTP request you have ever issued was, at the network level, indistinguishable from a postcard** — anyone who can see the wire can read the whole thing, front and back, without your ever knowing they looked.

### Naming the threats precisely

"Someone could see it" is not a threat model; it's a feeling. A threat model names *what kind* of interference is possible, because the countermeasure for each kind is different, and a countermeasure for one does nothing for another. There are exactly three, and getting them straight now is what makes the rest of this note make sense rather than feel like a pile of acronyms.

**1. Eavesdropping (a passive attacker reads).** Someone with access to the wire — a shared Wi-Fi network, a compromised router, an ISP, a network tap — copies your traffic without altering it. You never know it happened. The attacker doesn't need to be clever; they need only be present. Anyone on the same coffee-shop access point as you, running a packet capture tool, is an eavesdropper by this definition, and no cleverness on your part after the fact undoes what they already read.

**2. Tampering (an active attacker modifies).** Someone on the path doesn't just read your traffic, they change it before it arrives — rewriting a response body, injecting a script tag, redirecting a link, or altering a request. This requires the attacker to be *in* the path (or to insert themselves into it), not merely watching a copy, but it's a well-worn capability: a malicious or compromised router, a coercive network operator, or a classic man-in-the-middle (MITM) position are all sufficient.

**3. Impersonation (an active attacker pretends to be an endpoint).** Someone convinces you that they are your bank, or convinces your bank that they are you. This is the subtlest of the three because it doesn't require intercepting *existing* traffic at all — it requires only that you initiate a connection to the wrong party in the first place, something Day 12's DNS-cache-poisoning and DNS-rebinding discussion already showed you is entirely achievable by attacking the *name resolution* step rather than the connection itself. Impersonation is also the one encryption alone cannot fix, and that fact is the entire reason concept 3 (certificates) exists — hold that thought.

### Worked example — what a plaintext credential actually looks like on the wire

Abstractions make people complacent, so make this concrete once and it will stay with you. Suppose a browser submits an HTTP Basic-Auth-protected form over plain HTTP. The request header looks like this:

```
GET /admin HTTP/1.1
Host: intranet.example.com
Authorization: Basic dXNlcjpwYXNzd29yZDEyMw==
```

A beginner's first reaction is often "well, at least it's encoded, not sent as `user:password123` outright." That reaction is exactly the trap, and worth defusing precisely: **Base64 is an *encoding*, not an *encryption*.** It exists to let arbitrary binary data travel safely through text-oriented protocols (headers, JSON, email); it carries zero secrecy. Decoding it is not an "attack" in any meaningful sense — it's one command:

```bash
$ echo "dXNlcjpwYXNzd29yZDEyMw==" | base64 -d
user:password123
```

```powershell
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String("dXNlcjpwYXNzd29yZDEyMw=="))
# -> user:password123
```

Anyone who captured that header off a shared network — with a free tool, no cryptography, no brute force, just `tcpdump`/Wireshark and one shell command — has your credentials in plaintext, instantly. This is precisely the eavesdropping threat, made concrete: HTTP Basic Auth over plain HTTP is not "weakly protected," it is **completely unprotected**, dressed in a costume that merely looks like protection to someone who hasn't checked. TLS is the layer that turns that header into ciphertext an eavesdropper cannot reverse even with the entire capture and unlimited time (concept 4 explains exactly why the "unlimited time" part is true).

### Analogy: a postcard, a sealed envelope, and a wax seal — and where each one breaks

Sending plain HTTP is sending a postcard: anyone who handles it between you and the recipient — every postal worker, every sorting facility — can read every word, and if one of them is malicious, they can cross out a line and write a new one before it arrives, and you'd never know. That's eavesdropping and tampering, both, in one object.

Encryption is sealing the message inside an opaque envelope: the people handling it in transit can see the "to" and "from" (a passive observer can still see *that* you're talking to `bank.com`, even under TLS — Day 12's DNS lookup for the name, and the IP address in every packet's header, Day 10, are never hidden by TLS; TLS hides the *content*, not the *existence*, of the conversation), but not the contents. Sealing solves eavesdropping. It does not, on its own, prove who mailed the envelope.

A wax seal stamped with a signet ring is the attempt to solve the remaining problem: identity. If everyone recognizes the Doge of Venice's seal, a letter arriving with that exact seal, unbroken, is taken as authentically his. Two ways this breaks, and both are load-bearing for what follows:

- **Anyone can buy a signet ring and press it into wax.** A seal by itself proves nothing about authenticity unless there is some way to distinguish the real ring from a forgery — which in the physical world usually means trusting a person who has *seen* the real ring, i.e., a third party's vouching. That's exactly the shape of a certificate authority (concept 3): not magic, just a trusted third party whose job is to check identity before it lends its seal.
- **The wax seal only proves who sealed the envelope, once, at the moment of sealing — not that the letter wasn't opened, read, and re-sealed with a forged copy of the ring somewhere along a very long postal route.** A cryptographic signature doesn't have this weakness in the same way a physical seal does: forging a digital signature without the private key is not merely *inconvenient*, as forging a wax seal is — with a well-chosen key size, it is computationally infeasible with all of humanity's current computing power, for as long as the math underlying it holds up (concept 2 makes this precise). That distinction — "hard to forge because manufacturing a fake is expensive" vs. "hard to forge because it would require solving a problem believed to take longer than the age of the universe" — is exactly where the analogy is *not* just weaker but qualitatively different, and it's worth remembering the next time someone says "encryption is basically an envelope," because it undersells how strong the real guarantee is.

### Judgment: "just redirect HTTP to HTTPS" vs. HSTS — a decision most sites get wrong on the first try

Here is a genuine, common decision, worked through in miniature before the dedicated System Design block below gives it the full treatment: a site wants every visitor on HTTPS. The obvious move is a server-side 301 redirect from `http://example.com` to `https://example.com`. It looks sufficient. It is not, and the reason is a real, named, historically significant attack.

**The realistic alternative that looks fine and isn't:** rely on the redirect alone.

**The actual reason it fails:** the *first* request a browser sends for a domain it hasn't visited is plain HTTP, by default, because the browser doesn't yet know the site wants HTTPS-only — it's simply obeying whatever the user typed or clicked. That very first request is unencrypted, in the clear, on the wire, before the redirect ever has a chance to happen. An attacker positioned as a MITM (a rogue Wi-Fi access point is the classic staging ground) can intercept that first plaintext request, silently proxy it to the real HTTPS site behind the scenes, and return the plaintext response to the victim's browser with every `https://` link inside it rewritten to `http://` — so the browser never sees a reason to upgrade, and the entire session proceeds in the clear while the attacker relays it to the real site and reads (or modifies) everything. This is a real, named tool: **`sslstrip`**, demonstrated by Moxie Marlinspike at Black Hat DC 2009 in the talk *"New Tricks for Defeating SSL in Practice,"* and it worked precisely because a same-origin redirect is a promise the *server* makes *after* the vulnerable plaintext round trip has already happened.

**The trade-off, honestly:** the fix — **HSTS**, HTTP Strict Transport Security (RFC 6797) — is a response header, `Strict-Transport-Security: max-age=31536000; includeSubDomains; preload`, that tells the browser "for the next year, never issue a plain HTTP request to this domain again, even if the user types `http://` or clicks a plain link — silently rewrite it to HTTPS before any packet leaves the machine." That closes the sslstrip window completely, *after the first visit*. The trade-off is that it is a commitment with real teeth: once a browser has cached that header, there is no user-facing way to visit the plain-HTTP version of your site again until the `max-age` expires, so a broken or expired TLS certificate on that domain now locks out returning visitors entirely rather than merely serving an insecure connection — you have traded "vulnerable on first visit" for "an outage if you mess up TLS," which is a real cost, not a free lunch.

**The flip condition:** the residual gap — that first, unprotected visit — is closed by **HSTS preloading**: shipping your domain's HSTS policy baked directly into the browser's own source code (via `hstspreload.org`, which Chrome, Firefox, Safari, and Edge all consume), so there is never a first vulnerable request for *any* user of a modern browser, ever. The honest cost of preloading is that it is close to irreversible on any timescale you can operate on: removal requires the browser vendors to ship a new release with your domain removed from a hardcoded list, which has historically taken months. **The condition under which you should *not* preload:** you're not fully confident every subdomain, forever, will always be served over valid TLS (`includeSubDomains` applies preload-level enforcement to every subdomain you might ever create, including ones you haven't thought of yet) — plenty of real organizations have been burned by preloading before an internal tool or a forgotten subdomain was TLS-ready. Get the plain `max-age` header solidly working and monitored first; add `preload` once you're certain.

### System design — deciding how a site enforces HTTPS-only traffic

**Problem:** a consumer-facing site currently serves both `http://` and `https://`, with a same-origin redirect from the former to the latter. Design the actual enforcement policy.

**Requirements:** no user, ever, sends credentials or session cookies over plaintext, including on their very first visit from a link, a bookmark, or a typed URL; the policy must not create a permanent outage risk that outweighs its benefit; legacy embedded devices or old clients that genuinely cannot negotiate modern TLS must have a documented, bounded exception rather than silently breaking.

**Alternatives:**
1. **Redirect only** (the status quo above) — cheap, familiar, and leaves the sslstrip-shaped gap on every first visit.
2. **HSTS with a modest `max-age`, no preload** — closes the gap for every *returning* visitor after their first HTTPS visit, at the cost of leaving first-time visitors exposed and creating a bounded lock-out risk (bounded by `max-age`) if certificates break.
3. **HSTS with `includeSubDomains` and browser preload** — closes the gap for every visitor, including the first one, at the cost of a lock-out risk that is effectively permanent on operational timescales, extended to every subdomain that will ever exist.

**The decision:** (3) for anything handling authentication or payment data, phased in through (2) first. Start with a short `max-age` (a day) while confirming every subdomain genuinely serves valid, monitored TLS; extend `max-age` over weeks; only then submit to preload.

**The actual reason:** the cost asymmetry is severe and one-directional. A credential leaked via sslstrip on someone's first visit is an unrecoverable compromise — you cannot un-leak a password. A certificate misconfiguration causing a temporary HSTS lock-out is recoverable: you fix the certificate (concept 3's whole subject) and the lock-out ends the moment `max-age` next elapses for each affected client, and in the interim only *some* returning visitors on a *narrow* window are affected, not first-time visitors, since HSTS by definition only constrains clients that have already cached the policy from a prior successful visit.

**The trade-off, stated honestly:** you're accepting operational rigidity — TLS on this domain (and every subdomain, once `includeSubDomains` is set) is no longer optional infrastructure you can quietly fall back away from during an incident; it is now a hard dependency your incident responders must never "temporarily disable" as a mitigation, because doing so does nothing for clients that have already cached the policy and simply produces confusing, unrecoverable-looking errors for them.

**Flip condition:** the case for staying at (1) or (2) is genuine, not merely lazy, when your clients are things you don't control and that may be years out of date — set-top boxes, point-of-sale terminals, industrial IoT devices — where an HSTS-driven hard failure on a machine nobody can push an update to is worse than the residual sslstrip risk, and the honest mitigation is network-level segmentation of those clients rather than a browser-security header they may not even honor.

**Failure modes:** preloading before every subdomain is verified TLS-ready (a forgotten `beta.example.com` on plain HTTP becomes permanently unreachable for anyone with the preload list until you fix it and wait for the *next* browser release cycle); letting a certificate expire on an HSTS-preloaded domain (every returning visitor sees a hard, unbypassable browser error, not a "click through" warning — concept 3's Microsoft Teams case study is exactly this failure mode, one layer up); and the subtler mistake of enabling HSTS on a domain that still has active plain-HTTP-only internal tooling depending on it, which breaks silently and is often diagnosed hours later as "the network is broken" rather than "we shipped a security header."

### Case studies

**Firesheep (October 2010) — passive eavesdropping made trivial for anyone.** Security researcher Eric Butler released **Firesheep**, a Firefox extension demonstrated at the ToorCon security conference, that turned session-hijacking over shared Wi-Fi into a one-click affair: it sniffed unencrypted HTTP traffic on the local network, identified session cookies for popular sites (Facebook, Twitter, and others that served their *login page* over HTTPS but reverted to plain HTTP immediately afterward — the login credentials were protected, but the session cookie proving you were already logged in was not), and let anyone on the same coffee-shop Wi-Fi network click a name in a sidebar to instantly become that person on that site, no password required. The engineering lesson is precise and still routinely gotten wrong today: **encrypting only the login form and leaving the rest of the session in plaintext protects the password and gives up everything the password was protecting.** A session cookie is exactly as sensitive as the credential it stands in for, and eavesdropping doesn't care which HTTP request it was attached to. Firesheep is widely credited as a major forcing function behind the industry's subsequent multi-year move toward serving entire sessions over HTTPS by default rather than only the login page — Google had already defaulted Gmail to HTTPS in January 2010; Facebook and Twitter followed with HTTPS-by-default options in the roughly two years after Firesheep's release. *Primary source:* Eric Butler's own release announcement and technical write-up, October 2010 (`codebutler.com`); contemporaneous security-press coverage (e.g., Wired, Ars Technica) for the industry-response narrative. **Verify current** the exact rollout dates for individual companies' HTTPS-by-default policies if you cite them precisely — this is more than a decade of drift on secondary sources.

**The "Great Cannon" (2015) — active tampering with unencrypted traffic, at national-infrastructure scale.** In April 2015, researchers at Citizen Lab (University of Toronto) published a detailed technical analysis of a network-injection system, which they named the **Great Cannon**, operating alongside China's national internet infrastructure. The Great Cannon was documented actively **modifying unencrypted HTTP responses in transit** — specifically, injecting malicious JavaScript into plaintext traffic destined for computers making requests to Baidu's ad-network and analytics scripts (themselves served over plain HTTP) — turning ordinary visitors' browsers, unknowingly, into a botnet that flooded GitHub with traffic in a large-scale denial-of-service attack targeting pages that hosted censorship-circumvention tools. Nobody whose traffic was hijacked did anything wrong; their browsers requested an unencrypted script, and *any* party positioned on the path between them and the server was, structurally, able to substitute a different one. The engineering lesson generalizes past this one incident: **content served over plain HTTP is modifiable by anyone positioned on any link along its path, and "it's just a small script/ad/analytics tag, who'd bother" is precisely the reasoning that makes those small unencrypted assets the most attractive injection point** — attackers go after the weakest link in a page's supply chain, not its most important one. *Primary source:* Citizen Lab, *"China's Great Cannon,"* 10 April 2015 (`citizenlab.ca`), the primary forensic write-up. **Verify current** if citing attribution or ongoing-capability details — attribution in network-injection incidents is inherently harder to independently reverify years later than a vendor's own postmortem.

### In production — HTTPS-only enforcement, operationally

**Best practices:** default every new project to HTTPS-only from day one — retrofitting HSTS onto a mature site with unknown subdomains is strictly harder than starting there; verify `Strict-Transport-Security` is actually present with `curl -I`, not just "it's in the framework config" (frameworks silently fail to apply security headers behind reverse proxies more often than teams expect); treat your HSTS `max-age` and preload status as production configuration under change control, not a one-time setup step; and mix in **CSP's `upgrade-insecure-requests`** or simply audit for stray `http://` asset URLs in your own HTML/CSS/JS, since a page served over HTTPS that pulls even one image over plain HTTP creates "mixed content," which modern browsers block or warn on but which is still, at the protocol level, an eavesdropping/tampering surface you introduced yourself.

**Mistakes, beginner → senior:** *beginner* — assuming a login-page-only HTTPS deployment (the exact Firesheep-era mistake) protects a site; *intermediate* — shipping HSTS with `includeSubDomains` before auditing every actual subdomain, then discovering a forgotten staging host is now permanently unreachable for anyone who already loaded the policy; *senior* — treating "we're behind a corporate VPN, so internal tools don't need TLS" as sufficient (see the second system-design scenario's logic: a private network is a network segment, not a cryptographic guarantee, and it is trivially defeated by a single compromised device already inside it).

**Monitoring:** synthetic checks from outside your network confirming the redirect *and* the HSTS header are both present on every hostname you serve, not just the primary one; certificate-expiry alerting (concept 3 owns this in depth, because it is the single most common way HSTS enforcement turns into a self-inflicted outage); and periodic external scans (`securityheaders.com`, `ssllabs.com`) as a cheap, standard second opinion on your own configuration.

**Cost:** enforcing HTTPS-only costs essentially nothing in infrastructure once certificates are automated (concept 3) — the real cost is entirely organizational: an inventory discipline (know every subdomain that exists) that most organizations don't have until an HSTS rollout forces them to build one.

---

## 2. Symmetric vs. asymmetric cryptography, and the hybrid key exchange

**Depth: [WORKING]**

### Intuition

Concept 1 established what's needed: confidentiality (nobody but the two endpoints can read the traffic) and integrity (nobody can silently change it). The mechanism for both is encryption with a shared secret — scramble the message with a key only the two of you hold, and anyone without that key sees noise, while any tampering with the ciphertext is detectable when decryption fails to produce something sensible. That much is old cryptography, older than computers.

The genuinely hard problem is the one that comes *before* encryption can even start: **your browser has never spoken to this server before. Neither of you has a shared secret. And the only channel you have to agree on one is the exact channel Concept 1 says an attacker might be reading.** Whisper the secret over that channel in the clear, and the eavesdropper now has it too, and every subsequent "encrypted" message is exactly as private as if you'd never encrypted at all. This is not a minor wrinkle — it's the whole problem, and until it's solved, nothing else in this note is possible. Solving it is what the rest of this section is about, and the "just enough" instruction that shaped this section is deliberate: TLS's own security researchers spend careers on the number theory underneath this; you need the shape of the idea and the two primitives it's built from, not a derivation of why the underlying math is hard to invert.

### Two kinds of cryptography, and why neither one alone is sufficient

**Symmetric cryptography** uses one key for both directions: the same key that scrambles the message also unscrambles it. Concretely, TLS uses an **AEAD cipher** — Authenticated Encryption with Associated Data — such as AES-128-GCM, AES-256-GCM, or ChaCha20-Poly1305, which does two jobs in one pass over the data: it encrypts the bytes, *and* it produces a short authentication tag that lets the receiver detect if even one bit was altered in transit. This is blazingly fast — modern CPUs have dedicated instructions for AES (AES-NI, present in essentially every server and laptop CPU shipped in the last decade) that push multiple gigabytes per second through the cipher — and it's what actually protects every byte of your HTTP request and response once a TLS connection is established. But it has exactly one precondition: **both sides must already possess the identical key.** Symmetric crypto is a complete non-starter for two strangers meeting for the first time over a hostile network, because it assumes the hard problem — secure key distribution — is already solved.

**Asymmetric cryptography** (also called public-key cryptography) breaks that assumption. Instead of one key, you generate a mathematically related *pair*: a **public key**, which you can hand to absolutely anyone, including an adversary, with no loss of security, and a **private key**, which you never share with anyone. The two are related by a one-way mathematical relationship — computing the public key from the private key is easy; computing the private key from the public key is, for well-chosen parameters, believed to require more computation than is feasible with all of humanity's combined hardware for the foreseeable future. The classical scheme (RSA) bases this on the difficulty of factoring the product of two very large prime numbers; the schemes TLS 1.3 actually prefers today (elliptic-curve variants, discussed under "under the hood" below) base it on a related but distinct hard problem, the elliptic-curve discrete logarithm. **Either way, the payoff is the same:** you can publish your public key on a highway billboard, and an adversary who reads it gains nothing that helps them decrypt a message encrypted to it, or forge a signature that verifies against it.

This looks like it should solve everything — publish a public key, let anyone send you a secret, done. The catch is cost: asymmetric operations are dramatically more expensive per byte than symmetric ones, on the order of two to three orders of magnitude slower depending on the operation and key size (the exact multiplier is hardware- and library-dependent — **verify with `openssl speed rsa2048 aes-256-gcm` on your own machine before quoting a number for capacity planning**, but "roughly a thousand times slower" is the right order-of-magnitude intuition to carry). Nobody encrypts a multi-megabyte HTTP response directly with RSA; it would be needlessly slow and, for several of the classic constructions, isn't even semantically appropriate for streaming data of arbitrary length in the first place.

### The hybrid trick: use the slow one once, then the fast one for everything else

The resolution is the single most important idea in this section, and once you see it, half of what TLS does stops looking arbitrary: **use asymmetric cryptography exactly once per connection, for the narrow job of agreeing on a shared secret — then throw the asymmetric machinery away and use that secret to key a fast symmetric cipher for the entire rest of the session.** This is sometimes called "hybrid encryption," and it is not a TLS-specific trick — it's the standard pattern anywhere a fast bulk cipher needs a securely-established key (SSH does the same thing; so does age/GPG file encryption).

The specific asymmetric operation TLS uses for this isn't "encrypt the secret with the public key and send it" (that's an older, now-discouraged approach called RSA key transport, still found in legacy TLS 1.2 configurations) — modern TLS uses a **key exchange**, most commonly (Elliptic-Curve) Diffie-Hellman, which is more elegant: **neither side ever transmits the shared secret, not even in encrypted form.** Instead, each side generates a fresh, temporary ("ephemeral") key pair, they exchange only the public halves in the clear, and each side combines the other's public value with their own private value using a mathematical operation that is commutative in exactly the right way — so both sides land on the *same* number, despite an eavesdropper who saw every bit exchanged being unable to derive it.

### Worked example — Diffie-Hellman with numbers small enough to compute by hand

Real TLS uses numbers hundreds of bits long, which makes the mechanism feel like magic. Small numbers make it feel like arithmetic, which is exactly the point — this is the textbook toy example, done in full, so you can verify every step yourself with a calculator.

**Public parameters, known to everyone including the attacker:** a prime `p = 23` and a generator `g = 5`.

```
Alice picks a SECRET number:  a = 6   (never transmitted)
Bob   picks a SECRET number:  b = 15  (never transmitted)

Alice computes her PUBLIC value:  A = g^a mod p = 5^6  mod 23 = 8
Bob   computes his PUBLIC value:  B = g^b mod p = 5^15 mod 23 = 19

Alice sends A=8 to Bob, in the clear.  Bob sends B=19 to Alice, in the clear.
An eavesdropper now knows: p=23, g=5, A=8, B=19 — and nothing else.
```

Now each side combines the *other* side's public value with their *own* secret number:

```
Alice computes:  B^a mod p  =  19^6  mod 23  =  2
Bob   computes:  A^b mod p  =  8^15  mod 23  =  2
```

**Both land on 2.** That's the shared secret, and it was never transmitted in any form — only `A` and `B`, the public values, crossed the wire. Verify it yourself: `5^6 mod 23` really is 8 (5→25→2 mod 23, so 5²≡2, 5⁴≡4, 5⁶=5⁴·5²≡4·2=8), and the fact that `(g^a)^b ≡ (g^b)^a ≡ g^(ab) (mod p)` — exponents multiply regardless of which order you raise them in — is the entire mechanism. An eavesdropper who captured `p, g, A, B` needs to recover `a` (or `b`) from `A = g^a mod p` to compute the same secret, which is the **discrete logarithm problem** — trivial for `p=23` (you could brute-force all 23 possibilities in your head), and believed computationally infeasible for the 256-bit-plus curve groups TLS 1.3 actually negotiates.

**Honesty about scope, stated plainly:** this toy example is for building intuition about *why two strangers can agree on a secret over a public channel*, not a specification of anything TLS 1.3 actually runs. Real TLS 1.3 key exchange uses elliptic-curve groups (most commonly **X25519**, RFC 7748) rather than plain modular exponentiation over a prime, because equivalent security is achieved with far smaller keys and faster computation. **Deliberate stop:** deriving elliptic-curve point arithmetic, or proving why the discrete-log problem is hard, is a black box this note leaves closed — that's a semester of a cryptography course, not a paragraph, and per the study plan's own instruction, "just enough" means the mental model (ephemeral key exchange → shared secret, never transmitted) is the load-bearing part, not the group theory underneath it.

### Analogy: the padlock anyone can close but only one person can open

Bob wants to receive a private message from a stranger, Alice, who he's never met. Bob manufactures an open padlock and its one matching key, keeps the key, and leaves copies of the *open padlock* — not the key — at every post office in the country, for anyone to take. Alice takes a copy of the padlock, puts her message in a box, snaps the padlock shut, and mails it. Anyone who intercepts the box in transit can see the box and the closed padlock, but cannot open it — they never had the key, and having a copy of the open padlock (the public key) doesn't help them, because the padlock and its key are different objects. Only Bob, who kept the one matching key, can open it.

**Where the analogy breaks, and it's a genuinely important break:** a physical padlock and its key are two unrelated objects that happen to fit together because a locksmith manufactured them as a matched pair — you could, with enough machining skill, forge a key that opens a specific padlock, and doing so is a *manufacturing* problem, bounded by tooling and time. In RSA or elliptic-curve cryptography, the "padlock" (public key) and "key" (private key) are not independently manufactured objects — they are two numbers **mathematically derived from the same underlying secret**, related by a one-way function. Deriving one from the other isn't hard because nobody's built the right lockpicks yet; it's hard because it requires solving a specific, well-studied mathematical problem (factoring a 2048-bit number, or the elliptic-curve discrete log) that is currently believed, by the entire cryptographic research community, to require more computation than is physically achievable with any hardware humanity can build in the relevant timeframe. That's a categorically stronger and more precisely quantifiable guarantee than "we don't have the tools to pick this lock yet." (It's also not a *permanent* guarantee — this is exactly the assumption a sufficiently large quantum computer would break for the algorithms in wide use today, which is why post-quantum key-exchange algorithms are already being standardized and, as of recent TLS 1.3 deployments from major providers, starting to appear in production ClientHellos alongside X25519 as a hybrid. **Verify current** deployment status — this is one of the fastest-moving areas in the whole protocol.)

### Judgment: hybrid key exchange vs. direct asymmetric encryption of the payload

**The realistic alternative:** why not skip the whole "agree on a secret, then switch ciphers" dance, and just encrypt each HTTP message directly with the recipient's public key?

**The actual reason TLS doesn't do this:** throughput, at two levels. First, the raw speed gap above means directly asymmetric-encrypting a multi-kilobyte response would be substantially slower than symmetric encryption of the same data, on every single request. Second, and less obvious: asymmetric schemes like RSA have a maximum plaintext size tied to the key's modulus size (you cannot directly RSA-encrypt an arbitrary-length HTTP response any more than you can fit an arbitrarily long letter through a fixed-size envelope slot without folding it first) — bulk data needs a construction designed for streams of arbitrary length, which is exactly what a symmetric AEAD cipher, fed a continuous sequence of TLS records, is built for.

**The trade-off, honestly:** the hybrid approach is *more complex* — it requires an interactive handshake protocol (concept 4) to bootstrap the shared secret before any application data can flow, rather than the conceptual simplicity of "just encrypt with their public key and send it, no back-and-forth needed."

**The flip condition — and it's a real one, not hypothetical:** systems that are fundamentally **non-interactive** (no live back-and-forth session is possible) genuinely do use direct asymmetric encryption of a per-message key. Email encryption (PGP/GPG, S/MIME) is the clean example: when you encrypt an email, there's no live handshake to run — the recipient may not read it for days — so the sender generates a fresh symmetric key, encrypts the message body with it (fast, handles arbitrary length), then encrypts *just that short symmetric key* with the recipient's long-lived public key and attaches it. That's the same hybrid idea, but collapsed into a single one-shot message because there's no session to negotiate. **The condition that flips the decision is exactly "is there an interactive channel to run a handshake over, or does the message have to be fully self-contained the moment it's sent."** TLS has the former; store-and-forward email has the latter, and lands on a related but structurally different design as a direct consequence.

### Case study — the Logjam attack (2015): what happens when key-exchange negotiation is allowed to weaken itself

**What happened.** In May 2015, a team of researchers (Adrian et al., published at ACM CCS 2015 as *"Imperfect Forward Secrecy: How Diffie-Hellman Fails in Practice"*) disclosed **Logjam** (CVE-2015-4000). Decades earlier, U.S. export regulations had required software shipped internationally to support deliberately weak "export-grade" cryptography, including 512-bit Diffie-Hellman groups — small enough that, even in the 1990s, they were within reach of well-resourced attackers, and by 2015, trivially breakable with modern hardware. Those export-grade cipher suites were still supported, for legacy compatibility, in a large fraction of TLS servers on the internet in 2015, purely as a fallback option a server would only use if a client asked for it.

**The mechanism, and it should feel familiar.** An active man-in-the-middle attacker could intercept a normal TLS handshake, and — because many servers would still *accept* a request to downgrade to the weak export-grade Diffie-Hellman group even when the connecting client was a modern browser that never intended to ask for one — rewrite the negotiation messages in transit to force the connection down to the 512-bit group. Once downgraded, the researchers demonstrated they could solve the corresponding discrete-log problem (precomputing much of the work once per commonly-shared prime, since a huge fraction of servers reused a small number of standard primes) fast enough to break individual connections' key exchange in near-real-time, recovering the "shared secret" this whole section just explained the strength of, entirely because the *group size* backing it had been silently weakened by an attacker sitting on the path — exactly the tampering threat from concept 1, applied specifically to the parameters of the crypto itself rather than to application data.

**The engineering lesson, and it recurs across this entire note in different costumes:** a protocol that allows negotiating *down* to a weaker option, without cryptographically protecting the negotiation itself, is only as strong as the weakest option it's still willing to accept — no matter how strong its best option is. This is the exact same root cause as `sslstrip` (concept 1) forcing a downgrade from HTTPS to plain HTTP, and it recurs again in concept 4's discussion of protocol-version downgrade attacks. TLS 1.3's response (RFC 8446) was structural rather than a patch: it removed support for the weak legacy Diffie-Hellman groups and RSA key transport entirely, restricted the negotiable groups to a short list of modern, adequately-sized ones, and added a cryptographic signal (the downgrade-protection value embedded in `ServerHello.random`, RFC 8446 §4.1.3) that lets a client detect if an attacker tried to trick it into thinking only an older, weaker protocol version was available. *Primary sources:* Adrian et al., CCS 2015 (`weakdh.org` hosts the paper and an interactive tool to check your own server); CVE-2015-4000; RFC 8446 §4.1.3 and Appendix D for the downgrade-protection design.

---

## 3. Certificates and the chain of trust

**Depth: [CORE]**

### Intuition

Concept 2 solved a hard problem and left an easier-sounding one completely untouched. Diffie-Hellman lets two parties agree on a secret that an eavesdropper on the path cannot derive — but it says absolutely nothing about *who* is on the other end of that exchange. Generate a key pair, publish the public half, and you can run the exact same key-exchange math with anyone, including a browser that thinks it's talking to `yourbank.com`. **Nothing about asymmetric cryptography prevents an attacker from generating a keypair, presenting it as `yourbank.com`'s, and running a flawless, cryptographically unbreakable key exchange with your browser — flawless and unbreakable, and with entirely the wrong party.** This is impersonation, precisely as named in concept 1, and it is untouched by everything concept 2 built. Confidentiality against an eavesdropper is worthless if you've confidentially handed your password to an impersonator.

What's needed is a way to bind a public key to an identity — some assurance, checkable by a stranger's browser with no prior relationship to the site, that *this specific public key* really does belong to *this specific name*. That assurance can't come from the key itself (a key is just a number; numbers don't carry names). It has to come from somewhere else: a third party that both your browser and the website's operator already, independently, trust.

### What a certificate actually is

A **certificate** (specifically, an **X.509 certificate**, the format essentially the entire web uses) is a data structure that says, in effect: *"I, the issuer, having checked, assert that this public key belongs to this name, and I'm putting my own cryptographic signature on this assertion so anyone can verify I really said it and it hasn't been altered."* Concretely, it bundles:

- the **subject** — who or what this certificate is for, most importantly the domain name(s) it's valid for (in the **Subject Alternative Name**, or SAN, extension — the older `CN`/Common Name field for this purpose is deprecated and modern browsers ignore it in favor of SAN);
- the subject's **public key** (the other half of a keypair whose private half only the legitimate server holds);
- a **validity period** (Not Before / Not After timestamps — concept 3's whole reason for existing operationally, as the Microsoft Teams case study below demonstrates);
- the **issuer**'s identity; and
- the issuer's **digital signature over all of the above** — computed with the issuer's own private key, and verifiable by anyone holding the issuer's public key.

That last point is the entire mechanism in one sentence: **a certificate is a signed statement, and verifying it is exactly the signature-verification half of the asymmetric cryptography from concept 2** — no new cryptographic primitive is needed, just a new *use* of the one you already have. Let's see one for real. Here is the certificate this environment's outbound network actually presented when I ran `openssl s_client -connect example.com:443 -servername example.com` while writing this note — and what it turned out to be is itself an important lesson, covered right after:

```
$ echo | openssl s_client -connect example.com:443 -servername example.com

Certificate chain
 0 s:CN=example.com, O=Zscaler Inc., OU=Zscaler Inc.
   i:C=US, ST=California, O=Zscaler Inc., OU=Zscaler Inc., CN=Zscaler Intermediate Root CA (zscloud.net) (t)
   a:PKEY: RSA, 2048 (bit); sigalg: sha256WithRSAEncryption
   v:NotBefore: Aug 24 04:00:56 2026 GMT; NotAfter: Sep  6 03:53:42 2026 GMT
 1 s:C=US, ST=California, O=Zscaler Inc., OU=Zscaler Inc., CN=Zscaler Intermediate Root CA (zscloud.net) (t)
   i:C=US, ST=California, O=Zscaler Inc., OU=Zscaler Inc., CN=Zscaler Intermediate Root CA (zscloud.net), emailAddress=support@zscaler.com
   a:PKEY: RSA, 2048 (bit); sigalg: sha256WithRSAEncryption
 2 s:C=US, ST=California, ..., CN=Zscaler Intermediate Root CA (zscloud.net), emailAddress=support@zscaler.com
   i:C=US, ST=California, L=San Jose, O=Zscaler Inc., ..., CN=Zscaler Root CA, emailAddress=support@zscaler.com
   ...
---
Peer signing digest: SHA256
Peer signature type: rsa_pss_rsae_sha256
Peer Temp Key: ECDH, prime256v1, 256 bits
---
Verification error: unable to get local issuer certificate
...
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
Verify return code: 20 (unable to get local issuer certificate)
```

**Read that output slowly, because it is not what I expected to capture, and it teaches the chapter's central lesson better than a clean example would have.** I ran this against `example.com` — a real, unrelated domain with its own real certificate, issued by a public CA. What came back instead was a certificate for `CN=example.com` issued by **`Zscaler Intermediate Root CA`**, three levels deep to a `Zscaler Root CA`. This machine's network sits behind a corporate TLS-inspecting proxy, and that proxy is doing, quite legitimately in this context, exactly the thing concept 1 called impersonation: it terminates the "real" TLS connection to `example.com` on the corporation's own infrastructure, inspects the plaintext, and re-encrypts a *brand new* TLS connection to me, presenting its own freshly-minted certificate claiming to be `example.com` — signed not by any public CA, but by Zscaler's own private root, which IT administrators installed into this machine's trust store precisely so this substitution wouldn't trigger a browser warning. `openssl s_client`, run with no special trust configured, correctly reports `Verify return code: 20 (unable to get local issuer certificate)` — it doesn't have Zscaler's root in its own trust store the way the OS does, so it flags the chain as unverifiable, which is the *correct* behavior for a tool with no opinion about corporate proxies. **This is the exact mechanism a malicious impersonator would need to pull off the same trick — the only difference between "sanctioned corporate TLS inspection" and "a MITM attack" is entirely a matter of *whose root certificate is in the trust store and whether you consented to put it there*.** If you're on a network you don't administer and you ever see a certificate chain like this for a site you didn't expect it from, that's the observable signature of exactly this. (If you run this command yourself, on an unfiltered connection, you'll most likely see `example.com`'s own real chain — historically issued via DigiCert — rather than this proxy's substitution; what you get depends entirely on what sits on your own network path, which is itself the lesson.)

### The chain of trust, and why there are usually three certificates, not one

Notice the captured trace above wasn't one certificate — it was three, forming a chain: a **leaf** (or **end-entity**) certificate for `example.com`, signed by an **intermediate CA**, which is itself a certificate signed by a **root CA**. This three-tier structure is close to universal in the public web PKI, and the reason for the middle tier is a direct application of a lesson Day 12 already taught in a different context: **a root CA's private key is the single most valuable secret in the entire chain — if it's ever compromised, every certificate it has ever signed, and every certificate its intermediates have signed, becomes untrustworthy simultaneously.** Root keys are therefore kept extraordinarily well-protected (offline, in hardware security modules, in physically secured facilities, used only rarely and under audited ceremony) and are used almost exclusively to sign a small number of long-lived intermediate certificates, which then do the actual day-to-day work of signing the millions of leaf certificates issued every day. If an intermediate is ever compromised or needs replacing, it can be revoked and rotated without ever touching the root — the same blast-radius logic as Day 12's discussion of zones as a unit of authority, applied to signing authority instead of DNS delegation.

The verification a browser (or `openssl`, given the right trust anchors) performs is exactly walking this chain in reverse: it takes the leaf certificate the server presented, checks the intermediate's signature on it verifies correctly, checks the root's signature on the intermediate verifies correctly, and then checks whether that root is one of the roughly 100–150 root certificates baked into its own **trust store** — the fixed list of root CAs a given OS or browser has decided to trust unconditionally. If the chain reaches a trusted root with every signature verifying, the certificate is considered valid; if it can't reach one (exactly what `openssl s_client` reported above, `unable to get local issuer certificate`), verification fails.

```
   ROOT CA                          "the anchor" — self-signed, offline,
   (self-signed, in the OS/         extremely long-lived (often 20–25
    browser's trust store)          years), used almost exclusively to
        │  signs                    sign intermediates
        ▼
   INTERMEDIATE CA                  does the day-to-day signing;
   (signed by the root)             can be revoked/rotated without
        │  signs                    touching the root (blast-radius
        ▼                            containment, same idea as Day 12's
   LEAF / END-ENTITY CERT           zone-as-failure-unit)
   (e.g. api.example.com,
    signed by the intermediate,     ← what the server actually
    presented by the server)          presents in the TLS handshake
```

**Who decides which root CAs get to be in that trust store in the first place — and this is a genuine governance decision, not a technical one.** Browser and OS vendors (the CA/Browser Forum, a cross-industry body including Google, Mozilla, Apple, Microsoft, and CA operators, publishes the **Baseline Requirements** every public CA must meet) each maintain their own root program with audit requirements a CA must pass to be included, and each can — and periodically does — remove a CA that fails to meet them. This isn't abstract: it is exactly the enforcement mechanism behind the DigiNotar case study below, and it's the reason "which root programs trust this CA" is a real, checkable question (`crt.sh`, run by a major CA, lets you search all certificates a CA has ever issued that were logged to Certificate Transparency).

### How you actually get a certificate — validation levels, and what they really check

A CA won't sign "this public key belongs to `yourbank.com`" without checking *something*. What it checks defines the validation level:

- **Domain Validation (DV)** — the CA verifies only that whoever is requesting the certificate controls the domain itself, typically by having them serve a specific token at a well-known HTTP path, or publish a specific DNS TXT record (Day 12, concept 4, showed you exactly this pattern: `_acme-challenge.example.com` TXT records). It proves nothing about the legal identity of who runs the site — just that they control the DNS name or the web server at that name. This is what Let's Encrypt issues, entirely automatically, and it's what the vast majority of the web's certificates are today.
- **Organization Validation (OV)** — additionally checks the requesting organization's legal existence and details against business records, at the cost of a manual review process that can't be automated and doesn't scale to "renewed automatically every 90 days."
- **Extended Validation (EV)** — the strictest checks, historically rewarded in browser UI with a prominent green company-name display in the address bar. **Verify current:** essentially every major browser has removed that distinguishing UI treatment in recent years, on the finding that users didn't reliably notice or understand it, and a certificate's validation level is no longer meaningfully visible to an end user glancing at the address bar at all — a genuinely fast-drifting UI fact worth re-checking before you cite it, and a real example of a security mechanism whose usability assumption quietly stopped holding.

The load-bearing fact for engineers, not just for security researchers, is this: **DV proves domain control, and nothing more.** A certificate for `paypa1-secure-login.com` (note the digit) can be a perfectly valid, correctly-issued DV certificate — the CA checked, correctly, that whoever requested it controls that exact domain — and it will show no browser warning at all, because the browser's only question is "does this certificate's name match the domain in the address bar," not "is this domain a legitimate business." Certificates authenticate *names*, not *trustworthiness* or *intent* — a distinction phishing relies on constantly, and one worth internalizing precisely because "the padlock is there" is not the same claim as "this is who you think it is."

### What happens when a certificate needs to stop being trusted early: revocation

A certificate's Not-After date is a promise made in advance, exactly like a DNS TTL (Day 12, concept 5) — but unlike DNS, certificates need a mechanism for the CA to say "actually, don't trust this one anymore, before its natural expiry" — the canonical trigger being exactly what the Heartbleed case study below caused at internet scale: a private key leaking. Two mechanisms exist, and neither is fully satisfying, which is itself worth knowing rather than assuming revocation "just works":

- **CRLs (Certificate Revocation Lists)** — the CA publishes a signed list of every serial number it has revoked; a client is supposed to download and check it. This scales badly (the lists grow without bound) and is largely legacy at this point.
- **OCSP (Online Certificate Status Protocol)** — a client asks the CA directly, in real time, "is this specific certificate still valid?" This adds a live network round trip (and a privacy leak: the CA learns which sites you're visiting, from every client that asks) to every TLS connection, which **OCSP stapling** fixes by having the *server* periodically fetch a signed, time-stamped "yes, still valid" attestation from the CA and staple it directly onto the TLS handshake, so the client never has to call out at all.

**The honest limitation, stated plainly:** most browsers implement revocation checking as **soft-fail** — if the OCSP responder can't be reached (which is exactly what an attacker who's just stolen your private key, and who might also control your network path, would try to arrange), most browsers proceed with the connection rather than blocking it, on the reasoning that treating "I couldn't check" the same as "definitely revoked" would make an unreliable OCSP responder into a way to take down every site that depends on it. This is a genuine, debated trade-off between availability and security, not an oversight, and it means revocation is meaningfully weaker in practice than its description suggests — one more reason the certificate-lifecycle discipline in this concept's System Design section (short lifetimes, fast automated rotation) matters more than revocation infrastructure alone.

### System design — the two decisions every organization running TLS at scale eventually has to make

**Scenario 1 — certificate lifecycle management for 200 services.**

**Problem:** your organization runs roughly 200 internal and external services, each needing one or more TLS certificates. Today, certificates are requested manually through a CA's web portal by whichever engineer happens to be provisioning a new service, tracked (optimistically) in a spreadsheet, and renewed when someone remembers or when a service goes down.

**Requirements:** no certificate is ever allowed to expire unnoticed in production; adding a new service's TLS shouldn't require a human ticket and a wait; the process must survive the original engineer who set it up leaving the company.

**Alternatives considered:**
1. **Status quo — manual requests, a spreadsheet, human memory for renewal.**
2. **Long-lived certificates (multi-year validity) to reduce how often anyone has to think about this.**
3. **Fully automated issuance and renewal via the ACME protocol** (RFC 8555) — the protocol Let's Encrypt popularized, in which a piece of software on your infrastructure proves domain control programmatically (via an HTTP-01 or DNS-01 challenge, Day 12 concept 4) and receives a signed certificate with zero human interaction, on a schedule that renews well before expiry.

**The decision: (3), with short-lived certificates specifically, not long ones.**

**The actual reason:** this is a direct instance of the same lesson Day 12 built around DNS TTLs, applied to certificates instead of DNS records — **a human-driven process has a failure rate that scales with the number of things a human has to remember, and remembering doesn't scale past a handful of items.** With 200 services, "someone remembers to renew it" is not a plan, it's a probability distribution with a long tail that resolves, eventually, in an outage. Automating issuance removes the human from the loop entirely for the common case, which is the only way the failure rate stays low as the service count grows. And *short*-lived certificates (Let's Encrypt's default has been 90 days; some CAs and internal PKI systems now issue certificates valid for single-digit days) are the correct pairing with automation, not a separate preference: a short lifetime makes automated renewal something that runs constantly and is therefore continuously exercised and continuously alarms if it breaks, whereas a certificate that's valid for two years lets a broken renewal pipeline sit silently undetected for up to two years before it manifests as an outage — you want the failure mode to be "renewal broke, we noticed within a day" rather than "renewal broke, we notice when the site goes down."

**The trade-off, honestly:** automated ACME issuance requires infrastructure investment — DNS-01 automation needs API access to your DNS provider from your renewal tooling (itself a security surface: whatever can programmatically create `_acme-challenge` TXT records can, in principle, obtain a certificate for your domain), and every service needs the renewal daemon correctly deployed and its expiry-alerting correctly wired, which is nontrivial the first time and easy to get subtly wrong (a renewal job that fails silently is worse than no automation at all, because it removes the human vigilance the manual process at least had). **Short lifetimes also mean more frequent operations, which is more surface area for the automation to fail on** — you're trading "rare, manual, human-error-prone" for "frequent, automated, automation-error-prone," which is the right trade at scale but is a real cost, not a free upgrade.

**Flip condition:** manual issuance remains defensible for a genuinely small number of services with a named, accountable owner and a calendar reminder that's actually checked — but that "small number" erodes fast, and the honest threshold most organizations discover the hard way is somewhere well under 200: the moment you can't name, from memory, every certificate you're responsible for, you've already crossed it. The multi-year-certificate alternative (2) is essentially never the right answer today for anything internet-facing — beyond the failure-detection argument above, browser and CA/Browser Forum policy has been steadily *shortening* the maximum allowed public-certificate lifetime over the years specifically to force better hygiene industry-wide (**verify current maximum lifetime** before relying on a specific number — this is one of the fastest-moving policy areas in the whole PKI ecosystem).

**Failure modes:** an ACME renewal job with no alerting on failure (it silently stops working three renewal cycles before expiry, and the first anyone hears about it is the outage — this is precisely the shape of the Microsoft Teams case study below); DNS-01 automation credentials that are broader than they need to be (a compromised renewal service that can edit *any* DNS record, not just `_acme-challenge` TXT records, is a much larger blast radius than the certificate system needed to grant it); and treating "we automated issuance" as done, without separately verifying the automation actually completed successfully on a monitored schedule rather than merely being configured to run.

**Scenario 2 — where should TLS terminate: at the edge, or all the way to each service?**

**Problem:** a company runs a public API behind a load balancer, which fans requests out to a fleet of backend microservices inside a private VPC. Currently, TLS terminates at the load balancer, and traffic from the load balancer to each backend service travels as plain HTTP, on the reasoning that the VPC is a private network no external party can reach.

**Requirements:** protect customer data in transit; support a security posture that doesn't collapse if a single internal component is compromised; keep operational complexity proportionate to the actual risk.

**Alternatives:**
1. **Terminate TLS at the edge (load balancer/API gateway); plaintext inside the private network.** Simple: one certificate to manage (or one per public hostname), and every internal service is spared the CPU cost and code complexity of TLS entirely.
2. **End-to-end mutual TLS (mTLS)** — every internal service-to-service call is independently encrypted and *mutually* authenticated: not just does the client verify the server's certificate, the server also verifies a certificate presented by the client, so every hop knows cryptographically, not just by network position, exactly which service it's talking to.

**The decision:** (1) is defensible and extremely common for a straightforward internal network with a small number of trusted services and a conventional perimeter-security posture; (2) is the right call once you're operating under a **zero-trust** model — the explicit assumption that any single host or network segment might already be compromised, and that "it's inside our private network" must never be treated as a security boundary on its own.

**The actual reason:** option (1)'s entire safety argument rests on one assumption — that nothing malicious can ever get *onto* the private network — and that assumption has a well-documented, repeatedly-demonstrated failure mode: a single compromised container, a misconfigured security group, a supply-chain-compromised dependency running inside one service, or a insider threat, and an attacker is now on the "trusted" side of the perimeter, where every service-to-service call is plaintext and unauthenticated by anything other than network position. mTLS makes the compromise of one internal component *not* automatically extend to reading or forging every other internal call, because each service must independently prove its identity with a certificate, not merely its network location.

**The trade-off, honestly, and it is a large one:** mTLS is operationally heavy in a way option (1) simply isn't. Every service now needs its own certificate — meaning you almost certainly need an **internal/private CA** (concept 3's chain-of-trust machinery, but run by you, for names that only mean something inside your own infrastructure, rather than a public CA that has no reason to ever validate `payments-internal.svc.cluster.local`) — with its own issuance, rotation, and revocation pipeline, at internal-service scale rather than public-domain scale, which is often a *larger* number of certificates than a company has public-facing hostnames. Every service also now pays real TLS handshake and encryption CPU cost on every internal call, not just at the edge, and debugging a broken internal call now requires understanding whether the failure is at the application layer or the mTLS handshake layer — a genuine increase in operational surface area that teams reaching for "zero trust" as a buzzword frequently underestimate.

**Flip condition — this is where the decision actually turns:** the deciding factor is not company size, it's **the value of what's inside the perimeter and your actual confidence that the perimeter holds.** A small internal analytics service with no access to customer PII, behind a network only your own small engineering team can reach, genuinely doesn't need mTLS — the operational cost isn't justified by the risk. A payments processing path, or any service handling regulated data, crosses the line where "our network is private" stops being an acceptable security argument on its own, regardless of company size — this is precisely why service meshes (Istio, Linkerd, and similar) exist largely to *automate away* the operational burden described above, handling per-service certificate issuance and rotation transparently so that mTLS becomes closer to a checkbox than a project. In practice, most organizations land on a middle position: mTLS for the highest-value internal paths, plaintext-but-perimeter-protected for the rest — which is itself a real, deliberate risk decision, not an accident, as long as it's made explicitly rather than by default.

**Failure modes:** mTLS certificate rotation breaking silently for one service and taking it out of the mesh entirely (a much more confusing failure than a public-facing cert expiry, because the symptom is "service A can't reach service B" with no browser warning or user-facing signal at all); treating "we have a service mesh" as automatically meaning "we have zero trust" without verifying every service is actually enrolled and enforcing mTLS rather than accepting plaintext as a fallback; and the edge-termination model's classic failure — a load balancer or WAF misconfiguration that exposes the plaintext internal traffic to a network segment it was never meant to reach (a VPC peering misconfiguration, an overly broad security group), silently converting the "no external party can reach this" assumption from true to false with no alert generated, because nothing was watching for that assumption to break.

### Case studies

**DigiNotar (2011) — what happens when a link in the chain of trust itself fails.** In the summer of 2011, attackers breached **DigiNotar**, a Dutch certificate authority trusted by every major browser, and used the compromised systems to issue over 500 fraudulent certificates for domains they had no right to — including one for `*.google.com`. Those fraudulent certificates were subsequently used in the wild to conduct man-in-the-middle interception of HTTPS traffic, most notably against roughly 300,000 users in Iran, primarily targeting Gmail. The interception was first detected not by DigiNotar, but by Google Chrome's built-in certificate pinning for Google's own domains — Chrome had hardcoded which certificates were expected for `google.com`, and flagged the DigiNotar-issued one as anomalous the moment a user's browser encountered it, which is precisely a client-side mitigation for the exact class of failure this whole concept is built around. Once the fraud was confirmed, Mozilla, Google, and Microsoft revoked trust in DigiNotar's root certificates entirely, across every browser — meaning every certificate DigiNotar had ever legitimately issued to anyone else stopped being trusted too, an illustration of exactly the blast-radius argument made above for why root keys are so tightly guarded. DigiNotar declared bankruptcy within weeks. **The engineering lesson:** the entire security model rests on the assumption that CAs are not compromised, and that assumption is not automatically true — it has to be continuously monitored for, which is precisely why this incident directly motivated **Certificate Transparency** (RFC 6962, championed by Google): a public, append-only, cryptographically-verifiable log of every certificate ever issued by a participating CA, so that a fraudulently-issued certificate for your domain becomes *detectable* by anyone watching the logs (`crt.sh` is a public search interface over these logs), rather than silently exploitable until a browser happens to notice by luck, the way Chrome's Google-specific pinning did here. A related, smaller-scale echo of the same failure mode worth knowing by name: in 2019, Kazakhstan's government attempted to compel citizens to install a state-issued root certificate that would have allowed nationwide TLS interception of traffic to major sites — browser vendors (Mozilla and Google both) responded by actively blocklisting that specific certificate, refusing to trust it regardless of what a user's own device was configured to accept, a direct demonstration that the trust store is ultimately a policy decision enforced by the browser vendor, not merely a passive list. *Primary sources:* Fox-IT's forensic investigation report on the DigiNotar breach (commissioned by the Dutch government, published 2011); Mozilla's and Google's public security-advisory posts announcing root removal; RFC 6962 (Certificate Transparency). **Verify current** exact figures (the ~300,000-user estimate) against the original Fox-IT report if citing precisely — it has been rounded and re-reported inconsistently across secondary sources over the years.

**Microsoft Teams' outage (February 2020) — the case study that's really about operations, not cryptography.** On Monday, 3–4 February 2020, Microsoft Teams experienced a multi-hour global outage that prevented users from signing in. Microsoft's own public statement, posted to its Microsoft 365 status channel and widely reported by technology press at the time (e.g., ZDNet, The Verge), attributed the outage to an **internal authentication certificate that had expired** without being rotated in time, breaking the token-issuing/authentication path that every Teams client depends on to sign in — a service that was otherwise completely healthy was rendered globally unreachable by exactly one date field crossing exactly one threshold. **Verify current** the precise wording Microsoft used and the exact scope of the affected component before quoting it verbatim — this is the kind of vendor-statement detail that gets paraphrased slightly differently across secondary sources over time, and Microsoft's own original statement is the primary source to track down if you need the exact language. **The engineering lesson, and it's the one the study plan's framing is built around:** everything this concept just taught about chains of trust, validation levels, and cryptographic strength is *completely irrelevant* to this outage. Nobody broke the cryptography. Nobody forged a signature. The certificate was, right up until the moment it wasn't, a perfectly valid, correctly-issued, unbroken certificate — and it took down a product used by tens of millions of people anyway, because **an expiring certificate is not a security event, it's a scheduled event that a monitoring system either catches with days or weeks of runway, or doesn't catch at all, and there is no middle ground where "we noticed at the last minute" buys you anything.** This is precisely why the System Design scenario above insists on automation *and monitored, alerting renewal* rather than automation alone — an ACME renewal job that fails silently is architecturally identical to the manual process that caused this outage, just with a different-looking failure. **The pattern to internalize, in one sentence: certificate expiry is the single most preventable class of outage in this entire note, and organizations of every size, including ones with far more security engineering resource than most, keep having it happen anyway, because it's boring, it's invisible until it isn't, and "the cert is still valid for another 60 days" feels like a problem for future-you right up until it's everyone's problem at once.**

### In production — certificates and trust operationally

**Best practices:** automate issuance and renewal for every certificate without exception (Scenario 1 above) and alert on **days-until-expiry crossing a threshold with real lead time** (30 and 7 days is a common pairing) — never alert only on the certificate having *already* expired, because by then the outage has already started; monitor renewal-job *success*, not just its schedule (a cron job that ran and silently failed looks, from the outside, identical to one that succeeded, until the certificate expires anyway); prefer DNS-01 ACME challenges scoped to the narrowest possible DNS-provider API permission your tooling can request, rather than credentials broad enough to edit any record; and treat your CA choice and root-program coverage as a real architectural decision for anything beyond the public web (an internal CA's root needs to be distributed to every client that will validate against it — including non-browser clients like backend services and, per this note's framing note, autonomous agents making outbound calls, which don't get a human's judgment call on a certificate warning).

**Mistakes, beginner → senior:** *beginner* — confusing "the padlock icon is present" with "this site is legitimate," not realizing DV certificates only prove domain control (the `paypa1-secure-login.com` problem above); *intermediate* — manually-issued certificates tracked in a spreadsheet, discovered to have lapsed only when a service goes down; *senior* — treating a service mesh's mTLS as a completed security posture without verifying every service actually enforces it rather than falling back to plaintext, and underestimating how much a private/internal CA's own root-key protection matters just because "it's only used internally" — an internal CA compromise has exactly the same blast radius, inside your network, that a public CA compromise (DigiNotar) has on the internet.

**Monitoring:** external, from-the-internet certificate-expiry scanning independent of your own infrastructure's self-reported health (an internal monitoring agent that itself depends on the certificate it's checking can fail exactly when you need it most); Certificate Transparency log monitoring for your own domains (`crt.sh` or a commercial CT-monitoring service), specifically to catch a fraudulently-issued certificate for your name the way Chrome's pinning caught DigiNotar's, but generalized to every domain, not just Google's; and OCSP-stapling health as part of your standard TLS configuration audit, since a misconfigured staple is a subtle, easy-to-miss source of client-side slowdowns or soft-fail security gaps.

**Cost:** automated DV certificates from Let's Encrypt or similar ACME-compatible CAs are free; the real cost is entirely engineering time to build and monitor the automation once, which amortizes to near-zero per certificate at scale — versus the essentially unbounded cost of an outage caused by a lapsed certificate, which is precisely the asymmetry that makes "automate this" a decision that shouldn't require much deliberation once you've seen the Teams case study.

---

## 4. The TLS handshake: messages, round trips, and what it costs

**Depth: [CORE]**

### Intuition

Concept 2 built the tool (key exchange) and concept 3 built the tool (identity verification via certificates). Neither, alone, is a protocol — you still need an actual, precisely-ordered exchange of messages that negotiates *which* cryptographic algorithms both sides will use (a client and server built years apart, by different vendors, must agree on a common language before anything else can happen), performs the key exchange, transmits and verifies the certificate chain, and — critically — cryptographically binds all of that negotiation together so that nothing about *how* the connection got secured can itself be tampered with in transit. That entire choreographed exchange, run once at the start of a connection before a single byte of your actual HTTP request is trusted, is the **TLS handshake**. Day 11 promised, without explaining why, that adding TLS to a request would cost "2–3 RTTs" on top of Day 1's latency pyramid. This section is where that promise gets paid off, precisely.

### Under the hood: the TLS 1.3 handshake, message by message

RFC 8446 (the TLS 1.3 specification, August 2018) organizes the handshake into two phases, and the split matters: everything up through the key exchange happens in the clear (there's no shared secret yet to encrypt with), and everything after it is already encrypted, even though the handshake itself isn't finished. Here is the message flow, simplified from RFC 8446 §2's own figure:

```
CLIENT                                                        SERVER

ClientHello
 + key_share        (client's ephemeral DH/ECDHE public value)
 + signature_algorithms
 + server_name      (SNI — concept 5; sent in THE CLEAR)
 + supported_versions
 + alpn             (which APPLICATION protocol — concept 7/8)
                          ── flight 1 ──►

                                                          ServerHello
                                            + key_share  (server's ephemeral public value
                                                           — BOTH SIDES CAN NOW COMPUTE
                                                           THE SHARED SECRET, concept 2)
                                        {EncryptedExtensions}
                                                {Certificate}   ← concept 3's chain
                                          {CertificateVerify}   ← signature PROVING
                                                                   possession of the
                                                                   private key matching
                                                                   that certificate
                                                    {Finished}   ← MAC over the ENTIRE
                                                                   transcript so far
                          ◄── flight 2 ──
{Finished}                                            [Application Data ...]
[Application Data — e.g. the HTTP request]
                          ── flight 3 ──►
                                                       [Application Data — the HTTP response]

{}  = encrypted with keys derived from the handshake traffic secret
[]  = encrypted with keys derived from the (now fully established) application traffic secret
```

Walk what each message actually does, because each one is answering a specific requirement from concepts 1–3, not existing by convention:

- **`ClientHello`** proposes: which TLS version(s) it supports, which symmetric ciphers (AEAD algorithms, concept 2) it's willing to use, and — the part that makes 1-RTT possible at all — an ephemeral **`key_share`**: the client's freshly-generated Diffie-Hellman public value, sent *speculatively*, before it even knows which group the server will pick, for one or more of its preferred groups. Sending this speculatively, rather than waiting to be told which group to use, is precisely what collapses two round trips into one — the old TLS 1.2 flow (below) waited for the server to state its preferences before the client generated anything.
- **`ServerHello`** picks a version, a cipher, and replies with its own `key_share`. **The instant this message is fully received, both sides possess everything concept 2's Diffie-Hellman example needs to independently compute the identical shared secret** — this is the moment, one round trip in, where "unencrypted handshake" ends and "encrypted handshake" begins, which is why every message after `ServerHello` in the diagram above is wrapped in `{}`.
- **`alpn` (Application-Layer Protocol Negotiation, RFC 7301)** is the extension worth flagging now and returning to in concepts 7 and 8, because it answers a question this note hasn't addressed yet: once the handshake secures a connection, how do the two sides agree on *which HTTP version* to actually speak over it? The client lists every application protocol it supports, in preference order (`h2`, `http/1.1`, and, over QUIC, `h3`), inside this same `ClientHello`; the server picks one from the client's list and states its choice inside `EncryptedExtensions`. **This is why "does this server support HTTP/2" is, in practice, entirely a property of the TLS handshake, not a separate negotiation** — a browser and server settle on HTTP/2 versus HTTP/1.1 in the same round trip that establishes encryption, at zero extra cost, which is exactly why HTTP/2 deployment in the wild is overwhelmingly TLS-only: the cleartext alternative (`h2c`) has no equivalent zero-cost negotiation point to piggyback on, and essentially no browser implements it.
- **`Certificate`** and **`CertificateVerify`** are concept 3, delivered: the certificate chain itself, plus a fresh digital signature — computed *right now*, over this specific handshake's transcript, using the private key that matches the certificate's public key — proving the server genuinely holds that private key and isn't just replaying somebody else's captured certificate. This is the exact mechanism that closes the impersonation gap concept 3 opened: a certificate alone is a public document anyone can copy; `CertificateVerify` is the proof that *this* party, right now, actually controls the matching private key.
- **`Finished`**, sent by both sides, is a **MAC (message authentication code) computed over every single handshake message exchanged so far** — meaning it cryptographically commits to the entire negotiation: which version was chosen, which cipher, everything. **This is the direct, structural fix for the Logjam-style downgrade attack from concept 2's case study**: if an attacker tampered with any earlier handshake message — say, rewriting the `ClientHello`'s offered cipher list to exclude every strong option, forcing a weak one — the resulting transcript the two sides computed their MACs over would differ from what was actually sent, `Finished` would fail to verify, and the connection aborts before a single byte of application data is trusted. Downgrade attacks are a fight over *who controls the negotiation*, and `Finished` is TLS 1.3's answer: nobody gets the last unauthenticated word.
- Once the client has sent its own `Finished`, it can (and typically does) send its actual HTTP request in the very same flight — no additional round trip is spent purely on "finishing" the handshake before application data can move.

### Why TLS 1.2 cost one more round trip — the black box named, not fully reopened

TLS 1.2 (RFC 5246, now formally deprecated for new use by RFC 8996) used a shape that looks similar at a glance but has one structural difference with an outsized cost: the client's `ClientHello` proposed *only* which ciphers it supported, without speculatively sending a key-exchange value — it had to wait for the server's `ServerHello` (plus, in the classic flow, a separate `ServerKeyExchange` message) to learn which cipher and group were chosen *before* it could generate and send its own key-exchange material in a *third* flight (`ClientKeyExchange`), followed by its own `Finished`. That's the shape difference in one sentence: **TLS 1.2 negotiates, then exchanges keys, as two sequential round trips; TLS 1.3 does both in the same round trip by having the client guess and send its key share up front.** The full legacy message inventory (`ServerKeyExchange`, `ServerHelloDone`, the separate `ChangeCipherSpec` messages, static-RSA key transport as an alternative to Diffie-Hellman) is a deliberate stop for this note — TLS 1.2 is legacy-compatibility surface at this point, not something you'd design into new infrastructure, and per this concept's CORE tier the mechanism worth opening fully is TLS 1.3, which is what any new deployment should target; TLS 1.2's internals are named precisely enough to justify its round-trip cost and no further.

### Worked example — the exact round-trip cost, completing Day 11's forward reference

Day 11 used a server 100 ms away and said, without justifying the number, "add TLS and it's 2–3 RTTs." Use that same 100 ms one-way-trip-time round trip (RTT = 100 ms) and count precisely, in two different units that are easy to conflate: **RTTs of *setup* before the client can send its first HTTP request byte** (Day 11's exact framing), and **total RTTs before the first byte of the HTTP *response*** (one more round trip, for the request/response exchange itself, layered on top).

```
                                    SETUP RTTs        SETUP TIME       TOTAL RTTs TO
                                 (before 1st HTTP      @ RTT=100ms     1st RESPONSE BYTE
                                  byte can be SENT)                    (setup + 1 for HTTP)

Plain HTTP over TCP                    1                100 ms              2   (200 ms)
HTTPS, TLS 1.2 full handshake          3                300 ms              4   (400 ms)
HTTPS, TLS 1.3 full handshake          2                200 ms              3   (300 ms)
HTTPS, TLS 1.3, 0-RTT resumption       1                100 ms              2   (200 ms)
HTTP/3 (QUIC), fresh connection        1                100 ms              2   (200 ms)
HTTP/3 (QUIC), 0-RTT resumption        0                  0 ms              1   (100 ms)
```

Read the middle column first, because it's the exact number Day 11 forward-referenced: TCP's handshake is always 1 RTT on its own; TLS 1.3 adds exactly 1 more (2 total) and TLS 1.2 adds 2 more (3 total) — **"2–3 RTTs," precisely, and now you know exactly which number comes from which protocol version.** Two rows are worth sitting with a moment longer:

- **TLS 1.3 with 0-RTT resumption matches plain, unencrypted HTTP's total cost exactly (200 ms) — while staying fully encrypted.** This is only possible on a *repeat* connection to a host you've talked to recently enough to still hold a resumption ticket (a secret, derived from a previous session, that lets the client send encrypted application data in its very first flight, before the server has even replied) — and it comes with the replay caveat the second System Design scenario below exists to work through.
- **HTTP/3 over QUIC achieves TLS 1.3's *full-handshake* setup cost (200 ms) on your very first-ever connection to a host, with no prior visit needed at all** — because QUIC never pays for a separate transport handshake in the first place; its "Initial" packet carries the TLS 1.3 `ClientHello` from the very first UDP datagram, merging what TCP+TLS keeps as two sequential handshakes into one. That is precisely Day 11 concept 13's "TLS is integrated into it, not layered on top" statement, now quantified. Concept 8 below builds the rest of what changes at the HTTP layer to make this real.

**Honesty about the numbers:** this table assumes zero packet loss, a single coalesced flight per round trip, and no additional latency from certificate-chain size pushing a message across multiple packets — all realistic simplifications for a teaching example, all things that make real measurements noisier. **Verify with your own measurements** (`curl -w` timing fields, or a packet capture) before using these exact multipliers for capacity planning; the *shape* of the result (TLS 1.2 > TLS 1.3 > TLS 1.3 0-RTT ≈ plaintext, and QUIC 0-RTT as the floor) is the durable, protocol-mandated part.

### Analogy: a diplomatic handshake with a strict "no do-overs" rule

Picture two parties meeting for the first time to conduct sensitive business, under a rule imposed by both their governments: every round of back-and-forth communication is expensive and rationed, so waste none. The old protocol had them announce their intentions in one exchange, wait to hear the other's preferences, *then* separately work out a shared codeword scheme — two full rounds of correspondence before anything sensitive could be discussed. The new protocol has each party show up already holding a proposed codeword, computed in advance for a couple of likely scenarios, and offer it in the very first letter alongside their opening statement — so if the other side's preference matches one of the guesses (which, with only a handful of well-known "groups" in practice, it usually does), the codeword scheme is settled in the very same exchange that started the conversation. Each party also brings a sealed letter of introduction from a mutually recognized notary (the certificate) and, crucially, ends their very first reply with a phrase that could only be spoken correctly by someone who received *every earlier letter exactly as sent* — so if anyone intercepted and altered a letter along the way, that closing phrase won't check out, and the whole negotiation is thrown out before any business is discussed.

**Where the analogy breaks:** real diplomats care about the *content* of what's negotiated, not primarily the number of letters exchanged — a human negotiation optimizes for getting the terms right, and would happily take an extra round to double-check something important. TLS's design pressure runs the opposite direction: the *terms* (which specific cipher, which specific group) are drawn from a short, pre-agreed, publicly-specified list where getting it "wrong" barely matters as long as it's on the safe list — so the entire design effort goes into minimizing *how many trips the letters take*, because on a real network, latency (not deliberation) is the dominant cost, and Day 1's latency pyramid means every round trip is a fixed, unavoidable tax paid in milliseconds no amount of clever content can reduce.

### System design — two decisions that trade security guarantees for round trips

**Scenario 1 — reducing handshake latency for users on a high-RTT mobile network.**

**Problem:** a consumer app has a substantial user base in a region where the nearest datacenter is 300 ms away (a genuinely high but realistic RTT for some undersea-cable-limited regions), the app makes frequent short API calls, and mobile network conditions mean TCP connections are dropped and re-established constantly as users move between cell towers and Wi-Fi.

**Requirements:** minimize the latency users feel on each request; the fix must hold up under frequent reconnection, not just on a single long-lived connection; security guarantees can bend somewhat but must not silently disappear.

**Alternatives:**
1. **Do nothing structural; rely on TCP + TLS 1.3 full handshakes.** At 300 ms RTT, this is 200 ms of setup (using the table's formula: 2 RTT) *per reconnection*, before any request can even be sent.
2. **Enable TLS session resumption via tickets**, without 0-RTT — the abbreviated handshake still needs 1 RTT to complete (the shape doesn't fully collapse without 0-RTT specifically), but skips the expensive certificate-chain verification and signature computation, saving CPU on both ends without saving a round trip.
3. **Enable 0-RTT early data on top of resumption** — collapses setup to the 1-RTT-total row in the table above (matching plaintext HTTP's cost), at the cost of the replay exposure Scenario 2 examines in depth.
4. **Move to HTTP/3 (QUIC)** — gets the *full-handshake* setup cost down to what TLS-1.3-over-TCP needs *resumption* to achieve, on every fresh connection, *and* survives the mobile network's constant IP changes without dropping the connection at all (Day 11 concept 13's connection-ID mechanism), which directly attacks this scenario's "constantly reconnecting on mobile" requirement at its root rather than just making each individual handshake cheaper.

**The decision:** (4) as the structural fix, with (3) layered on top for genuinely idempotent, latency-critical requests specifically.

**The actual reason:** this scenario's requirements name two distinct pains — slow handshakes, and *frequent, disruptive reconnection* — and only QUIC addresses the second one at all. Shaving round trips off a TCP handshake that a flaky mobile network is going to tear down and rebuild every few minutes anyway is optimizing the wrong variable; eliminating the reconnection itself (via connection migration, Day 11 concept 13) removes far more accumulated latency over a session than any handshake optimization could, because it removes the *need* to re-run the handshake at all when the network changes underneath the user.

**The trade-off, honestly:** QUIC support requires both client and server infrastructure investment (Day 11 concept 13 already covered the real CPU-overhead and observability costs at length — cross-reference rather than re-litigate here), and layering 0-RTT on top for specific requests adds the replay-handling complexity Scenario 2 works through in full. This is not a free latency win; it's trading engineering and operational investment for a latency improvement that, at 300 ms RTT, is large enough in absolute terms (hundreds of milliseconds per interaction, repeatedly, for the app's entire affected user base) to justify that investment.

**Flip condition:** if the app's actual usage pattern is a small number of long, stable connections (a persistent WebSocket-style session, established once and held for the life of the app) rather than many short API calls, the reconnection-frequency problem this scenario is built around mostly disappears, and a one-time 300 ms handshake amortized over an hour-long session is not worth this investment — optimize elsewhere. The decision flips on *connection churn*, not on RTT alone.

**Failure modes:** enabling QUIC without a TCP+TLS-1.3 fallback path for the fraction of networks that actively block or throttle UDP (a real, documented interoperability concern — RFC 9308, "Applicability of the QUIC Transport Protocol," discusses this explicitly, and it's the reason every production HTTP/3 deployment advertises support opportunistically via `Alt-Svc` rather than requiring it); and rolling out session resumption without appropriately short ticket lifetimes, silently keeping stale session state usable far longer than intended.

**Scenario 2 — should a payment-processing API enable TLS 1.3 0-RTT early data?**

**Problem:** the same latency pressure as Scenario 1, but now the endpoint in question initiates a financial transaction, and the question is specifically whether to accept the fastest available handshake option.

**Requirements:** minimize latency where safe to do so; never allow a transaction to be executed twice from a single client action; make the security trade-off an explicit, reviewed decision rather than a default nobody examined.

**The trap, stated first:** 0-RTT data is submitted by the client *before* the server has had any chance to confirm the client isn't simply replaying a previously captured flight verbatim. Unlike every other part of the TLS 1.3 handshake, which the `Finished` MAC binds to a fresh, unique exchange, the 0-RTT data has no such freshness guarantee baked in by the protocol — RFC 8446 §8 states this limitation explicitly and puts the burden of handling it on the application. **A network attacker who captured your 0-RTT-carrying packet once can resend it, and a naive server has no protocol-level way to distinguish that from you genuinely reconnecting and repeating the same action.** If that packet's application data was "charge this card $500," the naive server processes it again.

**Alternatives:**
1. **Enable 0-RTT globally, for every request type, to capture the maximum latency win everywhere.**
2. **Disable 0-RTT entirely for this API**, accepting the full 1-RTT (or 2-RTT) handshake cost on every connection.
3. **Enable 0-RTT only for requests that are safe to execute more than once** — read-only lookups and other operations a repeat of causes no harm (this is exactly the **idempotent methods** distinction Day 13 builds in depth around HTTP's own method semantics) — while routing anything that mutates state (initiating a charge, submitting an order) through the full handshake, or additionally deduplicating via an application-level idempotency key regardless of what TLS layer it arrived on.

**The decision:** (3), and it is close to the industry-consensus answer for exactly this reason — it is also the choice several major TLS-terminating providers (Cloudflare among them, in their own public engineering writing on TLS 1.3 0-RTT) explicitly recommend by default, restricting 0-RTT to methods and endpoints known to be replay-safe.

**The actual reason:** the risk 0-RTT introduces is not diffuse — it's precisely and only a problem for actions where "this happened twice" is a real-world harm. A `GET` for a product's current price causes no harm if a network glitch (or an attacker) causes it to execute twice; a `POST` that charges a card absolutely does. Blanket-enabling (1) accepts unnecessary risk on the operations that can least afford it; blanket-disabling (2) discards a real, meaningful latency win on the large fraction of traffic (read-heavy APIs in particular) where the risk is genuinely zero.

**The trade-off, honestly:** option (3) requires your application layer to actually know, and correctly enforce, which of its own endpoints are replay-safe — TLS itself has no concept of your application's semantics, so the safety boundary has to be drawn and maintained by you, at the routing or application layer, and it's easy to get subtly wrong (an endpoint that *looks* like a read but has a side effect — logging an access that feeds into a billing calculation, for instance — is exactly the kind of case that slips through a hasty audit). Deduplication via an idempotency key is a stronger, defense-in-depth answer than method-based routing alone, but it's additional application-layer engineering, not something TLS or the web server gives you for free.

**Flip condition:** disable 0-RTT entirely, full stop, the moment you cannot confidently enumerate and continuously audit which of your endpoints are truly replay-safe — a fast-growing API surface where new endpoints are added faster than a security review can classify them is exactly the situation where option (2)'s blanket caution is the responsible default, and you re-enable (3) once your endpoint inventory and classification process is trustworthy enough to keep up.

**Failure modes:** classifying an endpoint as replay-safe based on its HTTP method alone without checking for side effects (analytics, rate-limit counters, cache warming) that a repeat execution would double-count; and the quieter failure of enabling 0-RTT and never building the monitoring to notice when it's actually being replayed in the wild — an attacker who understands this exposure and finds one exploitable endpoint will not announce it.

### Case studies

**Heartbleed (April 2014) — a bounds-check bug in the machinery this whole concept just described.** The TLS/DTLS **Heartbeat Extension** (RFC 6520, February 2012) lets either side of an established connection send a small message containing a payload and a stated payload length, asking the peer to echo it back verbatim — a lightweight way to keep an otherwise-idle connection alive. OpenSSL's implementation of this extension trusted the attacker-supplied length field without verifying it matched the data actually sent: a client could claim a payload length of up to roughly 64 KB while sending only a handful of bytes, and the vulnerable server would read past the end of the real payload into adjacent process memory and echo whatever it found back to the attacker — as a normal, protocol-legal response, indistinguishable in any standard log from ordinary traffic. That adjacent memory could, and in demonstrated proofs-of-concept did, contain the server's own TLS **private key** — the exact secret every certificate in concept 3 depends on staying secret, and the exact secret concept 2's entire hybrid-encryption design assumes an attacker cannot obtain. **CVE-2014-0160** was disclosed on 7 April 2014, simultaneously by Google security researcher Neel Mehta and the Finnish security firm Codenomicon (who registered `heartbleed.com` and gave the bug its name and logo — an unusually public disclosure style, chosen specifically because of how many ordinary users and site operators needed to act quickly). The flawed code had been present in the OpenSSL 1.0.1 branch since its release in March 2012, undiscovered for roughly two years. **The lesson the study plan's framing points at directly:** OpenSSL, despite implementing the cryptography underpinning a very large share of the internet's HTTPS traffic, banking infrastructure, and embedded devices, was at the time maintained by a small team with minimal dedicated funding — a mismatch between the software's criticality and the resources behind it that Heartbleed turned from an abstract concern into front-page news, and directly motivated the Linux Foundation's **Core Infrastructure Initiative** (announced 2014) to fund maintenance of critical open-source infrastructure, with OpenSSL among its first beneficiaries. The concrete operational lesson, tying back to concept 3: a leaked private key means every certificate built on it must be treated as compromised and reissued — Heartbleed forced this at a scale (hundreds of thousands of certificates needing revocation and reissuance within days) that stress-tested the entire certificate-lifecycle machinery concept 3's System Design argues you should already have automated, long before you're forced into it under incident conditions. *Primary sources:* CVE-2014-0160; `heartbleed.com` (Codenomicon's original disclosure site); RFC 6520; OpenSSL's own security advisory for the fix (shipped in OpenSSL 1.0.1g).

**POODLE (October 2014) — a protocol-downgrade attack, the same family as Logjam.** **CVE-2014-3566**, disclosed 14 October 2014 by Google security researchers Bodo Möller, Thai Duong, and Krzysztof Kotowicz in the paper *"This POODLE Bites: Exploiting the SSL 3.0 Fallback,"* exploited a padding-validation weakness specific to SSL 3.0's CBC-mode cipher suites — a protocol version already nearly two decades old by 2014, but still supported by browsers and servers as a compatibility fallback. An active man-in-the-middle attacker could force (or take advantage of) a client's automatic retry-on-failure behavior to trigger a downgrade to SSL 3.0, then use repeated, crafted requests exploiting the padding weakness to decrypt small amounts of data — such as a session cookie — one byte at a time. **This is structurally the same failure family as concept 2's Logjam case study, one layer up the stack:** Logjam downgraded the *key-exchange group*; POODLE downgraded the *protocol version itself* — and both worked because, at the time, negotiation could be steered down to an option weak enough to break, without the negotiation itself being tamper-evident. This is precisely the gap TLS 1.3's `Finished` message (this section's own mechanism) closes structurally: a transcript-binding MAC over the *entire* negotiation, including which version and cipher were chosen, means an attacker cannot quietly rewrite the negotiation to steer it toward a weak option without the resulting `Finished` failing to verify. The direct consequence for the ecosystem: SSL 3.0 was formally deprecated by the IETF (RFC 7568, June 2015), and every major browser removed the automatic-downgrade behavior that made the attack practical in the first place. *Primary sources:* Möller, Duong, Kotowicz, Google Security Team, 14 October 2014; CVE-2014-3566; RFC 7568.

### In production — the handshake, operationally

**Best practices:** disable TLS 1.0 and 1.1 entirely (both formally deprecated by RFC 8996, March 2021 — **verify current** which versions your load balancer or web server still enables by default, since defaults lag deprecations by years in practice); prefer TLS 1.3 wherever your entire client population supports it, specifically for the round-trip win this section just quantified; enable session resumption broadly (it's a pure win — same security properties for the abbreviated exchange, at lower CPU cost) while restricting 0-RTT specifically to audited, replay-safe endpoints per the System Design scenario above; and measure actual handshake latency in production (most CDNs and load balancers expose TLS handshake time as a distinct metric from total request time) rather than assuming the textbook RTT multiplier holds on your real network paths.

**Mistakes, beginner → senior:** *beginner* — not realizing "HTTPS is on" says nothing about *which* TLS version or cipher suite is actually negotiated, and never checking; *intermediate* — enabling 0-RTT globally as a quick latency win without auditing which endpoints are replay-safe; *senior* — treating the handshake as a fixed, un-optimizable cost rather than recognizing that connection reuse (Day 11, concept 11) and session resumption both exist specifically to make you pay this cost as rarely as possible, and under-investing in whichever one your actual traffic pattern would benefit from most.

**Monitoring:** TLS handshake failure rate and reason (version mismatch, cipher mismatch, certificate errors — most load balancers and web servers expose this as a distinct metric or log field from generic connection failures); the *distribution* of negotiated TLS versions and cipher suites across real traffic, so a legacy client population you don't know about doesn't silently block a future TLS 1.0/1.1 deprecation; and 0-RTT usage and rejection rates specifically, if enabled, since a spike in rejected early data can be an early signal of exactly the replay probing Scenario 2 warns about.

**Scaling and cost:** the asymmetric operations in the handshake (signature verification, key exchange) are the CPU-expensive part of TLS, not the symmetric bulk encryption afterward — at very high connection-establishment rates (many short-lived connections, rather than fewer long-lived ones), handshake CPU cost genuinely shows up on a capacity-planning spreadsheet, which is the concrete, measurable reason connection reuse (Day 11) and session resumption are treated as performance features, not merely conveniences.

---

## 5. SNI: how one IP address serves a thousand certificates

**Depth: [WORKING]**

### Intuition

Look again at the `ClientHello` in concept 4's handshake diagram: it carries a `server_name` field, sent in the clear, before the server has committed to a certificate. That ordering answers a question that's easy to skip past: how does a server sitting behind one shared IP address — the normal situation for a CDN edge node, or any shared-hosting provider, serving thousands of unrelated customer domains — know *which* certificate to present, if the only information it has when the handshake begins is "a TLS connection arrived on this IP address"?

### The problem SNI solves, and why it's a genuinely hard chicken-and-egg without it

The information that would normally disambiguate this — the `Host` header, naming exactly which site the client wants — lives inside the HTTP request. And the HTTP request is application data, which under TLS is only ever sent *after* the handshake completes and a shared secret has been established (concept 4's whole `{}`/`[]` distinction). But which certificate to present is a decision the server has to make **during** the handshake, before any encrypted application data exists to read. Without some other mechanism, a server fielding connections for many domains on one IP has no way to know, at the moment it must choose a certificate, which of its domains the connecting client actually wants — this is Day 10's IPv4-scarcity problem colliding directly with concept 3's one-certificate-per-identity model.

Before SNI existed (introduced as an extension in 2003, current text in RFC 6066, January 2011), the working answers were both genuinely bad: **dedicate one IP address per certificate** — wasteful given Day 10 already established how scarce IPv4 addresses are, and a hard non-starter for a CDN edge node that might terminate TLS for tens of thousands of customer domains from one anycast address; or **issue a single certificate listing every hostname sharing that IP** in its Subject Alternative Name list — which not only requires reissuing the whole shared certificate every time a customer is added or removed, but structurally cannot work at all once the hostnames belong to unrelated third-party organizations, because no CA will validate a certificate naming a domain the requester doesn't control, and even where it could, it would expose the full list of every co-tenant's domain name to anyone who inspected any one certificate.

### How SNI actually works

The fix is disarmingly simple, exactly the kind of thing that looks obvious only after someone's pointed it out: the **client** already knows which hostname it's trying to reach — it's right there in the URL the user typed or the request the code issued. So make the client say so, as part of the `ClientHello`, in a small extension called `server_name` (RFC 6066), sent before any encryption keys exist. The server reads that cleartext field, looks up the matching certificate from whichever set it holds for that shared IP, and presents exactly that one — all within the same handshake, at no extra round-trip cost, because it's carried inside a message that was already being sent.

### Worked example — watching a server refuse to hand over a certificate at all without the right SNI

Here's real, actually-captured evidence of SNI mattering, from three `openssl s_client` invocations against the same host and port, differing only in what SNI value was sent:

```
$ echo | openssl s_client -connect example.com:443 -servername example.com 2>&1 | grep subject=
subject=CN=example.com, O=Zscaler Inc., OU=Zscaler Inc.

$ echo | openssl s_client -connect example.com:443 -noservername 2>&1 | tail -5
---
no peer certificate available
---
...error:0A000410:SSL routines:ssl3_read_bytes:ssl/tls alert handshake failure:...:SSL alert number 40

$ echo | openssl s_client -connect example.com:443 -servername totally-different-name.invalid 2>&1 | tail -5
---
no peer certificate available
---
...error:0A000410:SSL routines:ssl3_read_bytes:ssl/tls alert handshake failure:...:SSL alert number 40
```

**With the correct SNI, a certificate comes back and the handshake completes** (this trace is running through the same corporate TLS-inspecting proxy discussed in concept 3, which is why the certificate names Zscaler rather than a public CA — the SNI mechanism itself works identically regardless of who's terminating the connection). **With no SNI, or the wrong one, the far side never even reaches "no peer certificate available" — it responds with TLS alert 40, `handshake_failure`, and aborts before presenting anything at all.** That is the cleanest possible demonstration of the chicken-and-egg problem this section opened with: whatever's terminating TLS for this hostname genuinely cannot decide what to send back without first being told, in the clear, which name the client wants — exactly the mechanism, and exactly the failure mode, that motivated RFC 6066 in the first place. (Run this yourself against any CDN-fronted site and you'll typically see the same shape of result, though the specific failure — a generic default certificate, a connection reset, or an alert — varies by provider.)

### Analogy: an office building's directory, and the one thing it leaks that a directory doesn't

Many businesses can share one street address. What lets a courier reach the right one isn't magic — you state which company you're visiting at the front desk before the elevator will take you anywhere, and the front desk (the server) routes you straight to that tenant's floor (the matching certificate) without ever needing to show you the other tenants' doors. SNI is exactly that spoken statement, made to the "front desk" before anything private (the TLS session, the HTTP request behind it) begins.

**Where the analogy breaks, and it's the load-bearing part of this section:** a real office directory doesn't broadcast which company you asked for to everyone standing in the lobby. SNI does, structurally, in the base protocol — it's sent in the clear specifically *because* it has to be readable before any encryption key exists, which means **anyone who can see the wire (concept 1's eavesdropper, exactly) learns which hostname you're connecting to, even though TLS otherwise hides everything about the conversation that follows.** TLS 1.3 encrypts the certificate, the request, the response — but the one piece of metadata that survives in plaintext, by the base protocol's own design, is the name of the site you're visiting. That gap is real, it's been used in the wild for exactly the kind of network-level blocking concept 1 introduced as "tampering" and "impersonation" territory, and it's the reason the case study below exists.

### Under the hood, named rather than opened: Encrypted Client Hello (ECH)

TLS's answer to the SNI privacy gap is **Encrypted Client Hello (ECH)**: rather than sending the real `ClientHello` (with the real hostname) in the clear, the client wraps it inside an *outer* `ClientHello` that names only a generic, shared "public name" the network can see, and encrypts the real, hostname-bearing inner `ClientHello` using a public key the server published in advance — specifically via DNS, in the **`HTTPS`/`SVCB` record type Day 12 (concept 4) already introduced** as carrying "encrypted-client-hello configuration." The server decrypts the inner hello using its corresponding private key and proceeds with the real handshake from there, so an observer on the path sees only the shared public name, not which specific site behind it you reached. **Deliberate stop:** the encryption construction ECH uses (HPKE, Hybrid Public Key Encryption, itself another layered application of concept 2's ideas) and the full mechanics of how a client discovers and validates a server's ECH configuration are a black box this note leaves closed — treat ECH as "the fix exists, here's the shape of the problem it solves," per this concept's WORKING tier, and **verify current deployment status before relying on any adoption number** — ECH support across major browsers and CDNs has been rolling out steadily but unevenly, and this is one of the fastest-moving areas in the whole protocol.

### Case study — SNI-based blocking, and why ECH's motivation is documented, not hypothetical

Because SNI leaks the hostname in the clear, a network positioned to observe (or actively interfere with) traffic can selectively block access to a *specific* HTTPS site without needing to decrypt anything at all — it only has to watch for a particular name in the `server_name` field and drop or reset the connection. This is not a hypothetical threat model; it's the explicitly stated motivation behind ECH's real-world deployment push. Cloudflare's own public engineering writing (published under a title along the lines of *"Encrypt it or lose it: how Encrypted Client Hello (ECH) can defend against SNI-based censorship"*) and Mozilla's Firefox engineering announcements about rolling out ECH both name plaintext SNI inspection as an active, observed tool used by network operators and state-level censors to block specific HTTPS destinations selectively, while leaving the rest of HTTPS traffic (to sites they don't want to block) untouched. **The engineering lesson:** SNI is a genuine two-sided mechanism — the exact cleartext field that makes shared hosting and CDN deployment practical (this section's opening problem) is, for the identical reason, the exact cleartext field that makes selective, content-aware blocking of individual HTTPS sites possible without breaking HTTPS as a whole. Closing that gap (ECH) and keeping the functionality it enables (efficient shared TLS termination) turns out to require the extra machinery this section named but didn't open. *Primary sources:* Cloudflare's and Mozilla's own public ECH engineering blog posts (search each vendor's engineering blog directly, since exact URLs and post titles shift over time); RFC 6066 for SNI itself. **Verify current** which browsers and CDNs have ECH enabled by default versus behind a flag — this list has been actively growing and is exactly the kind of fact that stales within months of being written down.

---

## 6. HTTP/1.1: persistent connections and the pipelining dead end

**Depth: [WORKING]**

### Intuition

Concept 4 just quantified how expensive establishing a secure connection is — a real, measurable cost in round trips, not a rounding error. That cost sharpens a question Day 11 (concept 10) already raised in passing: given that opening a connection is expensive, how many times per page load do you actually have to pay it, and how much use do you squeeze out of one connection once it's open? Day 11 already told you HTTP/1.1's headline limitation — exactly one request may be outstanding on a connection at a time — and named the six-connections-per-origin browser workaround it forced. This section covers what Day 11 didn't: the HTTP-level mechanics of connection reuse itself, and the specific, instructive way HTTP/1.1's own attempted fix for its own limitation — **pipelining** — failed, for reasons distinct from (and additional to) the TCP-level story Day 11 told.

### Persistent connections: paying the handshake cost once, not per resource

The earliest version of HTTP (1.0) closed the underlying TCP connection after every single response — a full TCP handshake (and, once HTTPS became standard, a full TLS handshake on top of it) for every image, script, and stylesheet a page needed, one after another. Given what concept 4 just quantified that cost as, this is easy to see as wasteful in hindsight, and HTTP/1.1 made the obvious fix the default: unless either side explicitly sends `Connection: close`, the underlying TCP connection stays open after a response completes, ready to carry the next request without paying the connection-setup cost again. This is what "keep-alive" means at the protocol level — not a special feature you opt into, but HTTP/1.1's default behavior, which HTTP/1.0 required an explicit, non-standard `Connection: keep-alive` header to approximate at all.

### The rule that survives persistence, and why it's genuinely different from Day 11's story

A persistent connection removes the *repeated handshake* cost, but it does not remove HTTP/1.1's one-request-at-a-time rule, and the reason why is worth being precise about, because it's a different mechanism from the TCP-buffer-ordering story Day 11 told. **An HTTP/1.1 response carries no identifier tying it back to which request it answers.** A status line reads `HTTP/1.1 200 OK` — nothing more — so the *only* way a client can know which response belongs to which request, on a connection carrying more than one, is strict positional ordering: the first response received must be the answer to the first request sent, full stop. There is no field to check, no way to reorder, no way to say "this response is for request #3." That absence of any correlation identifier is the direct, protocol-level reason true concurrency was never possible on an HTTP/1.1 connection without inventing one — and inventing one is exactly what HTTP/2's stream IDs, the next concept, are for.

### Pipelining: the fix HTTP/1.1 actually shipped, and why nobody uses it

HTTP/1.1 did define a partial answer, called **pipelining**: a client is permitted to send several requests back-to-back on one connection without waiting for each response first, so more than one request can be in flight on the wire at once. On paper this sounds like it should have been HTTP/2 a decade early. It failed to see real adoption, for two compounding reasons worth knowing by name because the pattern recurs:

1. **The ordering rule above still applies.** Pipelined responses must still come back in the exact order the requests were sent, so a slow response at the front of the queue blocks every already-sent request behind it from being delivered, even though their answers might already be sitting ready on the server. This is head-of-line blocking again, but now it's an *HTTP-protocol* rule, not the TCP-buffer mechanism Day 11 walked through — a distinct mechanism producing a similar symptom, and worth not conflating with Day 11's version.
2. **The deployed internet didn't cooperate.** A meaningful fraction of real-world proxies, load balancers, and other intermediate boxes already deployed across the internet had bugs handling pipelined requests correctly — misordering responses, or mishandling connection reuse in ways that silently corrupted which response went to which client. Browser vendors, burned by exactly this kind of interoperability breakage in the wild, largely never enabled pipelining by default, and it saw negligible real-world use before being superseded entirely by HTTP/2's actual multiplexing.

**The lesson worth extracting, because it's a repeat of something Day 12 already taught in a different context:** pipelining was a technically valid, standards-compliant answer to a real problem, and it lost anyway — not on technical merit, but because **a change that depends on every intermediate box on the existing internet behaving correctly is, in practice, undeployable at internet scale**, regardless of what the specification says those boxes are supposed to do. That is the identical shape of lesson Day 12 drew about DNS alternatives losing to DNS on deployability rather than technical elegance, and it will recur, in a stronger and more deliberate form, in concept 8's discussion of why QUIC runs over UDP rather than attempting to fix TCP directly (Day 11, concept 13).

**Deliberate stop:** the exact wire grammar of an HTTP/1.1 message — the request-line and status-line syntax, header folding rules, and chunked transfer encoding — is Day 13's material, not this note's; this section names the `Connection` header and the ordering rule specifically because they're what the concept's own point (persistent-but-serial connections, and why fixing that needed a new protocol) depends on, and stops there.

---

## 7. HTTP/2: binary framing, multiplexed streams, and HPACK

**Depth: [CORE]**

### Intuition

Concept 6 identified the exact, precise reason HTTP/1.1 cannot multiplex requests over one connection: a response is a bare status line with no identifier connecting it back to a specific request, so the only way to know which response answers which request is strict positional order. Day 11 (concept 10) already told you what browsers did about it at the *connection* level — open roughly six parallel TCP (and, once HTTPS became the norm, six parallel TLS) connections per origin, so six requests could genuinely be in flight simultaneously, at the cost of six separate handshakes, six separate congestion-control ramp-ups, and six times the server-side connection bookkeeping. **HTTP/2's core idea is to fix the actual root cause instead of working around it: give every request and response an explicit identifier, so many of them can be freely interleaved over a single connection, and the ordering ambiguity that forced one-at-a-time semantics simply stops existing.** (And per concept 4's `alpn` extension, the two sides agree to actually speak HTTP/2 at all inside the same TLS handshake that secures the connection — no separate negotiation step, no extra round trip.)

### Under the hood: frames, streams, and why interleaving is now safe

HTTP/2 (RFC 9113, June 2022, obsoleting the original RFC 7540) replaces HTTP/1.1's plain-text, line-based message format with a **binary framing layer**. Every unit of communication is a **frame**: a fixed 9-byte header (a 24-bit length, an 8-bit type, an 8-bit flags field, and a 31-bit **stream identifier**) followed by a type-specific payload. That stream identifier is the entire fix from concept 6, made concrete: **every frame says, explicitly, which logical request/response exchange it belongs to**, so frames from many different exchanges can be freely interleaved on the wire in any order, and each side simply sorts incoming frames back into their respective streams by reading that one field.

A **stream** is a bidirectional, independent sequence of frames sharing one stream ID — client-initiated streams use odd IDs, server-initiated ones (server push, discussed below) use even IDs — and, critically, a stream is a lightweight, in-protocol concept, not a new connection: opening stream 47 costs nothing resembling a TCP or TLS handshake, it's simply the next HEADERS frame carrying a new ID. The frame types worth knowing by name, because two of them come back later in this section: **`HEADERS`** (carries a request's or response's compressed header block), **`DATA`** (body bytes), **`SETTINGS`** (connection-wide parameters, negotiated once, such as the maximum number of concurrent streams a peer will accept), **`WINDOW_UPDATE`** (per-stream and per-connection flow control — the same receive-window idea Day 11, concept 6, already built, just applied at two granularities instead of one), **`RST_STREAM`** (cancel a single stream immediately, without touching the connection or any other stream on it — an efficient, ordinary, and entirely legitimate part of the protocol, whose abuse is this concept's first case study), and **`GOAWAY`** (a graceful shutdown signal naming the highest stream ID the sender will still process, so a connection can be retired cleanly without silently dropping in-flight requests).

### Worked example — two requests, genuinely interleaved on one connection

Concept 6 established that this is *impossible* on HTTP/1.1. Here it is happening, traced frame by frame, for two requests — `GET /page.html` (assigned stream 1) and `GET /style.css` (assigned stream 3, since client stream IDs increment by 2, reserving even numbers for server-initiated streams) — issued back to back on one HTTP/2 connection:

```
Frame 1:  HEADERS   stream=1   ":method: GET, :path: /page.html, ..."
Frame 2:  HEADERS   stream=3   ":method: GET, :path: /style.css, ..."
                                 ← BOTH requests are already fully sent — no
                                    waiting for page.html's response first.
Frame 3:  DATA      stream=1   (first chunk of page.html's response body)
Frame 4:  DATA      stream=3   (style.css's ENTIRE body — it's small; END_STREAM set)
Frame 5:  DATA      stream=1   (rest of page.html's response body; END_STREAM set)
```

Two things to notice, both direct payoffs of the stream ID: **stream 3's response (the small CSS file) completes entirely — frame 4 carries its END_STREAM flag — before stream 1's larger response finishes**, something strictly impossible under HTTP/1.1's ordering rule, where the *first-sent* request's response must be the *first-received* one. And the interleaving at frame 3/4 is exactly what "multiplexing" concretely means at the wire level: bytes belonging to two logically distinct exchanges, physically interspersed on one TCP connection, sorted back into their respective streams purely by reading each frame's stream ID. **This is HTTP-layer head-of-line blocking, solved** — Day 11 (concept 10) already told you the sequel to this exact story: because every one of these streams still shares a single underlying TCP connection, a single lost *packet* anywhere in that shared stream still stalls every stream multiplexed onto it, at the transport layer, which is precisely the "HTTP/2 could be slower than HTTP/1.1 on lossy networks" finding Day 11 attributed to Google's own measurements, and precisely the motivation concept 8 picks back up.

### HPACK: compressing headers across a connection's whole lifetime

Interleaving frames solves ordering, but it doesn't address a separate, real cost: HTTP headers are extremely repetitive. A browser sends nearly identical `User-Agent`, `Accept-Encoding`, `Cookie`, and `Authorization` headers on *every single request* to the same origin — often several hundred bytes, repeated verbatim, dozens or hundreds of times over the life of one connection, for information that never changes request to request. **HPACK** (RFC 7541) is HTTP/2's answer: a compression scheme purpose-built for this specific repetitiveness, using three mechanisms together. A **static table** of 61 predefined common header name/value pairs (`:method: GET`, `:scheme: https`, and similar) that never need to be transmitted at all — just referenced by a small fixed index both sides already agree on. A **dynamic table**, unique to each connection, that both sides build up incrementally: the first time a header (say, a specific `Cookie` value) is sent, it's transmitted as a literal and *added* to the dynamic table; every subsequent request on that same connection can reference it by a short index instead of resending the literal bytes. And **Huffman coding** of literal string values that do need to be sent, for further byte savings on top of the table mechanism.

**Worked example — the actual byte savings, made concrete.** Suppose a browser sends three requests over one HTTP/2 connection, and the `Cookie` and `User-Agent` headers together run to roughly 900 bytes, identical on every request (a realistic size once session cookies, tracking cookies, and a modern browser's user-agent string are combined):

```
WITHOUT header compression (HTTP/1.1, three separate requests):
  Request 1:  ~900 bytes of Cookie + User-Agent, sent as literal text
  Request 2:  ~900 bytes of Cookie + User-Agent, sent again, byte-for-byte identical
  Request 3:  ~900 bytes of Cookie + User-Agent, sent again, byte-for-byte identical
  TOTAL:      ~2,700 bytes spent on headers that never changed

WITH HPACK, on one HTTP/2 connection:
  Request 1:  ~900 bytes sent as literals, AND added to the dynamic table
  Request 2:  ~2–3 bytes: "same as dynamic table entry #62" (an index reference)
  Request 3:  ~2–3 bytes: same index reference again
  TOTAL:      ~906 bytes  — roughly a 66% reduction on just three requests,
              and the saving APPROACHES 100% of the repeated cost as more
              requests share the same connection.
```

**Why this works, and the subtlety that matters for what comes next:** the dynamic table is not a fixed lookup — it's built incrementally, in a specific sequence, by both sides watching the same stream of HEADERS frames go by. **The decoder must apply every table insertion in exactly the order the encoder made it, because a later header block can reference an index that was only added a few requests ago** — if entry #62 means "the 62nd thing added to this table so far," both sides have to agree, unambiguously, on that ordering, or every subsequent index reference decodes to the wrong header entirely, silently and without any error signal. **That ordering requirement is fine on HTTP/2, because HTTP/2 still runs over one ordered TCP byte stream (Day 11) — TCP already guarantees the HEADERS frames from every stream arrive at the decoder in the exact order they were sent, even though the streams themselves are logically independent.** Hold onto that dependency; concept 8 is built around exactly the moment it stops being free.

### Analogy: a shared customs checkpoint notepad, and the one rule it can't survive breaking

Picture a border checkpoint processing a tour group. The first traveler fills out a full form — name, passport number, nationality, tour operator. The officer notices later travelers in the same group share several of those fields exactly, and starts a shared notepad: from then on, a traveler can simply say "same as entry 12 for nationality and tour operator" instead of rewriting them, and the officer appends any genuinely new information to the notepad as it comes in. That's the dynamic table, doing exactly what it does in HPACK.

**Where the analogy breaks, and it's the one that matters most for this whole note's remaining sections:** a real customs officer, and a real paper notepad, can survive being read slightly out of order — a human can flip back a page and figure it out. HPACK's dynamic table cannot: "entry 12" is only entry 12 if every single earlier insertion was processed, by both sides, in the *exact same sequence*. If two travelers' forms arrived at the notepad in a different order than they were actually filled out, "entry 12" would refer to two different things on each side, and every reference to it from that point forward decodes silently into someone else's answers with no error raised. **HPACK's compression efficiency is bought at the cost of a strict, connection-wide ordering dependency across every stream sharing that connection** — a cost HTTP/2 pays for free because TCP already serializes everything for it. The moment you consider running this same idea over a transport that deliberately does *not* serialize independent streams against each other — which is precisely QUIC's whole design point, Day 11 concept 13 — that free ordering guarantee disappears, and concept 8 opens with exactly that problem.

### System design — two decisions HTTP/2 forces you to revisit

**Scenario 1 — should a site abandon domain sharding when it adopts HTTP/2?**

**Problem:** a page built for HTTP/1.1's six-connections-per-origin ceiling (Day 11, concept 10) serves its 90 images, scripts, and stylesheets from four fake subdomains (`img1.example.com`, `img2.example.com`, `static1.example.com`, `static2.example.com`, all pointing at the same servers) specifically to trick browsers into opening 6 connections *per subdomain* — 24 parallel connections total, rather than 6. The site is migrating to HTTP/2. Should the sharding stay?

**Alternatives:** (1) keep the existing sharded subdomains unchanged; (2) collapse everything back onto a single origin.

**The decision: (2) — remove the sharding.**

**The actual reason:** domain sharding's entire value proposition was multiplying the number of *connections* available, because HTTP/1.1 connections were the scarce, serializing resource. HTTP/2 multiplexing makes a single connection capable of carrying effectively unlimited concurrent streams (bounded only by the `SETTINGS_MAX_CONCURRENT_STREAMS` value each side advertises, typically in the hundreds), so the scarce resource sharding was optimizing for no longer exists — and worse, sharding now actively works against two things HTTP/2 specifically improved: each shard is a *separate* TLS connection, meaning a separately-paid handshake cost (concept 4) and a separate, cold TCP congestion window (Day 11, concept 7) instead of one connection that's already warmed up, and — directly following from this concept's own HPACK section — **HPACK's dynamic table is per-connection**, so splitting requests across four sharded origins means four separate, independently-building compression contexts instead of one, throwing away exactly the compounding header-compression benefit the worked example above demonstrated.

**The trade-off, honestly:** collapsing sharding is close to a pure win once every client reliably speaks HTTP/2, but "reliably" is doing real work in that sentence — a client population that includes a meaningful fraction of HTTP/1.1-only clients (old proxies, certain enterprise network appliances, unusually old software) loses sharding's benefit for exactly that fraction, since HTTP/1.1 clients still hit the six-connections-per-origin ceiling and now have fewer origins to spread across than before.

**Flip condition:** keep sharding only for as long as a *measured*, non-trivial share of real traffic is confirmed still negotiating HTTP/1.1 rather than HTTP/2 — check your own access logs' protocol field rather than assuming; once that share is negligible, sharding is pure downside.

**Scenario 2 — should an API gateway speak HTTP/2 to its backend services, or keep backend connections on HTTP/1.1 with a connection pool?**

**Problem:** a reverse proxy or API gateway terminates HTTP/2 (or HTTP/3) from the public internet, and must decide what protocol to use for the *second* hop, from itself to a fleet of backend application servers inside the private network.

**Alternatives:** (1) speak HTTP/2 end-to-end, multiplexing many client requests onto a small number of persistent HTTP/2 connections per backend instance; (2) terminate HTTP/2 at the gateway and speak HTTP/1.1, with a conventional connection pool (Day 11, concept 11), to each backend.

**The decision:** (2) remains extremely common and often correct, *unless* the backend framework is specifically built to handle HTTP/2's concurrency model well (this is precisely why gRPC, which is built directly on HTTP/2 framing, requires backend frameworks with genuine async, non-blocking request handling).

**The actual reason:** HTTP/2 multiplexing only delivers a real benefit if the *application* behind it can actually service many concurrent streams independently. A backend built around a traditional one-thread-(or-one-process)-per-connection model gains little from receiving many multiplexed streams over one HTTP/2 connection — if the framework processes requests on that connection serially regardless, you've reintroduced HTTP/1.1's exact head-of-line-blocking problem, just relocated from the wire protocol into the application's own request-handling loop, while adding HTTP/2's framing and HPACK-state complexity for no offsetting benefit. A connection *pool* of ordinary HTTP/1.1 connections (Day 11, concept 11), sized to match the backend's real concurrency capacity, is simpler to reason about, simpler to debug (every request maps to one whole connection, which is exactly the debuggability Day 11 flagged as a genuine QUIC cost, now showing up as a genuine HTTP/2-to-backend cost too), and loses nothing the backend wasn't going to bottleneck on anyway.

**The trade-off, honestly:** HTTP/2-to-backend genuinely does pay off, and pays off well, once the backend is built for it — an async framework (this stack's own FastAPI, running under an ASGI server with genuine concurrent request handling, is a reasonable example of the shape required) can use a single warm HTTP/2 connection per backend instance to serve many concurrent requests without paying a new connection's TCP+TLS setup cost (concept 4) for each one, which is a real win at high request rates. The cost of getting there is architectural, not incidental: you need to know, concretely, that your backend's request-handling model can actually exploit the concurrency HTTP/2 offers, not just accept the protocol.

**Flip condition:** default to (2) until you've specifically measured that backend connection-establishment overhead (not backend processing time) is a meaningful fraction of your total latency budget, or until you're deliberately adopting gRPC, which makes the choice for you by requiring HTTP/2 as its transport.

**Failure modes, for both scenarios:** assuming "we enabled HTTP/2" automatically delivers its benefits without checking whether the client population or the backend framework can actually exploit multiplexing; and — the scenario 2 failure specifically — deploying HTTP/2 to a synchronous backend and being surprised that latency didn't improve, because the bottleneck moved from the wire to the application's own concurrency model, which is invisible in a wire-protocol-level metrics dashboard.

### Case studies

**HTTP/2 Rapid Reset (October 2023) — a legitimate protocol feature turned into the largest DDoS attacks on record at the time.** **CVE-2023-44487**, disclosed jointly on 10 October 2023 by Google, Amazon Web Services, and Cloudflare, described an attack technique — dubbed **Rapid Reset** — that exploited exactly the `RST_STREAM` frame this section introduced above as an ordinary, legitimate part of the protocol: the ability to cancel one stream instantly without disturbing the rest of the connection. The attack: open a stream with a request, and *immediately* send `RST_STREAM` to cancel it, before the server has finished (or in some implementations, before it has even started) processing the request — then repeat this thousands of times per second, on one connection. Because each stream is cancelled essentially instantly, it never counts against a server's `SETTINGS_MAX_CONCURRENT_STREAMS` limit for more than a fleeting moment, letting an attacker force the server to *begin* an enormous amount of per-request work (routing, potentially invoking backend logic) far faster than any concurrent-stream limit or conventional per-connection rate limit was designed to catch, since those limits were built assuming a stream that finishes counts as "done" and one that's still open counts against a cap — never anticipating a stream that's opened and cancelled thousands of times per second **precisely to stay under both counters simultaneously**. Cloudflare and Google both publicly reported the resulting attacks as the largest volumetric DDoS attacks they had observed at the time, measured in the hundreds of millions of requests per second. **The engineering lesson:** a defensive mechanism built around counting *completed* or *concurrently open* units of work has a blind spot for work that is cheap to start and cheap to cancel — the cost that matters (the server-side processing `RST_STREAM` triggers before the cancellation is honored) was never the thing being metered. The fix, rolled out across major implementations within days of disclosure, involved rate-limiting stream resets themselves and, more fundamentally, counting a stream against concurrency limits based on requests *received* rather than requests *currently open*, closing the exact gap the counting model had. *Primary sources:* CVE-2023-44487; Cloudflare's engineering blog, *"HTTP/2 Rapid Reset: deconstructing the record-breaking attack"*; Google Cloud's and AWS's own disclosure posts, all published 10 October 2023. **Verify current** exact peak-traffic figures if citing them precisely — different providers reported different numbers from their own vantage points, and both have been re-cited with minor variation across secondary sources since.

**HTTP/2 Server Push's deprecation (2022) — a real, shipped feature that browsers removed after production experience showed it usually didn't help.** HTTP/2 originally defined **server push**, via the `PUSH_PROMISE` frame — a mechanism letting a server proactively send resources (say, a page's CSS and JS) to a client *before* it explicitly asked for them, on the theory that the server, knowing what a given HTML page references, could save the client a full request/response round trip for each pushed asset. In practice, production experience across the industry found the opposite far too often: servers routinely pushed resources the client's browser cache *already held* from a previous visit, wasting bandwidth rather than saving latency, and the complexity of getting push heuristics right in a way that reliably helped rather than hurt proved difficult to realize broadly. Chrome announced its intent to remove server push support starting with Chrome 106 (2022), and other major browsers followed a similar trajectory, effectively retiring a feature that had shipped, been standardized, and been implemented across the ecosystem. **The engineering lesson:** a mechanism can be well-specified, correctly implemented, and still fail in production for a reason that only becomes visible at scale — here, that the server's guess about what the client needs is frequently wrong in a way a client's own cache state trivially would have prevented, and there was no cheap way for the server to know that cache state in advance. **Verify current** exact browser support status for server push before relying on it existing at all — this is a feature actively being removed, not merely deprecated on paper, and the timeline differs slightly by browser. *Primary source:* the Chrome team's own "Intent to Remove: HTTP/2 server push" announcement on the Chromium project's `blink-dev` mailing list, and Chrome Platform Status entries tracking the removal.

### In production — HTTP/2 operationally

**Best practices:** enable HTTP/2 at the edge/load balancer as close to a default as your infrastructure allows, since the client-facing win (multiplexing over one connection, HPACK compression) requires no application code changes; audit and remove domain sharding as part of that migration, per Scenario 1 above, rather than leaving it as accidental legacy; and size `SETTINGS_MAX_CONCURRENT_STREAMS` deliberately rather than accepting a default, informed by the Rapid Reset case study's lesson that a stream-count limit alone is not a complete defense.

**Mistakes, beginner → senior:** *beginner* — assuming "we're on HTTP/2 now" automatically fixes performance without checking whether the client population or backend can actually exploit it; *intermediate* — leaving HTTP/1.1-era domain sharding in place after migrating, silently paying its costs (extra handshakes, fragmented HPACK state) with none of its original benefit; *senior* — under-provisioning stream-reset rate limiting, treating `RST_STREAM` as a purely benign protocol detail rather than a monitored resource, precisely the gap Rapid Reset exploited industry-wide.

**Monitoring:** stream-reset rate per connection, specifically, as its own metric (not folded into generic request-rate monitoring) — this is the direct, actionable lesson of the Rapid Reset case study; HPACK dynamic-table-related memory usage per connection at the server (a large number of long-lived HTTP/2 connections each maintaining their own dynamic table has a real, measurable memory cost); and the negotiated protocol version distribution across real client traffic, so a shift in your client population's capabilities doesn't go unnoticed.

**Scaling and cost:** HTTP/2 reduces the *number* of connections a server must maintain per client (one multiplexed connection versus up to six), which generally reduces server-side connection bookkeeping and memory at scale — offset partially by the per-connection HPACK dynamic table's memory cost and by the CPU cost of maintaining significantly more concurrent logical streams than an HTTP/1.1 server ever had to reason about on a single connection.

---

## 8. HTTP/3: QPACK, stream mapping, and TLS folded into QUIC

**Depth: [CORE]**

### Intuition

Concept 7 ended on a dependency worth restating precisely, because this entire concept exists to break it: **HPACK's dynamic table is safe to use only because every stream's HEADERS frames are delivered to the decoder in exactly the order they were sent — a guarantee HTTP/2 gets for free, because it runs over one ordered TCP byte stream (Day 11).** Day 11 (concept 13) already told you that QUIC's entire reason for existing is to give up exactly that guarantee *between* streams, on purpose — a lost packet carrying stream 5's data no longer stalls stream 7's, because QUIC tracks loss and retransmission per stream rather than for the connection as a whole. Put those two facts next to each other and a real problem falls out immediately: **if you ran HPACK unmodified over QUIC, a single lost packet carrying a dynamic-table update could stall every other stream's header *decoding*, even though QUIC just went to considerable trouble to deliver every one of those other streams' data perfectly fine.** You would have quietly smuggled connection-wide head-of-line blocking back in, through the one piece of the old design nobody thought to re-examine. This is not a hypothetical concern the standards process indulged out of caution — it's the concrete reason **HTTP/3 cannot simply reuse HPACK**, and it's the first of two places where "put HTTP/2 on top of QUIC" turns out to require real new engineering rather than a routine port.

### QPACK: decoupling compression state from the ordering QUIC deliberately gave up

**QPACK** (RFC 9204) solves this by physically separating two things HPACK bundled together: the *data* that updates the shared compression dictionary, and the *headers* that reference it. QPACK dedicates two special, unidirectional QUIC streams — an encoder stream and a decoder stream — to carry dynamic-table insertions and the acknowledgments confirming they've arrived, entirely separately from the ordinary bidirectional request streams that carry HEADERS and DATA frames. A HEADERS block on a request stream can reference a dynamic-table entry, but only one it can *prove* has already been safely delivered (via an explicit "required insert count" the encoder attaches, which the decoder can check against what it's actually received on the encoder stream) — and if a needed entry hasn't arrived yet, the encoder has an explicit choice: **either encode that particular header as a literal instead of a table reference (skipping compression on just that one field, but never blocking), or accept that this specific stream's headers will wait for the missing table update to arrive.** That choice is made deliberately, per header, per request — not forced implicitly by however each stream's packets happen to be lost or reordered, the way HPACK's silent ordering dependency would have forced it. **The core trade concept 7's analogy flagged — HPACK's compression efficiency bought with a strict ordering dependency — is now an explicit dial an implementation can tune, rather than a hidden assumption**, and the whole reason it needs its own RFC and its own two extra streams is that "genuinely independent streams" (QUIC's entire point, Day 11 concept 13) and "a shared, strictly-ordered compression dictionary" (HPACK's entire mechanism, concept 7) are not compatible without exactly this kind of deliberate decoupling. **Deliberate stop:** the precise instruction set QPACK uses on its encoder/decoder streams (`Insert With Name Reference`, `Insert With Literal Name`, `Duplicate`, and the exact required-insert-count arithmetic) is implementation detail this note leaves closed at CORE-appropriate depth for a *user* of the protocol; the mechanism worth carrying is the shape of the fix — separate the ordering-sensitive channel from the streams that must never block on it — not its bit-level encoding.

### Under the hood: how an HTTP/3 request actually maps onto QUIC, and how TLS rides inside it

The second piece worth making concrete, now that concept 4 has built the full TLS 1.3 message flow from scratch, is exactly how those messages physically travel once QUIC is the transport. Day 11 (concept 13) told you, correctly, that "TLS is integrated into QUIC, not layered on top of it" — here is precisely what that means, tied to the exact messages concept 4's diagram named. **RFC 9001, "Using TLS to Secure QUIC," specifies that TLS 1.3's own handshake messages — the very same `ClientHello`, `ServerHello`, `Certificate`, `CertificateVerify`, and `Finished` messages from concept 4's diagram, unchanged — are carried inside QUIC `CRYPTO` frames, packaged into QUIC's `Initial` and `Handshake` packet types, from the very first UDP datagram the client sends.** There is no separate transport handshake happening first and a TLS handshake riding on top of it afterward, the way TCP+TLS necessarily works (concept 4's own table shows this exactly: TCP pays 1 RTT, then TLS 1.3 pays 1 more, sequentially) — QUIC's *own* connection establishment and TLS's key exchange are the same exchange, which is precisely why concept 4's table showed a fresh QUIC connection matching TCP+TLS-1.3's total setup cost (2 RTT → 1 RTT) despite negotiating the identical TLS 1.3 handshake underneath: it simply never pays for a transport handshake as a separate, sequential step first.

Once that shared TLS state establishes a connection, an individual HTTP/3 request/response exchange maps onto **one bidirectional QUIC stream** — directly analogous to an HTTP/2 stream conceptually (an independent, bidirectional sequence carrying that exchange's HEADERS and DATA), but now genuinely independent at the *transport* level too, not merely multiplexed via frame interleaving over one shared, strictly-ordered TCP byte stream. This is the concrete mechanism behind Day 11 (concept 13)'s measured "roughly a 4× reduction in loss-induced latency" finding: a packet loss affecting one request's QUIC stream triggers loss recovery scoped to that stream alone (Day 11's own mechanism, not re-derived here), while every other concurrent request's stream proceeds unaffected — the exact property concept 7 showed HTTP/2 could *not* deliver, because HTTP/2's streams, however independent logically, still share one TCP connection's single ordered byte stream and therefore one shared fate against packet loss.

### How a client even finds out a server offers HTTP/3 in the first place

One more mechanical piece worth being precise about, because it explains real, observable browser behavior: HTTP/3 needs UDP, a different protocol entirely from the TCP connection a browser's very first request to a brand-new host would normally use. **A server can't be reached over QUIC by a client that doesn't yet know to try it.** Two real mechanisms exist, both worth naming: a server can advertise support over an *existing* HTTP connection via the response header `Alt-Svc: h3=":443"` ("Alternative Services" — telling the client "next time, try HTTP/3 on this port"), which only helps on a client's *second* visit onward; or a server can publish support directly in DNS, via the same **`SVCB`/`HTTPS` record type Day 12 (concept 4) already introduced**, which can advertise ALPN protocol support (including `h3`) as part of the DNS answer itself — letting a client attempt QUIC on the very first connection, with no prior visit needed. Because a meaningful fraction of real-world networks — corporate firewalls, some mobile carriers, certain restrictive middleboxes — silently drop or heavily throttle UDP traffic entirely (a real, documented interoperability concern; RFC 9308, "Applicability of the QUIC Transport Protocol," discusses this explicitly as a known deployment reality, not an edge case), browsers that support HTTP/3 do not treat it as mandatory once advertised: they typically **race** a QUIC connection attempt against a conventional TCP+TLS fallback, using whichever completes — the same Happy-Eyeballs-shaped instinct Day 12 (concept 4) already named for IPv4-versus-IPv6 racing, applied here to protocol choice instead of address family.

### System design — two decisions HTTP/3 forces onto real operators

**Scenario 1 — should a CDN/edge enable HTTP/3 by default, given that some networks block UDP entirely?**

**Problem:** a CDN operator wants to offer every eligible client the latency benefits concept 4's table quantified (matching TLS 1.3's full-handshake cost on a client's very first connection, and beating it entirely on repeat visits). Some fraction of real client networks will silently fail to establish a QUIC connection at all.

**Alternatives:** (1) require HTTP/3 exclusively wherever advertised, refusing to fall back; (2) never enable HTTP/3, staying on the well-understood TCP+TLS+HTTP/2 stack; (3) advertise HTTP/3 opportunistically (`Alt-Svc` and/or a `SVCB`/`HTTPS` DNS record) and let clients race it against a TCP+TLS+HTTP/2 fallback, exactly as described above.

**The decision:** (3), essentially universally, and it's the design every major browser vendor and CDN has converged on independently.

**The actual reason:** option (1) would convert "some networks throttle UDP" (a real, RFC-acknowledged interoperability fact) directly into hard connection failures for exactly the users on those networks, in exchange for a latency win the *other* users get instead — an unacceptable trade given that the population affected is neither small nor predictable in advance. Option (2) forfeits a real, measured performance improvement for the (generally larger) population on networks that handle UDP just fine. The race in option (3) is a strict improvement over both: it costs at most one wasted, cheap connection attempt when QUIC fails, is completely invisible to a user in the success case, and degrades gracefully to exactly the well-understood TCP+TLS+HTTP/2 behavior this whole note already built for the population where UDP genuinely doesn't work.

**The trade-off, honestly:** running the race means genuinely maintaining and monitoring *two* full protocol stacks in production rather than one, and — as Day 11 (concept 13) already quantified in detail — QUIC's own per-connection CPU cost is measurably higher than a mature, kernel-optimized TCP+TLS stack's, so "enable it for everyone who can use it" is a real infrastructure cost, not a free toggle, at CDN scale.

**Flip condition:** an organization operating entirely within a network environment it fully controls (e.g., an internal-only service where every client's network path is known and verified not to block UDP) can reasonably skip the race and require HTTP/3 directly, since the uncertainty the race exists to hedge against doesn't apply — but this is a narrow condition, and it does not hold for anything serving the general public internet.

**Scenario 2 — should internal gRPC-based microservices adopt QUIC now, or stay on HTTP/2 over TCP?**

**Problem:** an infrastructure team already runs gRPC (built on HTTP/2, concept 7) for internal service-to-service calls, and is evaluating whether to move to a QUIC-based transport for the same traffic, chasing the connection-migration and per-stream-loss-isolation benefits this section and Day 11 both described.

**Alternatives:** (1) adopt QUIC-based internal transport now; (2) stay on HTTP/2 over TCP for internal traffic and revisit later.

**The decision:** (2), for most organizations, as of this writing — and this is exactly the kind of call that should be re-examined on a fixed schedule rather than decided once and forgotten, because the honest reason it currently favors (2) is about ecosystem maturity, which moves.

**The actual reason:** internal service-to-service traffic on a well-run private network is precisely the traffic *least* likely to suffer from the specific problems QUIC solves — Day 11 (concept 13)'s connection-migration benefit matters enormously for a mobile client roaming between cell towers and matters close to not at all for two servers in the same datacenter with stable IP addresses that never change mid-connection; and packet loss on a well-provisioned internal network is typically far lower than on the public internet, shrinking the loss-isolation benefit's practical payoff for exactly this traffic. Against that reduced upside, the *costs* are unusually concentrated for internal infrastructure specifically: **debugging tooling for internal services (packet captures, existing observability stacks, `ss`/`netstat`-based dashboards) is overwhelmingly built around TCP**, and QUIC's own anti-ossification encryption (Day 11, concept 13) that protects it from middlebox interference on the public internet is, for an internal debugging session, exactly the property that makes an engineer's own inspection of their own traffic harder, not easier — a cost with no offsetting benefit when there's no adversarial middlebox to defend against in the first place.

**The trade-off, honestly:** deferring adoption means forgoing whatever real gains QUIC does offer even internally (marginally faster reconnection after any network blip, somewhat better tail latency under loss), and it risks becoming a decision nobody revisits once made, even after the ecosystem (QUIC-aware observability tooling, gRPC-over-QUIC library maturity) has genuinely caught up.

**Flip condition:** revisit this specifically when your organization's internal network genuinely experiences connection churn or loss patterns resembling the public internet's (multi-region internal traffic over real WAN links, rather than same-datacenter calls, is the concrete trigger), or when your observability stack has demonstrably closed the QUIC-tooling gap — and **verify current QUIC/gRPC ecosystem maturity directly before relying on this note's framing**, since library and tooling support in this specific area has been improving quickly and this is exactly the kind of claim that ages fastest.

**Failure modes, both scenarios:** enabling HTTP/3 without actually implementing (or verifying) the fallback race, silently breaking the fraction of users on UDP-hostile networks; and adopting QUIC internally before confirming your actual observability stack can debug it, discovering the gap only during a live incident — precisely the "teams underestimate this until their first HTTP/3 incident" warning Day 11 (concept 13) already issued.

### Case studies

**The honest state of this concept's case-study material, stated plainly rather than papered over.** The natural, dramatic, real case study for *why QUIC exists and what its deployment at scale actually looks like* — Google's own production rollout, and the specific lossy-network measurement that justified building QUIC in the first place — already lives in **Day 11, concept 13** ("why Google built QUIC (and the measurement that justified it)"), and repeating it here would be exactly the restatement Principle 9 exists to prevent; go there for it, and this note cross-references it rather than re-presenting it. What's left that's genuinely new at *this* concept's layer — the HTTP-semantics and deployment-friction layer, rather than the transport-design layer Day 11 already covered — is more structural than dramatic, and it deserves to be named as what it honestly is rather than dressed up as a single company's outage story it isn't:

**Middlebox and network interference with QUIC, as a documented, structural deployment reality rather than a one-time incident.** Unlike TCP, which decades of internet middleboxes (firewalls, NATs, traffic-shaping appliances) have been built and tuned to expect, QUIC is comparatively new UDP traffic on port 443 — traffic many of those same middleboxes have no established expectations for, and some networks throttle, deprioritize, or drop outright, whether from explicit policy (some corporate and institutional networks block UDP broadly for unrelated reasons) or from equipment that simply doesn't recognize high-volume UDP on that port as legitimate. This isn't a single dated postmortem — it's an ongoing, acknowledged characteristic of the internet QUIC has to operate on, **explicitly documented in RFC 9308, "Applicability of the QUIC Transport Protocol,"** which discusses exactly this interference as a first-class deployment consideration rather than a hypothetical edge case, and it is the direct, primary reason every major browser and CDN implements the opportunistic-race-with-fallback behavior described above rather than ever treating HTTP/3 as a hard requirement. **The engineering lesson generalizes cleanly from Day 11's own QUIC-design story:** QUIC was built to route around *protocol* ossification (middleboxes that had learned to expect exactly TCP's behavior and broke when anything deviated from it, Day 11 concept 13) — and its own real-world rollout has run straight into a close cousin of that same problem one layer up, at the level of "networks that have opinions about what UDP on port 443 is allowed to do." *Primary source:* RFC 9308 itself, which is unusually candid, for an RFC, about documenting real deployment friction rather than only specifying the ideal case.

### In production — HTTP/3 operationally

**Best practices:** never deploy HTTP/3 as the *only* path to a service — always pair it with the TCP+TLS+HTTP/2 fallback and the race behavior described above, treating HTTP/3 as a latency optimization layered on top of a working baseline, not a replacement for one; advertise support via both `Alt-Svc` and a DNS `SVCB`/`HTTPS` record where your DNS provider supports it, since the DNS route is what lets a client benefit on its very first connection rather than only its second; and budget real infrastructure capacity for QUIC's measured higher per-connection CPU cost (Day 11, concept 13) rather than assuming it's a drop-in, cost-neutral swap for your existing TCP+TLS termination layer.

**Mistakes, beginner → senior:** *beginner* — assuming "the browser says it used HTTP/3" means every client will, without verifying the fallback path actually works for clients that can't; *intermediate* — enabling HTTP/3 broadly without provisioning observability for it, then having no diagnostic path during the first real incident that involves it; *senior* — under-forecasting the CPU/hardware cost of QUIC termination at scale, treating Day 11's documented overhead figures as historical rather than re-verifying them against current kernel offload support (`sendmsg`/GSO and similar), which has been closing the gap but at a pace worth re-checking rather than assuming.

**Monitoring:** the QUIC-versus-fallback success ratio in real traffic, broken out by client network where possible, as the primary signal that the race behavior is working as designed rather than silently failing over for a larger population than expected; QUIC-specific connection-establishment latency and 0-RTT usage/rejection rates, mirroring concept 4's own 0-RTT monitoring guidance but now for QUIC's version of the same mechanism; and QUIC-aware tooling (`qlog`, `qvis`, a QUIC-capable packet-capture workflow) in place *before* the first production incident involving it, not scrambled together during one.

**Scaling and cost:** QUIC termination at high connection-establishment rates carries a real, measurable CPU premium over an equivalent TCP+TLS stack (Day 11, concept 13, already quantified this at "roughly 2× per byte in early deployments," improving with kernel offload support — **verify current figures** against your own load-testing rather than relying on a number that will have moved by the time you read this); the DNS-based discovery path (`SVCB`/`HTTPS` records) costs nothing beyond ordinary DNS hosting (Day 12) and is the cheapest lever for improving first-visit HTTP/3 adoption, making it worth enabling even before the CPU-cost question is fully resolved for your own infrastructure.

---

# Topic-wide wrap-up

## Glossary

**AEAD (Authenticated Encryption with Associated Data)** — a symmetric cipher construction (e.g., AES-GCM, ChaCha20-Poly1305) that encrypts data and produces a tamper-detection tag in one pass; what TLS actually uses to protect application data after the handshake.

**ALPN (Application-Layer Protocol Negotiation)** — a TLS extension, sent in the `ClientHello`, letting client and server agree which application protocol (`http/1.1`, `h2`, `h3`) to use over the connection, settled in the same round trip that establishes encryption.

**Asymmetric (public-key) cryptography** — a scheme using a mathematically related key pair, a public key safe to share with anyone and a private key never shared, where deriving one from the other is computationally infeasible for well-chosen parameters.

**Certificate (X.509)** — a signed data structure binding a public key to an identity (a domain name, via the Subject Alternative Name extension), issued and signed by a certificate authority, verifiable by anyone holding the issuer's public key.

**Certificate Authority (CA)** — a trusted third party that verifies a claim of domain (or organization) control before signing a certificate; the anchor that makes public-key cryptography useful for proving identity, not just exchanging secrets.

**Certificate Transparency** — a public, append-only, cryptographically verifiable log of every certificate issued by a participating CA, created in direct response to the DigiNotar incident, so a fraudulently issued certificate becomes detectable rather than silently exploitable.

**Chain of trust** — the sequence of signatures from a leaf certificate up through one or more intermediate CAs to a root CA already present in a client's trust store; verification means walking this chain and checking every signature.

**Diffie-Hellman (key exchange)** — a method letting two parties compute an identical shared secret using values exchanged entirely in the clear, without ever transmitting the secret itself; the mechanism underlying TLS's key exchange.

**Discrete logarithm problem** — the mathematical problem of recovering a private exponent from a public value in a Diffie-Hellman-style exchange; believed computationally infeasible for well-chosen groups, and the security foundation of the key exchange.

**DV / OV / EV (certificate validation levels)** — Domain Validation (proves only domain control, fully automatable), Organization Validation (additionally checks legal identity, manual), and Extended Validation (strictest checks; its once-distinct browser UI treatment has been removed in most modern browsers).

**ECH (Encrypted Client Hello)** — a mechanism that encrypts the real `ClientHello` (including the real SNI value) inside an outer, unencrypted one naming only a generic public name, closing SNI's cleartext hostname leak; configuration is distributed via DNS `SVCB`/`HTTPS` records.

**Eavesdropping / tampering / impersonation** — the three named threats plain HTTP is exposed to: a passive party reading traffic, an active party modifying it in transit, and an active party pretending to be one of the legitimate endpoints.

**`Finished` (TLS message)** — a MAC computed over the entire handshake transcript, sent by both sides at the end of the handshake; verifying it fails if any earlier handshake message was tampered with, which is the structural fix for negotiation-downgrade attacks.

**Frame (HTTP/2 and HTTP/3)** — the basic unit of communication in HTTP/2's binary framing layer: a fixed header (including a stream identifier) plus a type-specific payload (`HEADERS`, `DATA`, `SETTINGS`, `RST_STREAM`, `WINDOW_UPDATE`, `GOAWAY`, and others).

**HPACK** — HTTP/2's header-compression scheme (RFC 7541), using a static table of common header pairs plus a per-connection dynamic table that both sides update in strict order as headers are seen.

**HSTS (HTTP Strict Transport Security)** — a response header (RFC 6797) instructing a browser to never issue a plain-HTTP request to a domain again for a specified duration, closing the window an `sslstrip`-style downgrade attack depends on.

**Hybrid encryption** — using slow asymmetric cryptography once, to establish a shared secret, then a fast symmetric cipher for all subsequent bulk data; the pattern underlying TLS's entire design.

**mTLS (mutual TLS)** — a TLS configuration in which both sides present and verify a certificate, so each party cryptographically proves its identity rather than relying on network position alone.

**OCSP / OCSP stapling** — Online Certificate Status Protocol, a live query asking a CA whether a specific certificate is still valid; stapling has the server periodically fetch and attach that proof itself, avoiding a separate client-side query and its associated privacy leak.

**QPACK** — HTTP/3's header-compression scheme (RFC 9204), functionally similar to HPACK but carrying dynamic-table updates on separate, dedicated streams so that one stream's packet loss can never stall another stream's header decoding.

**`server_name` (SNI, Server Name Indication)** — a `ClientHello` extension (RFC 6066) stating, in the clear, which hostname a client intends to reach, letting a server sharing one IP among many certificates choose the right one during the handshake itself.

**Session resumption / 0-RTT (early data)** — reusing cryptographic material from a prior TLS session to skip parts of a future handshake; 0-RTT specifically lets a client send encrypted application data in its very first flight, at the cost of that data being replayable by a network attacker, making it safe only for actions with no harmful effect if repeated.

**Stream (HTTP/2 and HTTP/3)** — an independent, bidirectional sequence of frames identified by a stream ID, carrying one logical request/response exchange; many streams multiplex over one connection.

**Symmetric cryptography** — a scheme using one shared key for both encryption and decryption; fast, but requires the key to already be securely distributed to both parties.

**TLS (Transport Layer Security)** — the protocol (current version 1.3, RFC 8446; successor to the deprecated SSL) that negotiates cryptographic parameters, performs a key exchange, verifies identity via certificates, and thereafter encrypts all application data on a connection.

**TLS handshake** — the message exchange, run once per connection before any application data is trusted, that accomplishes version/cipher negotiation, key exchange, and certificate verification together.

**Revocation (CRL/OCSP)** — the mechanism by which a CA declares a certificate untrustworthy before its natural expiry, most urgently needed after a private-key compromise; meaningfully weakened in practice by browsers' soft-fail behavior when a revocation check can't be completed.

## Cheat sheet

**The three threats, and TLS's three answers**
```
EAVESDROPPING  → confidentiality → symmetric encryption of application data
TAMPERING      → integrity       → AEAD's authentication tag + the Finished MAC
IMPERSONATION  → authentication  → certificates + the chain of trust
```

**Round-trip cost, at RTT = 100 ms (concept 4's worked table)**
| Setup | Setup RTTs | To 1st response byte |
|---|---|---|
| Plain HTTP over TCP | 1 (100 ms) | 2 (200 ms) |
| HTTPS, TLS 1.2 full | 3 (300 ms) | 4 (400 ms) |
| HTTPS, TLS 1.3 full | 2 (200 ms) | 3 (300 ms) |
| HTTPS, TLS 1.3, 0-RTT | 1 (100 ms) | 2 (200 ms) |
| HTTP/3 (QUIC), fresh | 1 (100 ms) | 2 (200 ms) |
| HTTP/3 (QUIC), 0-RTT | 0 (0 ms) | 1 (100 ms) |

**Certificate validation levels**
| Level | Proves | Automatable? |
|---|---|---|
| DV | domain control only | yes — ACME/Let's Encrypt |
| OV | + legal organization identity | no |
| EV | strictest identity checks | no — and browser UI no longer distinguishes it (verify current) |

**HTTP evolution, one line each**
```
HTTP/1.1  1 request in flight per connection; keep-alive is the default; pipelining
          exists on paper, unused in practice (Day 11 concept 10 / this note concept 6)
HTTP/2    binary framing + stream IDs → real multiplexing over 1 TCP connection;
          HPACK compresses repeated headers; still shares TCP's fate under packet loss
HTTP/3    same idea, over QUIC; needs QPACK (not HPACK) because streams no longer
          share TCP's free ordering guarantee; TLS 1.3 IS the transport handshake
```

**Diagnostics**
| Task | Command |
|---|---|
| See a site's real cert chain | `openssl s_client -connect host:443 -servername host` |
| Verify against a specific CA file | `openssl s_client -connect host:443 -CAfile ca.pem` |
| Test without checking the SNI-selected cert | `openssl s_client -connect host:443 -noservername` |
| Inspect a cert's fields | `openssl x509 -in cert.pem -noout -text` |
| Test a self-signed local server | `curl -k https://localhost:PORT/…` |
| Trust one specific self-signed cert only | `curl --cacert cert.pem https://localhost:PORT/…` |
| Check HSTS/security headers | `curl -I https://host/` |

**Windows-native notes**
```
openssl        → WSL2, git-bash, or `winget install OpenSSL` — NOT native PowerShell
git-bash -subj → MSYS path-mangling breaks "/CN=..."; prefix with MSYS_NO_PATHCONV=1
curl (Windows) → often Schannel-backed, not OpenSSL: expect "SEC_E_UNTRUSTED_ROOT",
                 not "SSL certificate problem: self-signed certificate" (that's the
                 OpenSSL-linked curl message you'll see on WSL2/Linux instead)
```

## Build this

### Runnable example — serve FastAPI over TLS with a self-signed cert, and see exactly why the browser complains

This reuses the Day 7 skeleton (`src/skeleton/app.py`) unmodified:

```python
# src/skeleton/app.py  (Day 7's skeleton, unchanged)
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI(title="skeleton", version="0.1.0")


class Health(BaseModel):
    status: str
    service: str


@app.get("/health")
async def health() -> Health:
    return Health(status="ok", service="skeleton")
```

**Step 1 — generate a self-signed certificate.** On WSL2 or git-bash:

```bash
# pip install fastapi "uvicorn[standard]"   (if not already installed)

# On git-bash / MSYS2 specifically: the leading "/" in "/CN=localhost" gets
# mangled into a Windows path by MSYS's automatic path conversion unless you
# disable it for this one command. WSL2 and native Linux don't need this prefix.
MSYS_NO_PATHCONV=1 openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem \
    -days 365 -nodes -subj "/CN=localhost"
```

This is a genuinely real, executed command — here is the certificate it actually produced, inspected with `openssl x509`:

```
$ openssl x509 -in cert.pem -noout -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 67:25:03:47:3a:bf:e1:de:f2:39:ed:57:ff:5d:6a:b2:1d:a5:bf:dd
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN=localhost
        Validity
            Not Before: Aug 24 11:14:47 2026 GMT
            Not After : Aug 24 11:14:47 2027 GMT
        Subject: CN=localhost
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
        X509v3 extensions:
            X509v3 Subject Key Identifier: BA:82:5F:56:A3:2C:7E:4D:E0:B4:B4:80:2C:9C:D9:94:38:A4:86:10
            X509v3 Authority Key Identifier: BA:82:5F:56:A3:2C:7E:4D:E0:B4:B4:80:2C:9C:D9:94:38:A4:86:10
            X509v3 Basic Constraints: critical
```

**Note `Issuer: CN=localhost` and `Subject: CN=localhost` are identical, and the Subject Key Identifier and Authority Key Identifier are the same value** — this certificate signed itself. There is no chain, no CA, nothing for a client to check it against except itself. That single fact is the entire reason for everything that happens next.

**Step 2 — serve it with uvicorn's `--ssl-keyfile`/`--ssl-certfile`:**

```bash
PYTHONPATH=src uvicorn skeleton.app:app --host 127.0.0.1 --port 8444 \
    --ssl-keyfile key.pem --ssl-certfile cert.pem
```

Real startup output:

```
INFO:     Started server process [9596]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on https://127.0.0.1:8444 (Press CTRL+C to quit)
```

**Step 3 — hit it with `curl`, three ways, and watch three different outcomes:**

```bash
# 3a. Ask curl to verify the certificate the normal way — it can't, and refuses to connect
$ curl -v https://localhost:8444/health
# -> * schannel: SEC_E_UNTRUSTED_ROOT (0x80090325) - The certificate chain was
# ->   issued by an authority that is not trusted.
# -> curl: (60) schannel: SEC_E_UNTRUSTED_ROOT (0x80090325) - The certificate
# ->   chain was issued by an authority that is not trusted.
# -> curl failed to verify the legitimacy of the server and therefore could not
# -> establish a secure connection to it.

# 3b. Tell curl to skip verification entirely (demo/local only — NEVER in production code)
$ curl -vk https://localhost:8444/health
# -> < HTTP/1.1 200 OK
# -> < server: uvicorn
# -> < content-type: application/json
# -> {"status":"ok","service":"skeleton"}

# 3c. The CORRECT fix: explicitly trust THIS ONE certificate, by name, instead of
#     disabling verification globally
$ curl -v --cacert cert.pem https://localhost:8444/health
# -> * schannel: connection hostname (localhost) validated against certificate name (localhost)
# -> < HTTP/1.1 200 OK
# -> {"status":"ok","service":"skeleton"}
```

**These are all real, executed outputs, captured while writing this note — on Windows, via git-bash's bundled `curl.exe`.** Note the exact wording: `SEC_E_UNTRUSTED_ROOT` is **Schannel's** error, because this particular `curl.exe` is linked against Windows' native TLS library rather than OpenSSL — if you run the equivalent command from WSL2 or a Linux box (where `curl` is typically OpenSSL-linked), expect a differently worded, functionally identical failure, commonly `curl: (60) SSL certificate problem: self-signed certificate` (**verify current wording on your own setup** — this is exactly the kind of detail that varies by curl build and version, which is itself the point: the underlying *reason* for the failure is identical regardless of which TLS library reports it).

**Step 4 — confirm the same fact with `openssl s_client` directly:**

```bash
$ echo | openssl s_client -connect localhost:8444 -CAfile cert.pem 2>&1 | tail -5
# -> depth=0 CN=localhost
# -> verify return:1
# -> Verify return code: 0 (ok)
```

`Verify return code: 0 (ok)` **only** because we explicitly told `openssl` to trust `cert.pem` with `-CAfile`. Drop that flag and rerun it, and you'll get exactly the same `unable to get local issuer certificate` result concept 3's real capture against `example.com` showed — because the underlying situation is identical: a certificate with no path to anything the client already trusts.

**Step 5 — do it in a browser, and explain the warning precisely.** Navigate to `https://localhost:8444/health`. You'll see a full-page warning rather than a small icon change — modern browsers treat an unverifiable certificate as a hard stop, not a suggestion. The exact wording differs by browser and **drifts across versions, so verify current text on your own install**, but as of recent Chrome and Firefox releases the shape is: Chrome shows *"Your connection is not private"* with the technical code `NET::ERR_CERT_AUTHORITY_INVALID`, and an "Advanced" disclosure revealing a "Proceed to localhost (unsafe)" link; Firefox shows *"Warning: Potential Security Risk Ahead"* with the code `SEC_ERROR_UNKNOWN_ISSUER`, and an "Accept the Risk and Continue" button.

**Why this works, and why the warning is correct, not a false alarm — tied directly back to concept 1 and concept 3:** the browser's TLS stack did everything right. It received a syntactically valid certificate, checked whether it could build a chain from that certificate up to any root in its trust store, and found none — because this certificate signed itself, and nothing signed it. The browser cannot distinguish "this is genuinely your own local test server" from "this is an attacker on your network presenting a self-signed certificate claiming to be `localhost`" — **those two situations are cryptographically indistinguishable from the outside**, which is exactly concept 1's impersonation threat and exactly why concept 3's entire chain-of-trust machinery exists. The warning isn't the browser being overcautious about a harmless test certificate; it's the browser correctly reporting that it has no basis whatsoever for believing this certificate's claim, because — unlike a certificate from a CA in its trust store — nobody independent vouched for it. `curl -k` and `curl --cacert` don't make the certificate "valid" in any deeper sense; they simply tell the client "I already know and accept this specific risk," which is a decision only a human (or an explicitly-configured piece of infrastructure that's deliberately choosing to trust one specific, known certificate) should ever make — never a default.

**Honesty caveats:**
1. `curl -k` and disabling certificate verification in code (e.g., `verify=False` in `requests`, or `ssl.CERT_NONE` in Python's `ssl` module) remove protection against **both** eavesdropping-adjacent risks concept 1 named: not just impersonation, but any active MITM tampering, since there's no longer any check that you're even talking to the same server across the whole session. Never ship this in production code, and never do it against anything but a host you provisioned yourself, on this run, for this test.
2. This exercise generates a **self-signed** certificate — the simplest, but not the most realistic, way to test locally. A closer-to-production alternative is running your own tiny local CA (`mkcert` is a well-known tool built exactly for this) and installing *that* CA's root into your OS/browser trust store once, so every certificate it subsequently issues for `localhost` or any `*.test` domain verifies cleanly with no warning and no `-k` flag — worth trying once you're comfortable with the manual version above.
3. Uvicorn's TLS support here is fine for local development and this exercise; a production deployment almost always terminates TLS at a dedicated reverse proxy or load balancer (concept 3's TLS-termination-placement System Design) rather than inside the application server process directly.

### Task checklist

- [ ] Run every command above yourself and confirm you get the same shape of output (exact serials/ports/timestamps will differ — that's expected).
- [ ] Run `openssl s_client -connect <a real site you use>:443 -servername <that host>` and identify: how many certificates in the chain, which is the leaf, what's the leaf's validity window, and — per concept 3's own captured surprise — check carefully whether the chain you get is really that site's own, or something a network device between you and it substituted.
- [ ] Deliberately break the demo: let `curl --cacert wrong-cert.pem` point at a *different* certificate than the one the server is actually presenting, and read the resulting error message closely enough to explain, in your own words, exactly what check failed.
- [ ] Open your browser's DevTools Network tab against any HTTPS site you use daily and find the negotiated protocol (`h2` or `h3`) for at least three different requests — note whether they differ, and if so, why (concept 8's opportunistic-race behavior is the likely answer for `h3`).
- [ ] Using concept 4's worked-example table, compute your own expected setup latency using your actual measured RTT to a server you use (`ping` or `curl -w "%{time_connect}\n"`), and compare it to what `curl -w "%{time_appconnect}\n"` reports for the real TLS handshake time on that connection.

**Definition of done:** you can point to the exact moment in a captured `openssl s_client` trace where a certificate's signature is being checked against a chain, and explain — without looking anything up — why a self-signed certificate fails that check while a CA-issued one (usually) doesn't.

## Active recall and self-test

Answer from memory, in writing, before checking back against the note.

1. Name the three threats plain HTTP is exposed to, and which TLS mechanism answers each one.
2. Why is Base64 encoding not encryption? What's the one-command proof?
3. Why can't symmetric cryptography alone solve the "two strangers, one hostile channel" problem?
4. Walk through the toy Diffie-Hellman example with your own small numbers (pick a different prime and generator) and confirm both sides land on the same shared secret.
5. What does a certificate actually prove, and — precisely — what does it *not* prove about a website?
6. Why are there usually three certificates in a chain (leaf, intermediate, root), not one?
7. What's the honest limitation of OCSP-based revocation checking in most browsers today?
8. Name the TLS 1.3 handshake messages in order, and say which one closes the door on the Logjam/POODLE-style downgrade-attack family.
9. Complete the table from memory: setup RTTs for plain HTTP, TLS 1.2 full, TLS 1.3 full, TLS 1.3 0-RTT, HTTP/3 fresh, HTTP/3 0-RTT.
10. Why is 0-RTT data replayable, and what's the one property of an operation that makes it safe to allow over 0-RTT anyway?
11. What problem does SNI solve, and what does it leak as an unavoidable side effect of solving it?
12. What is ALPN, and why does it mean HTTP/2 support is really a property of the TLS handshake?
13. Precisely why can't HTTP/1.1 multiplex requests over one connection, even a persistent one?
14. Why did HTTP/1.1 pipelining fail to see adoption, despite being standardized?
15. What does a stream ID actually solve that HTTP/1.1 couldn't?
16. Explain HPACK's dynamic table, and why its correctness depends on strict message ordering.
17. Why can't HTTP/3 just reuse HPACK unmodified? What does QPACK do differently?
18. In one sentence, what does RFC 9001 actually specify, and why does it mean QUIC's handshake beats TCP+TLS 1.3's on a first-ever connection?
19. What made the Rapid Reset attack effective against defenses built around concurrent-stream limits?
20. Explain the sslstrip attack, and precisely how HSTS closes it.

### 60-second teach-back

> **"HTTP puts everything on the wire in the clear, which exposes it to three distinct threats: eavesdropping, tampering, and impersonation. TLS fixes the first two with encryption — but encryption needs two strangers to agree on a secret over a channel an attacker can also read, which is what asymmetric key exchange solves: both sides compute an identical shared secret from public values, without ever transmitting it, then switch to fast symmetric encryption for everything else. That still leaves impersonation completely open, because a public key alone proves nothing about who holds it — anyone can generate one. Certificates fix that: a trusted third party, a certificate authority, signs a statement binding a public key to a name, and your browser checks that signature chains up to a root it already trusts. All of that — negotiating versions and ciphers, exchanging keys, checking the certificate chain — happens in one handshake, once per connection, and it costs real round trips: TLS 1.2 needed two extra round trips beyond the TCP handshake; TLS 1.3 collapsed that to one, by having the client guess and send its key share before being asked. Meanwhile HTTP itself hit a wall: HTTP/1.1 allows only one request in flight per connection, so browsers opened six connections per site as a workaround. HTTP/2 fixed the real cause by giving every request a stream ID, so many can multiplex over one connection — plus HPACK compresses the repetitive headers every request resends. But HTTP/2 still runs over one ordered TCP connection, so one lost packet stalls every stream sharing it. HTTP/3 fixes that by running over QUIC instead, where streams are independent all the way down at the transport level — which meant reinventing header compression too, since HPACK's compression table secretly depended on the very ordering guarantee QUIC gives up. And because QUIC folds the TLS 1.3 handshake into its own connection setup instead of running it after a separate transport handshake, an HTTP/3 connection reaches application data as fast as a *resumed* TLS-over-TCP connection does — on your very first visit."**

If you can deliver that, and then answer "why does the browser correctly refuse my self-signed localhost certificate, given that I generated it myself and know it's safe" — you have this topic.

## Spaced-repetition flashcards

| Q | A |
|---|---|
| The three named threats plain HTTP is exposed to | Eavesdropping, tampering, impersonation |
| What does Base64 protect against? | Nothing — it's an encoding, not encryption |
| What does asymmetric key exchange actually transmit? | Only public values — the shared secret itself is never sent |
| What does a certificate prove? | That a specific public key belongs to a specific name, per the issuing CA |
| What does a certificate NOT prove? | That the name is legitimate, safe, or who you assume it is (DV certs prove domain control only) |
| Why are there usually 3 certs in a chain? | Leaf → intermediate → root; roots stay offline, intermediates do day-to-day signing |
| TLS 1.2 full handshake: extra RTTs beyond TCP | 2 |
| TLS 1.3 full handshake: extra RTTs beyond TCP | 1 |
| Why is TLS 1.3 faster? | Client sends a guessed `key_share` in the ClientHello instead of waiting to be told which group to use |
| What closes the Logjam/POODLE downgrade-attack family? | The `Finished` MAC, computed over the whole handshake transcript |
| Why is 0-RTT data risky? | It's replayable — no freshness guarantee against a captured, resent flight |
| What problem does SNI solve? | Lets a server sharing one IP pick the right certificate before the handshake is encrypted |
| What does SNI leak? | The hostname you're connecting to, in the clear, to anyone watching the wire |
| What does ALPN negotiate? | Which application protocol (http/1.1, h2, h3) rides on this TLS connection |
| Why can't HTTP/1.1 multiplex? | A response has no identifier tying it back to its request — only position tells you |
| What does an HTTP/2 stream ID solve? | Lets many requests/responses interleave over ONE connection, sorted by ID |
| Why did HTTP/1.1 pipelining fail? | Responses still had to return in order, AND real middleboxes handled it incorrectly |
| What does HPACK compress? | Repetitive HTTP headers, via a static table + a per-connection dynamic table |
| Why can't HTTP/3 reuse HPACK? | Its dynamic table needs strict cross-stream ordering; QUIC deliberately gives that up |
| What does QPACK do differently? | Puts table updates on separate streams, decoupled from request streams |
| What does RFC 9001 specify? | TLS 1.3's handshake carried inside QUIC's own packets, from the first datagram |
| Rapid Reset's exploited blind spot | Concurrency limits counted OPEN streams, not requests STARTED and instantly reset |
| What was Heartbleed's root cause? | A missing bounds check on an attacker-supplied length field in the Heartbeat extension |
| What did Heartbleed leak? | Arbitrary server process memory — including TLS private keys — with no trace in logs |
| Why did Microsoft Teams go down in Feb 2020? | An internal authentication certificate expired, unrelated to any cryptographic break |
| What's the single most preventable class of outage in this note? | Certificate expiry — automate issuance AND alert on it, don't just automate |

## Primary sources

**Core specifications**
- **RFC 8446** — TLS 1.3.
- **RFC 8996** — Deprecating TLS 1.0 and TLS 1.1.
- **RFC 6066** — TLS Extensions, including Server Name Indication (SNI).
- **RFC 7301** — TLS Application-Layer Protocol Negotiation (ALPN).
- **RFC 7568** — Deprecating SSL 3.0.
- **RFC 6797** — HTTP Strict Transport Security (HSTS).
- **RFC 8555** — Automatic Certificate Management Environment (ACME).
- **RFC 6962** — Certificate Transparency.
- **RFC 7748** — Elliptic Curves for Security (X25519).

**HTTP evolution**
- **RFC 9110** — HTTP Semantics.
- **RFC 9112** — HTTP/1.1.
- **RFC 9113** — HTTP/2.
- **RFC 7541** — HPACK: Header Compression for HTTP/2.
- **RFC 9114** — HTTP/3.
- **RFC 9204** — QPACK: Field Compression for HTTP/3.
- **RFC 9000** — QUIC: A UDP-Based Multiplexed and Secure Transport.
- **RFC 9001** — Using TLS to Secure QUIC.
- **RFC 9308** — Applicability of the QUIC Transport Protocol.
- **RFC 9460** — Service Binding and Parameter Specification via the DNS (SVCB/HTTPS records).

**Incidents and case studies**
- CVE-2014-0160 (Heartbleed); `heartbleed.com`; RFC 6520 (the Heartbeat Extension).
- CVE-2014-3566 (POODLE); Möller, Duong, Kotowicz, *"This POODLE Bites,"* Google Security Team, 2014.
- CVE-2015-4000 (Logjam); Adrian et al., *"Imperfect Forward Secrecy: How Diffie-Hellman Fails in Practice,"* ACM CCS 2015 (`weakdh.org`).
- CVE-2023-44487 (HTTP/2 Rapid Reset); Cloudflare's, Google's, and AWS's joint disclosure posts, 10 October 2023.
- Fox-IT's forensic report on the DigiNotar breach (2011); Mozilla's and Google's public root-removal advisories.
- Microsoft 365 status updates and contemporaneous press coverage (ZDNet, The Verge) on the February 2020 Microsoft Teams outage — **verify current** exact wording against Microsoft's own original statement.
- Eric Butler, Firesheep release announcement, October 2010 (`codebutler.com`).
- Citizen Lab, *"China's Great Cannon,"* 10 April 2015 (`citizenlab.ca`).
- Moxie Marlinspike, *"New Tricks for Defeating SSL in Practice,"* Black Hat DC, 2009 (the `sslstrip` talk).
- Chrome team, *"Intent to Remove: HTTP/2 server push,"* Chromium `blink-dev` mailing list, 2022.

**Operational references**
- `crt.sh` — Certificate Transparency log search.
- `dnsviz.net`, `zonemaster.net` — carried over from Day 12, still useful for auditing delegation/DNSSEC alongside the SVCB/HTTPS records HTTP/3 discovery depends on.
- `ssllabs.com`, `securityheaders.com` — external TLS/security-header configuration scanning.
- `mkcert` — a widely used tool for running a trusted local development CA, referenced in the Build This section's honesty caveats.
- `openssl speed` — for benchmarking symmetric-versus-asymmetric operation cost on your own hardware, referenced in concept 2.

**Fast-drifting facts — verify before relying on any of these**
- Browser TLS-version and cipher-suite defaults, and exactly which legacy versions are still silently accepted.
- EV certificates' browser UI treatment (largely removed across major browsers, but confirm current behavior).
- Maximum allowed public-certificate validity periods (actively being shortened by CA/Browser Forum policy over time).
- HTTP/2 and HTTP/3 adoption statistics (`W3Techs`, `Cloudflare Radar` publish current figures; both move quickly).
- ECH deployment status across browsers and CDNs.
- QUIC's measured CPU-overhead-versus-TCP figures, as kernel offload support (GSO/GRO, `sendmsg` segmentation) continues to close the gap Day 11 documented.
- Exact browser warning wording for untrusted/self-signed certificates (Chrome and Firefox both revise this UI periodically).

## Layered explanations

**10 seconds.** Plain HTTP exposes everything to eavesdropping, tampering, and impersonation. TLS fixes this with a handshake that exchanges a secret key (without ever transmitting it) and verifies identity via signed certificates — and it costs real round trips, which is exactly why HTTP itself evolved through HTTP/2 (multiplex many requests over one connection) and HTTP/3 (do the same thing over a transport, QUIC, where one lost packet doesn't stall every request sharing the connection).

**1 minute.** Two strangers can't agree on a secret over a channel an attacker can read — unless they use asymmetric key exchange, where both sides compute an identical shared secret from values exchanged in the clear, then switch to fast symmetric encryption for the actual data. That solves eavesdropping, but a public key alone proves nothing about who holds it, so certificates — signed statements from a trusted certificate authority binding a key to a name — solve impersonation, verified by walking a chain of signatures up to a root your browser already trusts. All of this is negotiated in one handshake, and the handshake's cost is measured in round trips: TLS 1.2 needed two extra beyond the TCP handshake, TLS 1.3 needs only one, because the client speculatively sends its key-exchange value before being told which group to use. Meanwhile, HTTP/1.1 allowed only one request in flight per connection — a real, separate limitation from anything TLS does — which HTTP/2 fixed by giving every request an explicit stream ID so many can multiplex over one connection, with HPACK compressing the repetitive headers every request resends. HTTP/2 still shares one TCP connection's fate under packet loss, though, which is why HTTP/3 moves to QUIC, where streams are independent all the way down — at the cost of needing a new header-compression scheme, QPACK, because HPACK's compression secretly depended on the ordering guarantee QUIC deliberately gives up.

**5 minutes.** HTTP puts every byte on the wire in the clear, exposed to three distinct threats — eavesdropping (passive reading), tampering (active modification), and impersonation (pretending to be an endpoint) — and TLS answers each one with a specific mechanism, not a single blanket fix. Confidentiality comes from hybrid encryption: asymmetric cryptography (public/private key pairs, whose security rests on problems like integer factorization or the discrete logarithm being computationally infeasible to invert) is used exactly once, via a Diffie-Hellman-style key exchange, to let two strangers compute an identical secret without ever transmitting it — then a fast symmetric AEAD cipher handles all the actual data, getting both speed and solved key distribution. That still leaves impersonation open, because a key pair alone carries no identity — anyone can generate one — so certificates bind a public key to a name via a CA's signature, and trust is established by walking a chain (leaf → intermediate → root) up to one of the roughly hundred roots a browser already trusts; DigiNotar's 2011 compromise showed what happens when that trust is misplaced, and Microsoft Teams' 2020 outage showed that the far more common failure mode is purely operational — a certificate simply expiring unnoticed. All of this negotiation, key exchange, and identity verification is bundled into one handshake, and Heartbleed and POODLE both demonstrate that the *implementation* and *negotiation* of that handshake are themselves attack surfaces distinct from the cryptography's own strength. The handshake's cost is measured in round trips on top of TCP's own: TLS 1.2 needs two, TLS 1.3 needs one (by having the client guess its key-exchange group upfront), and 0-RTT resumption needs zero — at the cost of replay exposure, safe only for actions with no harmful effect if repeated. SNI lets a server sharing one IP address pick the right certificate mid-handshake, at the cost of leaking the hostname in the clear, which ECH is closing. Meanwhile HTTP itself evolved independently of all this cryptography, but under the same round-trip-minimization pressure: HTTP/1.1's persistent connections still allow only one request in flight at a time, because a response carries no identifier tying it back to its request; HTTP/2 fixes this at the root with a binary framing layer and explicit stream IDs, adding HPACK to compress the headers every request repeats, but still inherits TCP's single shared fate under packet loss because every stream rides one ordered byte stream; HTTP/3 moves to QUIC specifically to give each stream independent loss recovery, which forces a new header-compression scheme (QPACK) because HPACK's dynamic table secretly relied on the cross-stream ordering QUIC deliberately abandons — and because TLS 1.3's own handshake messages are carried inside QUIC's packets from the first datagram rather than running after a separate transport handshake, a fresh HTTP/3 connection reaches application data as fast as a *resumed* TLS-over-TCP connection does, on the very first visit.

**Expert summary.** HTTPS is best understood as two separable engineering problems solved by one bundled protocol: establishing a shared cryptographic context between two parties with no prior relationship, and doing so with an authenticated binding to identity, both under a hard latency budget that makes every additional network round trip a real, felt cost rather than a rounding error. The first problem is solved by hybrid encryption — an ephemeral Diffie-Hellman-family key exchange amortizing asymmetric cryptography's cost over exactly one operation per connection, with symmetric AEAD ciphers carrying the actual byte volume — and its security is entirely contingent on computational hardness assumptions (integer factorization, elliptic-curve discrete log) that are believed, not proven, to hold, a caveat made concrete by the ongoing transition toward hybrid post-quantum key exchange. The second problem, identity, is solved by a certificate-based PKI whose trust model is fundamentally social and institutional rather than purely cryptographic: a signature only carries meaning because a browser vendor's root program decided, as a governance act, to trust the signer, which is precisely the property DigiNotar's compromise attacked and Certificate Transparency was built to make auditable after the fact. TLS 1.3's structural contribution over 1.2 is collapsing negotiation and key exchange into a single round trip by having the client speculate its key-exchange group, and closing the downgrade-attack surface (Logjam, POODLE) by binding the entire negotiation transcript into an authenticated `Finished` MAC rather than trusting an unauthenticated negotiation phase — a lesson in protocol design that generalizes past TLS: any negotiable parameter that isn't itself authenticated is available to an active attacker as a lever. HTTP's parallel evolution is a story about where an ordering guarantee is enforced rather than whether one exists: HTTP/1.1 enforces it implicitly via connection-wide serial delivery (no per-message identifier exists to do otherwise); HTTP/2 introduces an explicit identifier (the stream) to multiplex logically independent exchanges, but the ordering guarantee is still furnished for free by the underlying transport, which HPACK's compression state silently depends on; HTTP/3 relocates the ordering guarantee to per-stream granularity at the transport layer itself (QUIC), which is a strictly more precise match between the guarantee's cost and the correctness properties that actually need it, at the price of requiring every mechanism that implicitly assumed connection-wide ordering — header compression chief among them — to be redesigned around the new, narrower guarantee. The throughline across both halves of this note is the same: a system's correctness properties should be scoped as narrowly as the problem actually requires, because guarantees enforced more broadly than necessary (TCP's connection-wide byte ordering, an unauthenticated handshake negotiation) become exactly the surface where a change in requirements, or an adversary, later extracts a cost nobody budgeted for.
