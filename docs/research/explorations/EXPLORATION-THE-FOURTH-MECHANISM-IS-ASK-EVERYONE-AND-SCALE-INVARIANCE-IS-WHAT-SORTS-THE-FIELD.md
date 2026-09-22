# EXPLORATION — the fourth mechanism is *ask everyone*, and scale-invariance is what sorts the field

**Status:** EXPLORATION (2026-09-13). Written to be attacked. Nothing here is ruled. **No calls are
being made yet** — this completes the survey so that analysis has something to stand on.

**Fourth in a sequence.** `FIVE-KEYED-LOOKUPS` established what the fifth lookup is ·
`THE-CONTENT-LOOKUP-LANDSCAPE` read nine systems and found the three mechanism families ·
`THE-CONTENT-ADDRESSED-LANDSCAPE` added the outcome axis and corrected two rows. **This one adds a
fourth mechanism that our own archive already specified, a design criterion that sorts the field
better than any technical axis, and a narrowing of this arc's loudest claim.**

**Read for this document:** Hyphanet/Freenet (architecture, the darknet papers, and its adoption
record) · GNUnet R5N and CADET · Veilid · MaidSafe/Autonomi · Amazon Dynamo · **and two documents
from this project's own pre-split archive that specify the fourth mechanism in full.**

---

## §0 The result

**Nine findings. The first two change the frame; the third narrows this arc's loudest claim.**

1. ⭐⭐⭐ **There is a FOURTH mechanism and this project designed it twice: ASK EVERYONE.**
   Not compute, not look up, not publish — **propagate a capability-gated query, bound it, and
   aggregate the answers.** The pre-split archive carries it at two core revisions as
   **`QUERY { peer: "*", expand: true, ttl: 2, seen_peers: [...] }`**, with hop-decrement, loop
   prevention, progressive partial results streaming back before a `complete: true`, and — the part
   that makes it more than a flood — ***it doubles as peer discovery: "you learn about new peers by
   querying."*** **It is not in the current corpus**, but it is not forgotten either:
   `EXTENSION-QUERY` §1.2 lists *"cross-peer query handler (Level 3, future — types defined here are
   reusable)"* as declared future work. **The right correction to the previous documents is that
   *ask everyone* is SCALE-BOUNDED, not wrong** — it is the correct mechanism on a home network and
   it does not survive the open internet, which is a statement about its range rather than its
   validity.

2. ⭐⭐⭐ **Scale-invariance is the design criterion, and it sorts the outcome column better than any
   technical property we have tried.** *Does the system do something useful for one user, with no
   adoption at all?* **Every system alive in the niche column answers yes** (Git, Nix, git-annex,
   Syncthing, Perkeep, Spacedrive). **Every system in the dead or struggling column answers no**, and
   several say so in their own marketing — Veilid's value proposition is literally *"the more apps and
   nodes running Veilid, the more private the system becomes"*, and Freenet's anonymity set is its
   user count. **A design whose first user gets nothing has to cross an adoption chasm before it
   delivers value; a scale-invariant one delivers on day one and gets better.** §2 scores the field.

3. ⭐⭐ **This arc's *"the key is a confirmation oracle"* claim is NARROWER than it was written.**
   **Knowing a hash is not access here, and that is structural rather than incidental.** There is
   always a second hop: a capability, or an encryption key, or a relationship with the peer holding
   the bytes plus their permission to read out of a content store in a namespace that is itself
   resource-protected. **So what a holder claim leaks is EXISTENCE and INTEREST, not ACCESS** —
   which is a real leak worth designing against and **not the critical path this arc made it.** §4.

4. ⭐⭐ **The sharpest adoption lesson in the whole survey is Freenet's, and it is about what you make
   users host.** Its own design — every node stores encrypted blocks it did not choose and cannot
   inspect — **is simultaneously the source of its censorship resistance and its biggest adoption
   barrier.** The summary judgement in the review literature is worth carrying verbatim in substance:
   *it optimized for a threat model most people do not face, and charged them latency, disk space, an
   unfamiliar interface, and the discomfort of hosting unknown encrypted data to get there.*
   ⇒ ***A mechanism that requires users to host content they did not choose has a ceiling, whatever
   its technical merit.*** **Our model is the opposite and this is an argument for keeping it that
   way.**

5. ⭐⭐ **GNUnet supplies a SECOND, independent reason a Kademlia-style DHT does not fit here, and it
   is stronger than the one on record.** Our recorded reason is that our peers are static origins that
   cannot answer a live lookup. GNUnet's is structural: ***XOR proximity says nothing about whether a
   connection can actually be established***, so NAT'd nodes make Kademlia *"exhibit degraded
   performance or even fail"* — the **local-minimum trap**, where greedy descent stalls because no
   reachable neighbour is closer to the target. **`EXTENSION-NETWORK` §6.7 is entirely about restricted
   reachability, so we are the documented failure case, not an edge of it.**

6. ⭐ **The winners-have-no-lookup finding survives but its scope is narrower than stated.** OCI, Nix
   and the package registries are **curated corpora with a publisher and a bounded, append-mostly
   dataset** — selective about what enters the hash space at all. **Git is the general-purpose
   exception and its answer is still a configured remote.** ⇒ the finding proves *configuration works
   for a curated corpus*, **not** *configuration works for open content*, and the earlier document
   overstated the reach.

7. ⭐ **Dynamo contributes the most directly reusable mechanism in the survey: hinted handoff** — a
   substitute holder stores data **with a hint naming whom it really belongs to**, and hands it back on
   recovery. **That is a holder claim with an owner pointer**, and it is what lets a redundancy set
   tolerate churn without lying. Its **three-layer repair ladder** and its **per-request consistency
   knob** transfer too. §6.

8. ⭐ **Freenet resolves the erasure-coding tension rather than contradicting it.** It has **both**
   FEC redundancy and a content lookup, because it **erasure-codes into blocks and then
   content-addresses each block**: the holder set is per-share, and K-of-N assembly is the reader's
   job. **So the earlier warning stands as written but has a known answer** — code first, address the
   shares, and the holder question is unchanged.

9. **Timescales, for calibration.** MaidSafe/Autonomi — the closest comparator in ambition to this
   project — was founded in **2006** and reached mainnet in **February 2025**, roughly nineteen years,
   with its own executives calling it *the world's oldest startup* and long-time supporters noting the
   launch had been *"close"* since about 2018. **Veilid launched in 2023 after three years and its
   flagship application still has no released build.** *This is context for scope, not an argument
   about anything.*

---

## §1 The fourth mechanism, as we already specified it

**Two archive documents, at two core revisions.** The v1.0 core protocol specification, §7.3:

```
QUERY { type: "QUERY", peer: "*", expand: true, ttl: 2, seen_peers: ["Qm7X9a3b..."] }
```

with four rules: **decrement TTL on forward · add self to `seen_peers` · do not forward at TTL ≤ 0 ·
do not forward to anyone in `seen_peers`.** Results stream back progressively —
`RESPONSE { sender, partial: true, entities: [...] }` as each peer answers, then
`RESPONSE { complete: true }` at timeout — so *"applications can display results progressively."*

The v0.5 architecture document states the property that makes it more than a flood:

> **Local sends to peers it knows; a peer that knows someone the originator does not forwards to them;
> the answer comes back up the chain — so the originator learns about that peer's data AND discovers
> the peer exists.** *"This enables network discovery through queries."*

**And fan-out is listed there as a first-class communication pattern**, beside request/response, RPC,
streaming and pub/sub: *"Fan-out — `QUERY(peer:*)` → multiple `RESPONSE` — distributed discovery."*

### §1.1 What it is good for, and where it stops

| | |
|---|---|
| **Works** | a home network · a fleet you own · a small trusted group · **any bounded membership where the fan-out terminates** |
| **Fails** | the open internet, for the reason every flooding network failed — the query volume is `O(peers)` per lookup and the TTL that bounds it also bounds the answer |
| **Its unique property** | ⭐ **it needs no index, no map, no consensus, and no prior knowledge of who holds anything** — which is exactly why it is the right answer in the case where the other three mechanisms are overkill |
| **Its second property** | ⭐ **discovery falls out of it.** A query is also how you learn the graph — the only one of the four mechanisms with that side effect |

**The precedent is not encouraging at scale and is decisive at small scale.** Gnutella flooded and
did not survive contact with the open internet; BitTorrent's PEX is the bounded version — peers
exchanging swarm membership directly, *"faster and more efficient than relying on one tracker
alone"* — **and it cannot bootstrap**, which is the same TTL-bounded limitation seen from the other
end.

> ⭐ **So the honest taxonomy is four, and the fourth is the one the corpus already owns.** The
> previous document's three were **COMPUTE · LOOK UP · PUBLISH**; **ASK** belongs beside them, and it
> is the one that needs nothing and scales worst. **That is a coherent position, not a defect** —
> the scope decides, which is the same argument already made for the other three.

---

## §2 Scale-invariance, scored

**The question: does this system do something useful for its first user, before anyone else adopts
it?** Nothing else in this survey separates the outcomes as cleanly.

| System | useful at N=1? | why | outcome |
|---|---|---|---|
| **Git** | ⭐ **yes** — a local repository is the product | versioning is single-user value | **won** |
| **Nix / OCI** | ⭐ **yes** — reproducible local builds, local image store | caches are an optimization | **won** |
| **git-annex** | ⭐ **yes** — track large files in one repo | the location log has one entry and is still useful | **alive, 15 yrs** |
| **Syncthing** | **two** — needs a second device, which one person owns | no third party required | **alive, healthy** |
| **Perkeep** | ⭐ **yes** — a personal store, indexed | search over your own things | **alive, niche** |
| **Spacedrive** | **yes** — one device indexed, more is better | fleet value is additive | **alive, active** |
| **Ceph** | **no** — needs a cluster | but the cluster is one owner's | **won in the datacenter** |
| **Tahoe-LAFS** | **no** — needs a grid of storage servers | the grid may be your own friends | **alive, very niche** |
| **BitTorrent** | ⛔ **no** — a swarm of one is a file copy | value is strictly in others | **won anyway — see below** |
| **IPFS** | ⚠ **partly** — local CAS works; retrieval needs the network | | **alive, contested** |
| **Freenet** | ⛔ **no** — **the anonymity set is the user count** | and you must host others' blocks first | **niche, ~flat since 2013** |
| **Veilid** | ⛔ **no** — *"the more apps and nodes, the more private"* | stated as the value proposition | **launched 2023, flagship unreleased** |
| **Dat / Beaker** | ⛔ **no** — a peer web with no peers | *"Beaker apps had no backend"* | ⛔ **discontinued** |
| **Upspin** | ⛔ **no** — needed the key server and a community | | ⛔ **shutting down** |
| **Autonomi / SAFE** | ⛔ **no** — a storage market needs both sides | | **mainnet 2025, after ~19 yrs** |

**BitTorrent is the instructive exception and it proves the rule by a different route.** It is not
scale-invariant at all, and it won because it arrived with **an enormous pre-existing demand and a
bootstrap that was a single file you could email.** ⇒ **you can substitute a distribution channel for
scale-invariance, once, if the demand already exists.** Nothing in this survey did it twice.

> ⭐⭐ **Why this matters here specifically.** This architecture is scale-invariant by construction —
> one peer is a working system, two is a working system, an organization is a working system — and
> **that is a property to protect at every design decision in this arc, not a happy accident to
> mention.** A holder-set design that only pays off at internet scale would trade away the one
> property the field's outcome column says is decisive.
>
> **It also resolves the fleet-versus-global tension directly**: the fleet case is not a degenerate
> version of the global case that we tolerate. **It is the case that has to work first**, and the
> global case is the one allowed to be best-effort.

---

## §3 The restricted-route argument, which is stronger than the one on record

GNUnet's R5N exists because of a mismatch this corpus has documented extensively without connecting
it to the lookup question:

- **Restricted-route topologies arise when the underlay prevents direct connections between some
  nodes — commonly NAT.** Nodes behind NAT make *"common DHT routing algorithms such as Kademlia
  exhibit degraded performance or even fail."*
- ⭐ **The reason is precise: XOR proximity says nothing about whether a connection can be
  established.** Two nodes can be neighbours in the overlay metric and unable to speak at all.
- ⭐⭐ **The failure mode is the local-minimum trap**: greedy descent stalls when the reachable
  neighbour set contains nobody closer to the target, so *"a node may only be able to publish and
  retrieve data in the proximity of its local minima."*

**R5N's fix is to randomize before it optimizes** — a random walk for a network-size-estimated number
of hops, *then* deterministic XOR routing — with a **Bloom filter in the message** for loop
prevention (needed because random-walk-plus-branching would otherwise revisit peers), **path
recording** so applications can use the DHT to discover routes, and ⭐ **an explicit replication level
`r` carried in the request**, achieved by probabilistic branching. Its cost is stated plainly: **it
requires a high number of replications.**

**CADET is the other half** — end-to-end channels over multi-hop forwarding where *"peers
retransmitting traffic on behalf of other peers cannot access the payload."*

> **The unifying thesis is directly ours: both subsystems reject the assumption that any node can dial
> any other node.** `EXTENSION-NETWORK` §6.7 exists because we know our underlay is restricted — the
> observed-address reflection, the dial-back, the held-outbound-socket asymmetry. ⇒ **We are the
> documented failure case for XOR-greedy routing, and that is a second independent reason to decline
> the DHT, on different grounds from the one already recorded.**

**Two R5N mechanisms are worth keeping regardless of the DHT decision:** a **replication level carried
in the request** rather than configured at the node, and **path recording** as a discovery byproduct —
which is the same insight as the archive's *"you learn about new peers by querying."*

---

## §4 What a holder claim actually leaks

**The correction, stated precisely, because this arc overstated it.**

| | is it leaked by publishing *"I hold `H`"*? | |
|---|---|---|
| **the bytes** | **no** | ⭐ a hash is not a capability here — retrieval needs a grant, or a key, or a relationship plus permission, over a namespace that is itself resource-scoped |
| **that `H` exists** | **yes** | to anyone who can guess or compute `H` |
| **that this peer has it** | **yes** | the association is the disclosure |
| **that YOU want `H`** | **yes, on the reader side** | and this is the one with a second-order effect |

⭐ **The second-order effect is the interesting half and it is not about access at all.** Asking *"who
has `H`"* tells the answerer that `H` is worth having — **so the query itself creates interest in a
thing the answerer may not have known existed and may now go looking for.** That is an *amplification*
leak rather than a *disclosure* leak, and none of the four derivations in the previous document is
aimed at it. Tor's blinding hides *which* service; it does not hide *that someone wants something at
this ring position*, which the Tor literature states as surviving leakage.

**So the honest position:**

- ⛔ *"the content hash as a key is a privacy DEFECT that must be fixed before anything is built"* —
  **overstated, and withdrawn as written.**
- ✅ *"a holder set keyed on the content hash is an existence-and-interest disclosure, it is the one
  surface where this architecture's usual second hop does not protect you, and the deployed
  derivations exist if we want them"* — **supported, and it is a design input rather than a blocker.**
- ⭐ **And the public/private split is already expressible**: the two-tier default seen in the
  capability-aware systems — *no token means public* — maps onto an architecture where most content
  is already namespace-scoped. **The question is not how to protect everything; it is what the
  default is.**

---

## §5 What Freenet costs its users, and why that is the survey's best adoption lesson

**The architecture, briefly, because parts of it are ours under different names.**
**CHK** is a content-hash key for immutable data; **SSK** is a signed namespace for updates under a
publisher's key; **USK** builds versioned updates on SSK. ⭐ *That is the pinned reference, the live
reference and the versioned root — designed in 2000.* Blocks are 32 kB (CHK) and 1 kB (SSK) with
**Vandermonde FEC** for redundancy and **LRU eviction favouring frequently-accessed content**.

**Routing is a small-world DHT** — nodes take locations in `[0,1)`, greedy routing steers toward the
location closest to the key, **successes leave routing hints and caches that improve future
requests.** ⭐ **And the darknet mode is the field's best answer to the scale question**: nodes connect
only to peers they trust, added manually by invitation, *"not harvestable because your node reference
is never passed on"* — **and routing works over a MIXTURE of darknet and opennet links, so people with
only a few friends get adequate performance while keeping some of the security.** That is a
graded-membership design rather than a binary one, and it is the closest thing in the survey to
serving the fleet, the group and the world with one mechanism.

**Its known weakness is worth recording beside our own computed-placement interest:** the **Pitch
Black attack** on its location-swapping algorithm was published in 2007 and the fix landed in a much
later build, with outside funding raised specifically to address the routing problem. ⚠ **Deterministic
or swapped placement is attackable, and this is the second instance in this survey** — the first being
the key-grinding attack that Tor's shared-random value exists to stop.

**And the adoption record is the lesson.** Measured node counts peaked in the low thousands
concurrently — roughly 2,500–3,600 active in 2012, with no large-scale public metrics after 2013 —
against a project in continuous development since before Tor. The stated barriers: it is slow; it
handles static content with dynamic features emulated; the interface is nothing like the web; **and
every user stores encrypted fragments they did not choose and cannot inspect.** A 2020 traffic
analysis reporting that a large fraction of requests concerned illegal material, and prosecutions
following node infiltration, added reputational drag *and* undercut the anonymity that justified the
cost. A 2023 rename to Hyphanet split the remaining mindshare.

> ⭐⭐ **The transferable rule: the price you charge a user before they get value is the thing that
> caps adoption, and hosting other people's opaque content is the most expensive price in this
> survey.** Freenet charged it for a threat model most users do not have.
>
> **Our position is the opposite by construction and should stay there.** A peer holds *its own
> namespace*, things it deliberately fetched, and things it deliberately chose to mirror. **If a
> placement or redundancy design ever implies "and you will also store blocks you did not choose,"
> that is the moment to stop and check this row.**

---

## §6 Dynamo, and the mechanisms that transfer

**Partitioning.** Consistent hashing with **virtual nodes**, so a join moves only about `1/N` of the
data and rebuild work spreads across many peers rather than one successor.

**The preference list** is the set of nodes a key replicates to, **deliberately constructed to span
failure domains** — across data centres, skipping virtual nodes that map to the same physical host.
*Same idea as CRUSH's rules over a topology, reached independently.*

⭐ **Sloppy quorum.** Operations go to the first `N` **healthy** nodes in the preference list, not the
first `N` on the ring. `R + W > N` still holds, but over healthy nodes.

⭐⭐ **Hinted handoff — the most directly reusable mechanism in this survey.** A substitute node stores
the data **with a hint recording whom it actually belongs to**, and transfers it back when the owner
returns, discarding its copy. This is what makes the system *"always writeable."* **In our vocabulary
that is a holder claim carrying an owner pointer** — *"I have these bytes, they are not mine, and they
belong to `X`"* — which is exactly the claim a redundancy set needs in order to be honest about
temporary custody. ⚠ Its stated limits: it covers **transient** failure only, it needs low churn, and
**the substitute may itself go offline before handing back.**

⭐ **The repair ladder is three layers and the survey has nothing else like it:** hinted handoff for
transient failures · **read repair** for hot keys · **anti-entropy over Merkle trees** as the
exhaustive background backstop, reconciling cheaply by tree-hash comparison. *We have the Merkle
layer and neither of the other two.*

⭐ **Push the consistency knob to the caller.** Per-request `ConsistentRead`, or `ONE` / `QUORUM` /
`ALL` — *the caller decides request by request, and that per-request control is the feature.* **Our
expose-knobs-not-values discipline, confirmed from the datacenter side, and the numeric form of
acceptance-policy-is-the-reader's.**

⚠ **And the operational lesson:** ring membership changes were **manual by choice**, to stop automated
partitioning mistakes cascading. **Even with total control over the fleet, the map was changed by a
human.**

---

## §7 What remains unread

**Named so nobody re-derives the gap.** **Private information retrieval** beyond the two abstracts
read · **Tahoe's grid membership** in practice · **Perkeep's sharing model** (share claims) ·
**Hypercore's sparse replication** mechanics, which the outcome read did not reach ·
**Coral CDN and OpenDHT** as the two deployed survivors of the academic wave · **Autonomi's actual
data model** post-launch, now that it has one · and **NNTP flood-fill**, still the largest
ask-everyone network ever operated and the obvious comparator for §1.

---

## §8 What would test this

- **Scale-invariance is falsifiable on the outcome column**: name a system that is useless at N=1 and
  nevertheless won without a pre-existing demand and a single-file bootstrap. **We did not find one;
  BitTorrent is the near miss and it had both.**
- **The fourth mechanism is testable on a fleet**: implement the archived fan-out query with its TTL
  and `seen_peers`, on three peers, and measure whether the answer set is complete. If it is, the
  small-scale case needs nothing else built.
- **The leak narrowing is testable by attack**: hold a content hash for private content and attempt
  retrieval without a capability. If any path succeeds, §4's first row is wrong and the earlier,
  louder claim was right.

---

## §9 Sources

**Read for this document.** Hyphanet/Freenet — [project site](https://www.hyphanet.org/index.html),
[Wikipedia](https://en.wikipedia.org/wiki/Hyphanet),
[darknet wiki](https://github.com/hyphanet/wiki/wiki/Darknet),
[security summary](https://github.com/hyphanet/wiki/wiki/Security-summary),
[*The Dark Freenet* (Clarke, Sandberg, Toseland, Verendel)](https://www.cs.kent.edu/~javed/class-P2P13F/papers-2012/PAPER2012-freenet-0.7.5-paper.pdf),
[research papers index](https://freenet.org/about/papers/),
[NLnet, fixing the Pitch Black attack](https://nlnet.nl/project/Freenet-Routing/) ·
GNUnet — [R5N Internet-Draft](https://datatracker.ietf.org/doc/html/draft-schanzen-r5n-07),
[LSD0004](https://lsd.gnunet.org/lsd0004/),
[Grothoff's R5N lecture notes](https://grothoff.org/christian/teaching/2012/2194/r5n.pdf),
[CADET documentation](https://docs.gnunet.org/v0.20.x/developers/cadet/cadet.html),
[subsystems](https://docs.gnunet.org/latest/users/subsystems.html) ·
Veilid — [cDc announcement](https://cultdeadcow.com/tools/veilid.html),
[The Register](https://www.theregister.com/2023/08/12/veilid_privacy_data/),
[CyberScoop](https://cyberscoop.com/cult-of-the-dead-cow-veilid/) ·
MaidSafe / Autonomi — [repository](https://github.com/maidsafe/autonomi),
[FAQs](https://docs.autonomi.com/learn-more/faqs),
[FutureScot on the launch preparation](https://futurescot.com/ayr-tech-firm-building-new-internet-prepares-to-launch-network-after-18-years-of-development/),
[TechCrunch, 2016 alpha](https://techcrunch.com/2016/08/12/after-a-decade-of-rd-maidsafes-decentralized-network-opens-for-alpha-testing/) ·
Dynamo — [MIT 6.824 notes](https://pdos.csail.mit.edu/archive/6.824-2007/notes/l21.txt),
[insights on the Dynamo paper](https://www.hemantkgupta.com/p/insights-from-paper-part-ii-dynamo) ·
BitTorrent — [peer exchange](https://en.wikipedia.org/wiki/Peer_exchange).

**From this project's own record.** The pre-split archive's **v1.0 core protocol specification §7.3
*Query Expansion*** and **v0.5 architecture document, *Recursive Query Expansion***, both located via
the archive title index · `EXTENSION-QUERY` §1.2 (the declared Level-3 deferral) ·
`EXTENSION-NETWORK` §6.7 (restricted reachability) · `EXTENSION-ROUTE` §3, §4 ·
`APP-CONVENTION-REFERENCE` §1 · and the three predecessor explorations named at the head of this
document.
