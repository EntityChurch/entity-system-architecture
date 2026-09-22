# EXPLORATION — web caching ran all four mechanisms, side by side, and measured them

**Status:** EXPLORATION (2026-09-13). Written to be attacked. Nothing here is ruled.

**Fifth and last of the survey sequence.** The four mechanisms — **compute · ask · look up · publish** —
were not derived here and were not first tried by the peer-to-peer field. **Three of them were
deployed protocols in the same software, on the same workload, in the 1990s, and compared against each
other.** This document reads that literature, adds the modern set-reconciliation answer, and records
where the research stops paying.

**Read for this document:** Squid's inter-cache protocol family — **ICP** (RFC 2186/2187), **Cache
Digests**, **CARP**, HTCP — and the operational record of why cache hierarchies declined · **Erlay**
and **minisketch** (BIP 330) · the **private information retrieval** literature, 2023–2026 ·
**CoralCDN**'s five-year operational retrospective and **OpenDHT**'s public-service design.

---

## §0 The result

**Six findings. The first reframes the whole survey; the fifth is the one that changes a design row.**

1. ⭐⭐⭐ **Web caching implemented three of our four mechanisms as competing protocols and published the
   comparison.** **ICP** is *ask everyone* — query all siblings, wait, take the first HIT. **Cache
   Digests** is *publish* — a Bloom filter of what you hold, exchanged periodically. **CARP** is
   *compute* — deterministic URL hashing, always the same neighbour for the same URL. **The
   trade-offs they measured are exactly the ones this survey re-derived**, and their fitness verdict
   is our scope argument in a single line: **CARP *"works well for Intranet hierarchies, but less well
   for loosely coupled, non-autonomous Internet cache peerings."*** Compute needs tight coupling; ask
   and publish are what loose peering gets.

2. ⭐⭐ **Their sharpest failure mode is the capability problem, and they hit it in production thirty
   years ago.** ICP's **false hits**: *"ICP does not convey information about HTTP headers… an object
   present in cache but not accessible for a sibling cache"* — **the sibling answers HIT and then
   refuses to serve it.** ⇒ ***a holder claim that does not model the authorization gate is a claim
   that will be honoured and then declined***, which is precisely the shape our capability model would
   produce if a holder set were built naively.

3. ⭐⭐ **They also ran the load-bearing-hint failure and logged it.** A Cache Digest false positive
   produced `CD_SIBLING_HIT … TCP_MISS/504` — **the client got an error instead of the request falling
   through to another sibling or a parent.** Their stated remedy is the rule this corpus already
   carries in different words: *"use lightweight techniques, optimise for the common case, and
   robustly handle the unusual case"*, plus a protocol affordance — the **`only-if-cached`** request
   header, so a probe that misses returns cleanly instead of causing work.

4. ⭐⭐⭐ **Set reconciliation is the bridge between *ask* and *publish*, and this corpus was already
   there — further than the last three documents credited.** `THE-CONVERGENT-DESIGNS` §7 established
   range-based set reconciliation, scored it against Bloom/IBLT/PinSketch, and reached the finding that
   matters: RBSR *"depends critically on the storage backend's ability to summarize arbitrary
   ranges,"* and ***our canonical Merkle trie is exactly such a backend*** — a total order with a
   fingerprint already computed at every interior node, for other reasons. **What Erlay adds is new
   and worth having** (§4): the **hybrid**, the **measured numbers**, and the **eclipse-resistance
   argument**.

5. ⭐⭐ **The universal deployed answer to holder-set staleness is not probing and not trust levels — it
   is SOFT STATE WITH MANDATORY REFRESH, and it appears in four independent systems.** Kademlia
   provider records expire and must be re-announced · OpenDHT's **"revolving door"** where old data is
   overwritten and *"clients must periodically refresh, with TTLs keeping clients informed"* · Squid's
   digests, which **cannot delete** and so must be **rebuilt from scratch periodically to erase stale
   bits and prevent digest pollution** · Freenet's LRU eviction. ⇒ **a holder claim should be soft
   state by default**, and the design question is *who pays for the refresh* — which is exactly the
   cost IPNI was built to reduce.

6. ⚠ **PIR's barrier is a proven lower bound, not an engineering gap — and it is nevertheless
   shipping in narrow cases.** *If the server learns nothing about which record is fetched, the server
   must compute over every record.* So cost scales with database size, per query, forever. **But
   Apple's Live Caller ID Lookup and its private-search work are deployed PIR**, and the 2024–2026
   schemes are an order of magnitude better than the 2022 ones. ⭐ **And the practical note that
   matters most to us: for small databases, "just download the whole thing" beats PIR on total cost**
   — which is Cache Digests again, and which is the fleet case.

> ⭐ **A correction to my own list: three items I named as unread had already been read here.**
> Usenet/NNTP flood-fill, Scuttlebutt, and the BitTorrent tracker→DHT→PEX progression are all marked
> **READ** in `REFERENCE-PRIOR-ART`, with a verdict I independently reproduced without noticing —
> *"it is **not** a progression; PEX structurally cannot bootstrap, and no layer won."* **This is the
> same defect the register exists to stop, committed by the session that wrote the row about it.**

---

## §1 The three protocols, as they were actually compared

| | **ICP** | **Cache Digests** | **CARP** |
|---|---|---|---|
| **our mechanism** | ⭐ **ASK EVERYONE** | ⭐ **PUBLISH** | ⭐ **COMPUTE** |
| mechanism | UDP query/reply **per request** | **Bloom filter** exchanged periodically | **deterministic URL hashing** |
| per-request network cost | **high** — query all peers and wait | **none** | **none** |
| false hits | **yes** — no HTTP header semantics, plus a race | **yes** — Bloom false positives **and** staleness | **none, by construction** |
| object duplication | yes | yes | **avoided** |
| best fit | legacy / loose peering | **loose peering with less chatter** | **tightly-coupled intranet arrays** |

**ICP's mechanics, because they are our archived fan-out query almost exactly.** A cache that lacks a
document **queries its siblings**; siblings answer `HIT` or `MISS`; the cache picks a source from the
replies. It **sends all queries at once** — *it cannot query serially without a linear slowdown as
siblings are added* — and **holds the client request until the first positive reply**, with a
configurable timeout **defaulting to two seconds**. **Sibling MISS replies are ignored entirely.**

⭐ **And it carries a role distinction we have under other names:** *a neighbour hit may be fetched from
a parent or a sibling, but a neighbour miss can never be fetched from a sibling* — **a sibling may
only be asked for what it already holds; a parent may be asked to go and get anything.** That is
peer-versus-relay, in a 1997 RFC.

**Its stated costs:** the per-request latency and the fan-out — *every added sibling multiplies UDP
query fan-out per miss, the classic argument against scaling these meshes* — plus **UDP's security
properties**, and the false hits below. **Cache digests were proposed specifically to replace that
two-second wait.**

---

## §2 The failure modes they found, which are ours

### §2.1 The false hit is the capability problem

> *"ICP does not convey information about HTTP headers associated with a web object; HTTP headers may
> include access control and cache directives… false cache hits may occur — an object present in cache
> but not accessible for a sibling cache being one example."*

⭐⭐ **Read that as a holder set and it is the exact failure our capability model would produce.** A peer
answers *"I have `H`"* truthfully, and then the retrieval is refused because the asker's capability
does not cover it. **The claim was true and useless**, and the asker paid a round trip to find out.

**Two consequences, and they pull in opposite directions:**

- **A holder claim should say what it can honour, not merely what it holds** — otherwise the answer
  set is padded with peers that will decline.
- ⚠ **But saying so leaks the authorization state**, which is a finer-grained disclosure than the
  existence leak already recorded. **This is a genuine tension and it is new to this survey.**

**There is also a timing variant worth keeping:** an ICP `HIT` means *a subsequent request for that URL
would hit* — **a claim about the future**, which the object may not survive. A holder claim has the
same property and the same honest fix: a timestamp and an expiry, not a promise.

### §2.2 The load-bearing hint, observed in the field

A Cache Digest false positive was seen to produce `CD_SIBLING_HIT … TCP_MISS/504` — **the client
received an error rather than the request falling through to the next sibling or the parent.**

⇒ **The non-load-bearing rule this corpus already states for reference hints is not theoretical, and
this is its production instance.** Their design philosophy is worth adopting verbatim in substance:
***lightweight techniques, optimise for the common case, robustly handle the unusual case*** — and the
protocol affordance that makes it cheap is **`only-if-cached`**, a request header meaning *serve this
only if you actually have it*, so a false positive is answered rather than escalated.

### §2.3 What Bloom filters cost, operationally

- ⛔ **No deletions.** *"Squid does not support deletions from the digest, so the digest must
  periodically be rebuilt from scratch to erase stale bits and prevent digest pollution."* The classic
  limitation — no unset without counting.
- ⭐ **Deltas exist and are the optimization**: a digest diff *"built by comparing a new digest with
  the old one, consisting of aggregated deletions and additions since the previous digest, requiring
  less bandwidth and enabling more frequent updates."*
- **Refresh policy interacts with correctness**: *be conservative — do not add objects that might
  become stale soon*, which reduces false hits.
- ⚠ **A content-addressing quirk worth knowing:** Squid indexes objects by their **MD5 key**, so
  *"there is no URL actually available for each object"* — which breaks URL-pattern refresh rules and
  falls back to the default. ***Content-addressing the index costs you the ability to apply policy by
  name***, and that is a real trade rather than a bug.

---

## §3 Why the hierarchies declined, and why that failure does not transfer

**The deployment-level causes are clear and none of them is the algorithm:**

- ⭐⭐ **HTTPS.** *Encrypted content cannot be cached.* Transparent interception does not rescue it —
  the proxy becomes a man-in-the-middle re-encrypting with its own key, which clients must trust and
  which some services forbid outright.
- **CDN and embedded-cache displacement.** Origin-controlled reverse proxies moved **into** the very
  networks the hierarchies were built to serve; *the more content served that way, the less effective
  a demand-based cache becomes.*
- **Economics**, and protocols optimizing the client-origin connection directly.

> **The summary in the operational literature: *the miss-cost of sibling meshes stopped paying for
> itself once content moved behind TLS and origin-controlled reverse proxies moved into the same
> networks.***

⭐⭐⭐ **And the first cause is the one that does not transfer, which is the most useful thing in this
document.** Web caches failed on encrypted content because **an HTTP cache must understand the bytes
to know whether it may serve them.** A content-addressed store does not: **an intermediary can hold,
serve and be verified on ciphertext it cannot read**, because the hash settles identity and the
capability settles access — separately. **The capability-aware systems in this survey demonstrate it
already**: storage servers that get *"no automatic ability to read or modify"* what they hold, and
block-level tokens where absence of a token means public.

⇒ ***Encryption killed intermediary caching on the web and does not kill it here.*** That is a
structural advantage of content addressing over URL caching, and it should be stated somewhere
durable rather than rediscovered.

**Not dead, repurposed:** forward proxies persist, aimed at **policy** — authentication, filtering —
rather than bandwidth. *The mechanism survived; its economic justification did not.*

---

## §4 Set reconciliation — and crediting what this corpus already had

**`THE-CONVERGENT-DESIGNS` §7 reached this first and reached it well.** It establishes **range-based
set reconciliation** — two peers hold fingerprints over ranges of a **totally ordered** set, equal
ranges fingerprint equal, unequal ranges are recursively subdivided until the difference is isolated,
with a short-circuit to *just send the items* below a size threshold — scores it against the sketch
family (**no false positives · work proportional to the difference · logarithmically many rounds**
against Bloom/IBLT/PinSketch's one-or-few rounds and linear build), and lands the finding:

> **RBSR's efficiency *"depends critically on the storage backend's ability to summarize arbitrary
> ranges"* — and our canonical Merkle trie is exactly such a backend**, a total order over paths with
> a subtree-hash fingerprint at every interior node, computed already for other reasons. **The
> marginal cost would be the protocol messages and close to nothing else.**

### §4.1 What Erlay adds, which is genuinely new here

**Erlay is the same family applied to flooding, and its contribution is that it did not replace the
flood — it bounded it.**

- **The problem, stated as a topology cost:** every node announces every transaction to every peer, so
  announcement bandwidth is **linear in connection count** — which ⭐ ***discourages nodes from
  increasing connectivity, and connectivity is what makes eclipse attacks hard.*** **The cost of
  flooding suppresses the security property you wanted from the topology.**
- ⭐⭐ **The design is a deliberate hybrid:** announce directly over a **small number of connections**
  (eight outgoing), and reach everyone else by **periodically reconciling the withheld set** over every
  connection. Stated reason: *a fully reconciliation-based protocol would be inherently very slow,
  even if highly efficient.*
- **The primitive:** **minisketch** implements **PinSketch** (BCH codes), communicating a set to a peer
  with an unknown but similar set **using bandwidth equal to the size of the difference, not the size
  of the sets** — and more bandwidth-efficient than IBLT. **Sketch extension** handles a failed decode
  by sending a higher-capacity sketch minus what was already sent.
- **Measured:** roughly half the bandwidth, about **75% at 32 outbound peers** in the paper, and a more
  conservative **~40%** for the Core implementation, *which was needed for better security.* Latency
  went from about **4 seconds to 6**, judged acceptable against a ten-minute block time.
- ⭐ **Graceful negotiation:** a `sendtxrcncl` handshake; peers that do not support it **fall back to
  flooding seamlessly.**

### §4.2 The distinction this forces, and it resolves an earlier tension

An earlier document in this sequence concluded that **a flat hash space cannot aggregate**, on NDN's
evidence. **That is true of routing and false of reconciliation, and the difference is precise:**

| | needs | do content hashes have it? |
|---|---|---|
| **routing aggregation** (NDN's FIB, CIDR) | the key's **prefix must correlate with topology** | ⛔ **no** — a hash prefix implies nothing about location |
| **reconciliation aggregation** (RBSR, digests) | only a **total order** with summarizable ranges | ✅ **yes, trivially** — hashes sort |

⇒ **You cannot route on a hash prefix and you can absolutely reconcile on one.** The earlier finding
stands for placement and does not bar a reconciled holder set — **which is a materially different
conclusion from the one the leaf-count discussion was heading toward.**

---

## §5 Soft state with mandatory refresh

**Four independent systems, one answer.**

| system | the mechanism | what it costs |
|---|---|---|
| **Kademlia / IPFS** | provider records expire; **re-announce continuously** | the cost IPNI exists to reduce |
| **OpenDHT** | ⭐ the **"revolving door"** — old data is overwritten by new, **clients must periodically refresh**, TTLs keep clients informed | *designed in, because* **"if it offered persistent storage semantics it would eventually fill up with orphaned data — garbage collection of which seems difficult to do efficiently"** |
| **Squid digests** | **periodic full rebuild** — Bloom cannot delete | bandwidth, mitigated by deltas |
| **Freenet** | **LRU eviction** favouring frequently-accessed content | unpopular content disappears |

⭐⭐ **And garbage collection is a recurring killer rather than a detail.** Upspin's maintainers name it
directly — *GC had always been their sore point.* OpenDHT designed its whole storage model around the
impossibility of collecting orphaned data efficiently. **Two of the survey's shutdowns and one of its
core designs turn on it.**

⇒ **For a holder claim the implication is concrete: make it soft state with an expiry, and make the
holder re-assert.** The alternative — a grow-only set of permanent claims — is the orphaned-data
problem with extra steps, and it is the thing OpenDHT refused to build.

---

## §6 PIR, honestly

**The lower bound is the whole story.** *If the server learns nothing about which record a user
fetches, the server must compute over every record* — so PIR is **fundamentally memory-bound** and
dollar-cost per query scales with database size. Preprocessing buys sublinear online computation but
**assumes an immutable database**, and per-client hints do not amortize cheaply.

**It is nonetheless moving, and it has shipped.** OnionPIRv2, FlashPIR and YPIR are substantially
faster than the 2022 state of the art — YPIR reporting an **8× server-cost reduction with no offline
communication** — and **Apple's Live Caller ID Lookup and its private-search work are deployed PIR.**

⭐ **The line to carry: *for small databases the trivial "just download it" baseline often wins on total
cost.*** **That is Cache Digests, again, arrived at from cryptography** — and it is the fleet case,
where the honest answer to reader privacy is *fetch the whole digest and query it locally.*

---

## §7 Coral, OpenDHT, and *the algorithms were not the problem*

**CoralCDN ran for five years on PlanetLab**, serving *several million client IPs per day* and
accounting for the majority of that platform's bandwidth, with publishing as simple as **appending to
a URL's hostname** and a DNS layer steering browsers to nearby caches.

⭐ **Its lookup is a three-level scope hierarchy** — a node queries its **level-2 cluster**, then its
**level-1 cluster**, then the **global level-0 system**, stopping at the first hit. *That is
local-then-regional-then-global as a lookup structure*, and it is the closest deployed thing to the
scope model this survey arrived at independently.

⭐⭐ **The retrospective's own emphasis is the finding:** *rather than focusing on the self-organizing
algorithms, the majority of the paper analyzes CoralCDN as an open web service on a virtualized
platform — which is where most of the "what went wrong" material lives.* **The distributed algorithm
was not what hurt. Operating an open service was** — resource management, abuse, and a hosting
substrate that eventually went away.

**This corpus reached the same verdict about a different system**, in its Hyper-G reading of p-flood:
***"it is a good algorithm and it was never the problem."*** ⇒ **Three independent instances now say
the mechanism is rarely the thing that kills these systems.**

---

## §8 Where the research stops paying

**Diminishing returns are here, and naming that is part of the job.** The last three documents each
overturned a conclusion; this one overturned none — it **confirmed, quantified and credited**. That is
the signal to stop surveying and start synthesizing.

**Genuinely unread, with why each is now low-value:**

- **Matrix's state-resolution and federation scaling** — a different problem (room state convergence),
  and this corpus has read it at the bridge level already.
- **Perkeep share claims · Hypercore sparse replication · Autonomi's post-launch data model** —
  mechanism detail on systems already scored; unlikely to move a row.
- **Tahoe grid membership in practice** — would sharpen §5 and nothing else.
- **HTCP** — the fourth cache protocol; named in the literature, and the three read here span the
  design space.
- ⚠ **The one that might still pay: the abuse and resource-exhaustion axis.** CoralCDN and OpenDHT
  both say it is where the real difficulty lives, and **this survey has looked at it only glancingly.**
  *If one more research pass happens, make it that one.*

---

## §9 What would test this

- **The false-hit/capability claim is testable by construction**: write a holder claim for content the
  asker is not entitled to, and check whether the answer set can distinguish *I do not have it* from
  *I have it and will not give it to you*. If it cannot, §2.1's tension is real and unaddressed.
- **The reconciliation claim is testable against our own trie**: take two peers' content-hash sets and
  reconcile by subtree fingerprint. If range summarization over hashes is not cheap on the existing
  structure, §4's conclusion is wrong.
- **The soft-state claim is falsifiable by counterexample**: name a deployed shared content index whose
  entries never expire and which did not accumulate orphaned data. **We did not find one.**

---

## §10 Sources

**Read for this document.** Inter-cache protocols — [RFC 2187, *Application of ICP v2*](https://www.rfc-editor.org/rfc/rfc2187),
[Squid: Cache Digests](https://wiki.squid-cache.org/SquidFaq/CacheDigests),
[Squid FAQ: Cache Digests](https://flex.phys.tohoku.ac.jp/texi/faq-squid/FAQ-16.html),
[Squid's inner workings](https://wiki.squid-cache.org/SquidFaq/InnerWorkings),
[Squid: linking into a cache hierarchy](https://wiki.squid-cache.org/Features/CacheHierarchy),
[*Inter Cache Communication Protocols* (IETF draft)](https://www.ietf.org/proceedings/44/I-D/draft-melve-intercache-comproto-00.txt),
[*Cache digests* (Rousskov & Wessels)](https://www.sciencedirect.com/science/article/abs/pii/S0169755298002517),
[*Is transparent web caching dead?*](https://www.senki.org/transparent-web-caching-dead/) ·
Erlay — [BIP 330](https://bips.dev/330/),
[Scaling Bitcoin transcript](https://scalingbitcoin.org/transcript/telaviv2019/erlay),
[bitcoin/bitcoin#30249 tracking](https://github.com/bitcoin/bitcoin/issues/30249) ·
PIR — [FrodoPIR (PoPETs 2023)](https://petsymposium.org/popets/2023/popets-2023-0022.pdf),
[YPIR](https://eprint.iacr.org/2024/270.pdf),
[OnionPIRv2](https://eprint.iacr.org/2025/1142),
[Sion, *On the computational practicality of PIR*](https://www.ndss-symposium.org/wp-content/uploads/2017/09/On-the-Practicality-of-Private-Information-Retrieval-Radu-Sion.pdf) ·
CoralCDN — [*Experiences with CoralCDN: A Five-Year Operational View* (NSDI '10)](https://www.usenix.org/legacy/event/nsdi10/tech/full_papers/freedman.pdf) ·
OpenDHT — [*OpenDHT: A Public DHT Service and Its Uses* (SIGCOMM '05)](https://dl.acm.org/doi/pdf/10.1145/1080091.1080102).

**From this project's own record — and §0's correction means these should have been consulted first.**
`EXPLORATION-THE-CONVERGENT-DESIGNS-WILLOW-SSB-USENET-MATRIX-AND-THE-FIVE-VERDICTS` §7 (range-based set
reconciliation, the sketch-family comparison, and the Merkle-trie-as-backend finding) ·
`REFERENCE-PRIOR-ART-FEDERATED-AND-P2P-PUBLISHING-SYSTEMS-BY-AXIS` (Usenet, Scuttlebutt and the
BitTorrent progression, all marked READ) · `EXPLORATION-HYPER-G-THE-SYSTEM-THAT-SHIPPED-WHAT-XANADU-DESIGNED`
(p-flood, and *"it is a good algorithm and it was never the problem"*) ·
`EXPLORATION-THE-P2P-COVERAGE-AUDIT-WHAT-WE-HAVE-STUDIED-AND-THE-FIVE-AXES-WE-HAVE-NOT` ·
and the four predecessor explorations of this sequence.
