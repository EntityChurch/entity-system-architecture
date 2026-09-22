# EXPLORATION — the convergent designs: Willow, Scuttlebutt, Usenet, Matrix, BitTorrent, and five verdicts

**Status:** Exploration (design record). Not a proposal, not normative.

**Companion:** `EXPLORATION-THE-HYPERTEXT-LINEAGE-…` covers the dead designs. This covers the **living
and the operationally-retired** ones — systems that ran long enough to produce a verdict rather than a
theory. **The five documents `REFERENCE-PRIOR-ART-BY-AXIS` §9 named as its honest gaps are four-fifths
closed here** (SSB, NNTP, Matrix, BitTorrent), plus one it did not know to name: **Willow**, which
turns out to be our nearest living neighbour by a wide margin.

**Why this matters more than the Xanadu read.** Xanadu tells us which requirements are unachievable.
These tell us **which achievable choices cost what**, because their authors made them and then lived
in the result for years.

**Sourcing discipline (D12).** Every claim with a URL was opened. Verdicts are labelled with who
reached them. Claims marked *(our reading)* are this record's inference.

---

## §0 The five verdicts, stated first

| # | Question we had open | Verdict | Whose |
|---|---|---|---|
| **1** | Was making the hash chain **opt-in** right? (§2.3.2) | **Yes, and two independent designs agree** — SSB made it mandatory and paid; Willow rejected it explicitly, for a reason we had not written down | SSB's own retrospective; Willow's design rationale |
| **2** | Is content-addressing + naming the right stack? | **Yes, and the diagnosis is published** — Willow eschews content-addressing *at the protocol level* to escape a bootstrapping problem we do not have, because our tree is `path → hash` | Willow authors |
| **3** | Does the corpus need state resolution / consensus? | **No, and Matrix is the cost of the alternative** — seven years of repairing state resolution, and the repairs are still landing in 2025 | Matrix.org's own record |
| **4** | Retention — *"what do I hold and why"* (capstone §4.4) | **Nobody solved it; it became a market.** Usenet's answer to unbounded replication was to turn retention into the product's price | Usenet's operational history |
| **5** | Should there be one canonical discovery route? | **No — and the field's verdict is explicit: no layer won.** Tracker, DHT and PEX all persist because each covers the others' failure mode, and PEX structurally cannot bootstrap | BitTorrent's measured literature |

**And one finding that is not a verdict on an open question but on a closed one:** our *"aggregating
two observers is set union"* result (capstone §4.2) is the same claim Willow implements as **3-D
range-based set reconciliation**, which has a published complexity analysis (Meyer, SRDS 2023). **The
instrument for the pocketed bloom-filter question already exists, is analyzed, is deployed twice, and
is a better fit for our substrate than bloom filters are** — §7.

---

## §1 Willow — the nearest living neighbour, and it is very near

**Willow (`willowprotocol.org`) grew out of Earthstar; one of its two authors is Earthstar's core
maintainer.** It is a general-purpose synchronisation protocol with a three-dimensional
(user × path × timestamp) data model, a capability system called **Meadowcap**, and range-based set
reconciliation for sync. **It is the only system in this survey whose authors published a
point-by-point rationale against every other system in it**, which makes it the single most useful
document in this read.

### §1.1 Where they agree with us, and it is most of the load-bearing surface

| Their sentence | Our equivalent |
|---|---|
| *"Willow is based on **naming data**"* (contra Nostr) | load-bearing invariant #1: `tree = path → hash` |
| *"After agreeing on an initial name, I can continuously bind new data to the name, which automatically reaches those interested"* | the signed root + `path → hash` binding; capstone §1 |
| *"Willow deliberately avoids any cryptographic proofs of completeness, allowing for **traceless data removal**"* | the trie is canonical *"regardless of insertion or deletion history"* — removal is byte-identical (FEED-10a) |
| *"write **capabilities** rather than public keys allows for flexible delegation"* | `system/capability` — four grant dimensions, delegation with strict attenuation, `delegation-caveats` |
| ActivityPub's names *"address a particular server that supplies the data… this introduces a certain brittleness"* | capstone §1: every route terminates at the key; losing a domain costs a label, not an identity |

**Five independent convergences on the load-bearing choices is the strongest external validation this
design has received.** Note especially the third row: **we and Willow independently concluded that
traceless removal is worth giving up completeness proofs for, and neither of us learned it from the
other.**

### §1.2 The content-addressing sentence, and why it does not indict us

This is the one that looks like a contradiction and is not:

> *"Pure content-addressing faces a **bootstrapping problem**: in order to actively distribute data to
> others, I must first inform them about the hash of that data."* … *"Content-addressed data excels at
> immutability, statelessness, and global connectivity. Willow embraces **mutability and statefulness,
> locality, and structure**."*

and, on deletion:

> the authors consider Willow to have the strongest story for mutability, *"as Willow is the only one
> to completely eschew content-addressing on the protocol level."*

**Read precisely, that is a diagnosis of IPFS, not of us.** The bootstrapping problem is *"a CID names
bytes and nothing tells you which CID to want."* **Our answer is the same as theirs — a name — and we
have it at the layer they say is missing:** a peer publishes a signed root over a `path → hash` trie,
so a reader who knows the key knows every name, and the hash is what the name *resolves to*. We are
not pure content-addressing and never were; the third of our five load-bearing invariants
(*local-view authority*) and the first (`path → hash`, **not** `path → entity`) exist precisely
because content addressing alone is insufficient.

**Where they went further than us and it is a real difference:** Willow eschews content addressing
*at the protocol level* to get mutability and deletion. We kept it and got deletion anyway, via a
canonical trie whose byte representation is independent of history. **So our position is strictly
between IPFS and Willow, and it is the only one of the three that has both self-verifying content and
traceless removal.** *(our reading — and it should be checked by someone who reads Willow's spec
rather than its comparison page.)*

### §1.3 Their statement on append-only logs, and it closes our §6.5 question 2

> *"A hash chain of data cryptographically authenticates that authors never produce differing
> extensions of the same log. In itself, this can be a laudable feature, but it **precludes the option
> of concurrent writes from multiple devices**."* … *"With 3-D range-based set reconciliation, Willow
> can achieve efficient data synchronisation without the need to restrict updates to a linear
> sequence."*

**We made `prev` opt-in for a different reason** — FEED §2.3.2 rejected a mandatory chain because *a
chain makes an old object permanently referenced by a newer published one, so a retroactive edit
cascades to the whole archive* (the curation cost, FEED-10).

**Willow's reason is one we never wrote down: multi-device.** A mandatory single-writer chain means a
person with a phone and a laptop cannot publish from both without forking their own log — and SSB's
own ICN paper lists *how nodes should react to forked feeds* as **unresolved future work**.

> **This is a real gap in our derivation, not just a missing citation.** Our §2.3.2 argument is
> correct and it is *incomplete*: had `prev` been mandatory we would have shipped a design in which
> **one human with two devices is a protocol violation**, and we would have found out from an
> implementer. The rule survives; the derivation should gain the second reason.

### §1.4 Meadowcap versus our capability model — we are ahead

Meadowcap: an unforgeable token granting read or write over a three-dimensional *area*, issued by the
data's owner, delegable and restrictable, and **a capability confers the ability to mint further
capabilities for the same resources.**

Ours: `system/capability` grants over **four** dimensions (`handlers`, `resources`, `operations`,
`peers`) with two typed scope kinds (`path-scope`, `id-scope`), delegation under a strict narrowing
rule — *"child MUST retain all parent constraint keys… MUST NOT add keys parent doesn't have"* — plus
`delegation-caveats` (`no_delegation`, `max_delegation_depth`, `max_delegation_ttl`).

**We are strictly more expressive.** What Willow has that we do not is **read capabilities used to
control *propagation*, with private-set-intersection for metadata privacy** — the authors note *"few
if any protocols devote the attention to controlling data propagation through read capabilities, and
even fewer extend this care to metadata privacy."*

> **That is the one thing in Willow we do not have an answer to, and it is on our Stage 4 path.**
> Our sync surface reveals *what you are interested in* to whoever you sync with. Willow treats that
> as a first-class privacy leak and solves it. **Filed as an open question (§8.1)** — it is exactly
> the kind of thing that is cheap now and expensive after a wire format is deployed.

### §1.5 The namespace framing worth stealing

> Meadowcap's **owned** namespaces were designed *"to mimic the curative function of a fediverse
> instance host without tying it to computational resources,"* while **communal** namespaces provide
> *"a model that couldn't realistically be replicated in the fediverse unless everybody ran their own
> server."*

**That is our forum problem and our moderation result, stated from the other end.** Owned ≈ our
single-author space (capstone use case 1). Communal ≈ our forum (use case 3). And *"the curative
function without the computational resources"* is precisely what our mirror does: **moderation without
hosting.** Their sentence is better than ours and should be borrowed.

---

## §2 Secure Scuttlebutt — the natural experiment on the mandatory chain

**SSB is the system that made the opt-in choice mandatory and lived with it for a decade.** This is
the §6.5 question-2 verdict the bearings document asked for, and it comes back clearly.

### §2.1 What it costs, from SSB's own documentation and papers

- **Onboarding.** The protocol guide is blunt: replicating whole feeds from the initial message is
  *"a major friction point for onboarding new users"* — the download plus the indexing cost. The
  mitigation was **retrofitted partial replication** (fetch slices from the most recent message).
- **Deletion is structurally impossible.** *"Append-only really means append-only"* — deletion must be
  layered above, via delta encoding.
- **The human cost, in their own ICN 2019 paper.** The *"cocktail of pseudonymity, non-refutability
  and immutability can be a serious risk to users"*; immutable feeds complicate *remediation of
  harmful posts such as harassment once replicated*; and the authors concede participation
  *"currently favors privileged users for whom privacy issues are not critical."*
- **Forks are unresolved.** How nodes should react to forked feeds is listed as future work.
- **Multi-writer mismatch.** Conversations among many participants spread across many logs, requiring
  replication of the whole **transitive interest graph** — and participants do not want the same
  transitive set, because past disagreements mean someone may not want another's content arriving via
  a common friend.

### §2.2 What we get right by construction, and one place we should look harder

| SSB's cost | Our position |
|---|---|
| whole-feed replication on join | **navigation is by key, never by chain** (FEED §3.3 rule 1) — a reader starts at any page. The onboarding cost SSB retrofitted away, we never incur |
| deletion impossible | traceless removal, byte-identical trees (FEED-10a) |
| immutability as a harassment vector | `prev` is opt-in; a removal leaves no trace |
| transitive interest graph replication | **the mirror inverts this**: you read *one* view someone published, not everything their friends hold |
| forked feeds unresolved | our `prev` claim is scoped to one author's sequence and is optional; a fork is a broken claim, not a broken protocol |
| **schema fragmentation from uncoordinated evolution, splintering subgroups** | **unaddressed** — see the companion document on vocabularies; this is the one row where SSB's failure is live for us |

**And the one that should worry us.** SSB's partial-replication work surfaced a correctness hazard
that generalizes: in sliced replication, sending messages at depth 100–200 *also requires sending a
"certificate pool" giving the shortest path back to depth 0*, and **any authorization message must not
be dropped from replication.** In other words: **partial replication is not free — you must preserve
the authorization/validation skeleton even when you prune content.**

> **Cross-reference for us.** Our analogue is a mirror or a reader holding a *subset* of a tree. Our
> verification skeleton is a detached `system/signature` per entry plus (for capability-bearing paths)
> a transportable authority chain — and `ENTITY-CORE-PROTOCOL` §5.1's *"a signature stored only at an
> extension-private path is invisible to that machinery, so the chain cannot be transported"* is the
> **same hazard, already found here, by a different route.** SSB is independent corroboration that the
> class is real. Worth checking whether FEED's mirror rules say what a mirror must carry *besides*
> entries in order to remain independently verifiable. *(open — §8.2)*

---

## §3 Usenet / NNTP — the flood-fill lineage, and the only real answer to retention

**Usenet is the forum's ancestor and the one prior art for our §4.2 convergence-through-redundancy
claim.** It is also the only system here that ran unbounded replication to its economic conclusion.

### §3.1 The mechanism, and how close it is to ours

- **No centre.** Each server passes articles to its peers, who pass them on: *flood fill*.
- **Dedup by immutable identifier.** Every article carries a `Message-ID`; a server logs seen IDs in a
  history file and **discards anything already seen.** Loop prevention is reinforced by the `Path:`
  header listing the systems already traversed.
- **Offer-before-transfer.** Raw flooding wastes bandwidth, so `ihave`/`sendme` and later streaming
  NNTP send **only the IDs** first; the peer asks for what it lacks.

**Read that third bullet against our design.** *Exchange identifiers, transfer only the difference* is
set reconciliation, invented operationally in the 1980s. **Usenet's `ihave` is the crudest member of
the family whose most refined member is range-based set reconciliation (§7)** — and our capstone §4.2
"a thread is a grow-only set, merging is union" is the same structure with content hashes in place of
`Message-ID`s. **A content hash is a `Message-ID` you cannot forge and do not have to allocate.**

### §3.2 The verdict — and it is about economics, not protocol

Full replication at every server made Usenet *cheap and censorship-resistant* while the payload was
text. Then it wasn't:

- posting volume **~5 TB/day (Jan 2009) → ~9 TB/day (Jan 2011)**
- retention as the competitive product axis: **120 days (2007) → 900 days (2011) → 5,000+ days
  today**, with one provider putting 2,000 days at **25 PB**
- and the honest caveats buyers learned: *"full retention"* claims can cover only popular articles or
  only text groups, and nominal retention overstates real availability because of takedowns, disk
  failures and propagation errors

**The system went from a free ISP amenity to a subscription storage market where retention days are
the price.** Nobody solved retention. **The market solved it by charging for it.**

> **This is the sharpest available answer to capstone §4.4's *"what do I hold and why"* and §8.3's
> *"republication makes storage grow monotonically and nothing reclaims it."*** The finding is not a
> mechanism, it is a shape: **in a fully-replicating network, retention is not a policy, it is a
> price** — and it will be set by whoever pays for the disk, per-participant, with no global answer.
> Our design is already better positioned than Usenet's because **replication here is selective by
> construction** (you mirror what you chose to gather, not every article in every group you carry).
> **What we lack is any vocabulary for expressing a retention decision**, and Usenet says the
> vocabulary that matters is *how far back*, not *what*.

---

## §4 Matrix — the price of contended mutable state, paid in public for seven years

**Matrix is the most-deployed federated system with a real distributed-consistency answer**, which is
exactly why it is the right control for our claim that we need none.

**Why it needs one:** rooms do not belong to a server; they are replicated across all participating
servers, each of which must independently compute room state and *"arrive, as far as possible, at the
same state."* Forks happen whenever servers cannot keep each other up to date, and users keep sending
**state** events into the drift.

**What it cost, from Matrix.org's own record:**

- state resolution **v1** had bugs where a merge could favour historical over current state — **"state
  resets"** — giving attackers a way to maliciously revert room state
- designing and implementing **v2** in 2018 *"was a bit of a mission"*; the fixes were treating
  one-sided state as a conflict, separating access-control events from regular events, and ordering
  events by their **authorisation chain** (the auth DAG)
- **2025: still repairing it.** MSC4297 (*State Resolution v2.1*, room version 12) protects against
  state-reset classes where delayed federation traffic reverts state — symptoms include *users
  re-added to rooms they had left* and *access control resetting*. MSC4291 changes room IDs to be
  hashes of the create event as a precaution against `m.room.create` hijacking
- the counterintuitive core: merging two room versions means treating state events as **unordered
  sets**, even though the room DAG orders them
- plus **soft failure**, a whole sub-mechanism to stop evaded bans from re-entering state

### §4.1 The cross-reference, and it is the cleanest one in this document

**Our capstone §3 says three things must be distinguished — set, order, selection — and only *set*
must converge.** Matrix is what it costs when a fourth thing is in the room: **contended mutable
state with authority semantics** (membership, power levels, bans).

| | Matrix | Us |
|---|---|---|
| what is replicated | room **state**, mutable, multi-writer, authority-bearing | signed immutable **entries**, single-writer per space |
| merge operation | state resolution v2.1 + auth DAG + soft failure | **set union** |
| can a merge produce a *wrong* answer? | **yes** — that is what a state reset is | **no** — union is idempotent, commutative, associative |
| can a peer inject a forged element? | guarded by auth events and event signing | **no** — every element carries its author's signature |
| still being fixed? | **yes, room version 12, 2025** | n/a |

**The transferable rule, and it is worth a charter line:** *the moment a design has state that more
than one party may change and whose value is authoritative, it acquires a consensus problem, and that
problem is never finished.* We do not have one **because we never made a shared object mutable** — a
mirror is one author's view, a thread is a grow-only set, and membership (Stage 4) is the one place
this will bite. **Matrix is the map of what Stage 4 costs if we get it wrong**, and its verdict is
that the expensive part is not encryption, it is **membership**.

---

## §5 BitTorrent — the discovery-route verdict, and it endorses our §1

**Our capstone §1 says: many routes to a key and no canonical one.** BitTorrent is the field's largest
natural experiment on exactly that, and it is usually told wrong (as a *progression*, tracker → DHT →
PEX, in which each replaces the last).

**It is not a progression, it is a stack, and the reason is structural:**

- **Peer exchange cannot bootstrap.** *"PEX first requires a peer to know at least one other peer via
  some other discovery form."* To make initial contact a peer must use a tracker or a DHT bootstrap
  node. **PEX is an amplifier, never an entry point.**
- **PEX converges slowly.** Peer-reviewed measurement: clients obtain ~30–40% of tracker and DHT peers
  *immediately*, and the fraction rises slowly — **under 60% even after extended time**.
- **Private swarms disable the DHT deliberately**, because standard DHT has no authentication or
  access control, so any peer could join and undermine reputation-based admission. Discovery is forced
  through a trusted tracker.
- **No layer won.** Most clients use PEX *in addition to* trackers and DHT; each covers the others'
  failure mode — PEX reduces tracker load and keeps swarms alive when the tracker is down.

**Cross-reference:** this is direct external support for capstone §1's *"many routes and no canonical
one… all of them terminate at the key, and once they do the route is discardable."* **The system that
ran this experiment at internet scale for twenty years arrived at exactly that architecture, and the
one thing it proves is that the always-on route (the tracker) cannot be removed** — it is the only one
that works from a cold start. **Our equivalent of the tracker is the signed root at a known
location**, and BitTorrent says: do not expect a gossip route to replace it.

**Sourcing caution:** the strongest quantitative pro-decentralization figures returned for this
question (a *"4.1× median TTFB reduction"*, *"3.8× initial peer count"*) come from content-farm domains
with unverifiable methodology and are **not** relied on above. The 30–40%/60% figures are from the
peer-reviewed PEX measurement literature.

---

## §6 The composite scorecard

Where this design sits against every system in this survey, on the axes that decide interop.

| Axis | SSB | Willow | Nostr | ATProto | Matrix | Usenet | **Us** |
|---|---|---|---|---|---|---|---|
| identity = key, not account | ✓ | ✓ | ✓ | DID (~) | ✗ (server) | ✗ | **✓** |
| data named, not only hashed | ✗ (log offset) | **✓** | ✗ (event id) | ✓ (repo path) | ✓ | ✗ (Message-ID) | **✓** |
| self-verifying content | ✓ | ✗ (by design) | ✓ | ✓ | ✓ | ✗ | **✓** |
| traceless deletion | ✗ | **✓** | ~ (relay policy) | ✓ | ~ (redaction) | ~ (cancel) | **✓** |
| multi-device write | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ | **✓** (`prev` opt-in) |
| partial replication native | ✗ (retrofit) | ✓ | ✓ | ✓ | ✓ | ✗ | **✓** |
| no consensus problem | ✓ | ✓ | ✓ | ✓ | **✗** | ✓ | **✓** |
| capability-based access | ✗ | ✓ (Meadowcap) | ✗ | ✗ | ✗ (power levels) | ✗ | **✓ (4-dim)** |
| read-capability / metadata privacy | ✗ | **✓** | ✗ | ✗ | ~ | ✗ | **✗ — §8.1** |
| retention answer | ✗ | ✗ | relay policy | PDS operator | server policy | **priced** | **✗ — §8.3** |
| vocabulary governance | **✗ (splintered)** | n/a | NIP registry + fallbacks | Lexicon + NSID | room versions | newsgroup names | **partial** |

**Two empty cells are ours and both are already on the board:** metadata privacy on the sync surface,
and retention. **The third — vocabulary governance — is the subject of the companion document and is
the largest open risk in the whole arc.**

---

## §7 The pocketed question, answered enough to file

**The operator's framing was right and the instinct pointed at the wrong instrument.** The question —
*"internet-scale data sets, we can't replicate everything, people need to know what they're interested
in and what others have"* — is **set reconciliation**, and the field has converged on an answer that is
not a Bloom filter.

**Range-based set reconciliation (RBSR).** Two peers hold fingerprints over ranges of a **totally
ordered** set; equal ranges fingerprint equal, unequal ranges differ, so they recursively subdivide
non-matching ranges until the difference is isolated and only that is exchanged. Short-circuit: below a
size threshold, just send the items. Generic treatment and analysis: **Meyer, *Range-Based Set
Reconciliation*, SRDS 2023 / arXiv:2212.13567** — which also reduces local-computation time by a
logarithmic factor over prior work.

**Why it beats the sketch family for our shape:**

| | Bloom / IBLT / PinSketch | RBSR |
|---|---|---|
| false positives | **yes** (Bloom); IBLT must be over-provisioned to keep failure probability low | **none** |
| local work | **linear in set size** to build the sketch | proportional to the **difference**, times log factors |
| rounds | one or few | **logarithmically many** |
| decode cost | PinSketch O(d²); IBLT O(d) but over-provisioned communication | cheap, incrementally maintainable |
| requirement | none | a **totally ordered set with range-summarizable fingerprints** |

**That last row is the finding.** RBSR's practical efficiency *"depends critically on the storage
backend's ability to summarize arbitrary ranges."* **Our canonical Merkle trie is exactly such a
backend** — it is a total order over paths with a fingerprint (the subtree hash) already computed at
every interior node, for other reasons. **We may already have the data structure the technique needs,
which would make the marginal cost of RBSR close to the protocol messages and nothing else.**
*(our reading — unverified against the trie's actual node structure, and that verification is the
first step if this is ever picked up.)*

**Deployed twice, which answers "does it work":** Willow's `range-reconcile`, and **Negentropy** in the
Nostr ecosystem, which was directly inspired by Meyer's work.

**Filed, not pursued** — the operator scoped this as exploratory and it is correctly behind the
vectors and the two seats' reviews. **What changes is the shape of the eventual item:** it is no
longer *"do bloom filters help?"* but *"our trie is a range-summarizable ordered store; RBSR is the
published technique for that; measure it."*

**The Chrome Safe Browsing parallel the operator raised is a different problem and worth separating.**
That is *membership testing against a large mostly-static remote set with a privacy requirement* — one
party has the set, the other has a query. **Set reconciliation is symmetric: two parties each hold
part of a set and want the union.** Bloom filters are right for the first and wrong for the second.
**Our problem is the second.** *(our reading; the Safe Browsing design was not opened this session and
is owed a read if the first shape ever becomes relevant — e.g. for a shared block-list distribution.)*

---

## §8 What this read leaves open

1. **Metadata privacy on the sync surface.** Willow uses read capabilities plus private set
   intersection so that *what you ask for* does not leak. We have no equivalent, and our four-dimension
   grant model has the right shape to carry one. **Cheap now, expensive after deployment.** Stage 4
   adjacent; should not wait for Stage 4.
2. **What must a mirror carry to stay independently verifiable?** SSB's certificate-pool hazard and
   `ENTITY-CORE-PROTOCOL` §5.1's transportable-chain rule are the same class. FEED §4.1 says a mirror
   republishes entries unmodified with their signatures; it should be checked against the case where a
   referenced entry's *authority* (not its signature) is needed and was pruned.
3. **Retention has a shape now, not a solution.** Usenet says: in a replicating network retention
   becomes a price, expressed as *how far back*, set per-participant. A vocabulary for that is missing.
4. **`prev`'s derivation should gain the multi-device reason** (§1.3) — the rule is right and the
   argument is half-length.
5. **Willow's spec proper is unread.** Everything above about Willow comes from their comparison page
   and their rationale prose. **The data model, the sync protocol and Meadowcap's actual grammar were
   not opened**, and §1.2's *"we sit between IPFS and Willow"* claim is exactly the kind that should be
   checked against a spec rather than a comparison page.

---

## §9 Sources opened

**Willow / Earthstar:**
[Willow — comparison to other protocols](https://willowprotocol.org/more/willow_compared/index.html) ·
[range-reconcile](https://github.com/earthstar-project/range-reconcile) ·
[willow-rs](https://github.com/earthstar-project/willow-rs) ·
[meadowcap-js](https://github.com/earthstar-project/meadowcap-js) ·
[Willow & Earthstar's Spring](https://gwil.garden/posts/willow-earthstar-big-year.html)

**Set reconciliation:**
[Meyer, *Range-Based Set Reconciliation*, arXiv:2212.13567 / SRDS 2023](https://arxiv.org/pdf/2212.13567) ·
[*Range-Based Set Reconciliation via Range-Summarizable Order-Statistics Stores*, arXiv:2603.19820](https://arxiv.org/html/2603.19820) ·
[*Practical Rateless Set Reconciliation*, SIGCOMM 2024](https://arxiv.org/html/2402.02668) ·
[Log Periodic — RBSR overview](https://logperiodic.com/rbsr.html)

**Secure Scuttlebutt:**
[Scuttlebutt Protocol Guide](https://ssbc.github.io/scuttlebutt-protocol-guide/) ·
[ssb-db docs](https://ssbc.github.io/ssb-db/) ·
[*Secure Scuttlebutt: An Identity-Centric Protocol…*, ACM ICN 2019](https://conferences.sigcomm.org/acm-icn/2019/proceedings/icn19-19.pdf) ·
[Kermarrec et al., *Gossiping with Append-Only Logs in Secure-Scuttlebutt*, DICG 2020](https://dicg2020.github.io/papers/kermarrec.pdf) ·
[ssb-subset-replication-spec](https://github.com/ssbc/ssb-subset-replication-spec) ·
[ssb2-discussion-forum — tangle auth / sliced replication](https://github.com/ssbc/ssb2-discussion-forum/issues/24)

**Usenet / NNTP:**
[*How Does Usenet Handle News?* (Linux Network Administrator's Guide)](https://tldp.org/LDP/nag/node258.html) ·
[INN — the RFC set: 3977, 4643, 4644, 5536, 5537, 8315](https://github.com/InterNetNews/inn) ·
[Giganews retention 900 days (2011), volume figures](https://techcrunch.com/2011/01/24/giganews-retention-has-reached-900-days) ·
[Usenet retention guide](https://www.downloadprivacy.com/usenet-retention)

**Matrix:**
[Project Hydra — improving state resolution (2025)](https://matrix.org/blog/2025/08/project-hydra-improving-state-res/) ·
[State Resolution v2 for the Hopelessly Unmathematical](https://matrix.org/docs/older/stateres-v2/) ·
[Room Versions](https://spec.matrix.org/latest/rooms/) ·
[Federation API — soft failure](https://spec.matrix.org/legacy/server_server/r0.1.4.html) ·
[Jacob, Becker, Grashöfer, Hartenstein — *Matrix Decomposition*, SACMAT '20](https://matrix.org/blog/2020/06/16/matrix-decomposition-an-independent-academic-analysis-of-matrix-state-resolution/)

**BitTorrent:**
[*Understanding Peer Exchange in BitTorrent Systems*](https://scispace.com/pdf/understanding-peer-exchange-in-bittorrent-systems-43ng7d0de5.pdf) ·
[Peer exchange — overview](https://en.wikipedia.org/wiki/Peer_exchange) ·
[*Persistent BitTorrent Trackers*, arXiv:2511.17260](https://arxiv.org/pdf/2511.17260)
