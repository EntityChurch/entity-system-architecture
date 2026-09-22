# EXPLORATION — the content-lookup landscape: compute it, look it up, or publish it — and why the key cannot be the content hash

**Status:** EXPLORATION (2026-09-13). Written to be attacked. Nothing here is ruled.

**Companion to** `EXPLORATION-FIVE-KEYED-LOOKUPS-ARE-ONE-MACHINE-AND-THE-CONTENT-ADDRESS-LOOKUP-IS-THE-REDUNDANCY-CASE`,
which established *what* the fifth lookup is. **This one asks what the field has already built, and it
changes two of that document's conclusions.**

**Sources opened at primary or near-primary:** Spacedrive V2 (philosophy + architecture writeups) ·
git-annex (preferred-content manual, balanced-preferred-content design note, assistant design blog) ·
Ceph (architecture docs, CRUSH map docs, the CRUSH paper) · Named Data Networking (project technical
report NDN-0001, the 2014 CCR paper, a 2025 naming-and-packet-lookup survey) · Tahoe-LAFS (file-encoding,
mutable-file and dirnode specifications) · IPNI (IPFS docs, the IPNI reader-privacy spec, IPIP-337 /
IPIP-421) · iroh (content-discovery blog and experiments, iroh-blobs 0.90 notes) · Peer2PIR
(arXiv:2405.17307). **Plus two documents from this project's own pre-split archive that turn out to
carry half the answer** — see §3.3.

---

## §0 The result

**Six findings. The third and the fourth change the design; the first is about us.**

1. **The coverage is lopsided in a measurable way, and it is the shape a hyper-focus leaves.**
   Measured across all three regions of the search path: **IPFS ≈ 274 documents, BitTorrent ≈ 85,
   Kademlia ≈ 43.** Against that: **Ceph/CRUSH placement, Named Data Networking, Tahoe-LAFS,
   git-annex, Spacedrive, Upspin, Peergos, erasure coding, `numcopies`, and reader privacy are at or
   near zero in every region.** Four whole traditions — datacenter placement, the personal fleet, the
   capability-aware store, and the network layer — were never opened. §1 has the table.

2. ⭐ **There are exactly three ways to answer *"who has these bytes,"* and they differ by what they
   require, not by how clever they are: COMPUTE it, LOOK IT UP, or PUBLISH it.** Ceph computes (no
   lookup table at all), DHTs and trackers and NDN look up, git-annex publishes. **This corpus has
   already chosen *publish* everywhere else, for reasons that hold here too.**

3. ⭐⭐ **The scope of agreement decides which of the three is available — and the map is itself
   publishable, so all three are reachable from one mechanism.** CRUSH needs an agreed cluster map;
   a fleet of your own machines *has* one; the open internet does not. **Because a map can be a signed
   entity like anything else, "compute" is available at exactly the scope where a map can be agreed,
   and degrades to "publish" beyond it.** *This is not a new idea here: this project's own archive
   proposed `system/sharding/config` — a prefix → peer-set table as entity data — at three separate
   core revisions, and it was never brought forward.*

4. ⭐⭐ **The lookup key cannot be the raw content hash.** Three independent systems reached this,
   two of them after deploying the naive version: publishing *"I hold `H`"* is a **confirmation
   oracle** for anyone who can guess `H`, and asking *"who has `H`"* tells the answerer what you want.
   Tahoe-LAFS solves it structurally (**the storage index is derived from the encryption key, so
   lookup capability and read capability are the same capability**); IPFS/IPNI retrofitted **double
   hashing** plus encrypted provider records; Willow does **private interest overlap**, which this
   corpus has already read and scored. **`content-coord = hash(bytes)` as drafted is a privacy defect,
   not a simplification** — and for a general-purpose system this is the finding that matters most.

5. ⭐ **Possession is cheaply PROBEABLE, and content addressing is what makes it so.** iroh's content
   tracker **downloads a random 2 KiB BLAKE3 chunk before it will even store an announce.** That
   dissolves most of the staleness objection against a published holder set: a claim is a belief until
   probed, and probing costs two kilobytes. **git-annex, which has the same problem, does not have
   this answer** — its backends are not uniformly probeable, so it uses *trust levels* instead.

6. ⭐ **You cannot aggregate over a flat hash space, and NDN is the proof at network scale.** NDN's own
   scaling answer is that *hierarchical naming permits name aggregation, which is primary in scaling
   the routing system.* Content hashes are flat. **So per-blob holder claims can never aggregate, and
   the only aggregation available is over the hierarchical term sitting beside the hash — the
   namespace or the path.** The companion document's §5.2 lead was a hunch; this makes it structural.

> **And the framing the landscape supports.** These lookups are **infrastructure** — the layer an
> application takes as given, in the way an application takes routing as given. **But NDN is the
> cautionary tale rather than the model:** it put the content lookup *inside the network*, one table
> set per router, and paid in name-lookup cost, PIT memory, FIB aggregation limits, mobility
> breakage, and cache-induced non-determinism. **The layer you put the content lookup at is the
> largest architectural choice available here, and NDN is the measured price of putting it at the
> bottom.**

---

## §1 The coverage measurement

**Run across all three search regions**, because an absence measured inside one scope is evidence
about the scope and not about the subject. Counts are documents containing the term.

| Term | this corpus | pre-split archive | paper corpus |
|---|---|---|---|
| **IPFS** | 27 | 142 | 105 |
| **BitTorrent** | 28 | 36 | 21 |
| **Kademlia** | 7 | 33 | 3 |
| provider record | 13 | 4 | 0 |
| Perkeep | 5 | 1 | 0 |
| Ceph | 0 | 4 | 19 |
| Syncthing | 0 | 7 | 0 |
| replication factor | 0 | **5** | 0 |
| Kademlia-adjacent: Veilid | 0 | 1 | 0 |
| Tahoe-LAFS | 1 | 1 | 0 |
| double hashing | 0 | 2 | 1 |
| **CRUSH** | **0** | **0** | **0** |
| **Named Data Networking / NDN** | **0** | **0** | **0** |
| **git-annex** | **0** | **0** | **0** |
| **Spacedrive** | **0** | **0** | **0** |
| **Upspin** | **0** | **0** | **0** |
| **Peergos** | **0** | **0** | **0** |
| **`numcopies`** | **0** | **0** | **0** |
| **erasure coding** | **0** | **0** | **0** |
| **reader privacy** | **0** | **0** | **0** |
| **confirmation oracle** | **0** | **0** | **0** |

**Two honest qualifications, because a bare count inherits the shape of its search.**

- **"Replication factor" is NOT unstudied — it is unbrought-forward**, and the difference matters
  (§3.3). The five archive hits are real design work, opened and read for this document.
- **Ceph appears 19 times in the paper corpus and zero times as CRUSH.** Read: Ceph is cited as *a
  system that exists*; **its placement algorithm — the part that bears on this question — is what
  nobody opened.** A term census would have scored that axis as covered.

> **What the shape says.** Three systems account for the overwhelming majority of the corpus's
> peer-to-peer reading, and **all three are open-membership, no-capability, public-content designs.**
> That is one corner of the space. The traditions that assume *bounded membership* (Ceph, Spacedrive,
> Syncthing), *capability-scoped reads* (Tahoe-LAFS, Peergos, Willow), or *declared placement policy*
> (git-annex, Ceph) were not read — and those are precisely the assumptions a general-purpose system
> with a capability model actually has.

---

## §2 The three answers, and what each costs

| | how *"who has `H`"* is answered | what it requires | worked example | fails when |
|---|---|---|---|---|
| **COMPUTE** | a deterministic function of the key and a **map** | **an agreed map** — membership plus a way to keep it consistent | **Ceph CRUSH**: `(object name, cluster map) → placement group → OSD set`. *"Ceph needs no lookup tables, since object placement is determined by a deterministic function"* | membership is open, or the map cannot be agreed |
| **LOOK UP** | ask infrastructure that answers live | **something online at query time** | Kademlia provider records · a BitTorrent tracker · **IPNI** · **NDN's FIB/PIT** | the infrastructure is absent, decayed, or becomes a concentration point |
| **PUBLISH** | read an artifact the holder wrote | **nothing live** — bytes on an origin | **git-annex's location log**: the `git-annex` branch records which repository UUIDs are *believed* to hold each key | the artifact is a **belief**, and beliefs go stale |

### §2.1 COMPUTE is the strongest answer available and it is not usually available

CRUSH's property is worth stating exactly: **any client with an up-to-date cluster map can
independently calculate which devices hold a given object, with no metadata-server query.** It buys
no single point of failure, no broker, direct client-to-device traffic, and placement policy expressed
as **rules over a topology** — failure domains being chassis, rack, power, network.

**The price is the map.** CRUSH replaces a lookup table with a *consensus-maintained* map, and Ceph
pays for that map with a monitor quorum. **The lookup did not disappear; it moved into membership.**
That is the honest accounting, and it is why COMPUTE is a datacenter answer.

It also carries an aggregation lesson this corpus needs: **Ceph does not place objects, it places
placement groups.** Object → PG → device set, with the PG as a deliberate indirection layer. **Nobody
tracks per-object anything.** §5.2 is the same lesson arriving from NDN.

### §2.2 LOOK UP is what the studied corner does, and it is the corner that keeps concentrating

The measured story on the most-studied system is now two steps long, and the second step is recent:

- Kademlia provider records decay — the figure already in this corpus is **>70% unreachable** — and
  keeping them alive means continuous re-announcement.
- **The deployed response was to add an indexer.** IPNI *"is designed to create an alternate routing
  and discovery infrastructure outside and independent of the Kademlia DHT"*, offloading provider
  records to a high-performance key-value store. It is **complementary, not a replacement** — the
  sources are consistent about that and an earlier assumption otherwise would be wrong. But one
  academic reading calls it *"a centralized version of the DHT for indexing provider records, serving
  primarily large content providers."*

> ⭐ **Read plainly: the most-studied open network's answer to lookup decay was to add a fast central
> index, and the cost it paid was concentration.** That is this corpus's own concentration finding
> happening again, in the field, in the last two years — and it is the strongest available argument
> for not solving this with a lookup at all.

### §2.3 PUBLISH is what this corpus already does, and git-annex is the mature version

git-annex has shipped the fifth lookup for over a decade, and its vocabulary is worth taking whole:

- **The location log** lives in the `git-annex` branch and records which repository UUIDs are
  **believed** to hold each key. *Believed* is the design's own word and it is load-bearing.
- **Trust levels qualify the belief** — `trusted` / `semitrusted` (default) / `untrusted` / `dead`.
  **This is an acceptance policy held by the reader**, which is the same position this corpus takes.
- **`numcopies`** is the minimum number of copies, **synced through the whole repository network**,
  and settable per file type.
- **Preferred content** is a small DSL per repository saying what that repository *wants*, with terms
  like `lackingcopies`, `balanced=group:lackingcopies`, and `lackingcopies=backup:1`.
- **Rebalancing is explicit and user-triggered**, and the size calculation it needs is slow enough to
  print *"(calculating repository sizes)"*.

**Two operational scars worth more than the feature list:**

- **The trusted-repo paradox.** If a repository with a preferred-content setting is itself *trusted*,
  receiving a file raises its trusted-copy count, which makes the file eligible to be dropped again.
  **A placement policy and a copy-counting policy can fight each other.**
- **The `or present` clause.** Expressions in practice take the form
  `(balanced=group:N and not copies=group:N) or present` — **and the `or present` is what stops
  endless drop/get churn.** A policy that is evaluated identically by every holder, with no term for
  *what is already here*, oscillates.

---

## §3 Scope decides which answer is available — and the map is publishable

### §3.1 The three scopes are not three modes

| scope | membership | so what is available |
|---|---|---|
| **A fleet you own** — five or six machines | **closed, known, complete** | **COMPUTE.** The map is small, agreeable and yours |
| **An organization** — tens to hundreds of peers, governed | **bounded and administered**, not global | **COMPUTE within the boundary; PUBLISH across it.** Placement policy is a real requirement here, and so is capability scoping |
| **The open network** | **unbounded, unknowable** | **PUBLISH, merged by union, best-effort.** Nothing can be computed because no map can be agreed |

⭐ **The move that makes this one mechanism rather than three: the map is an ordinary signed entity.**
A membership set already exists in this architecture as a capability-governed artifact, and a
placement *rule* over it is data. So:

> **COMPUTE is available exactly where a map can be agreed, and the map's scope is a capability scope.
> Past that boundary the same holder claims are still published and still merge by union — you lose
> determinism, not the mechanism.**

**That is the answer to whether one design covers the fleet case, the organization case and the
public case: it is not three designs, it is one artifact plus a map that may or may not exist.**

### §3.2 What each tier in the field actually does, which confirms the table from the other end

- **Spacedrive** — *the library is metadata; the files stay home.* Indexes in place, BLAKE3 content
  hashes with adaptive sampling for large files, `SdPath` for device-transparent addressing,
  **libraries as isolated namespaces**, and devices connected directly over Iroh/QUIC. Its sync is
  **domain-separated on purpose**: device-specific data (the filesystem index) uses **state
  replication**, shared metadata (tags, ratings) uses a **lightweight HLC-ordered log with
  deterministic conflict resolution** — a stated rejection of writing a custom CRDT.
  **There is no content-lookup service anywhere in it, because with a complete device list there is
  nothing to look up.** And there is **no capability model**: authority is a single user approving
  previewed operations.
- **Syncthing** — device-level, no content addressing, complete membership per folder. Same reason.
- **git-annex** — the one that does publish a lookup, because its repositories genuinely come and go.

> **So the entire personal-fleet tradition answers the fifth lookup by not having it**, and that is
> not an oversight — **it is what a complete membership list buys you.** The lookup is a cost you pay
> for openness.

### §3.3 This project's archive already proposed the map — three times

**Found while measuring §1, and it changes the status of §3 from *proposed here* to *never brought
forward*.** The pre-split archive carries, at three separate core revisions:

- **A sharding design with a replication factor.** Peers subscribe to paths matching a hash range;
  *"the number of peers per shard is the replication factor"*; roles within a shard are left open —
  **symmetric, leader-follower, or quorum** — with the explicit posture that *the entity system
  doesn't mandate; peer groups choose.*
- **The map as entity data.** `system/sharding/config`, holding `prefix_bits` and a list of
  `{id, prefix, peers[]}`. ⭐ **That is a CRUSH map expressed as an entity**, written here
  independently and years earlier.
- **A topology-to-use-case table** that already separates the scopes §3.1 rediscovers — including
  *"large file storage → sharded → can't store everything everywhere"* and *"personal device sync →
  spanning tree → your devices, rooted at primary."*

**None of it is in the current corpus.** The honest status of §3 is therefore *recovered and
corroborated*, not *derived* — and the corroboration is strong, because Ceph reached the same shape
from datacenter storage and the archive reached it from the entity model.

---

## §4 The systems, by the axes that matter here

| System | key it looks up on | who may answer | capability model | placement policy | staleness handling |
|---|---|---|---|---|---|
| **BitTorrent** | infohash | tracker, then DHT, then peers | **none** | none — every peer is interchangeable | peer churn, re-announce |
| **IPFS / Kademlia** | CID | DHT nodes | **none** | none | **>70% record decay**; re-announcement |
| **IPNI** | CID / multihash | an indexer | none; **moving to encrypted records** | none | advertisement chains, gossip announce |
| **NDN** | hierarchical **name** | any router on the path | none (signatures on data) | caching is per-router policy | cache eviction; **stale FIB on mobility** |
| **Ceph / CRUSH** | object name **+ map** | **nobody — you compute it** | cluster-internal | ⭐ **rules over a topology, by failure domain** | **map epochs**; no staleness by construction |
| **Tahoe-LAFS** | ⭐ **storage index = tagged hash of the encryption key** | storage servers | ⭐ **read-cap / verify-cap / write-cap** | **K-of-N erasure coding** (3-of-10 typical) | servers can deny, not read or forge |
| **git-annex** | annex key | **nobody — you read the log** | git-level | ⭐ **`numcopies` + preferred-content DSL** | ⭐ **trust levels**; explicit rebalance |
| **iroh** | BLAKE3 hash | a pluggable **tracker**, outside the blobs layer | node identity | none yet | ⭐ **probe 2 KiB before storing an announce**; partial/complete announce kinds |
| **Spacedrive** | BLAKE3 hash | **nobody — membership is complete** | **none** (single user consent) | redundancy *reporting*, not policy | index sync |
| **Willow** | namespace + path + time | peers you sync with | ⭐ **Meadowcap** | none | range-based set reconciliation |
| **this corpus, drafted** | content hash | any peer you can already reach | capability-typed throughout | **none yet** | union merge; **unaddressed** |

**Reading the last row against the rest is the point of the table.** The architecture has the
capability model that the two most-studied systems lack, and it has **neither a placement policy nor a
staleness answer** — which are the two columns every system that ran in production had to fill.

---

## §5 The five things the landscape adds

### §5.1 Possession is probeable, and this is the answer to staleness

iroh's tracker **verifies before it records**: it downloads a random 2 KiB BLAKE3 chunk over a
dedicated protocol to confirm the announcer actually holds the data, and the design note recommends
*requiring at least one successful probe before even storing an announce*, so that the worst a fake
announcer achieves is a futile connection attempt. An announce is a **signed content-availability
statement** carrying host, content, kind and timestamp — and **`kind` distinguishes partial from
complete**, with partial content verified only for its unverified size, which is all a node that has
just begun downloading can offer.

> ⭐ **Why this transfers here and does not transfer to git-annex.** Probing is cheap **because
> everything is content-addressed and chunked**: a 2 KiB sample plus the hash is a proof of
> possession, and the prover cannot fake it without the bytes. git-annex cannot do this uniformly
> because its backends are arbitrary (a cloud bucket, a USB disk in a drawer), so it falls back to
> **trust levels** — a human judgement — where we can use a **measurement**.
>
> **This substantially answers the companion document's sharpest objection.** A holder claim is a
> belief; a *probed* holder claim is a measurement with a timestamp. The two should not be the same
> object, and **`partial` / `complete` is a discriminator the drafted design did not have.**

### §5.2 A flat key cannot aggregate — NDN paid for this at network scale

NDN's three tables per router are the **Content Store**, the **Pending Interest Table** and the
**Forwarding Information Base**, with FIB doing longest-prefix matching on name prefixes. Its measured
problems are, in order: **name lookup cost** (variable-length hierarchical names, LPM being far more
expensive than IP matching), **PIT memory** (fast *and* large is the hardware problem), **FIB size**,
**mobility** (relocating producers leave stale prefix trails and Interests are dropped), and
**cache-induced non-determinism**.

And its stated mitigation is the sentence that matters:

> *hierarchical naming permits name aggregation, which is primary in scaling the routing system*

**Content hashes are flat and have no aggregation structure — which this corpus already knows and
says, in the routing extension, as the reason it refuses prefix matching on peer ids.** The two
statements are the same statement.

⇒ **A per-blob holder index cannot be made to scale by any amount of engineering, because the term it
would aggregate on does not exist.** The only hierarchical terms available beside a content hash are
the **namespace** and the **path**. The companion's §5.2 lead — *publish per estate, not per blob* —
is therefore not a cost optimization; **it is the only shape with an asymptote.** And it lands on the
same indirection Ceph uses: **place groups, not objects.**

### §5.3 ⭐ The key cannot be the content hash — three systems, two of them retrofitting

**Two distinct leaks, and the drafted design has both:**

| | the leak | who has it |
|---|---|---|
| **publisher side** | announcing *"I hold `H`"* confirms possession to anyone who can guess or compute `H` | BitTorrent, IPFS, iroh, and **`content-coord` as drafted** |
| **reader side** | asking *"who has `H`"* tells the answerer exactly what you want | all of the above, plus **any delegated router** |

**Three answers, in increasing structural strength:**

1. **Double hashing** *(IPFS / IPNI, retrofitted)*. Look up `hash(hash(content))` rather than the CID,
   and encrypt the provider records so the router can decrypt only when given the original CID. *An
   external observer cannot know what a user is looking up without already knowing the original CID.*
   IPIP-421 makes this a client obligation; cid.contact has moved toward storing **only** encrypted
   provider records. The IPNI spec is candid about the motive: because an indexer catalogues provider
   content behind a central API, **the need for reader privacy is amplified** — without it, an indexer
   is a hard sell against more decentralized routing precisely because it enables mass query snooping.
   Academic work (Peer2PIR) argues the guarantee is only **k-anonymity over a prefix**, and applies
   private information retrieval for a stronger one.
2. ⭐ **Derive the lookup key from the capability, not from the content** *(Tahoe-LAFS, by
   construction)*. The **storage index is a tagged hash of the encryption key**; the grid is *"a table
   mapping Storage Index to shares."* **You cannot look it up unless you already hold a cap that
   entitles you to read it** — and storage servers get *no automatic ability to read or modify* what
   they hold. The three-level split — **read-cap, verify-cap, write-cap** — even lets a third party
   verify integrity **without** the decryption key, which is exactly the auditor role a
   redundancy set wants.
3. **Match interests without revealing them** *(Willow's private interest overlap)*. Peers exchange
   **salted hashes of their interests** and match before anything is disclosed — already read and
   scored in this corpus, and scored as a capability the corpus lacks.

> ⭐ **So the drafted `content-coord = {kind:"content", hash}` is a confirmation oracle over every
> byte any peer holds, and publishing one at a constructable path makes it a *world-readable* one.**
> For public content that is harmless — it is already public. **For anything else it is a defect**, and
> it is precisely the concern that separates a general-purpose system with a capability model from the
> public-content systems this corpus over-studied.
>
> **The shape of the fix is Tahoe's, and it composes with what is already here:** the lookup key
> should be derived such that **holding the key to look it up is itself the authorization to know the
> answer.** What that derivation is, here, is an open question — §7 — but *"just use the content
> hash"* is answered, and the answer is no.

### §5.4 The placement vocabulary is missing, and git-annex has the mature one

`numcopies` · `preferred content` as a per-repository DSL · trust levels · `balanced=group:N` ·
`lackingcopies=backup:1` · explicit rebalancing · the `or present` anti-churn clause · the
trusted-repo paradox. **Zero of these concepts appear anywhere in this project's three regions.**

**Three of them transfer nearly unchanged and are worth naming now:**

- **A desired copy count is a property of the DATA, not of a peer** — git-annex sets it per file type,
  and Ceph sets it per pool. Either way **a peer does not decide how replicated someone else's data
  should be**, which fits a model where authority follows the namespace.
- **What a peer WANTS is a separate declaration from what it HAS**, and it is the peer's own. That is
  a second artifact beside the holder claim, and conflating them is what makes placement
  unexpressible.
- **Any policy evaluated identically by every holder oscillates unless it can see what is already
  placed.** That is the `or present` clause, and it is the cheapest available lesson from a decade of
  production.

### §5.5 The personal tier does not look up at all — and that is data, not a gap

Covered at §3.2. **Spacedrive and Syncthing have no content-lookup service because complete membership
makes one unnecessary**, and Spacedrive has no capability model at all. **Neither is a model to copy**
— they are single-authority systems — **but both confirm the scope argument**, and Spacedrive's
domain-separated sync (state replication for the index, an HLC-ordered log for shared metadata, an
explicit refusal to write a custom CRDT) is a sober engineering data point for anyone tempted to reach
for a general convergent type where two simpler mechanisms will do.

---

## §6 What this changes for the fifth lookup

**Against the companion document's conclusions:**

| companion said | after the landscape |
|---|---|
| it is a fifth coordinate kind and a third predicate on the walk | **still right, and better supported** — iroh's signed announce is nearly the same object |
| the sharpest objection is staleness | ⭐ **substantially answered — possession is probeable at 2 KiB.** A claim and a *probed* claim are different objects, and `partial` / `complete` is a needed discriminator |
| the leaf-count question is the open one, with per-estate as a lead | ⭐ **upgraded from a lead to a structural requirement.** A flat key cannot aggregate; Ceph places groups and NDN measured the cost of not doing so |
| the key is the content hash | ⛔ **WRONG, and this is the finding.** That is a confirmation oracle on both sides. The key must be derived so that looking it up is itself authorized |
| *(no position)* | ⭐ **placement policy is a real, separable deliverable** — desired copy count on the data, wanted-content on the peer, and an anti-churn term |
| *(no position)* | ⭐ **scope decides mechanism, and the map is publishable** — so the fleet, the organization and the open network are one design, not three |

---

## §7 Is a proposal ready?

**Close, and it is worth saying exactly what is and is not settled**, rather than drafting around the
gaps.

**Settled enough to write:** the three-way taxonomy and its scope mapping · the map as an ordinary
entity · the holder claim as a signed, timestamped, partial-or-complete statement · probe-before-record
· aggregation over the namespace rather than the hash · the separation of *what I hold* from *what I
want* from *how many copies this data wants* · the anti-churn requirement.

**Not settled, and a proposal must decide them:**

1. ⛔ **What the lookup key actually is.** Tahoe derives it from the encryption key, which works
   because everything there is encrypted. **Most content here is not**, so the derivation cannot be
   copied — and *"hash the hash"* only buys k-anonymity against someone who does not already know the
   content. **This is the hard open question and it should not be hand-waved.**
2. ⛔ **How a holder claim composes with the capability system.** A capability-scoped read and a
   world-readable *"I have it"* are in tension, and no system read here has both a capability model
   and a content lookup. **Willow is the closest and it has no content lookup; Tahoe has one and its
   answer is to make the key the capability.**
3. ⚠ **Whether placement policy is the same proposal or a second one.** The evidence says second —
   git-annex's placement layer sits *above* its location log, and Ceph's rules sit *above* CRUSH.
   **Two proposals stacked, not one document.**
4. ⚠ **Whether the probe is normative.** It is the staleness answer, it is cheap, and it is also a new
   obligation on whoever records a claim.

**Recommendation: two proposals, and the first one is small.** *The holder claim and its probe* is
nearly writable today. *The lookup key and its authorization* is a real design question and deserves
its own document — it is where the capability model earns its keep, and it is the thing that makes
this system different from the three it over-studied.

---

## §8 What would test this

- **The confirmation-oracle claim is testable by inspection**: take any drafted holder claim, assume
  an adversary who knows a candidate document, and check whether publication confirms possession. If
  it does not, §5.3 is wrong.
- **The probe claim is testable at 2 KiB**: a prover holding the bytes and a prover holding only the
  hash must be distinguishable in one round trip. If they are not, §5.1 is wrong.
- **The aggregation claim is a measurement, not an argument**: leaf counts for one real peer's blob
  inventory versus its namespace count. If per-blob is payable at realistic scale, §5.2's consequence
  is optional.
- **The scope claim is falsified by a counterexample**: any system that computes placement without an
  agreed map, or that maintains an agreed map across open membership. **None of the systems read here
  is one**, which is suggestive and not a proof.

---

## §9 Sources

**Read for this document.** Spacedrive V2 — [philosophy](https://v2.spacedrive.com/overview/philosophy),
[v2 release notes](https://spacedrive.com/blog/spacedrive-v2-release),
[repository](https://github.com/spacedriveapp/spacedrive) ·
git-annex — [preferred content](https://git-annex.branchable.com/git-annex-preferred-content/),
[balanced preferred content design](https://git-annex.branchable.com/design/balanced_preferred_content/),
[assistant design blog, day 98](https://git-annex.branchable.com/design/assistant/blog/day_98__preferred_content/) ·
Ceph — [architecture](https://docs.ceph.com/en/reef/architecture/),
[CRUSH maps](https://docs.ceph.com/en/latest/rados/operations/crush-map/),
[CRUSH paper (Weil et al., SC '06)](https://ceph.com/assets/pdfs/weil-crush-sc06.pdf) ·
NDN — [project technical report NDN-0001](https://named-data.net/techreport/TR001ndn-proj.pdf),
[Named Data Networking (ACM CCR 2014)](https://dl.acm.org/doi/pdf/10.1145/2656877.2656887),
[naming and packet-lookup survey (PeerJ CS, 2025)](https://peerj.com/articles/cs-3551/) ·
Tahoe-LAFS — [file encoding](https://tahoe-lafs.readthedocs.io/en/tahoe-lafs-1.12.1/specifications/file-encoding.html),
[mutable files](https://tahoe-lafs.readthedocs.io/en/latest/specifications/mutable.html),
[dirnodes](https://tahoe-lafs.readthedocs.io/en/latest/specifications/dirnodes.html),
[the LAFS paper](https://tahoe-lafs.org/~trac/lafs.pdf) ·
IPNI — [IPFS docs](https://docs.ipfs.tech/concepts/ipni/),
[reader-privacy spec](https://github.com/ipni/specs/blob/main/reader-privacy.md),
[IPIP-337](https://specs.ipfs.tech/ipips/ipip-0337/),
[IPIP-421](https://github.com/ipfs/specs/pull/421/files),
[content-routing recap, IPFS þing 2023](https://blog.ipfs.tech/2023-ipfs-thing-content-routing-track/) ·
iroh — [content-discovery experiments](https://www.iroh.computer/blog/iroh-content-discovery),
[iroh-experiments](https://github.com/n0-computer/iroh-experiments/tree/main/content-discovery),
[iroh-blobs 0.90](https://www.iroh.computer/blog/iroh-blobs-0-90-new-features) ·
[Peer2PIR: Private Queries for IPFS (arXiv:2405.17307)](https://arxiv.org/pdf/2405.17307).

**From this project's own record.**
`EXPLORATION-FIVE-KEYED-LOOKUPS-ARE-ONE-MACHINE-AND-THE-CONTENT-ADDRESS-LOOKUP-IS-THE-REDUNDANCY-CASE` ·
`EXPLORATION-WILLOW-THE-NEAREST-NEIGHBOUR-READ-AGAINST-ITS-SPEC` (private interest overlap, §5/§7.5) ·
`EXPLORATION-THE-OPERATING-MODELS-AND-THE-ALWAYS-ON-TIER` (the DHT decay measurement) ·
`EXPLORATION-THE-P2P-COVERAGE-AUDIT-WHAT-WE-HAVE-STUDIED-AND-THE-FIVE-AXES-WE-HAVE-NOT` (the prior
coverage measurement, and the scope lesson this document applies) ·
`PROPOSAL-THE-COORDINATE-AND-WALK-DISCOVERY` · `EXTENSION-ROUTE` §3 (flat key, no aggregation) ·
`APP-CONVENTION-REFERENCE` §1, §2.3 · **and two pre-split archive documents** on sharding with a
replication factor and on replication topologies — located via the archive title index, §3.3.
