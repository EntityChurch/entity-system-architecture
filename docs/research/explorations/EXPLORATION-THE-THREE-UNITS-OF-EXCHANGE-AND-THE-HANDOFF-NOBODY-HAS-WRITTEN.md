# EXPLORATION — three units of exchange, not one: where the model stops, and the handoff nobody has written

**Status:** analysis, 2026-09-13. **Opens by correcting three claims made in a prior summary of this
arc**, because two of them made the design look broken in a way it is not, and the third compared the
wrong layer to the wrong prior art.
**Question it answers:** *is this one mechanism for all peer-to-peer data exchange, and if not, where is
the seam and what is missing at it?*
**Rests on:** `EXTENSION-CONTENT` §1–§3 · `APP-CONVENTION-FEED` §7.6 · the two-level-index exploration
§2–§5d · the interaction-landscape exploration §2 · `EXTENSION-SIGNALING` §9 · `EXTENSION-RELAY` §11.1 ·
the peer-compositions body in the **pre-split archive**.

> **In one sentence.** There are **three** units of exchange — the **entity**, the **chunk**, and
> **nothing** — the first two are built and compose, the third is where the model correctly stops, and
> **the seam between the second and third is the one thing in this architecture that is neither
> specified nor deferred with a reason.**

---

## §0 Three corrections, because two of them are why this looked worse than it is

**① *"~4.2 files per item, against a tracker's ~6 bytes per peer"* was a category error.** Those are not
comparable quantities and putting them in one sentence implies a cost the design does not have.

| quantity | what it actually is |
|---|---|
| **~4.2 files per entry** | **publisher write-amplification in one implementation's current emitter** — an entry, its signature, an index-page binding, changed trie nodes. It is a *build* number, and that emitter is known to re-emit the whole archive rather than the 4 bindings that moved |
| **~6 bytes per peer** | **index storage**, in a DHT, per participant per subject |
| **~119 bytes per pin** | **our index storage**, per *item* — ***this*** is the one comparable to the 6 bytes, and the comparison is §4's, not a cost of the whole design |

**② *"The reader pulls the archive"* is false, and the design has said so in normative text the whole
time.** §1.

**③ FEED was compared to IPFS and BitTorrent. That is the wrong layer** — but the follow-up was also
wrong in a way that matters more. The comparable layer is `EXTENSION-CONTENT`; **and the useful answer is
not "we have that extension," it is that content addressing, dedup and retrieval-by-hash are UNIVERSAL
and type-agnostic — a blog entry needs none of that extension to be fetched by hash.** §2.1. **What is
absent is the distribution layer: which peer has these bytes.** §3, §4.

**④ *"The entity system is a control plane and a durable plane, not a data plane for ephemeral ordered
state"* — WITHDRAWN.** It asserted a capability limit from no measurement. **There is no architectural
boundary there; there is a cost envelope, its terms are hardware-dependent, and not one of them has ever
been measured.** §6.

**⑤ The "durable half vs ephemeral half" split — WITHDRAWN.** *Ephemeral* was invented for the sentence.
A replicated database on the other side of a seam is not ephemeral; **it is not entity-native**, which is
a different claim with different consequences. §6.3.

---

## §1 The read path already is *read the article, not the encyclopedia*

**`APP-CONVENTION-FEED` §7.6, the light-client property, normative:**

1. Fetch the publisher's signed root — **one signature**, check `seq` has not gone backward.
2. Read the index head — **a known key, tree-depth cost**.
3. **Read down to your cursor and stop.**

*"Everything read under that root is committed to by that one signature, because the trie is hash-linked
from the root. There is no chain to sync, no history to replay and no genesis to reach."*

⇒ **Reading one post is: one root fetch, one page (≈32 entries, ~4 KB), the entry, its signature.** Four
fetches. **A ten-year archive next to it costs nothing**, which is precisely the property the
key-addressed index exists to produce and which the measurements confirm holds exactly.

**So where did *pull the archive* come from? Three places, and only one of them is a reader:**

| | who pays | status |
|---|---|---|
| the publisher re-emits every binding to add one post | **the publisher**, invisible to readers | a build inefficiency; the property that fixes it is measured present and unused |
| the live leg enumerates a whole prefix to show 50 | **the reader** ⛔ | **a real defect, open** — the index is a publish artifact, so a live read has no index by construction |
| a mirror was one flat list fetched whole | **the reader** ⛔ | **fixed this week** — head plus pages |

⇒ **One of the three is a reader-facing archive pull and it is a bug in a fallback path, not the
model.** The other two are a publisher's own cost and a defect already closed. ***The alarm was
justified by the summary and not by the system.***

---

## §2 There are three units of exchange, and conflating them is what made this confusing

**The system does not have "a unit of exchange." It has three, layered, and you choose by what you are
moving.**

| | unit | addressed by | what moves | built? |
|---|---|---|---|---|
| **1** | **entity** — `{type, data}`, ECF-canonical, individually signed | content hash, bound at a tree path | a post, a claim, a record, a manifest, a pointer | ✅ landed |
| **2** | **chunk** — 1 MiB, content-addressed, deduplicated | content hash, in the shared content store | bytes: music, images, video, a 500 MB blob | ✅ landed (`EXTENSION-CONTENT`) |
| **3** | **nothing** — the system carries no payload | — | ephemeral, high-rate, ordered state | ⛔ **the seam is unwritten — §6** |

**They compose, and the 500 MB post is the worked example that proves it:**

```
app/feed/entry                      the post — a signed entity in your namespace
  └── body: Embed                   the escape hatch for what markdown cannot express
        └── pointer-payload         { tag: "pointer", hash: H }   ← one content hash
              └── system/content/blob        the chunk-list manifest  ← "analogous to a torrent file"
                    └── system/content/chunk × 500     1 MiB each, content-addressed, dedup'd
```

**Every property you would want falls out of layer 2 and is already normative:** deduplication (identical
chunks are identical entity hashes *regardless of which handler created them* — the content store is
shared), **resumable transfer** (*"a receiver tracks which chunks it has and only fetches missing
ones"*), per-chunk verification by entity hash, and **multi-peer distribution** (*"any peer with the
chunks can serve them"*).

⇒ **A reader who wants your post's text and not your 500 MB video fetches the entry and stops.** The
blob is one hash in it. **That is the whole point of the indirection and it is why the answer to *"do I
have to download it"* is no, at every layer.**

### 2.1 ⭐ Retrieval by hash is TYPE-AGNOSTIC — a blog entry does not need the blob path

**The diagram above shows how BYTES get inside an entry. It is not how an entry is FOUND, and reading it
as though it were is the misdirection worth removing.**

**The content store is `Hash → Entity` and it holds everything**, not just blobs and chunks. The system
content handler's `get` is, in the spec's own words, *"hash-addressed entity retrieval. Returns **any**
entity type — blobs, chunks, capabilities, type definitions, or any other entity in the content store.
The operations are **type-agnostic**."*

⇒ **A blog entry published as an ordinary record is retrievable by its hash, with no chunking extension
anywhere in the story.** *"Give me hash H"* is answered for a 400-byte post exactly as for a 1 MiB chunk.

**So the three properties people mean by *"does it work like a content network"* sort like this:**

| property | for a blog entry | why |
|---|---|---|
| **content addressing** | ✅ **universal** | every entity is addressed by the hash of its canonical `{type, data}` — there is no opt-in |
| **deduplication** | ✅ **universal, and cross-type** | identical bytes produce an identical hash *regardless of which handler wrote them*; the store is shared |
| **retrieval by hash** | ✅ **type-agnostic** | one operation, any entity |
| **finding WHO HAS IT** | ⛔ **absent — §4** | |

### 2.2 So what is chunking actually for, and is it mandatory?

**Chunking is a SIZE strategy, not an addressing requirement, and it is forced by exactly one thing.**

- **It is not required for addressing.** Addressing is uniform at every size.
- **It is not required for dedup at the object level** — two identical entries already dedup.
- **What it buys is sub-object granularity**: dedup *within* and *across* large payloads, and
  **resumable transfer** — a receiver tracks which chunks it holds and fetches only the missing ones,
  which an indivisible 500 MB object cannot offer.
- ⭐ **What forces it is the connection's configured frame budget.** An entity that cannot fit a frame
  cannot be moved as one, so above that threshold chunking stops being a choice. **The threshold is a
  transport parameter, not a design rule** — which is the right place for it to live.

⇒ ***Nothing says "everything must be chunked content," and nothing should.*** A small record is one
content-addressed object and is already everything the content layer would make it.

⚠ **The remaining judgement call is the author's and the corpus does not rule it:** a publisher deciding
whether a given payload is *"big enough to chunk"* is choosing between one indivisible fetch and
resumability plus sub-object dedup. **Below the frame budget both are conformant**, and two publishers
making opposite choices about the same 5 MB image produce different objects that do not dedup against
each other. *That is a real interop wrinkle, it is not currently addressed anywhere, and it is smaller
than it sounds only because nobody has two publishers doing it yet.*

> ⚠ **The one real constraint at layer 2, and it is named in the site convention: chunker agreement.**
> A blob hash is a function of which chunker ran, so the canonical 1 MiB default is a **deduplication
> identity**, not a tuning parameter. Two publishers on disagreeing chunkers store the same bytes twice
> and share nothing. *This is the same failure IPFS has with its own chunker settings, and we have the
> same exposure with a narrower default.*

---

## §3 The prior art, compared at the layer that actually corresponds

**The earlier comparison was wrong because it put a social convention next to a block store.** Placed
properly, the landscape sorts into three bands and **we are in all three, at different maturities:**

| band | them | us | state |
|---|---|---|---|
| **bytes** — move large opaque content | BitTorrent pieces · IPFS blocks / UnixFS | `EXTENSION-CONTENT` blobs + chunks | ✅ **landed, and it is the direct analogue** |
| **records** — signed, typed, per-author streams | Nostr events · AT Protocol records/lexicons · ActivityPub objects | `app/feed/entry` + the key-addressed index + a signed root | ✅ **landed, and stronger on mutability** |
| **finding** — *who has hash H*, and *who contributed to subject S* | BitTorrent DHT/tracker · IPFS Amino DHT · AT Protocol Relay+AppView · Nostr relays | ⛔ **absent for BOTH, and this is the one real hole** | §4 |

⭐ **And the hole is more fundamental than the social framing made it look.** It is not merely *"who
replied to this thread"* — it is ***"which peer has these bytes"***, and it is missing for a blob and a
blog entry alike. The retrieval operation exists and is type-agnostic (§2.1); **what does not exist is
any way to learn whom to ask.** The handler is also optional and capability-gated, so a peer may not
serve hash-addressed reads at all. ⇒ **we have the data architecture of a content network and none of
its distribution layer.**

**Where we are genuinely better, and it is one thing:** *mutability with a verifiable, self-certifying,
statically-servable head.* An infohash is immutable; the block store's mutable-pointer layer is its
acknowledged soft spot; the relay/AppView model requires a live service. **A signed root with `seq` and
`predecessor` is a file on a web server**, and the corpus's own operating-models study found that **the
always-on tier that survives is the one that is also useful for something else** — a plain web server —
while *"the tier that exists only to serve the network is a donation with a half-life."*

**Where the record band differs from the relay lineage, and it is the structural claim:** in the surveyed
systems an aggregator's output is **a different kind of object** from its input — an API, a query
response, a firehose socket — so the next layer cannot consume it and **there is exactly one of them**.
Here an aggregator's output is signed entries in its own namespace, so **an aggregator can be
aggregated**. That is the closure property, and it is the reason the composition-tier document exists.

> ⚠ **And the honest cost, already measured in this corpus and worth not losing: a DHT decays.** In a
> documented investigation of the largest deployed one, **over 70% of provider records pointed at
> unreachable peers**, with NAT'd and residential nodes the largest contributor — announced but
> undialable. **Every deployed mitigation moves back toward addressability.** So *"we lack a DHT"* is
> not obviously a deficit; §4 is about the *index*, and the index does not have to be a DHT.

---

## §4 `coordinate → who` — the mechanism, precisely, and what is actually missing

**The earlier statement *"we lack it"* was imprecise. The design exists in full; the fold and the build
do not.**

### 4.1 What the mechanism is

**Three things are needed and only the third was ever missing:**

1. **A coordinate anyone can compute** — the content hash of the subject. ✅ have it.
2. **Contributions naming it** — a reply is a signed entry in the replier's own namespace carrying a
   pinned reference to the subject. ✅ have it.
3. **The reverse edge** — *given a coordinate, who contributed?* ⛔ this one.

**And the move is that the reverse edge is TWO indexes, where every surveyed system builds one:**

- **A: `coordinate → WHO`** — a grow-only set of peer ids. Small, **merged by set union**, and
  **able to omit but never forge** — a fabricated participant costs one wasted fetch, because you verify
  by fetching their signed entry and checking it names the coordinate.
- **B: `WHO → WHAT`** — a stream fetch. **Already built.** That is §1.

⭐ **Fusing A and B into `coordinate → CONTENT` is what forces the firehose** — an aggregator must ingest
everything because it cannot know what will be queried. **A is separable and you cannot selectively
fuse**, so the split is what buys *selectivity*: a gatherer covers only the coordinates it chose and is
complete-ish there while holding nothing else.

### 4.2 The shape it takes here, and why it is not a DHT

**A published artifact, not a live lookup.** A participant set is an entity in some peer's namespace,
fetched like everything else. Not because our peers are static — that is a deployment choice that will
expire — but because **an artifact is not a service:**

| | published set | live lookup |
|---|---|---|
| after the publisher goes away | **still there** | gone |
| verifiable offline | **yes** | no |
| re-servable by a third party | **yes, by anyone** | no |
| merges with a partial answer | **union** | one answer |
| same type as everything else | **yes** | **no — it is an API** |

⇒ **A live overlay is an accelerator over the same artifact, never a replacement**, because the last row
is the type break that causes centralization in the first place.

### 4.3 ⚠ Where the tracker analogy breaks, and it is load-bearing

> **A torrent's peer set is a REDUNDANCY set. Ours is a COMPOSITION set.**

Every BitTorrent peer serves **the same bytes**, so a partial peer list costs you nothing — same file,
fewer sources. **Here every participant wrote something different, so a partial participant list means
you literally get less content.** ⇒ *a tracker is enough* transfers as a **mechanism** and not as a
**completeness argument**, and our participant set has an importance/ordering dimension theirs never
needed.

### 4.4 So what is actually missing

| | state |
|---|---|
| the concept, the split, the precedent, the cost model | ✅ worked out in full |
| the coordinate's four canonical preimages and its constructable path | ✅ **a DRAFT proposal**, unfolded |
| the published-walk object | ✅ **a DRAFT proposal**, unfolded |
| **the fold, and any implementation** | ⛔ **nothing** |
| **how a reader finds a gatherer it does not already follow** | ⚠ **the real open question** — the argument that forward traversal solves it (a gatherer's output is an ordinary entry in an ordinary stream, so pulling any stream that references it tells you it exists) is **derived and has never been run** |

⭐ **And the standing caution is worth more than the gap: solving discovery with a registry lookup would
reintroduce exactly the privileged tier the whole design avoids**, and should be suspected on that ground
before it is designed.

---

## §5 The firehose question has two answers, because there are two firehoses

**These get conflated and they are not the same problem.**

### 5.1 The consumer firehose — *"everything published, so I can index it"*

**Answered, and the answer is: publish it in chunks instead of streaming it.** A page chain — a head
naming the current page, pages at key addresses, a reader resuming at its cursor. Same machinery as a
personal feed with walks as the page contents.

| | streamed socket | published chunks |
|---|---|---|
| **cacheable** | **no** — a socket cannot be cached | **yes — immutable content-addressed objects, ideal CDN objects** |
| consumer must be online | yes, or the server buffers per cursor | **no** — resume at any page |
| history re-servable | only by the origin | **by anyone who fetched the chunks** |
| gaps | invisible | **visible** — page numbers are dense |
| latency | sub-second | **the chunk period** |

⇒ **The trade is latency for cacheability, offline operation and replay-by-anyone** — and for an indexer
or a search corpus, minute-latency is irrelevant while **cacheability is what makes serving a firehose
nearly free instead of an operating cost.** *A streamed firehose is a per-consumer connection; a chunked
one is one static object served identically to everybody* — the same economics that make the rest of the
publishing model work.

⚠ **And key-addressing matters more here than for a personal feed:** a hash back-chain makes page *N*'s
identity depend on *N−1*, so **correcting one old chunk republishes the entire archive.** For a
high-volume firehose that is fatal.

### 5.2 The real-time firehose — *"nine peers, a game, events every tick"*

**Different problem, and the answer is that it does not belong in this layer at all.** §6.

---

## §6 Real-time — there is no architectural boundary here, there is a COST ENVELOPE

> ⛔ **A previous revision of this section said *"the entity system is a control plane and a durable
> plane, not a data plane for ephemeral high-rate state."* That sentence is withdrawn.** It asserted a
> capability limit the system does not have, from no measurement, and it is the kind of claim this corpus
> has a standing rule against in both directions — a note can be wrong by being *ahead* of reality as
> easily as behind it.

### 6.1 What is actually true

**Nothing forbids any of these**, and all of them are ordinary uses of the system:

| you want | you can |
|---|---|
| tick state in a handler | write a handler |
| tick state as entity-native types | define the types |
| the whole simulation entity-native and versioned | run it as entity-native compute over a revision DAG |
| the payload outside the system entirely | terminate in whatever transport you like |

**What differs between those is COST PER ITEM, and nothing else.** The system's distinguishing
properties — every item content-addressed, individually signed, bound into a hash-linked trie — are
each **work per item**, and that work is what sets the rate a given latency budget can carry.

**The terms of the envelope, which is what *"where is the boundary"* actually resolves to:**

```
per-item cost ≈ hash(bytes) + sign(if authored) + verify(on read) + trie update(O(log_K N)) + encode/decode
sustainable rate ≈ latency budget ÷ per-item cost   ... × however many peers multiply it
```

⭐ **Every term on the top line is a primitive with a hardware answer.** Content hashing, signature
generation and verification are exactly the operations that dedicated silicon has historically collapsed
by orders of magnitude. **On general-purpose hardware with no acceleration, they are the dominant cost;
with acceleration they may be close to free** — and that moves the boundary rather than removing or
creating one. ⇒ ***the envelope is a function of the hardware, and a sentence about what the system "is"
cannot be, which is why the withdrawn sentence was the wrong shape of claim.***

### 6.2 ⛔ *"Nobody has measured this"* was FALSE — and the error is the one this arc already catalogued

> **A previous revision said per-item cost *"has never been measured in any implementation."* That is
> wrong, and it was asserted without searching — which is the exact failure this corpus has a standing
> rule against.**

**Measurement exists and is substantial.** A heterogeneous-actor compute probe — a real-time game ported
to entity-native compute — was run by an implementation team in July 2026, produced a port taxonomy with
per-step operation counts, **self-corrected four headline numbers**, and was validated and absorbed by
this seat against the actual builtin set. There is also compute-budget work, sharding measurement at
k=2/5/8, and implementation-side performance testing beyond it.

**What that body actually establishes, and it is more useful than a rate ceiling:**

- ⭐ **Cost and parallelism are the same property.** A gather is expensive *because* every output cell is
  independent — which is exactly why it shards. A fold-scatter is cheaper-looking but **sequential by
  construction**. An indexed-update scatter would be cheaper still and would **kill sharding** for the
  same reason. ⇒ ***you cannot have cheap scatter and free sharding out of the same structure*** — a real
  invariant, and it is architecture rather than measurement.
- **Sharding survives heterogeneity** — boundary-hash-identical at k=2, 5 and 8, in parallel, with
  cross-actor reads and a length-changing set.
- **A display list is `O(actors)` and resolution-independent**, sitting between "the renderer must know
  what the object is" and "the renderer blits a framebuffer" — and native rendering is a drop-down, not
  something compute does per pixel.

⛔ **And the two framing corrections that body already recorded are the ones to carry, because a later
session repeated both.**

1. **An extrapolation showing per-pixel rendering in compute is far out of budget is a correct rejection
   of *that formulation*, not a statement that the workload is out of reach.** The architecture never
   rasterizes per-pixel in compute; the display-list path fits trivially. **A budget wall measured against
   a formulation the architecture does not use is a strawman.**
2. **A default operation budget is a negotiable per-evaluation parameter, not "the cap."** Pricing
   everything as a multiple of a default and calling the result a ceiling is over-anchoring.

⇒ ***The rule the probe team wrote for themselves, and the one this document had to be corrected by
twice: every wrong number was reporting a property of their own expression as a property of the
architecture.*** That is the same defect as §6.1's withdrawn sentence, and it has now occurred at both the
measurement layer and the prose layer. **Quote shapes, ratios and per-unit constants; never a bare
absolute, and never a formulation's ceiling as the system's.**

⚠ **What is genuinely not established** — stated narrowly this time, and as *not searched exhaustively*
rather than as absent: **an isolated per-entity figure for hash, sign, verify and bind, on named
hardware, with and without acceleration.** The existing body measures *compute evaluation* under an
operation budget, which is a different quantity from the per-entity cost of publishing and verifying.
**Before anyone asserts that gap again, they are to search the implementation repositories' own
performance work — it is not in this corpus and was missed twice by looking only here.**

### 6.3 The axis is *in the entity store or not*, and "ephemeral" was a made-up word

> ⛔ **A previous revision split the world into a "durable half" and an "ephemeral half." Withdrawn —
> that is not a real axis.** If what sits on the other side of a seam is a replicated database, it is not
> ephemeral; **it is simply not entity-native**, which is a different statement with different
> consequences.

**The real axis, stated as properties rather than as a dichotomy:**

| in the entity store | not in the entity store |
|---|---|
| content-addressed, so identical bytes are one object everywhere | addressing is whatever that system does |
| individually signed, so authorship survives being carried by a third party | authorship is whatever that system provides, usually nothing |
| covered by a signed root, so a reader verifies a whole subtree from one signature | verification is out of band or absent |
| revocable at the dispatch point, because access is a grant evaluated per operation | governed by that system's own access model |
| republishable under closure — a gatherer's output is the same kind of object | not composable with any of the above |
| **costs the per-item work in §6.1** | **costs whatever that system costs** |

**Neither column is "the system" and neither is a failure.** A composition that keeps ownership records
entity-native and runs simulation in a native engine is a legitimate architecture; so is one that puts
everything entity-native and pays for it; so is one that puts almost nothing there. **Which to choose is
an engineering judgement against the envelope in §6.1, and that is a question the corpus can help with
only once §6.2 is measured.**

### 6.4 What IS genuinely missing, and it is narrower than a boundary claim

**If you do put the payload outside the entity store, the seam has no specification.** That is a real
gap, and it is a gap about *interoperability of a choice*, not about a limit:

1. **How does a handler declare that its payload terminates outside the system?** Nothing in the
   dependency contract or the type system expresses it, so a second implementation cannot know.
2. **What does a capability authorize** when the thing authorized is a flow rather than an operation on a
   path? A grant is evaluated at dispatch; a flow has no dispatch.
3. **Is the negotiated result an entity?** ⭐ *It should be — that makes the session auditable, revocable
   and inspectable by the same machinery as everything else.* Nothing says so.
4. ⚠ **Termination and mid-flow revocation — the one with a real hazard.** The authorization model's
   value is that revocation bites at the dispatch point, and **a flow that has left the dispatch point has
   left the revocation point with it.** Nothing specifies what a peer owes when a grant is revoked under a
   session already running.
5. **What may a peer claim about a session it conducted out of band?** An attestation about an *outcome*
   is expressible; a claim about the transport is not.

**There is precedent for a deliberately non-entity component** — the public reflector, described in its
own spec as *"not an entity peer … a stateless service with its own small protocol, outside the entity
system entirely."* So the pattern exists; what is missing is the general statement of it.

**And the related slot is named and empty:** circuit relay, *"deferred from v1 … when a driver
materializes."* ⚠ **That deferral is correct and should hold** — a circuit is a *transport* for a payload
whose contract (1–5 above) does not exist yet, so building it first would be building the pipe before the
thing that goes in it.

### 6.5 ⛔ *"A contended many-writer state has no home here"* — WITHDRAWN, and compositions are why

> **A previous revision called contention "structural" and said a faster machine cannot create a shared
> authority. The second half is true and the conclusion drawn from it was wrong.**

**Local-view authority says each peer is the authority for its own namespace. It does not say a system
built from peers cannot hold contended state** — it says *where the authority sits* is always some named
peer. **That is a placement rule, not a prohibition**, and the way you build contended state out of it is
the ordinary way anyone builds contended state out of anything: **shard it, parallelize at the boundary,
and aggregate.**

⭐ **That is what peer compositions are for, and it is why the catalog is architecture and not
deployment trivia.** The parallelization boundary is the peer. A service pool shards a keyspace; a
hub-and-spoke names an authority; a compute pool isolates the expensive half. **The costs are real —
cross-peer communication, an aggregation step, additional operational complexity — and they are the same
costs every parallel architecture pays.**

> **The comparison that makes this obvious: a per-process actor runtime never claimed everything lives in
> one actor.** Its whole model is many small processes with boundaries between them. **A peer-to-peer
> actor system is the same shape one level up, and "an application is a single peer running everything"
> was never the model.** A session that reasons about limits by imagining one peer doing everything is
> measuring a deployment choice nobody recommends.

**What IS a real limit, stated so it is not confused with an architectural one:**

- **Complexity classes.** A sort is a sort; the physics of information does not bend. **This is true of
  every system and is not a property of this one.**
- ⭐ **The known bottleneck is the EMIT PATHWAY**, and it is specific and nameable: a great deal of
  reactive machinery listening on emit inside a single peer, with a lot of compute behind it, is where
  this design gets slow. **That is exactly the Class I site the composition catalog is organized around,
  and the catalog's answer is to put a boundary there.**
- **Whatever you install, you pay for.** Turn on revision, turn on history, version your compute and merge
  the result — **the system will let you, and it will cost what it costs.** Whether that is the right
  trade depends on a requirement the architecture does not know.

⇒ ***Without knowing the workload and its requirements, a statement about what this system can or cannot
carry is not an architectural finding.*** The contention question is a placement-and-sharding question
with a known bottleneck and a catalog of answers, not a boundary.

**What survives from the earlier framing, narrowed to what it can support:** in a game, inventory,
ownership, rosters, replays and **adjudicated outcomes as attestations** are natural fits with primitives
already built; an authoritative world simulation needs **an authority**, and naming one is the live rung
of a dial the design already has. **That is advice about where to put things, not a list of what is
possible.**

> **The signature rule that makes the attestation half work:** *a player signing "I scored 4,000,000" is
> worth nothing; the game server signing "this player scored 4,000,000" is worth whatever the server is
> worth to you.* **"Trust the signature" is never the right instruction. "Decide whose attestations you
> accept, then check the signature" is.**

## §7 *"How do I build on this"* — three different questions, and only one of them has a document

### 7.0 ⭐ Three questions get asked as one, and they have different audiences

**This is the distinction that was missing, and collapsing it is why the answer kept sounding evasive:**

| # | The question | Audience | Where it is answered |
|---|---|---|---|
| **A** | *I have several peers — how do I wire them, what do I grant, what does the arrangement give me?* | whoever **runs** the peers | ✅ **`GUIDE-PEER-COMPOSITIONS.md`** — transferred into this corpus 2026-09-13 |
| **B** | *What do I install, which conventions apply, and what can I build on top?* | whoever **writes the application** | ⚠ **partial** — the extension and convention guides cover their own tiers and nothing assembles them |
| **C** | *When do I write a handler, versus an extension, versus a convention, versus terminating outside the system?* | whoever is **choosing an architecture** | ⛔ **nowhere** |

⇒ **The composition catalog answers A. It was being offered as the answer to B, and it is not** — seven
ways to wire peers together is a deployment topology, not an application architecture. **B and C are the
questions an application developer actually has, and C is the one with no home at all.**

### 7.1 The layer stack, as a developer would have to assemble it today

| you want | you install |
|---|---|
| read and verify someone's published tree | core + tree |
| large binary content, dedup, resumable | **+ content** |
| be reachable / reach others | **+ network** (+ signaling for NAT traversal, + relay if unreachable) |
| a name that resolves to a peer | **+ registry** |
| third-party claims — scores, labels, credentials, reputation | **+ attestation** (+ quorum for K-of-N) |
| publish and follow feeds | + the feed convention — **format only, no new extension** |
| a website | + the site and embed conventions — **format only** |
| a gathered view of a subject | the data-exchange rules — **and the walk, which is unfolded** |

**The important structural fact, and it is easy to miss: the application conventions install nothing.**
They define **format and only format** — no kernel machinery, no SDK machinery. **A feed is not an
extension.** That is why *"is this even an extension?"* is a fair question with a real answer: **an
extension adds operations and types to a peer; a convention adds an agreement about bytes.** The social
tier is entirely the second kind.

### 7.2 ✅ The composition catalog — found in the archive, transferred, and it answers question A

**Found in the pre-split archive as four documents** — a guide, a closed proposal and two explorations,
~2,240 lines — **and the guide is now `guides/GUIDE-PEER-COMPOSITIONS.md` in this corpus.** It carries:

- **A peer is a configuration, not a role** — identity + installed handlers + installed extensions +
  grants held and issued + tree state + continuations + subscriptions + operational state. *Two peers
  identical in identity and extensions still differ if any other facet differs.*
- **A composition is a family of configurations** wired by grants and coupling that yields a property no
  single peer can. **Compositions are not hierarchical — there is no parent peer.**
- **A named catalog of seven** — operational peer · observer · service pool · hub-and-spoke · recovery
  cluster · **bridge** · compute pool — each with what it gives you, its coupling, its capability grants,
  when to use it, **when not to**, and its liveness class. **None requires protocol changes.**
- **Two structurally distinct stall classes with two non-interchangeable prevention mechanisms**, and the
  warning that reading one as the answer leaves the common case unprevented.
- **Three delivery classes** — durable at-least-once, synchronous request-response, async
  fire-and-forget, chosen per call.
- **K1, a liveness invariant:** *no single component can indefinitely stall a peer; saturation surfaces
  as an error to the caller, never an indefinite block and never a silent drop.* **Two-team-converged,
  and its own guide records that it never reached normative spec text.**

**It had been cited by filename from a guide in this corpus for the whole intervening period, as though a
reader could open it.** The split moved conclusions forward and left derivations in place, and this is a
case where **the conclusions were the part that got left.**

**Transferred with three changes, stated so the transfer is auditable:** internal tracking identifiers and
cross-repo paths removed; citations re-pointed at documents that exist here or marked archive-only; and
**K1 restated as an open gap rather than as a pending sign-off**, which is what it has become. The
catalog, the vocabulary, the two stall classes and the coupling rule are carried unchanged — they
describe configurations of primitives, and the primitives have not moved.

⇒ **Question A is now answered in-corpus. B and C are not, and the catalog was never going to answer
them.**

---

## §8 The gap list, ordered by what it blocks

| # | gap | blocks | state |
|---|---|---|---|
| **1** | ⛔⛔ **Per-item cost has never been measured** — hash, sign, verify, bind. Every claim about a rate ceiling, including one withdrawn in §0, is speculation until it is | *any* judgement about whether a workload fits, and the whole real-time question | **a microbenchmark, not a build — the cheapest high-value measurement in the arc** |
| **2** | ⛔⛔ **Provider discovery — *which peer has hash H*** — absent for a blob and a blog entry alike. Retrieval by hash exists and is type-agnostic; **learning whom to ask does not exist** | content distribution of any kind; the "find it in the network" property | ⚠ the subject-scoped half is designed in two DRAFT proposals; **the general bytes-scoped half is not** |
| **3** | ⛔⛔ **Question C has no home** — handler vs extension vs convention vs terminate-outside | every architecture decision an application developer makes | ⛔ **nowhere**, §7.0 |
| **4** | ⛔ **The out-of-system seam** (§6.4) — five unanswered questions, of which **mid-flow revocation** is a real hazard | interoperating any choice to put payload outside the store | ⛔ unspecified |
| **5** | ⚠ **Question B is partial** — nothing assembles the tiers into *"what do I install to build a thing"* | a newcomer's first hour | ⚠ per-tier guides exist; no assembly |
| **6** | ⛔ **K1 is not normative** | the composition catalog's correctness is unauditable | ⚠ converged across two implementations, never folded |
| **7** | ⛔ **The live leg has no index** (§1) | every live read | ⛔ open; the repair is a design question |
| **8** | ⚠ **Chunk-or-not below the frame budget is unruled** (§2.2) | two publishers making opposite choices about one 5 MB image do not dedup | ⚠ real, small today, nobody has hit it |
| **9** | ⚠ **254 citations in the published surface name documents that do not resolve here** | a reader following any of them | ⚠ measured; one block fixed, one document transferred |
| **10** | ⚠ **Circuit relay deferred** | two peers who cannot dial each other at all | ✅ **correctly deferred** — downstream of #4 |

**#1 and #3 are the two that change what someone can decide tomorrow**, and neither is a build. **#2 is
the one that changes what the system can *do*.**

## §9 Is there one mechanism for everything — and the honest answer

**No, and the design is better for it, but only two of the three seams are drawn.**

| seam | drawn? |
|---|---|
| entity ↔ chunk (*a record vs the bytes it points at*) | ✅ **cleanly** — a pointer payload, and either side is fetchable alone |
| entity ↔ published aggregate (*mine vs what I gathered of yours*) | ✅ **cleanly** — closure, the republication rules, and the growth rule |
| **durable ↔ ephemeral** (*what is published vs what is streamed*) | ⛔ **not drawn** |

⇒ **The unifying claim that survives, and it is a claim about ARTIFACTS rather than about limits:**

> **Content-addressed signed artifacts are primary; transports are interchangeable.** A live overlay is
> an accelerator over the same artifact, never a replacement for it.

**That principle says nothing about what may be built and it was never a boundary.** What it does say is
that anything expressed as an artifact inherits addressing, dedup, authorship, verification and closure
for free, and **anything not expressed as an artifact inherits none of them and has to supply its own** —
which is a trade to make deliberately, at a price §6.1 states and §6.2 admits nobody has measured.

**What would falsify the frame**, stated so it is attackable: if an application put its owned state in
signed entities and its high-rate state elsewhere and found **the two could not be kept coherent in
practice** — that the seam cost more than either side saved — then this is two systems wearing one name
and the artifact principle does not reach as far as claimed. ⚠ **Equally falsifying in the other
direction:** if the §6.2 microbenchmark came back showing per-item cost is *small* on ordinary hardware,
then much of the hedging in this document is unnecessary and the entity-native path is simply the right
one for far more workloads than anyone has assumed. ***Both are open, both are cheap to settle, and
nobody has done either.***
