# EXPLORATION — the content-addressed landscape: what survived, what died, and the lookup the winners never built

**Status:** EXPLORATION (2026-09-13). Written to be attacked. Nothing here is ruled.

**Third in a sequence.** `FIVE-KEYED-LOOKUPS` established *what* the fifth lookup is;
`THE-CONTENT-LOOKUP-LANDSCAPE` read nine systems and corrected one of its conclusions. **This one adds
the axis neither had: what actually survived, what died, and why** — and it corrects a conclusion of
its own predecessor, for the second time running.

**Systems read for this document, at primary or near-primary sources:** Perkeep · Upspin (including
its shutdown notice) · Peergos · Hypercore/Dat/Beaker · Syncthing global discovery v3 · BitTorrent's
tracker / DHT / PEX / magnet split · the 2001–07 academic DHT wave and its post-mortems · the
crypto-storage networks (Filecoin, Storj, Arweave, Sia, Swarm) · OCI registries and Nix binary caches ·
**Tor v3 onion-service descriptors.**

---

## §0 The result

**Six findings. The first is the one to carry, and the third and fourth correct our own rows.**

1. ⭐⭐⭐ **The content-addressed systems that won at planetary scale have no content-address lookup at
   all.** OCI registries, Nix binary caches, Git, the Go module proxy, Debian archives — every one is
   hash-keyed, deduplicating, mirror-friendly and enormous, and **not one of them can answer *"who has
   `H`"***. They have three things instead: content addressing for verification, **a configured list of
   places to ask**, and a signed Merkle root. The property that makes this work is stated best by the
   OCI/Nix literature: ***verification is decoupled from the source of bytes*** — a pull pinned to a
   digest can be served by any mirror or pull-through cache. **So "who has it" is answered by
   configuration, and content addressing is what makes that safe.** *This is the previous document's
   accelerator finding, confirmed at the only scale that matters: the winners did not build the
   accelerator either.*

2. ⭐ **The systems that tried to build the lookup have a consistent failure record, and the causes
   are not incidental.** The academic DHT wave's own post-mortem gives three: peer latency and
   bandwidth versus datacenter networks, peer unreliability, and ***lack of trust in peers' correct
   behaviour — securing DHT routing is "hard and unsolved in practice."*** Summary: *decentralization
   seemed good for load and fault tolerance, but security and churn undermined both.* §2 has the
   scoreboard.

3. ⭐⭐⭐ **The hard open question from the previous document is ANSWERED, and deployed for a decade.**
   That document said the lookup key cannot be the content hash and that nobody had a usable
   derivation. **Tor v3 onion services have one: the lookup index is a BLINDED public key**, derived
   by the client from (identity key, time period, shared random), daily-rotating, **from which the
   identity cannot be recovered** — and the descriptor it fetches is encrypted to the identity key.
   ***Holding the address is simultaneously the ability to compute where to look and the ability to
   read what is there.*** **And v2 was our drafted design**: a descriptor id derived directly from the
   service identity, which *"allowed harvesting of onion addresses"* — harvesting that the Tor
   Project classifies as malicious and sanctions relays for. **They shipped our key, watched it get
   enumerated, and replaced it.**

4. ⭐⭐ **The previous document's *"no system read here has both a capability model and a content
   lookup"* is WRONG. Two do.** **Peergos** layers **block access tokens** under IPFS — *you need a
   secret token to retrieve a block at all*, the token is derivable from the block so the server
   authenticates statelessly, and **blocks without a token are treated as public**. **Tor v3** is the
   other. Both are worth more than the systems that lack it.

5. ⭐⭐ **The field's own verdict on combining open discovery with access control is that they are
   incompatible — and it is a verdict by surrender, not by proof.** When BitTorrent needed access
   control it **turned distributed discovery off**: the `private` flag disables DHT and PEX so every
   peer goes through one trusted tracker, *because standard DHT protocols lack authentication.* A 2025
   paper proposing an authenticated Kademlia so private communities can have a DHT fallback shows the
   problem is **still open research.** ⇒ **We are not failing to find a known answer. There mostly
   isn't one** — which is a very different position to be designing from.

6. ⭐ **Price is not why decentralized storage has not displaced the centralized kind.** Filecoin is
   cited at as little as **$0.19/TB/month against S3's ~$23** — two orders of magnitude — and stores
   **~2.1 EiB against 7.6 EiB of raw capacity, roughly 28% utilization.** The named barriers are
   retrieval latency, operational complexity, and — the one that matters here — ***an inability to
   provide asynchronous forward replication to ensure availability against failures.*** **That is the
   placement-policy gap, priced: 121× cheaper and it still does not win.**

---

## §1 The scoreboard

**Outcome is judged on deployment and continued operation, not on design quality.** Several of the
best designs here are in the dead column.

| System | what it is | the "who has it" answer | outcome |
|---|---|---|---|
| **Git** | content-addressed object store | **a remote URL you configured** | ⭐ **won, totally** |
| **OCI registries** | hash-keyed blob store | **a registry/mirror you configured** | ⭐ **won, totally** |
| **Nix / Guix caches** | hash-keyed store paths | **a substituter list you configured** | ⭐ **won in its niche** |
| **BitTorrent** | swarm distribution | **tracker first, DHT as fallback, PEX once in** | ⭐ **won; still the reference** |
| **Ceph / CRUSH** | datacenter object store | ⭐ **computed from an agreed map** | ⭐ **won in the datacenter** |
| **Syncthing** | device-pair sync | **a global discovery service, `device → addresses`** | **alive, healthy, niche** |
| **git-annex** | location-tracked annexes | ⭐ **a published location log of beliefs** | **alive, 15 years, niche** |
| **Tahoe-LAFS** | capability-based grid | **storage index from the encryption key** | **alive ~18 years, very niche** |
| **Tor onion services** | anonymous services | ⭐⭐ **computed blinded index over a hash ring** | ⭐ **won in its niche; v2 replaced** |
| **Perkeep** | personal permanode store | **your own index — a disposable projection** | **alive, niche, low activity** |
| **Peergos** | encrypted P2P filesystem | **IPFS + block access tokens** | **alive, small** |
| **Spacedrive** | fleet file manager | **nobody — membership is complete** | **alive, active, single-user** |
| **IPFS** | content-addressed P2P | **Kademlia provider records, then an indexer** | ⚠ **alive; the lookup layer kept being replaced** |
| **Filecoin / Storj / Arweave** | paid decentralized storage | **market + indexers** | ⚠ **alive; ~28% utilized, pivoting** |
| **Dat / Hypercore / Beaker** | P2P web | **Hyperswarm DHT** | ⛔ **Beaker discontinued 2021** |
| **Upspin** | global name + store | **DirServer, one global KeyServer** | ⛔ **being shut down** |
| **Academic DHT wave** | CAN, Chord, Pastry, Tapestry, OceanStore, PAST, CFS, Pond | **the DHT** | ⛔ **essentially none deployed** |

### §1.1 The pattern in the outcome column

**Read the winners' column and the answer is the same four words every time: *a place you configured*.**
Git, OCI, Nix, and BitTorrent's tracker are all *"ask this address."* Ceph and Tor compute against a
map that a membership process agreed. **Nothing in the won column discovers holders in an open network
without either configuration or consensus.**

**And the two systems in the dead column that were best-designed died of the same thing:**

- **Upspin** federated storage and naming — run your own DirServer and StoreServer — **and centralized
  identity in one global KeyServer.** The maintainers' own shutdown note is candid: it *"was intended
  to foster communities of sharing but has been less successful at that than hoped,"* so the central
  infrastructure is being turned down. ⭐ **The single point whose shutdown ends the public namespace
  was the key server** — and a second stated reason was crypto: continued uncertainty about a
  cryptographically-relevant quantum computer, a risk they felt they could not responsibly leave
  ordinary users exposed to. *(Garbage collection, they add, had always been the sore point.)*
- **Beaker** shut down in September 2021, and the reason its author gave is the one this corpus has
  reached independently: ***Beaker apps had no backend.*** His follow-on project was a server.

> ⭐ **Both failures are ours to avoid and we have partial answers to both already.** Against Upspin's:
> this corpus's registry is *"just a peer"*, **anyone can publish a binding claiming any name, and
> receiver policy decides** — there is no key server to turn off. Against Upspin's crypto reason: the
> wire carries a `key_type` and a `content_hash_format`, so the agility is structural rather than a
> migration. Against Beaker's: the always-on-tier finding is the same conclusion from the other
> direction. **Three of our least glamorous decisions are the ones the dead column argues for.**

---

## §2 Why the built lookups did not work

### §2.1 The academic wave's own post-mortem

2001–02 produced CAN, Chord, Kademlia, Pastry and Tapestry, and the following five or six years
produced filesystems (CFS, Ivy, OceanStore, Pond, PAST), naming systems, query processors and
CoralCDN on top of them. Teaching retrospectives now answer *"why don't services use P2P?"* with:

1. **latency and bandwidth** between peers, against intra- and inter-datacenter networks
2. **peer unreliability** against managed servers
3. ⭐ **lack of trust in peer behaviour** — *securing DHT routing is hard and unsolved in practice* —
   plus **churn**, worse as `log(n)` grows

**Two details worth more than the summary.** *"How to Make Chord Correct"* exists because **the ring
is disrupted when nodes join, leave or fail** — written a decade after a paper that was the
fourth-most-cited in computer science and won a Test-of-Time award. **Citation impact and deployability
came apart completely.** And the design space exploded — CAN, Chord, Pastry, Tapestry, Plaxton,
Viceroy, Kademlia, Skipnet, Symphony, Koorde — *each extensively analysed in isolation*, which is why
comparing them needed its own paper.

### §2.2 The deployed one kept replacing its lookup layer

IPFS is the surviving member and its lookup layer has been rebuilt around it twice: Kademlia provider
records **decay** (the >70%-unreachable figure already in this corpus), the response was **IPNI**, an
indexer explicitly *"outside and independent of the Kademlia DHT"* — **complementary, not a
replacement**, though one academic reading calls it *"a centralized version of the DHT… serving
primarily large content providers."* **Reader privacy then had to be retrofitted on top** (§3).

### §2.3 BitTorrent is the honest comparison, and it never removed the tracker

- **The tracker is still the fast path.** Public swarms announce to trackers *and* DHT simultaneously;
  DHT is the thing that keeps a swarm alive when a tracker dies. **Hybrid, not replacement.**
- ⭐ **PEX cannot bootstrap** — initial contact requires a tracker or a DHT bootstrap node.
- ⭐⭐ **The `private` flag disables DHT and PEX**, so all peers go through one tracker with full
  visibility of the swarm — *because standard DHT protocols lack authentication and any peer could
  otherwise join and undermine reputation-based admission.* **The moment access control was required,
  the distributed lookup was switched off.**

---

## §3 ⭐⭐ The privacy answer exists, is deployed, and corrects our open question

The previous document established that the lookup key cannot be the content hash — a confirmation
oracle on the publisher side and a disclosure on the reader side — and left *what the key should be*
as the hard open question. **There are now four answers in front of us, and they agree on the
principle.**

| | the derivation | what it buys | what it costs |
|---|---|---|---|
| **Tahoe-LAFS** | storage index = **tagged hash of the encryption key** | looking it up and being entitled to read it are **one capability**; servers can deny, never read or forge | everything must be encrypted |
| ⭐⭐ **Tor v3** | index = **blind(identity key, time period, shared random)**, client-computed, daily-rotating | **unlinkability** — the identity cannot be recovered from the blinded key, so a directory cannot learn what it is serving; descriptor encrypted to the address | needs an agreed **time period** and an agreed **shared random**, i.e. a consensus |
| **Peergos** | **block access token**, derivable from the block, carried in the capability | **stateless server-side auth on retrieval**; no token ⇒ the block is public | a token layer beneath the content layer |
| **IPFS / IPNI** | `hash(hash(content))` + encrypted provider records | an observer *"cannot know what a user is looking up without already knowing the original CID"* | **k-anonymity over a prefix only**, per the PIR literature |

> ⭐ **The principle all four share, and it is the design rule to carry: THE LOOKUP KEY IS DERIVED FROM
> THE THING THAT AUTHORIZES READING.** Tahoe derives it from the decryption key, Tor from the identity
> key, Peergos from the block, IPNI weakly from the content. **Three different domains — distributed
> storage, anonymity networks, encrypted filesystems — reached the same construction independently.**

**Tor v3 is worth reading closely because it is the closest to our shape**, and because **v2 was our
drafted design and failed in exactly the predicted way:**

- v2 asked an HSDir for a descriptor id **derived directly from the service identity**, which
  *"allowed harvesting of onion addresses"* — and collecting them via HSDirs is **classified as
  malicious behaviour and sanctioned.**
- v3's client **computes the current index itself** from the address and the date. **No lookup service
  is consulted to learn what to ask for** — Strategy-B construction, with privacy.
- Placement is a **deterministic hash ring** with `hsdir-n-replicas` and `hsdir-spread-fetch` as
  consensus parameters. ⭐ **That is CRUSH with a replication factor, inside an anonymity network.**
- ⭐ **The shared-random value exists to stop an attack on deterministic placement**: in v2 an adversary
  could **grind relay keys to land on a target's ring position.** *Anyone building computed placement
  needs unpredictable-but-agreed randomness, and this is why.*
- **Residual leakage is real and stated**: an HSDir still learns *that something exists at that ring
  position* and *when it is fetched*, it can selectively deny queries, and an adversary who already
  knows the address can compute the blinded key and watch for it. The academic proposal on top is
  **PIR**.

**The honest limit for us: the Tor construction needs a time period and a shared random, which means a
consensus we do not have.** ⚠ **But the requirement is much weaker than Tor's** — we need agreement on
a *rotation epoch*, not on a routing table, and a coarse wall-clock epoch may be enough. **That is a
tractable design question rather than an open one, and it is a different sentence from the previous
document's.**

---

## §4 What the capability-aware systems do that the studied corner does not

| | read authorization | metadata protection | who can fetch a raw block |
|---|---|---|---|
| **BitTorrent / IPFS** | none | none | anyone |
| **Tahoe-LAFS** | **read-cap / verify-cap / write-cap** — and a **verify-cap audits integrity without the decryption key** | filenames in dirnodes are encrypted; **transitive read-onlyness** by encrypting child write-caps with the directory's write-cap | anyone, but shares are useless without a cap |
| **Peergos** | **cryptree** — each node holds the keys that unlock its children, so a capability grants exactly a subtree and nothing above | extended beyond the original cryptree to cover **file size, names, thumbnails and directory structure** | ⭐ **nobody without a block access token; no token ⇒ public** |
| **Willow** | **Meadowcap** | **private interest overlap** — salted interest hashes matched before disclosure | peers you sync with |
| **this corpus** | capability-typed throughout | namespace and tree scoping | ⚠ **undecided for a holder claim** |

**Three transferable mechanisms, none of which we have:**

- ⭐ **The verify-cap as a third authority level.** *Audit the integrity of a thing you may not read* is
  exactly the role a redundancy set needs — a third party confirming a holder's claim without being
  entitled to the content. **This composes with probe-before-record: the probe is a verify-cap
  operation.**
- ⭐ **Cryptree's shape** — keys that unlock children, so authority flows down a tree and never up.
  **This corpus's authority already follows the namespace tree**, and cryptree is what that looks like
  when the enforcement is cryptographic rather than by policy.
- ⭐ **Peergos's "no token ⇒ public" default.** A single mechanism covering both the public-content case
  and the private case, with the discriminator on the object. **That is the two-tier split a
  general-purpose system needs**, and it is one bit rather than two systems.

---

## §5 Placement, replication, and the gap the market priced

**The previous document established that the placement vocabulary is missing here and mature in
git-annex.** Three additions:

- ⭐ **The market has priced the gap.** Decentralized storage is **two orders of magnitude cheaper** and
  has roughly **28% utilization**, and one of the named technical barriers is *"an inability to
  provide asynchronous forward replication to ensure availability against failures."* **Placement
  policy is not a convenience feature — its absence is a cited reason a 121×-cheaper product does not
  displace the incumbent.**
- **Tor's `hsdir-n-replicas` is the minimal form**: a replication factor as a *consensus parameter*
  rather than a per-object setting. **Ceph puts it per pool, git-annex per file type, Tor per network.**
  Three granularities; the common factor is that **none of them is per-peer**.
- **Erasure coding is the other half and is absent from this corpus entirely.** Tahoe's K-of-N
  Reed-Solomon with 3-of-10 typical is the reference. ⚠ **It interacts badly with a pure holder-set
  model** — with erasure coding *no single holder has the bytes*, so *"who has `H`"* stops being the
  right question and becomes *"can I assemble K shares."* **A design that assumes whole-object holders
  forecloses erasure coding, and that should be a deliberate choice rather than an accident.**

---

## §6 The two personal-fleet mechanisms worth stealing

- ⭐ **Syncthing's announcement does not carry the device id.** The id is **a hash of the device's TLS
  certificate**, so the mutual-TLS handshake itself proves it and ids cannot be spoofed. **Announce is
  an HTTPS POST of an address list; lookup is a GET with the device id; 404 means well-formed but not
  registered.** Two further details: an **empty address means "use the source IP of this
  announcement"**, and the discovery server is authenticated by **certificate pinning through a
  synthetic `id=` parameter that is never sent to the server.** *(The same trick as a capability in a
  URL fragment: the verification material rides in the locator and is not transmitted.)* **Their docs
  state the leak plainly** — whoever runs the discovery service can map device ids to IP
  addresses and deduce which devices connect to each other.
- ⭐ **Perkeep's index is a disposable projection.** *"You can lose your index at any time — delete it,
  re-replicate from storage, and search works again."* A permanode is **a signed random number**; its
  state is *the result of combining all attribute-modifying claims that reference it, in order*; **the
  signer is the owner, others may publish their own mutation claims, and the owner decides the policy
  on how mutations are respected.** GC is anchored by explicit `keep` claims, and deletion resolves at
  query time — **a delete claim can itself be deleted.** ⚠ **The operational detail worth copying: an
  out-of-order indexing loop, because blobs arrive before the claims and permanodes that reference
  them.**

---

## §7 What this changes in our own record

| we said | after this read |
|---|---|
| *no system read here has both a capability model and a content lookup* | ⛔ **wrong — Peergos and Tor v3 both do**, and both are more relevant than the systems that lack it |
| *what the lookup key is, is the hard open question* | ⭐ **substantially answered — four deployed derivations agreeing on one principle**, and the residual for us is a rotation epoch, not a construction |
| *the content lookup is an accelerator, not a resolution path* | ⭐ **confirmed at the largest available scale — the systems that won never built it**, and answer "who has it" from configuration |
| *placement policy is a separable deliverable* | ⭐ **strengthened — its absence is a cited reason a 121×-cheaper product has ~28% utilization** |
| *(no position on erasure coding)* | ⚠ **a whole-object holder set forecloses it**; decide deliberately |
| *(no position on outcomes)* | ⭐ **the two best-designed dead systems died of a central trust root and of having no backend** — and three of our least glamorous decisions are the defences |

**Two corrections in two sessions, both from reading the field, both of confident negatives.** The
pattern is worth naming: **this corpus's negatives about the outside world have a poor record, and they
fail in the same direction every time — asserting that something does not exist because we had not
looked.**

---

## §8 The frontier — what nobody has

**Stated so it is clear we are not failing to find a known answer.**

1. ⛔ **An authenticated, capability-scoped content lookup in an open network.** BitTorrent switched
   discovery off rather than solve it; a 2025 proposal for an authenticated Kademlia exists precisely
   because it is unsolved. **Peergos is the closest deployed thing and it authorizes *retrieval*, not
   *discovery* — you still have to know which block to ask for.**
2. ⛔ **Reader privacy without a consensus.** Tor's blinding needs a time period and a shared random;
   IPNI's double hashing needs an indexer to hold encrypted records; PIR is the only construction that
   needs neither and it is expensive and unshipped.
3. ⛔ **A holder set that composes with erasure coding.** Every system does one or the other.
4. ⚠ **Placement policy that does not oscillate**, which git-annex reached empirically with `or
   present` and which nobody has stated as a general rule.

---

## §9 What would test this

- **The §0.1 claim is falsifiable by one counterexample**: a planetary-scale content-addressed system
  that answers *"who has `H`"* without configuration or consensus. **We did not find one.**
- **The blinded-index construction is testable on paper**: derive an index from an epoch and check
  whether a party who does not already hold the content can enumerate holders. If they can, §3 does
  not transfer.
- **The erasure-coding tension is a design test, not a measurement**: write a holder claim for a peer
  holding 3 of 10 shares and see whether the schema can express it. If `partial` suffices, §5's
  warning is weaker than stated.

---

## §10 Sources

**Read for this document.** Perkeep — [schema](https://perkeep.org/doc/schema/),
[permanodes](https://perkeep.org/doc/schema/permanode), [terms](https://perkeep.org/doc/terms),
[overview](https://perkeep.org/doc/overview), [`pkg/index`](https://perkeep.org/pkg/index),
[`pkg/search`](https://perkeep.org/pkg/search) ·
Upspin — [overview](https://upspin.io/doc/overview.md),
[**"Turning down Upspin infrastructure"**](https://groups.google.com/g/upspin/c/Whma_O-iexM/m/lSConHZ5DwAJ),
[LWN, "Giving Upspin a spin"](https://lwn.net/Articles/716409/) ·
Peergos — [README](https://github.com/Peergos/Peergos/blob/master/README.md),
[block access control (issue 862)](https://github.com/Peergos/Peergos/issues/862),
[architecture slides](https://speakerdeck.com/ianopolous/peergos-architecture) ·
Hypercore / Beaker — [protocol rename](https://blog.dat-ecosystem.org/dat-protocol-renamed-hypercore-protocol/),
[Beaker on IndieWeb](https://indieweb.org/Beaker) ·
Syncthing — [Global Discovery v3](https://docs.syncthing.net/specs/globaldisco-v3.html),
[security principles](https://docs.syncthing.net/users/security.html) ·
BitTorrent — [peer exchange](https://en.wikipedia.org/wiki/Peer_exchange),
[private trackers](https://wiki.pulsedmedia.com/wiki/Private_tracker),
[TorrentFreak on DHT/PEX/magnets](https://torrentfreak.com/bittorrents-future-dht-pex-and-magnet-links-explained-091120/),
[*Persistent BitTorrent Trackers* (arXiv:2511.17260)](https://arxiv.org/pdf/2511.17260) ·
the DHT wave — [Princeton COS 418, *P2P Systems and DHTs*](https://www.cs.princeton.edu/courses/archive/fall19/cos418/docs/L8-dhts.pdf),
[Stoica, CS 268 DHT notes](https://people.eecs.berkeley.edu/~istoica/classes/cs268/06/notes/21-DHTsx2.pdf),
[*How to Make Chord Correct* (arXiv:1502.06461)](https://arxiv.org/pdf/1502.06461) ·
decentralized storage — [TechTarget comparison](https://www.techtarget.com/searchstorage/tip/Comparing-4-decentralized-data-storage-offerings),
[Securities.io comparison](https://www.securities.io/decentralized-storage-filecoin-arweave-storj-comparison/) ·
content addressing in packaging — [Nesbitt, *Content addressing in package managers*](https://nesbitt.io/2026/07/07/content-addressing-in-package-managers.html),
[*What package registries could borrow from OCI*](https://nesbitt.io/2026/02/18/what-package-registries-could-borrow-from-oci.html),
[NixOS/nix#8400, OCI registry as binary cache](https://github.com/NixOS/nix/issues/8400) ·
Tor — [V3 onion services usage](https://blog.torproject.org/v3-onion-services-usage/),
[*Improving the Privacy of Tor Onion Services* (IACR ePrint 2022/407)](https://eprint.iacr.org/2022/407.pdf).

**From this project's own record.**
`EXPLORATION-FIVE-KEYED-LOOKUPS-ARE-ONE-MACHINE-AND-THE-CONTENT-ADDRESS-LOOKUP-IS-THE-REDUNDANCY-CASE` ·
`EXPLORATION-THE-CONTENT-LOOKUP-LANDSCAPE-COMPUTE-LOOK-UP-OR-PUBLISH-AND-WHY-THE-KEY-CANNOT-BE-THE-CONTENT-HASH` ·
`EXPLORATION-WILLOW-THE-NEAREST-NEIGHBOUR-READ-AGAINST-ITS-SPEC` ·
`EXPLORATION-THE-OPERATING-MODELS-AND-THE-ALWAYS-ON-TIER` ·
`EXPLORATION-CONTENT-ADDRESSED-SUPPLY-CHAIN` · `EXPLORATION-THE-SHAREABLE-REFERENCE-WHY-A-NAME-AND-A-KEY-ARE-NOT-REDUNDANT` ·
`EXTENSION-REGISTRY` §1, §5 · `APP-CONVENTION-REFERENCE` §1.
