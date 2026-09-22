# PROPOSAL — keeping a copy current is one mechanism, and the absence it must explain is what shapes it

**Proposes:** **`DATA-EXCHANGE`** — a data layer spanning four documents, not one extension:
`EXTENSION-DATA-EXCHANGE` (Tier 2b), `SYSTEM-DATA-EXCHANGE` (the composition half),
`SDK-DATA-EXCHANGE` (the entry point an application actually touches) and a developer guide — plus
corrections to `EXTENSION-SUBSCRIPTION`, `APP-CONVENTION-FEED` and `APP-CONVENTION-EMBED`.
**Status:** DRAFT 2026-09-12 · **revision 5.** **PARTIALLY FOLDED and stays `active/`** (L3): the
closure chapter has landed as `specs/SYSTEM-DATA-EXCHANGE.md` v0.1 (`D20`, `D24`) and the gathered-view
corrections into `APP-CONVENTION-FEED` (`D25`, `D26`); everything else is open. Revision 3 restored the
closure property as the spine; **revision 4 is what happened when one seat BUILT it and the other read
the substrate underneath it**; **revision 5 is what happened when both seats then built against
revision 4** — and the sentence revision 4 called load-bearing turned out to be false at one of its two
layers (§5.0-a). §14 carries rounds 1–3, §16 round 4, §17 round 5.
**Tier:** extensions + system + SDK. **This is deliberately not a single-capability extension**, and
§15.5 is the layer map.
**Depends:** `ENTITY-CORE-PROTOCOL.md` §1.2 (content hash) · §1.4 (paths) · §3.5 (signatures) ·
§6 (dispatch) · `EXTENSION-TREE.md` §1, §3.3a (signed root), §3.7.1 (subtree hash by
reconstruction), §3.8 R1 (enumeration completeness) · `EXTENSION-CONTENT.md` §6.5.3 (closure) ·
`EXTENSION-SUBSCRIPTION.md` §5.5, §6.3 · `EXTENSION-SUBSTITUTE.md` §1, §2.1, §6 ·
`EXTENSION-REGISTRY.md` §6a.3a · `EXTENSION-IDENTITY.md` §1 (agents are per-device peers) ·
`EXTENSION-ENCRYPTION.md` §4.4 (a device is a private-key holder) · `EXTENSION-REVISION.md` §2.3 ·
`SYSTEM-ARCHITECTURE.md` §13.1 · `SYSTEM-COMPOSITION.md` §1, §5 ·
`GUIDE-APPLICATION-DEVELOPMENT.md` §3 · `SPECIFICATION-FORMAT.md` §8.8, §8.9
**Kind**: proposal · **Authority**: non-binding until folded · **Governed-by**:
`SPECIFICATION-FORMAT.md`

---

## §0 Summary

**One sentence:** seven independent implementations across two code bases, sharing no code, have each
built the same loop for *keeping a local copy of another peer's data current* — **and not one of them
replays a change log**, because the substrate is content-addressed and the question is answerable by
comparison. This proposal specifies that loop once, at the layer where it composes, and reduces the
application conventions that currently contain copies of it to vocabularies over it.

**But the mechanism is not shaped by the loop.** It is shaped by a failure the loop makes visible:

> **You cannot detect, from one source, what that source did not tell you.**

A cursor is silent about an element never sent. A sequence number is silent about a root withheld. A
signature proves *authentic* and never proves *current*. **The design obligation is therefore not
"stay current" — it is *make absence attributable*, and every layer below earns its place by naming
an absence that would otherwise be silent.**

⛔ **Revision 2 said "the mechanism is READER-SCOPED" and that was the error that produced two bad
names in a row.** A name for half a loop cannot come out right. What a peer acquires **it may
publish**, and the published result is **the same kind of object as what it consumed** — that closure
(§5.0) is not a feature of this mechanism, it is the property that decides whether the network
federates or centralizes, and scoping the mechanism to the reader deletes it.

⇒ **The cycle is the unit: publish → propagate → consume → gather → publish.** The same peer does all
of it, on the same rails, and *"which one am I"* is a statement about what a peer is doing this minute,
never about a tier it belongs to (§5.1).

**The closest honest analogy is the transport layer one level down, and it is structural rather than
decorative:** packets take heterogeneous paths, arrive out of order and incomplete, and are reassembled
into something coherent that the layer above extracts meaning from. **Here the packets are signed
entities, the paths are a live peer / a published origin / another peer's republished walk, the
reassembly is content-addressed convergence, and the extraction is LOWERING.** What is *not* borrowed:
there is no connection, no ordering guarantee, and no delivery guarantee — convergence is established
by comparison, not by acknowledgement (§6.3).

**What this proposal contains:**

| | |
|---|---|
| **§1–§3** | the problem, the evidence, and why it is one mechanism rather than several — decided by a landed `[MUST]`, not by taste |
| **§4** | where it belongs, and why the answer is two documents |
| **§5** | the five layers and the one seam |
| **§6** | the normative content — **beginning with the single entry point**, because that is what decides whether this reduces anyone's work |
| **§7** | what is deliberately excluded, with the reason each exclusion is a trap |
| **§8** | the emergent half — a `SYSTEM-COMPOSITION` chapter, and why one landed `MUST` is currently unsatisfiable |
| **§9** | corrections to landed text |
| **§10** | what a conformance check set must discriminate |
| **§11** | open questions |
| **§12** | deltas |
| **§13–§14** | what this does not establish · what revisions 2 and 3 changed |
| ⭐ **§15** | **the traces — four applications run end to end, the layer map, and where the rungs fit** |

---

## §1 The problem — four absences that are indistinguishable from their opposites

Each row is a pair of conditions that are **byte-identical at the consumer** today. Each was found
independently, from a different symptom, by a party not looking for the others.

| # | Layer | These two are indistinguishable | Consequence observed |
|---|---|---|---|
| **A1** | **omission within a version** | a node **hidden** from a walk · a node that is **not there** | a reader is told a thing does not exist by a host that chose not to serve it |
| **A2** | **staleness of the version** | an **older signed root served as current** · **nothing new to serve** | *"no new posts"* is not a fact a reader may assert |
| **A3** | **delivery** | a notification **dropped before the wire** · **nothing to send** | 2,000 records written, **676 arrive**, transfer stops permanently, **no error on either side**, status reporting healthy throughout |
| **A4** | **presentation** | *the publisher offered only a format I cannot decode* · *this renderer declines it* · *something broke* | a caption where a figure should be, with nothing anywhere saying which of the three is true |

**These are one shape, and the corpus already names it.** The absent-vs-withheld collapse is on its
seventh recorded instance in this corpus's design record; A2, A3 and A4 are the eighth, ninth and
tenth. The rule already derived for it is the one this proposal builds on:

> ***Absence yields no conclusion unless someone paid for one to be drawn.***

### 1.1 ⭐ Who pays differs per layer, and two of the four cannot be paid by the same party

| | who must pay | with what |
|---|---|---|
| **A1** | **the publisher** | declare the prefix **tracked** under the signed root, so that **`EXTENSION-TREE` §3.8 R1**'s enumeration-completeness rule binds: a host *"cannot omit a node from the walk without the walk failing"* — **silently hidden becomes visibly incomplete**. `EXTENSION-REGISTRY` §6a.3a is a **consumer** of that rule, not its authority |
| **A2** | **the reader** | **a second source that answered**, compared |
| **A3** | **the receiver** | a comparison pass that does not depend on having been told |
| **A4** | **the renderer** | state **which** payload it declined and **why** |

### 1.2 ⚠ A1's instrument does not solve A2, and assuming it does is the trap

A tracked prefix converts a missing node into a failed walk — **within one version**. It **cannot see
a stale version at all**: an older signed root is internally consistent and complete, every node it
names is present, nothing is omitted, and **the walk succeeds.**

⇒ **Omission and staleness are different absences requiring different instruments, and a mechanism
that ships one while reading as though it shipped both is worse than one that ships neither.** A2 is
the reason plurality of sources is structural in §6.4 rather than an optional resilience feature.

⭐ **And A1 has a reader-side twin that a reader inflicts on itself** — recorded independently while
this proposal was under review, and it is why §6.3's memoization rule is normative: *a partial walk
that is cached as though it were complete manufactures a withholding origin locally, permanently,
with nothing upstream wrong.* The reader then serves itself A1 forever, out of its own store.

### 1.3 ⚠ The fifth class is real, is the mirror image, and is NOT in this document

**A publisher cannot distinguish *nobody has read this* from *nobody can read this.*** Three recorded
instances, from three unrelated subsystems:

- a publisher who cannot serve live has **no way to stop every reader trying**, and experiences the
  failure as load they cannot attribute;
- a sender that discarded 2,327 notifications, **counted them**, and had nowhere to report it (§8.1);
- a file-serving side measured at **52 deposits / 0 collects**, unable to tell *"nobody wants this"*
  from *"nobody can reach me."*

⇒ **Named as a distinct class with no owner, deliberately excluded, so that it is not re-derived
piecemeal from whichever symptom surfaces next.** Two of this proposal's own open questions were
circling it before it had a name. §7 carries the exclusion; §11 Q2 carries the one part of it this
mechanism could host.

---

## §2 The evidence — seven implementations, and what they agree on

**Measured across two independent code bases at their current tips, in implementations that share no
code with one another.** Six of the seven carry no social vocabulary of any kind.

| Consumer | Subject | Social? |
|---|---|---|
| a durable cache of foreign artifacts — site manifest, application catalog, application bundle | three fixed keys | **no** |
| a static-origin content reader | one fixed key | **no** |
| *"is a newer build of this application available?"* | one fixed key | **no** |
| a folder replicated between two machines | a growing prefix | **no** |
| peer discovery on a local network | a candidate set | **no** |
| a declaration-to-substrate reconciler | a declared set | **no** |
| a reader following an author's publications | a growing prefix | **yes** |

**What they agree on, without having coordinated:**

1. ⭐ **None replays a change log.** Every one answers *am I current?* by **re-deriving current state
   and comparing by hash.** This is not taste — it falls out of content-addressing: when identity is a
   hash, currency is answerable by comparison, so a change stream is an **optimization** and not a
   requirement.
2. **Each was written because no shared mechanism existed.** One code base measured the cost inside
   itself: with no shared entry point owning the refresh trigger, *"it is re-decided at every call
   site, and **six consumers disagreed six ways**"* — after four production incidents.
3. **Two of them have already performed the extraction this proposal proposes, one layer down**, each
   naming the trigger as *a second consumer appeared*. In one case the two consumers were **file
   transfer and publication reading** — which is precisely the pair a prior design document names as
   the test for whether this is a mechanism.
4. ⭐ **Both code bases independently built the same ENTRY POINT** — one operation, called
   unconditionally, with the layers inside it and none of them visible to a caller. Neither was asked
   to; both report the call site, not the layering, as the thing that stopped the divergence. §6.0 is
   that finding promoted to a requirement.

### 2.1 ⚠ What the evidence does not establish

- **Two code bases, and not every implementation.** Reference implementations were examined and
  excluded by the stated definition: their loops are **connectivity** currency — liveness, keepalive,
  connection retry — not **data** currency.
- **Fit was assessed from each implementation's own framing, not by attempting the port.** The honest
  verdict for the non-social consumers is *"does not contort because it never came near the
  vocabulary"*, which is weaker than *"fits"*.
- **Two of the seven are boundary judgements** — one is partly connection-establishment retry, one
  reconciles against a local declaration rather than a remote sequence. **Strike both and the result
  survives on five.**

---

## §3 Why this is ONE mechanism — decided by a landed rule, not by taste

`SPECIFICATION-FORMAT.md` §8.8 `[MUST]` makes type identity **behavioural**:

> *"A new entity type is minted only when a conformant consumer must **behave** differently… The test
> is **behavioural, not taxonomic**: two things that a consumer treats identically are one type
> regardless of how differently a human would describe them."*

**Applied to the seven subjects above:**

| | do consumers behave identically? |
|---|---|
| subject identity · source resolution · currency · position · intent | **Yes** — and this is measured, not argued: seven implementations, no shared code, converging on the same answer to every one |
| **lowering** (what an arriving change *means* here) and **disposition** (may it be republished, cached, warned over) | **No** — and §8.9 already rules that disposition lives on the entity: *"containers do not travel; entities do"* |

⇒ **§8.8 says the loop is one thing. §8.8 and §8.9 together say meaning is many.** That is the
boundary, and it is a landed `[MUST]` rather than a new judgement. Both rules were promoted to
corpus-wide scope from a single-tier standard precisely because they were always true of every tier.

### 3.1 ⚠ A guard, because §8.8 alone under-splits — and there is a controlled case

**Two independent implementations applied §8.8 to the same pair** — a conversation message and a
publication entry — **and reached opposite conclusions.** One found *"a consumer holding one and a
consumer holding the other do not behave differently; they would be one model with one render path."*
The other found that they do: a message from a **closed** conversation **MUST NOT** be republished,
while a publication entry **MAY**, and republication is the entire mirror model.

**Neither was careless.** §8.8 under-splits because the tester enumerates the behaviours **they can
see**, and **disposition is the behaviour most easily missed, because it is about what a *downstream*
holder may do rather than what *this* consumer does.**

⇒ ***§8.8 must be applied together with §8.9, and a proposal invoking one must invoke both.*** Applied
as a guard against this proposal's own conclusion it **strengthens** it: across the seven subjects the
loop is identical **while disposition diverges completely** — an application bundle, a publication
entry, a replicated file and a discovery candidate share nothing about what a holder may do with them.
**That disposition diverges while the loop does not is the sharpest available confirmation of where
the line falls.**

---

## §4 Where it belongs — and the answer is two documents

### 4.1 The extension: Tier 2b, operational network

`SYSTEM-ARCHITECTURE.md` §13.1's own tests decide it:

| test | answer |
|---|---|
| Does a **single peer** need it to run? (Tier 1 is *"irreducible… together with Tier 0 and V7 they define what a peer is"*) | **No.** A lone peer acquires nothing ⇒ **not Tier 1** |
| Is it needed for *"production multi-peer deployments"*? | **Yes** — it is what makes a held copy converge ⇒ **Tier 2** |
| Which sub-group? | **2b**, whose own description is *"Cross-peer connection, **sync**, discovery, relay"* |

**Its siblings** — `NETWORK`, `DISCOVERY`, `RELAY`, `REGISTRY`, `ROUTE`, `SUBSTITUTE` — are each a
*cross-peer capability built on Tier 1*. This is the same shelf.

### 4.2 ⭐ The emergent half belongs in `SYSTEM-COMPOSITION.md`, and one landed `MUST` proves it

`SYSTEM-COMPOSITION.md` §1 states its own subject:

> *"cross-extension **emergent properties** — the behaviors that arise when multiple system extensions
> compose… Individual extension specs define what each extension does…; **this document defines how
> those extensions interact.** … normative for peers implementing multiple system extensions."*

**Every implementation surveyed in §2 is a composition**, not a new primitive: tree walks, content
addressing and closure, subscription, transport, continuation, and merge. **No single extension owns
the loop.**

⭐⭐ **And this explains a defect rather than merely classifying one.** `EXTENSION-SUBSCRIPTION` §6.3
property (ii) requires:

> *"Convergence **MUST** hold under delivery loss: a peer that misses notifications **MUST** still
> reach the latest state, **via §5.5 gap detection and re-read**."*

**That is a cross-extension emergent property stated inside a single extension's spec — which is
exactly why it names a detector that its own extension owns and that cannot deliver the property**
(§8.1). ⇒ **The property belongs at the composition layer; the detector stays with subscription. The
defect is a layering defect, and the tier model already had the right home for it.**

### 4.3 ⛔ The word *convergence* already means something else in that document, and the chapter must not collide

`SYSTEM-COMPOSITION.md` **§5 is titled *Convergence and Termination*** and is about **local reactive
cascade termination** — *does the write-loop stop at one peer*, with a convergence check that
suppresses a write whose content hash already matches. **This chapter is about *do two peers hold the
same bytes*.** Same word, different subject, one document.

⇒ **The chapter is named for its question, not for the word** — *Holding another peer's data* — and
**§5 gains one disambiguating line** pointing at it (§12 D2a). This corpus has now had four recorded
instances in one arc of *a name that already means something else*, three of them caught by a reader
rather than by review; declining to add a fifth costs one sentence.

### 4.4 The conventions keep their tier and lose scope

Application conventions that currently *contain* the loop **stop being it and start registering
against it**. Their tier does not change. What changes is that the mechanism is no longer restated
per convention, which is what produced the divergences in §9.

---

## §5 The shape — one closure property, five layers, one seam

### 5.0 ⭐⭐ THE CLOSURE PROPERTY — the reduction, and the reason this federates

**This is the load-bearing sentence in the document, and revision 2 did not contain it.**

> **[MUST] What a peer obtains through this mechanism, it MAY publish. A published result is the
> SAME KIND OF OBJECT as the sources it was built from — signed entries in the publishing peer's own
> namespace — and MUST be verified, attributed and rendered by the IDENTICAL code path.**
>
> **[MUST] The closure is over the AGGREGATOR's output.** A gatherer's result **MUST** be consumable
> by another gatherer with no new type and no new code path. **The fixed point is
> `gathered → gathered`.**
>
> **[MUST NOT]** Closure does **not** require that a gathered set be **walked** the way an author's
> own set is, and an implementation **MUST NOT** publish an author's own set-layer object under its
> own namespace in order to satisfy it.

#### 5.0-a ⛔ Which LAYER is closed — revision 4's sentence was literally false, and taking it literally builds a forgery

**Revision 4 said *"consumable by the identical code path"* flat. The seat that built the path measured
it and the sentence does not survive contact:**

| layer | identical? |
|---|---|
| **entry** — decode, fetch the author's detached signature, attribute, render | ⭐ **YES, literally.** One `finish_entry`; no mirror-flavoured copy, no weaker path |
| **set** — how you find the entries | ⛔ **NO.** An author's feed is an index head plus key-addressed pages; a gathered view is one record naming pins. **Two walks** |

⛔ **And the literal reading is not merely imprecise, it is dangerous: a reader who took it at face
value would go looking for the author's own index shape UNDER THE GATHERER** — which is the gatherer
**forging the author's index**, and unauthenticated besides, since the authorship instrument signs
*entries* and not the set.

⇒ **The property closure actually needs is that the output carries the same authorship evidence the
input did** — which is the entry layer, and which is exactly what §5.0.0's two `MUST`s pin. **The set
layer is where an aggregator is allowed to be a different shape, because that shape is its own and it
is signing it in its own name.** *The anti-centralization argument is untouched: the failure in the
table below is that the aggregator's output is not fetchable, verifiable, republishable content at
all. A gathered record is all three.*

⚠ **So the trace's sentence needs the same scoping.** *"Erin follows Bob the same way she follows a
person"* is **true of how she verifies and renders** and **false of how she walks** (§15.1).

**Why it is the whole design and not a nicety.** If a peer's *output* is a different type from its
*input*, aggregation cannot be aggregated: one party ends up holding what everyone needs, and there is
exactly one of them. That is not a policy failure in the systems that have it, it is a typing failure:

| system | its aggregator's output | can the next layer consume it? |
|---|---|---|
| a firehose-consuming index service | **an API** | **no** |
| a relay | **a query response** | **no** |
| a federated instance | **its own database** | **no** |
| ⭐ **here** | **signed entries, exactly like a source's** | **yes — and that is the difference** |

⇒ ***They did not centralize because someone chose to. They centralized because their aggregator's
output was a different type from its input.*** A mechanism that is closed under its own output has no
privileged tier to centralize into: a peer subscribes to an aggregating peer **the same way it
subscribes to a person**, an aggregator can aggregate aggregators, and one code path merges both
because they are one type.

⭐ **And the degenerate case is the same mechanism at zero infrastructure.** A reader that trusts
nobody runs the walk itself over the peers it already follows. **The spectrum from *no infrastructure
at all* to *a well-resourced public index* has no discontinuity and no protocol change in it** — only
how much work a peer chose to do.

⭐⭐ **And it is the read-amplification fix, which is the reason a developer reaches for it first.**
Arithmetic over the measured 2-request no-op check:

| a reader rendering an unchanged timeline | requests |
|---|---|
| following **500** authors, one subject each | **1,000** |
| following **one republished walk** | **2** |

⇒ **250×.** *Closure is argued above as the anti-centralization property. It is also the scaling
mechanism, and that is the more immediate reason to build it.*

⚠ **The obvious objection, answered — and marked DERIVED, not measured.** If a gatherer publishes
**one gathered record per subject**, does Erin not pay per subject again? **No, and the reason is
which layer the witness sits at: the witness is the GATHERER's signed root, not the subject's.** One
root comparison answers *did anything Bob gathered change* for every subject at once, because every
gathered record is an ordinary binding in Bob's own tree — §5.0's closure supplying the property a
second time. **The 2-request no-op is per-gatherer and does not scale with the number of subjects
gathered.** *Arithmetic over a measured 2-request no-op plus a derivation about where the witness
lives; nobody has built a 500-follow reader (§16.4), and the delta case is root-plus-descent, not 2.*

#### 5.0.0 ⛔ Closure has TWO preconditions, and revision 3 asserted both were already handled

**Revision 3 said *"the publication/authorship split is already landed… nothing new needed to do it."*
Both consuming seats went at that sentence — one by reading the substrate, one by running the path —
and between them they established that it is half true in a way that matters.**

> **[MUST] BYTE PRESERVATION.** A republished entity **MUST** be bound **byte-identically** to the form
> in which it was obtained. An implementation **MUST NOT** re-encode it — **including by decoding it
> through a type it does not fully declare.**

**Measured, with a control arm, and this is the finding that would have cost somebody a week.** The
closure path was built A → B → C: B republishes byte-preserving, C consumes with the same consumer, no
branch, **every entity hash byte-identical at both hops.** The control arm republished *the way a real
gatherer would* — decode the body through a struct, put it back, with a struct that does not declare
one of the publisher's fields, **which is the realistic case because a gatherer aggregates types it has
never heard of**:

⛔ **3 of 3 hashes moved. No error anywhere.**

**And the consequence is not a lost field — it is authorship.** A detached signature is bound at
`/{author}/system/signature/{hex(entry_hash)}`; move the hash and **the signature no longer names the
entity**, so a conformant renderer must present every republished entry as **unattributed**. ⇒ ***a
gatherer that re-encodes produces a complete, verifiable, correctly-walked publication in which nobody
wrote anything.*** **That is the centralization failure in a different costume: the output stopped being
the same kind of object, because it lost its authorship.**

> **[MUST] AUTHOR-ANCHORED EVIDENCE.** An object republished through this mechanism **MUST** carry
> evidence of authorship that **survives detachment from the author's signed root** — a detached
> per-entity signature at the invariant pointer, or an inclusion proof. **A republished object with
> neither carries integrity without authorship and MUST NOT be presented as attributed.**

⚠ **The framing of the finding needed one correction, and it makes the fix cheaper rather than
weaker.** It was reported as *"the substrate supplies none of it."* **The substrate supplies the
INSTRUMENT — it is corpus-wide, in twenty documents, and one landed extension states it as a general
fact: *"everywhere else in this ecosystem a signature binds at the invariant pointer
`/{signer_peer_id}/system/signature/{target_hash_hex}`."*** A canonical peer id is a commitment to its
public key, so such a signature verifies **with no key distribution and no second fetch** — which is
exactly why it survives detachment.

⇒ **What is application-scoped is the OBLIGATION, not the mechanism.** The requirement that a
republished entry *carry* it, that a mirror may never *supply* it, that its absence renders as
*unattributed*, and that republished bytes are *never re-encoded* are four rules that exist **only in
`APP-CONVENTION-FEED`.** **So this is a promotion of existing normative text to the tier where closure
is claimed, not an invention** — the same move already accepted for that convention's audit rule one
section over, and more expensive: *that one's absence costs an audit line; this one's absence costs the
property the whole design rests on.*

**The failure without it is silent and specific:** a second convention ships a gathered view, inherits
closure, does not know about a feed convention's rules, and renders correctly-verified entries
attributed to whoever handed them over. **No gate sees it, because the bytes check out.**

#### 5.0.0a ⚠ Closure is a CAPABILITY. It is not a permission

**[MUST]** Closure states what the mechanism **can** carry. **Whether a given object may be
republished is that object's disposition** (§3.1, and `SPECIFICATION-FORMAT` §8.9) — *the mechanism
neither grants nor withholds it.*

⚠ **Stated because the closure sentence is billed as the load-bearing one and will therefore be read
alone** — and read alone, an unqualified *"what a peer obtains, it may publish"* licenses republishing
a message from a closed conversation, which §3.1 of this document `MUST NOT`s two sections earlier.

### 5.0.1 The reduction underneath it, and it is a theorem rather than a preference

> **The pattern reduces to MONOTONE ACCUMULATION of independently-signed claims over coordinates any
> party can compute.**

**Monotonicity is what buys coordination-freeness, and the equivalence is exact** — a program has a
consistent, coordination-free distributed implementation **iff** it is monotone (the CALM result,
Hellerstein & Alvaro, arXiv:1901.01930). Every part of this mechanism inherits from that:

- **comparison-primary currency** (§6.3) works because more information never retracts an answer;
- **partial source sets compose by union** (§6.2) — associative, commutative, idempotent, no ordering,
  no conflict, no trust in either source;
- **plural sources are safe** rather than a consistency hazard;
- ⚠ **and the places the mechanism must coordinate are exactly the non-monotone ones — deletion,
  retraction, and any "have I heard everything?" question.** That is why §6.7 guarantees eventual and
  refuses bounded: *a bound is a claim to have heard everything, and nothing here can make it.*

⭐⭐ **And the theorem locates the one place this mechanism is NOT coordination-free, which is worth
more than stating the theorem.**

> **The reconciliation rule a `shared` subject declares IS the coordination point the reduction
> predicts.** `owned` and `ownerless` subjects are coordination-free. **A `shared` subject is not, and
> its declared rule is where the coordination was moved to.**

**Reconciliation is non-monotone by construction** — a last-arrival-wins policy converges a folder by
**replacing** a file, and more information retracts an earlier answer at the surface a person sees.
⇒ **an implementer who reads §5.0.1 alone will conclude the whole mechanism is coordination-free and
then build a `shared` subject expecting it, which is the one place it is not.** *It also explains,
rather than asserts, why §6.5 requires the rule to be the subject's: **a coordination point each party
picks for itself is not one.***

### 5.1 ⭐ The roles are ACTS, not tiers — and the vocabulary survives intact

`publisher · consumer · gatherer · tracker` are the words this project has used throughout and they
are **kept**, with one rule that keeps them from hardening into an architecture:

| The act | What it is, mechanically | Required? | Privileged? |
|---|---|---|---|
| **publish** | write signed entries into your own namespace and make them reachable at one or more sources | every peer does it | no |
| **consume** | the §6 loop against a subject | every peer does it | no |
| **gather** | **publish a walk** — consume from N sources and publish the result, which by §5.0 is the same kind of object | **no** | **no** |
| **track** | answer **`coordinate → who`** — the participant set for a subject nobody owns (§6.2.4) | **no** | **no** |

> **[MUST NOT] The mechanism MUST NOT name a role as a required participant, and MUST NOT give any
> role a distinct output type.** *A noun for a role invites a tier, and a tier is the thing §5.0 makes
> unnecessary.*

⚠ **This is deliberately narrower than "roles do not exist."** No role is *required* and none is
*privileged* — **and dedicated operators are a legitimate, expected deployment shape.** There is at
least one job only an always-on party can do at all: carrying a non-publishing client's writes into
the tree. **A peer that cannot publish cannot participate in the write direction without one.** Delete
the role from the *mechanism*; do not delete the possibility from the *deployment*.

⭐ **The publication/authorship split that makes this safe is already landed** — a republished entry is
attributed to its author, never to the peer that carried it, and a renderer that does otherwise is
non-conformant. **So gathering is publishing, on the same rails, with nothing new needed to do it.**

---

### 5.3 The layers — five, one seam, and each justified by an absence it makes attributable

**Each layer earns its place by naming something that, without it, is indistinguishable from
something else.** A layer that cannot answer that column does not belong — **and that test removed two
of the eight this proposal opened with** (§5.2).

| | Layer | The job | Without it, these collapse |
|---|---|---|---|
| **1** | **SUBJECT** | the **source-independent** name of the mutable thing, its **kind**, and its **authority class** (§6.5) | *nothing changed* from *everything changed* — observers minting one identity each |
| **2** | **SOURCE** | a place that may answer about a subject. **Ordered, plural**; resolution is the source's, not the subject's; every attempt yields one of **six outcomes** | *could not ask* from *asked, nothing there* from *refused* (A2, A3) |
| **3** | **WITNESS** | whatever a source offers that compares **equal iff the subject is unchanged** — this is what currency is | **authentic-and-stale from authentic-and-current** (A2) |
| **4** | **POSITION** | held on the **(reader, subject) pair**, and **droppable** | *I stopped following this* from *I have not read it lately* |
| **5** | **INTENT** | a **durable declaration** plus a loop that reconciles substrate to it | declared from merely attempted; **process state dies with the process** |
| **seam** | **LOWERING** | what an arriving change **means** here, and the right to **decline out loud**. **The convention's, not the mechanism's** | *not offered* from *declined* from *broken* (A4) |

### 5.3.1 ⭐ Resolution belongs to the SOURCE, and that is the load-bearing separation

**One derivation produced a subject with no address; the other produced addressing with no subject —
and measured that essentially all of its vocabulary coupling lived there:** thirteen address
constructors, each a pure projection with no branch and no protocol step, against one piece of
transfer logic.

**Plural sources require the separation.** You cannot have many sources for one subject unless the
subject's identity is separable from its address at any one of them.

⭐ **Revision 2 makes this structural instead of prohibitive.** The first draft carried addressing as
its own layer with a `MUST NOT` forbidding a subject to encode an address. **Making resolution a
property of the SOURCE says the same thing by construction** — there is no place on a subject for an
address to go — and it is the stronger form, because a prohibition is obeyed and a structure is
inherited.

**And in this substrate the separation is real, measured before the question was asked:** a publish is
a **projection of the tree**, so the key a signed-root consumer resolves and the path a live reader
dispatches at are **the same string with a peer segment in front.**

⭐⭐ **The corpus already pinned this exact shape one noun over.** `EXTENSION-SUBSTITUTE` §2.1:

> *"`endpoint` is **OPAQUE**… The substrate **MUST NOT** pin `endpoint` to a single concrete type…
> **An implementation that tightens `endpoint` to a single endpoint type is non-conformant (it
> forecloses non-HTTP conventions).**"*

**The dispatch key dispatches; the endpoint is opaque and per-source.** §6.2 is that ruling applied to
subjects instead of substitute sources, and §9 records where landed text does the opposite.

### 5.3.2 ⚠ Two layers were removed, and saying why is the test for the other five

- **COMMITMENT is not a layer of this mechanism; it is a dependency on the core.** Verification is
  `ENTITY-CORE-PROTOCOL` §1.2 and §3.5 plus `EXTENSION-TREE` §3.3a, and **this mechanism adds nothing
  to it**. Listing it as a layer implied it did. Its one load-bearing statement — *authentic is not
  current* — is where it belongs, as the sentence that motivates WITNESS (§6.3).
- **ADDRESS folded into SOURCE**, per §5.1.

⇒ **The remaining five each answer the "without it, these collapse" column with a distinct absence.**
That is the whole membership test, and it is the one to apply to any future addition.

---

## §6 Normative content

### 6.0 ⭐⭐ THE ENTRY POINT — one operation, and it is normative

**This section is first because it is the one that decides whether the mechanism reduces anyone's
work.** The first draft specified eight layers and **zero call sites**, and both consuming
implementations returned the same objection independently: a specification that describes the parts
and requires no chokepoint **permits the exact defect it exists to remove**, and its check set cannot
tell a conformant implementation from the pre-mechanism state.

**[MUST]** An implementation **MUST** expose this mechanism as **one operation**. Its inputs are a
subject and the reader's intent; its result is exactly one of:

| result | means |
|---|---|
| **current** | the held copy is the current one — nothing was transferred |
| **updated** | a newer value was obtained; the held copy is now it |
| **unavailable** | no source served, carrying **the per-source attempts** and their §6.2 outcomes |

⭐ **[MUST] `current` and `updated` carry qualifiers — at minimum `truncated` and `disagreement`.**

⚠ **Found by trying to write the signature down, which is the only way this surfaces.** §6.2.3 `MUST`s
that disagreement between two served sources be **reportable**, and §6.3.4 `MUST`s that a truncated
walk be **reported as truncated** — but a bare three-valued result can express neither, so **the only
legal API could not satisfy two of the specification's own requirements**, and an implementation would
have to bypass the entry point to comply with §6.3.4 — which §6.0 makes non-conformant. *Small,
mechanical, and it makes two `MUST`s reachable that were not.*

**[MUST NOT]** A consumer of this mechanism **MUST NOT** issue a source read for an acquired subject
directly. **An implementation in which a consumer can bypass the entry point is non-conformant.**

**[MUST]** ⭐ **Holding a copy is never a reason not to ask.** The check is issued unconditionally; the
**source** may answer *unchanged*, the **consumer** may not assume it. *This closes the skip that four
production incidents were bought with, and it is invisible to every other requirement here — §6.6's
witness cannot see it, because **a pass that never ran reports nothing.***

⚠ **The trigger is not the schedule, and the two requirements above are not a cadence.** When a
consumer needs the subject, the operation is called. How often a **background** pass runs is local
policy (§7). One qualifier is normative, because the general form is an outage generator:

**[MUST]** An implementation **MUST** state which of its reconciling entry points may be driven by a
timer, and **a pass that establishes outbound connections MUST NOT be timer-driven by default.**
*Observing and dialing are two operations; a wake-up that dials turns an opened panel into a
connection storm.*

#### 6.0.1 ⭐ The default profile — a worked composition, so that the ordinary case is not eight decisions

**[SHOULD]** An implementation **SHOULD** offer a **default profile**, and **MAY** declare conformance
against one as a unit. They are **recognition, not invention.**

⚠ **Revision 3 offered ONE profile and claimed *"four of the seven surveyed consumers already are
exactly this."* Re-measured against the four one seat owns, the true count is ONE — and the correction
matters less than what it revealed: there are TWO common profiles, and the second is the more common.**

| | **A — the published prefix** | ⭐ **B — the single artifact** |
|---|---|---|
| **SUBJECT** | a prefix in a tree, under a peer | **one mutable artifact at a stable key** |
| **SOURCE** | ⭐ `[live peer, static origin, a peer republishing this subject's walk]` | the same three, wherever offered |
| **WITNESS** | the signed root pointer, then trie descent by hash equality | ⭐ **an opaque pointer at a stable key** |
| **requires** | a publication act, a root, a signature | ⛔ **none of those** |
| **POSITION** | `(reader, subject)`, droppable | often none — there is no backlog |
| **INTENT** | a durable declaration plus an idempotent reconcile | a declaration, **or none at all** (§12 `D16`) |
| **LOWERING** | identity — the bytes are the thing | identity |
| **instances** | a feed (§15.1) | **a durable cache of foreign artifacts · an application-build check · an index head · ⭐ §15.4's game board — *class 1, fixed-key mutable*, in this document's own trace** |

⭐ **Profile B is what most consumers are, and revision 3 did not offer it.** *A developer whose subject
is one mutable artifact reads the default profile, finds a signed-root requirement they have no use
for, and composes their own five decisions — the exact failure a default profile exists to prevent, at
the case it does not cover.*

⭐⭐ **And the third SOURCE leg is not decoration — it is the 250×.** Revision 3 listed
`[live peer, static origin]` only, so **a developer implementing §6.0 and §6.0.1 as written builds the
1,000-request version** (§5.0). The republished-walk leg is what makes the 2-request version reachable,
and it is the centre of §15.1's trace.

**Why this is normative and not a guide note.** Without it the failure the mechanism was extracted to
fix **recurs one layer up**: every implementation composes the five decisions differently and *all of
them are conformant.* A mechanism with no default is a substrate specification, and the stated goal is
that an application developer does not rebuild this.

### 6.1 SUBJECT — a subject is named independently of where it is reached

**[MUST]** A **subject** is identified by a value that does not change when its current bytes change
and does not encode any one source's addressing.

**[MUST]** A subject declares a **kind**. The kind is the dispatch key; a handler for that kind
resolves a subject to bytes **at a given source**.

**[MUST]** A subject declares its **authority class** — `owned`, `shared` or `ownerless` (§6.5).

**[MUST NOT]** An observer's own observation state — when it looked, what it saw — **MUST NOT** be a
field of the subject entity. *A subject observed by N readers must have one identity, not N.*

**[SHOULD]** Dispatch follows `EXTENSION-SUBSTITUTE` §6's pattern: **no new registry surface**; a
convention ships a handler at a conventional operation path and is found by ordinary core-protocol
dispatch. A subject kind with no installed handler yields `not_found` and the source chain advances.

### 6.2 ⭐ SOURCE — the ordered set, and the outcome taxonomy is normative

**[MUST]** A subject **MAY** have more than one source, and sources are **ordered**.

**[MUST NOT]** The mechanism **MUST NOT** pin a source's addressing to a single concrete scheme. An
implementation that does so is non-conformant, for the reason `EXTENSION-SUBSTITUTE` §2.1 already
gives: it forecloses the source kinds nobody has thought of yet.

#### 6.2.1 Eight outcomes, and the rule that generates them

**[MUST]** Each attempt yields exactly one of **eight outcomes, and an implementation MUST NOT merge
any two of them:**

| outcome | means | whose fact is it — *where does it send a person?* | terminal? |
|---|---|---|---|
| **served** | here it is | nobody's | — |
| **unreadable** | bytes arrived and **cannot be used** — they do not decode, or do not verify | **the publisher's encoder** | **no** |
| **absent** | *we do not have this* | **the publisher's intent** — they hold nothing here, by choice | **yes** |
| **refused** | *we have it and have not shared it with you* | **the asker's authority** | **yes** |
| **faulted** | they answered, with a failure | **the source's operation** | **no** |
| **unheard** | nothing came back at all | **nobody's — no durable conclusion may ever be drawn from it** | **no** |
| ⭐ **declined** | the source served, the bytes are usable, **and the reader refused them under its own policy, which it names** | ⭐ **the READER's POLICY** — *which writer am I talking to?* | **no** |
| ⭐ **unsupported** | the source served, the bytes are authentic and well-formed, **and this reader cannot use or keep them** — an unimplemented payload arm, a cap, a budget, a full disk | ⭐ **the READER's CAPABILITY** — *this build, this machine* | **no** |

⭐ **The generative rule, so that the next person does not re-derive the list:**

> **Two outcomes may be merged only if they send a person to the same place *and* agree on whether
> asking again can change the answer.**

⭐⭐ **And the seventh was found by the rule being FALSIFIED as asked — the gap was not a missing row,
it was a missing OWNER.**

> **Every one of revision 3's six is the source's fact, the publisher's fact, or nobody's. There was no
> outcome whose owner is the READER.**

**Two live conditions are reader facts.** The sharp one: a source **served**, the bytes are authentic,
they decode, **the signature verifies** — and the reader refuses them on its own monotonicity policy,
*deliberately after the signature check, because a rollback is a correctly-signed root being replayed.*
Run it through the generative rule: it sends a person to *"which writer am I talking to — has this
publisher rolled back, or is this their second device?"* — **a destination none of the six names, and
precisely the one §6.5.2's multi-device ruling created.** Asking again may change it ⇒ non-terminal. It
merges with nothing: `unreadable` sends you to a publisher's encoder that is working, `faulted` to an
operation that is fine, and `refused` **inverts the authority** — there *we* refused *them*.

⚠ **Why this is not tidiness: a reader-owned outcome had nowhere to be reported, so on one of two
otherwise-identical legs the refusal simply did not exist.** *An absent row in a taxonomy is an absent
obligation.*

##### 6.2.1a ⭐⭐ The reader owns TWO rows, not one — and the generative rule is what splits them

**Revision 4 filed *"served bytes the reader cannot store — a cap, a budget, a full disk"* as a second
instance of `declined`, evidence that the reader-owned row was a class rather than a one-off. The seat
that shipped both of them separately pointed out that it is not one class, and the document's own rule
is what decides it:**

> **Two outcomes may be merged only if they send a person to the same place *and* agree on whether
> asking again can change the answer.**

| | **`declined` — reader POLICY** | **`unsupported` — reader CAPABILITY** |
|---|---|---|
| the sentence | *I could, and I will not* | *I would, and I cannot* |
| the instance | the anti-rollback floor refuses a correctly-signed replayed root | an authentic payload whose arm this build has not implemented |
| where it sends a person | **to the WRITER** — has this publisher rolled back, or is this their second device? | **to THIS MACHINE** — upgrade the build, raise the cap, free the disk |

⇒ **Different destinations, so by the rule they do not merge.** *Merging them tells a person to go and
investigate a publisher when the answer is that their own disk is full — which is the taxonomy's
founding complaint, one row further in.*

⭐ **And the coverage rule is sharper than revision 4 stated it, which is what the split teaches:**

> **[MUST] The taxonomy is generated by DESTINATION, not by party. Every party that can cause an
> attempt to end owns *at least* one row, and a party with two destinations owns two.**

*Stated the old way — one row per party — the taxonomy is closed at seven and the eighth row is
unreachable by construction.* Source, publisher's encoder, publisher's intent, asker's authority,
nobody — **and the reader, twice.**

⭐ **A second seat has now produced a reader-owned outcome independently, in an unrelated application**,
which is the evidence §16.4 said the row deserved: a delivery that would land on a local edit while the
subject's reconciliation rule is unreadable is **held** — served, authentic, usable, refused under a
named reader policy, released on the next pass. *`declined` exactly, arrived at without reference to
this taxonomy. We are classifying their construct, not claiming they built against the row.*

**This is why four was too few.** The first draft's *"could not ask — we could not reach them, or they
faulted"* merged two conditions inside the row whose own rule forbids merging: a failure the source
reported is a fact you may act on, and silence is a fact about nothing. **`unreadable` is the other
addition**, and it is this corpus's own tier-wide rule (`GUIDE-APPLICATION-DEVELOPMENT` §3 — *a
deliberate schema violation is not a corrupt byte*) arriving at the transport attempt: bytes that
reached you and cannot be used are the publisher's defect, and folding them into *faulted* sends
somebody to inspect a network that is working.

**Why merging is a defect and not untidiness:** folding *refused* or *unheard* into *absent* converts
*we do not know* into the confident claim *they have nothing* — which sends a person to inspect the one
place the problem is not.

#### 6.2.2 Terminality is a property of the answer, not of the schedule

**[MUST]** An implementation **MUST** distinguish *is there any point asking again* from *when to ask
again*, and **MUST NOT** derive the first from the outcome's transport spelling.

**Only an answer the source *chooses* is terminal** — *absent* and *refused*. A failure, a dropped
connection, a truncated body arriving as a decode error and a mangled one arriving as a hash mismatch
are **transport faults wearing a content fault's name**, and are not terminal. **Measured, in both
directions:** a ladder that branched on the error variant rather than on terminality spent **~9 s
waiting on an answer that arrived in the first 200 ms**, and the health finding that would have
reported it could not appear until the ladder ended — while the opposite error, calling a transient
fault terminal, turns a momentary origin blip into a permanently missing application.

#### 6.2.3 Serving stops the walk; answering does not

⭐ **[MUST]** An **empty** answer from one source **MUST NOT** end the ladder. *A publisher who serves
from a static origin because they cannot carry live load has a live tree with no publication in it; a
reader that stopped at the first source that answered would report "this author has published nothing"
to somebody one hop from their whole archive.*

**[MUST]** Where two sources both **served**, an implementation **MUST** be able to compare them, and
the specification **MUST** define what disagreement means. **Whether a deployment enables the
comparison is a reader policy decision; the semantics are not optional.**

⭐ **[MUST] Disagreement is an OUTCOME, not a resolution.** Two legs serving different bytes yields a
**served-class** outcome carrying **both** answers, and the requirement is that it is **reportable**.
Which one a reader uses is policy; **that they were told is not.** *Merging them would put a reader in
the business of adjudicating between two publishers' origins, which is the ownerless column (§6.5)
arriving where it does not belong.*

⚠ **Qualified for `owned`, and the seat that proposed the rule is the one that qualified it.** For an
`owned` subject **there are not two publishers** — there is one author, one signed succession, and two
sources holding different points in it.

> **[MUST]** For an `owned` subject, two served answers are **ordered by the author's succession**
> (§6.3.1a), the later wins, and the earlier source is reported as **behind** — not as **disagreeing**.
> **Disagreement is reserved for two answers at the SAME succession point with different content.**
> For `shared` and `ownerless` the unqualified rule stands.

***A source that is merely behind is the normal steady state of a federated network, and calling it
disagreement makes the ordinary case look like a fault.*** **And reporting it unresolved hands a reader
a decision the author already made.** *Note where the reserved case lands: two answers at the same
succession with different content is exactly the equal-sequence cell that has no defence — so the
disagreement outcome now names precisely the condition nothing else detects.*

**Rationale — this is A2, and it is the only instrument that sees it.** A second source that *answered*
is the only way a reader can distinguish a withheld newer root from a genuinely quiet publisher. §1.2
shows why the publisher-side instrument cannot substitute.

#### 6.2.4 ⭐⭐ Where the source set COMES FROM — and for a subject nobody owns it must be discovered

**For an `owned` subject the source set is given:** the writing peer, its published origin, any peer
republishing it. **For a `shared` or `ownerless` subject it is not** — the contributors are N
independent writers, each owning only their own namespace, and *the reader holding the subject has to
find them.* **That discovery is a SOURCE-layer problem, and it is the same problem in every
application:**

| Application | The subject | The contributions |
|---|---|---|
| forum / social | a thread's root entry | replies |
| forge | a repository | patches, issues, reviews |
| wiki / encyclopedia | a page | edits |
| marketplace | a listing | offers, bids |
| chat | a room | messages |
| **a game** | **the board** | **each player's moves** |

**Three things are needed and only the third is unsolved.** ① a **coordinate** — a name for the subject
any party can compute without coordinating, which is what a content hash is; ② **contributions that
name the coordinate** — signed entries in the contributor's own namespace; ③ **the reverse edge,
`coordinate → contributions`.**

⭐ **The move: the reverse edge is TWO indexes, and every surveyed system builds the fused one.**

| | maps | size per edge | trust needed | who can answer |
|---|---|---|---|---|
| **A · `coordinate → WHO`** | subject → the peer-ids that contributed | **~32 bytes** | **none** | anyone |
| **B · `WHO → WHAT`** | a peer → their entries | their stream | **none** (signed, content-addressed) | the peer, or any source republishing it |

**Index B is this mechanism, already specified above.** ⇒ **the only genuinely missing thing in the
whole system is index A, and index A is small.** Fusing A and B is what forces a firehose, and the
firehose is what makes that tier the centralized one.

**Three properties, and the second is why this is safe to take from anyone:**

1. **Index A can OMIT but cannot FORGE.** A claim *"Bob contributed to C"* is verified by fetching
   Bob's entry and checking its signature and that it names C. A forgery fails; a fabrication costs one
   wasted fetch. **This is the same *omit but never substitute* property already ratified for
   republished content, arriving on a much smaller object.**
2. **Omission is repairable by UNION.** A participant set is a set of peer-ids, so merging two partial
   indexes is set union — §5.0.1's monotonicity, exactly. **Two partial answers compose into a better
   one with no coordination and no trust in either.**
3. ⭐ **It is SELECTIVE, and this rather than size is the structural win.** A fused index must ingest
   everything because it cannot know what will be asked. **A peer tracking `coordinate → who` covers
   only the coordinates it chose** and is complete-ish there while holding nothing else. *You cannot
   selectively fuse — a fused index is all-or-nothing by construction.*

⚠ **The precedent is the one that actually scaled, and where it breaks is load-bearing.** A
peer-discovery tracker keyed by a content identifier stores contact information and never content —
that is `coordinate → who`, it ran on hobbyist hardware, and it is why that system's content layer
never had to centralize. **But its peer set is a REDUNDANCY set and ours is a COMPOSITION set:** there
every peer serves the same bytes, so a partial list costs nothing; here **each participant wrote
something different, so a partial list means you get less content.** ⇒ **coverage is a real cost for us
and the participants are not fungible** — which fifty you got matters.

⭐⭐ **And the base case needs no index at all — this is the thing not to lose.** A reply is an entry in
the replier's own stream, so **if you follow them, it arrives in the fetch you were already doing.**
Better: every entry carries its **outbound** edges, so pulling a stream you already follow tells you
that subjects you have never seen exist, and you can fetch them. **Forward traversal over the content
graph is free and it discovers peers you do not follow.** ⇒ the reverse index answers exactly **one**
query and should be scoped that narrowly:

> **Given a coordinate, find contributors I do NOT already follow.**

**That is a far smaller claim than "how do threads work." Threads work by following people. Tracking
buys reach past your own graph, and nothing else.**

**[SHOULD]** A peer **MAY** publish `coordinate → who` sets for coordinates it chose. **[MUST]** Such a
set is **signed entries in the publishing peer's own namespace** — §5.0's closure, so that a
participant set is itself consumable, mergeable and re-publishable by the same code path as any other
subject. **[MUST NOT]** An implementation **MUST NOT** treat a participant set as authoritative or
complete; it is a hint that can only omit.

⚠ **`coordinate → who` is absent from the corpus today** — the existing indexes are `name → peer`,
`peer → next hop`, and a store-local `path → referrer`. **This is the one genuinely new object in the
whole design**, and it is ~32 bytes per edge.

### 6.3 ⭐⭐ WITNESS — currency is comparison, and the witness is opaque

**[MUST]** Verification establishes that bytes are **authentic**. It **does not** establish that they
are **current**, and an implementation **MUST NOT** present a verified result as a current one.

**[MUST]** Convergence is established by **comparison of current state** at a subject. A change stream
is an **optimization** that reduces how often comparison must run. **A subscriber that ignores the
stream entirely MUST still converge.**

#### 6.3.1 The definition, and it replaces revision 1's two-row table

> **[MUST]** A source offers zero or more **witnesses** over a subject. A **witness** is an **opaque
> token**, obtained at a **stable address**, that compares **equal iff the subject is unchanged at that
> source**. A reader establishes currency by comparing the witness a source offers against the witness
> of the copy it holds. **Where a source offers no witness, the reader MUST establish currency by
> enumerating the subject and comparing by content.**

**[MUST NOT]** An implementation **MUST NOT** require a witness to be a content hash. *A content hash
is this substrate's spelling of a witness and is not the only conformant one* — an entity-tag plus a
conditional read at an ordinary web origin is a witness, and a rule spelt as *"compare the held copy's
own content hash"* cannot express it. **This is §6.2's opacity rule one column over: pinning the
witness type forecloses exactly as much as pinning the address type**, and it decides whether an
application developer can point this mechanism at a web origin that has never heard of this protocol.

⭐ **[MUST] The witness is selected by the *(source, subject)* pair — never by the subject's shape
alone.** Revision 1 keyed the check on the subject and **both consuming implementations returned the
same counterexample from opposite ends**: a growing prefix whose publisher offers a moving fixed-key
head is a *prefix subject* with an *artifact-shaped check* — which is what an index is for — and a
live peer that has performed no publication act offers **no root at all**, so the cheap arm is
unavailable in the topology one product actually ships in. *One keyed on the subject cannot express
either.*

#### 6.3.1a ⭐⭐ Three witness classes — because revision 3's blanket clause forbade the instrument another `MUST` requires

**Revision 3 said flatly: *"a witness is a fact about one source; witnesses from two sources are not
comparable."* Both seats found that unimplementable, from opposite directions, and one of them found it
by measurement.** §6.2.3 `MUST`s that where two sources both served, an implementation **MUST** be able
to compare them — **so the blanket clause forbids the only instrument that satisfies it.**

| class | examples | comparable across sources? |
|---|---|---|
| ⭐ **content-derived** | a trie root over a binding set · a content hash | ✅ **YES** — and it is what makes the two-source comparison implementable at all |
| ⭐ **author-anchored** | an author-signed succession counter carried inside a signed root | ✅ **YES, across every source serving that author's subject** — it is a fact about the **writer**, not the source |
| **source-minted** | an entity-tag · a `Last-Modified` · any token the source invented about its own history | ⛔ **NO** |

⭐ **The content-derived row is measured, and the result was not predicted.** In the closure run, **two
peers published the IDENTICAL trie root under different peer-ids** — trie keys are prefix-relative and
the canonicalization is permutation-invariant, so the same binding set yields the same root whoever
holds it. *That is the landed reconstruction property (§9.7) showing up as a cross-source witness.*

⭐ **The author-anchored row is what makes closure usable, and without it §15.1's cycle cannot work.**
Alice's feed is available from Alice's live peer, Alice's origin, and two peers' republished walks.
**Which of the four is freshest?** Only a witness anchored to Alice's own succession can say — so
without this row a reader has four sources and **no ordering over their answers**, and a walk
republished a month ago is indistinguishable from one republished a minute ago.

⚠ **And the implementation trap, which is one sentence and will otherwise be found the hard way:**
a published root has **both halves**. The *manifest* — peer-id, sequence, signature — is
**source-minted and differs per peer.** The `root_hash` it commits to is **content-derived and
identical.** ***A reader comparing manifests sees disagreement where there is none; a reader comparing
`root_hash` sees the truth.***

**[MUST]** An implementation **MUST NOT** treat a **source-minted** witness as comparable across
sources.

#### 6.3.2 The three currency classes, because the third is routinely misread

| class | example | currency owed? |
|---|---|---|
| **fixed-key mutable** | an index head, a manifest | **yes** |
| **content-addressed immutable** | an entry keyed by its own hash | **no** — a different entry is a different key |
| ⚠ **keyed by another entity's hash, mutable** | a detached signature, keyed by its **target's** hash | **yes** — *it looks content-addressed and its key is not its own* |

**[MUST]** An implementation **MUST NOT** treat class 3 as class 2.

#### 6.3.3 ⭐ The substrate properties comparison depends on — stated, not assumed

**Measured during review, at three scales, with a control arm** (§9.2): a returning reader holding the
previous pass's nodes pays **5 · 6 · 7 requests** for a one-entry delta over **1,000 · 10,000 ·
50,000** entries, against **3,346** for a reader holding nothing. **The returning reader pays the
root-to-leaf path, not the tree.** The affordability of this whole mechanism rests on that, and on two
properties that are **not** implied by content-addressing alone:

⚠ **Scoped, and the scope was missing.** These clauses arrived verbatim from an implementation whose
source is a content-addressed tree, **into a section that became more general around them** — so as
written they `MUST`ed a structure that a conformant source need not have. *An implementation whose
source is a web origin with entity-tags has no trie nodes, no root and no walk, and §6.3.1's whole
point is that such a source is legal.* **The conformance statement and the opacity rule contradicted
each other; the fix keeps every word and adds a condition.**

> **Where a subject is reached as a content-addressed tree**, the following three clauses bind. **Where
> it is not, they do not apply and their absence is not a defect.**

**[MUST]** An implementation **MUST** retain trie nodes keyed by their own hash **across a root move**,
and **MUST** descend a moved root by **hash equality**, so that an unchanged subtree is neither fetched
nor re-verified.

⛔ **[MUST]** **Only a COMPLETED walk may be memoized.** A partial one manufactures a withholding origin
locally and permanently — §1.2's twin — and every read afterwards is confidently wrong with nothing
upstream broken. *Invisible to every test that does not kill the process between passes.*

#### 6.3.4 The enumerating fallback is bounded, and a bound has three obligations

**[MUST NOT]** A walk over a **remote-supplied** tree **MUST NOT** recurse. *The depth is chosen by the
peer being read; a recursive walk over a remote-supplied structure is a stack overflow with someone
else's finger on the trigger.* This is a security property, and it costs one explicit queue.

**[MAY]** An implementation **MAY** bound the walk. The number is **implementation freedom** — the right
value differs by three orders of magnitude between a photo folder and an index — but a bound carries two
obligations:

**[MUST]** A truncated walk **MUST** be reported as truncated. *A truncated walk reported as complete is
a withheld node wearing an absent node's clothes, generated locally — A1 at the smallest available
scale.*

⭐ **[MUST] Repeated passes MUST make progress.** ✅ **MEASURED since revision 3** — 25,000 entities
under one prefix, a 20,000 cap, two passes: **identical sets, ZERO paths reached by the second that the
first did not, 5,000 permanently unreachable**, truncation honestly reported and every gate green.
*The prediction this requirement was written from is now a run.* A bound that resumes where it stopped, or a partition
that rotates — **a bound that re-walks the same prefix forever is a silent ceiling on the subject size
the implementation supports.** *Reported against itself by the only party who has built one: a
deterministic breadth-first walk restarting from the base prefix each pass converges to a fixed,
permanently incomplete prefix, with the truncation honestly reported and the loop making no progress.*
**Invisible to every fixture smaller than the cap**, which is every fixture anyone has.

##### 6.3.4a ⭐ The requirement binds the SCHEDULER too, and the seat that fixed the walk nearly shipped the starved version

**`C15` is fixed and the fix was nearly defeated one layer up, by a back-off that is correct
everywhere else.** A catch-up loop that slows down when a pass *recovered nothing* meets an oversized
folder whose **first segment is entirely already-current** — a legitimate and common state — reads it
as *nothing to do*, and backs off. ⇒ **§6.3.4 satisfied to the letter, progress on every pass, and an
hour between passes: the subject fills one segment per hour and every gate is green.**

> **[MUST]** *Repeated passes make progress* binds the **loop that schedules the passes**, not only
> the walk. **An implementation MUST NOT derive its back-off from whether a pass transferred
> anything.** A pass that **advanced the resumption point** has made progress whether or not it moved
> a byte.

*This is the cheapest possible fix — the back-off signal changes from "recovered nothing" to "the
cursor did not advance" — and it is invisible from inside the walk, which is where every other clause
in this section lives.* **We are not naming a rate.** A rate would be a cost figure in a normative
document and there is no measurement behind one (§16.1); the defect is not *too slow*, it is
**deriving liveness from the wrong signal**, and that is checkable without a number.

⭐ **And the truncation disclosure needs a TENSE, which is the transferable half:**

> **[MUST]** A truncated result **MUST** carry its **resumption point**, not only the fact of
> truncation.

**The tell, and it generalizes past this mechanism: a disclosure written in the PRESENT tense.**
*"This is a prefix of the folder"* is **true, complete about the pass that just ran, and silent about
the only thing the reader needs** — whether the rest is coming. The honest sentence is future: *"the
next pass resumes after X."* ⇒ *a present-tense disclosure of an incomplete result is how a permanent
ceiling passes review*, and it is why §6.0's `truncated` qualifier carries a value rather than a flag.

### 6.4 POSITION — it belongs to the (reader, subject) pair

> **[MUST]** Position is held on the **(reader, subject) pair.** It **MAY** live inside the subject
> **only** when the subject is `owned` and the position is that single writer's own. It **MUST NOT** be
> stored inside a multiply-observed subject, and it **MUST NOT** be stored in the reader's declaration
> of intent.

⭐ **[MUST] Advancing a reader's position MUST NOT change the bytes of that reader's declaration of
intent.**

**The second form is the one to conform against.** It states the property rather than the location: a
sibling entity satisfies it, a field does not, **it holds for storage shapes nobody has proposed**, and
it is checkable by a fixture — snapshot the canonical serialisation of the persisted declaration across
a position advance and compare — rather than by reading a layout.

**Earned on three instances in three unrelated subsystems:**

| where position was put | outcome |
|---|---|
| a publisher's sequence number, inside its own published root | ✅ correct — one writer, and the position is that writer's own |
| an observation timestamp, inside an observed candidate entity | ❌ N observers mint N identities; *nothing changed* and *everything changed* become byte-identical and no downstream deduplication can recover |
| a reader's cursor, inside the reader's follow declaration | ❌ the declaration is rewritten every pass, so **stopping following becomes indistinguishable from not having read lately** |

**[MUST]** Position is **droppable**: an implementation that loses every position record **MUST** still
converge. *A specification that cannot state that sentence about its own position record has placed the
position at the wrong layer.*

⚠ **The mechanism does not own the position's path.** Where a convention requires that two applications
under one profile share a reader's position, **that convention names the path**; absent such a rule, two
applications holding two positions for one subject is a presentation matter and not a correctness one,
because position is droppable and re-derivable.

### 6.5 ⭐ The authority axis — three values, declared, never inferred

**[MUST]** A subject declares its authority class.

| | **owned** | **shared** | **ownerless** |
|---|---|---|---|
| writers | exactly one, authoritative | **many, each authoritative over their own edits** | many, no authority |
| convergence between two readers | **REQUIRED** | **REQUIRED** | **FORBIDDEN** |
| a differing second reader is | **a defect** | **a defect** | **the design** |
| reconciliation rule | not needed | ⭐ **MUST be declared on the subject** | n/a |
| position may live inside the subject | yes (§6.4) | **no** | **no** |
| a **monotonic floor** is a valid witness | **yes** | ⚠ **over a single writer's LEG, yes — over the aggregate, no** (§6.5.2a) | **no, there is no succession to anchor to** |

**[MUST NOT]** An implementation **MUST NOT** infer the axis from a subject's shape. *Two subjects with
identical substrate shape can sit on opposite sides of it, and the consequence is a `MUST` inverting.*

**[MUST]** A `shared` subject **MUST** name its reconciliation rule, and an implementation **MUST NOT**
infer one.

⭐ **[MUST] How a reader OBTAINS the rule — ruled, because the `MUST` above said the rule is the
subject's and said nothing about where a reader gets it, and for a subject with no single published
copy that is the whole difficulty.**

> **The authority-class declaration names a party, and THAT party's copy of the rule is the subject's.**
> A reader **MUST** obtain the rule from the named party over a channel it already has. **A reader that
> cannot read the rule MUST refuse to reconcile rather than guess** — a guess creates durable
> divergence and a refusal is recoverable on the next pass.
>
> ⭐ **[MUST] The refusal is scoped to the RECONCILING ACT, not to acquisition.** A reader that cannot
> obtain the rule **MUST** continue to acquire and apply everything that does not collide, and
> **MUST** hold only the change that would resolve a collision. **An implementation MUST NOT stop the
> subject.**
>
> **[MUST]** A held change is **reported**, as a `declined` outcome (§6.2.1) naming the reader policy
> that held it. *A hold nobody can see is indistinguishable from a delivery that never arrived.*

⚠ **Scoped because the implementing seat asked rather than assuming, and the strong reading is
available in the words.** *Refuse to reconcile the **subject*** — stop the folder — is grammatical,
and it is wrong here, for a reason worth stating rather than leaving to each implementer:

- **The rationale only reaches the act.** Durable divergence is created by *applying a guessed
  resolution policy.* A non-colliding change creates none; holding it buys nothing and costs
  everything.
- ⛔ **The strong reading fires on the ordinary case.** *The other device is asleep* is most of the
  time in the topology these products ship in, and the strong reading converts it into a stopped
  subject — **which from the user's side is indistinguishable from the divergence the rule exists to
  prevent, and is less recoverable.** *A safety rule whose failure mode is the same shape as the
  hazard, arriving more often, is a net loss.*
- ⭐ **And it is `declined` doing exactly what the taxonomy predicts** — served, usable, refused under
  a named reader policy, non-terminal, released on the next pass. *The row and the ruling were derived
  independently and they land on the same behaviour, which is the cheapest kind of corroboration
  there is.*

⚠ **This was asked for rather than invented, and the asking seat is non-conformant against it today** —
their per-folder rule is stored **per peer**, the view that carries the other side's declaration does
not carry the field, and **two peers can declare different rules for one folder with nothing anywhere
noticing**: one converges to a single file, the other keeps both, and they never converge. *The `MUST`
is what exposed it.* **The fix is available because their folder identifier already designates an
owner even in the two-way mode** — which is the general shape above. **The rule is part of the SUBJECT, not the reader** — *two readers that reconcile differently
do not converge, which is the whole point of the column.* **What the rule is, and how a conflict is
resolved, is `EXTENSION-REVISION`'s and the application's** — this mechanism specifies only that the
subject names one.

#### 6.5.1 ⚠ Why `shared` exists: it was missing, and four unrelated cases fell into the hole

**Revision 1 had two values.** Four cases, derived independently by two parties, land in neither:

| the case | writers | convergence | revision 1 put it |
|---|---|---|---|
| **a folder shared in both directions** — a shipped product today | two, each over their own edits | **REQUIRED** — it is the entire feature | ownerless ⇒ **FORBIDDEN** |
| **one person's notes across their own laptop and phone** | two devices, one human | **REQUIRED** | ownerless ⇒ **FORBIDDEN** |
| **a turn-based game with an authoritative host** | one authority, N submitters | **REQUIRED** | owned requires *exactly one* ⇒ neither |
| **the multi-party record** — a co-authored post, a mutual follow, a confirmed invitation, a receipt | two signing parties | **REQUIRED** | recorded independently at the social tier as *"the substrate is complete and no shape exists"* |

⇒ **Revision 1 made *"keep my own notes on my own two machines"* a configuration the specification
forbids from converging.** The declaration already exists in shipped code — a per-subject conflict
policy with two values, **absent meaning the quiet one**, because defaulting to the loud option starts
writing files a person never asked for.

#### 6.5.2 ⭐⭐ What a *writer* is — and this is why one owner with two devices is `shared`, not broken

**Asked precisely during review: does `owned` mean one writer, one identity, or one process?**

> **[MUST]** A **writer** is a **single serialized authority over the subject's succession** — one
> process's durable state. **Not one keypair, and not one identity.**

**Measured:** two publication trees under **one keypair** are two independent sequences, and a consumer
that has read one refuses the other as a **sequence rollback** — *the publisher's own second machine is
indistinguishable from an attack.* And the cell with no defence is **equal** sequence, not lower: two
devices at one number with different content are both accepted, and nothing in the chain has anything to
say.

⭐ **The corpus already holds the model that resolves this, and it was not connected to convergence.**
`EXTENSION-IDENTITY` makes an identity a peer graph in which **agents are per-device peers**, each with
its own key; `EXTENSION-ENCRYPTION` §4.4 states it outright — ***a device is a private-key holder***, and
deduplication is by public key because *"two live certs over one key are one device."* Without the
identity extension installed, *"one peer = one keypair = one identity"*, and there is no multi-device
case to have.

⇒ **Two devices under one identity are TWO WRITERS. The subject is `shared`**, its reconciliation rule is
declared, and **the monotonic floor is not its witness** — which is exactly why the table above makes the
floor `owned`-only. *The finding is not repaired by tightening the floor; it is classified, and the
classification removes the instrument that was breaking.* **The general form is §6.3.1's last clause: a
floor compares witnesses across writers, and witnesses from two sources are not comparable.**

#### 6.5.2a ⛔ The floor is a property of a LEG, not of an authority class — and the blanket row was a security regression

**Revision 3's row said a monotonic floor is not a valid witness for `shared`. That generalized one
position too far, and the seat that would have benefited from the loophole is the one that closed it.**

**Follow §6.2.4:** a `shared` subject's contributions are *signed entries in each contributor's own
namespace.* So a reader of a `shared` subject reads **N legs, each `owned` by exactly one writer** —
and §6.5.2 defines a writer as a single serialized authority over succession, **which is precisely the
granularity at which a floor is meaningful.**

> **[MUST]** A monotonic floor is a valid witness **over a single writer's leg, whatever the subject's
> authority class.** It is **not** a valid witness over an aggregate with more than one writer.

⛔ **Why this is not pedantry: as written, the blanket row made a live gap conformant.** One
implementation ships rollback refusal on its static leg and **nothing on its sync leg — same repo, same
threat.** Under the blanket row the sync leg is a `shared` subject, so *having no floor was correct*
and the gap stopped being a defect. **One clarification restores rollback protection to the exact
configuration that has none.**

#### 6.5.2b ⛔⛔ *What is the monotone quantity on a delivered leg?* — the leg carries no witness, and that is the DEFECT, not the answer

**The narrowing above was adopted partly on a seat's source reading that they had explicitly marked
unperformed. They then performed it, and it holds:**

> `REPLAY: dispatch status=200, err=nil` — and the file on disk after the replay is **the older
> version.** Two peers, real bootstrap, real connection. The newer file is gone, **silently, with a
> success code.**

**Asked back, precisely, and it is the right question: their delivery leg has no published root, no
sequence, and nothing monotone in the protocol.** The only local candidate is the file's `modified_at`
— *a number a filesystem chose* — and refusing on it turns a restored backup into a file that silently
stops syncing. **And their control arm forecloses the cheap answer: a sender that GENUINELY reverts
its own file must still be followed** — that is convergence, not an attack — **and in their probe the
revert is byte-identical to the replay.** ⇒ *content cannot be the discriminator.*

⭐ **The answer is already in §6.3.1a and needed one word moved: a witness minted by the SENDER of a
delivered leg is AUTHOR-ANCHORED, not source-minted — because on that leg the sender IS the writer.**

| candidate | class | valid floor? |
|---|---|---|
| the file's `modified_at` | ⛔ **neither** — minted by a **third party**, the filesystem | **no** |
| the bytes | content-derived | **no** — *it cannot order two states, only distinguish them, and the control arm makes them identical* |
| ⭐ **a per-`(sender, subject)` counter from the sender's own durable state, carried on the delivery** | ⭐ **author-anchored** | ⭐ **YES** |

⭐⭐ **And it settles the control arm exactly, which is how you know it is the right quantity.** A
genuine revert is **a new write by the writer**, so the counter *advances* while the bytes go
backwards ⇒ **accepted, correctly.** A replay is *the same write arriving twice*, so the counter is
**stale** ⇒ **refused.** *The two are indistinguishable by content and trivially distinguishable by the
writer's own succession — which is what §6.5.2 said a writer IS.* `modified_at` fails not for being a
number but for being minted by neither the content nor the writer.

**So the ruling, and it is deliberately not a floor built out of what is lying around:**

> **[MUST]** A delivered leg **SHOULD** carry a per-`(sender, subject)` monotone witness minted from
> the sender's durable state. **Where it does not, the receiving implementation MUST report the leg's
> witness as `not_supported` (§6.3) and MUST NOT present the leg as rollback-protected.**
>
> **[MUST NOT]** An implementation **MUST NOT** synthesize a floor from a quantity minted by neither
> the writer nor the content. *A floor over the wrong authority refuses correct data and admits the
> attack it was built for.*

⇒ **The honest state: this leg has no witness, so today it has no floor, and the specification says so
out loud rather than letting an undefended leg read as a defended one.** **Carrying the witness is a
subscription-tier change and needs its own proposal** — it lands beside `Q4`, and it is now the second
independent reason to open that one. ⚠ **Marked, because this is the third round in which a leg's
protection was inferred rather than read: the seat measured that the *handler* has no ordering check,
and did NOT measure who may dispatch to it.** *Until that is measured, the interim defence of this leg
is the authorization on its entry point, which is an assumption and is written here as one.*

#### 6.5.3 May an `owned` subject's writer change?

**Yes, and this mechanism does not specify the handover.**

**[SHOULD]** An `owned` subject's writer **SHOULD** be named by an **identity or a role** rather than by
a peer key. Where it is, *which peer writes* is a cert-level fact, **handover is the identity
extension's**, and the subject's name never moves. Where a deployment runs without the identity
extension, one peer is one identity and **the writer cannot change** — which is a bounded, honest
answer rather than an unspecified transition.

⚠ *Both readings available without this clause are bad: a new subject breaks every reader's identity and
position, and an unspecified transition is decided differently by every implementation. The host of a
turn-based game closing their laptop is the ordinary case, not an exotic one.*

### 6.6 INTENT — a durable declaration plus a reconciling loop

**[MUST]** A reader's intent to keep a subject current is a **durable record**, not process state.

⭐ **[MUST]** ***A DECLARED intent is durable. A DERIVED intent MUST NOT be made durable unless its
inputs are.***

**The discriminator is a withdrawal act, and it is testable:** a declared intent has one — the reader
who started it can stop it, and the removal is the same kind of act as the addition. **A derived intent
has none.** *"I have met this peer, so I should be reachable to them"* is a pure function of session
events that nobody ever retracts, so a durable set of them only ever grows — persisting it is caching a
derived value with no invalidation, and it re-grows a stale set nobody clears. **This qualifier was
returned by an implementation against its own two intent surfaces: the `MUST` fits one exactly and is
actively wrong for the other.**

**[MUST]** An implementation **MUST** provide a reconciliation pass that makes substrate match the
declaration and that is **idempotent** and **safe to run at start-up, after any change, and on a
connection event**. *On a timer is §6.0's qualifier, and it is not unconditional.*

**[MUST]** Reconciliation **MUST** report a witness distinguishing *a pass that did nothing because
everything was current* from *a pass that did nothing because it could not look.* **Nothing in the
substrate reports "I would have done nothing";** without it, a second pass cannot be shown to be a no-op.

### 6.7 ⭐ The guarantee is EVENTUAL, not BOUNDED — and the specification must say so

**[MUST]** This mechanism guarantees **eventual** convergence. It does **not** guarantee convergence
within any bound.

**[MUST]** A convention requiring a latency bound **MUST** state that requirement itself, and **MUST
NOT** obtain it from this mechanism.

**Why this is normative rather than a caveat.** A reconciliation interval that backs off with idle time
means *"delivery is expected and its absence is a failure"* is a guarantee an implementation **can**
meet, while *"delivered within N"* is one it **cannot**. **Without this clause a future conversation or
messaging convention will harden a bound no implementation can honour** — and the implementation that
measured the ceiling filed the warning specifically to prevent it.

⚠ **This section is placed before the exclusions on purpose.** The extension's name (§11 Q7) and the
family of systems a reader will associate it with both carry an expectation of a bounded staleness
guarantee. **This mechanism refuses it, and the refusal should be read before the word is.**

---

## §7 What is deliberately NOT in this mechanism

| Excluded | Why, and why the exclusion is not obvious |
|---|---|
| ⭐ **The schedule / cadence** | Surveyed implementations use intervals four orders of magnitude apart **over four genuinely different cost structures**, and they may be four **correct local answers**. A mechanism imposing one number would be worse than what exists. **What is extracted is the TRIGGER (§6.0) and the OUTCOME TAXONOMY (§6.2)** |
| **…but the drop signal is INCLUDED** (§11 Q4) | One adaptive back-off exists solely because *"the receiving peer cannot see the sender's drop counter."* ⇒ **standardize the signal, not the schedule**, and keep a local-signal fallback for peers that do not carry it. ***A blind heuristic promoted into a specification is a workaround with a MUST on it*** |
| **The SUBMIT path's *transaction* semantics — multi-peer atomicity** | **Not the same thing as writing**, and revision 2 conflated them (§7.1). **Publishing is in; cross-peer atomic commit is the cluster-transaction extension's** |
| **The publisher-side absence class** (§1.3) | ⚠ **Named, not excluded — and not solved either.** *A publisher cannot tell "nobody has read this" from "nobody can read this."* The mechanism does not answer it; §11 Q2 carries the one part it could host. **Named as a distinct class so it is not re-derived from whichever symptom surfaces next** |
| **Lowering and rendering** | §9.4 |
| **Meaning** — republication rights, audience, policy, reply, context, collection | `SPECIFICATION-FORMAT` §8.8 + §8.9. **A mechanism that grew a `reply` field would be this proposal's own defect with the arrow reversed** |
| **Reader-side policy** — selection, ranking, cross-publisher merged order | per-reader by design and **MUST NOT** converge (§6.5). A mechanism that converged them would be one with a central moderator |
| **Conflict RESOLUTION** | A `shared` subject names its reconciliation rule (§6.5); what the rule does is `EXTENSION-REVISION`'s. **Not a merge engine and not a convergent-replicated-datatype library** |
| **Progress plumbing** — synchronous surfaces over asynchronous reads | Three expressions exist and the duplication is real, but what would be lost is each surface tuning retry against its own user-visible cost. **Lowest value; excluded on the implementers' own recommendation** |
| **Retention** | A mechanism keeping durable copies current raises a retention question that touches operational management concerns not yet specified. **Named so nobody assumes it was considered** |

### 7.1 ⚠ The submit path — revision 2 excluded it, and that was wrong in the way that matters

**Revision 2 said *"every layer is reader-side, therefore submitting is out of scope."* That inverted
cause and effect.** §5.0's closure says a peer publishes what it obtained, in its own namespace, as the
same kind of object — **so writing is not a separate mechanism; it is the other half of this one.** A
party that "submits" a game move, a reply, an acceptance or an edit is **publishing an entry in their
own namespace that names a coordinate** (§6.2.4), and the holder of the subject acquires it like any
other source. *There is no submit primitive because there does not need to be one.*

**What genuinely is not here**, and the distinction is the whole correction:

| | in scope | where |
|---|---|---|
| **contributing to a subject you do not own** | ✅ **yes** — publish into your own namespace, naming the coordinate | §5.0, §6.2.4 |
| **the owner incorporating your contribution** | ✅ **yes** — they acquire you as a source | §6.2 |
| **a direct write into someone else's namespace** | **not needed, and not offered** — it would be the one operation that breaks single-writer authority | — |
| ⛔ **multi-peer ATOMIC commit** — *these three writes land together or not at all* | **no** | `EXTENSION-TRANSACTION` §1.2 defers this to a future extension, **zero consumers today** |

⭐ **And *"did my move land?"* is answered.** A contributor is a **reader of the result subject**: they
publish, then acquire. **§6.0's result tells them *whether* — `updated` carrying the incorporated move
is the confirmation — and §6.7 refuses to tell them *when*.** *A real answer with a real limit: a
contributor who needs a bound needs it from somewhere else.*

---

## §8 The `SYSTEM-COMPOSITION` chapter — the emergent half

### 8.1 ⛔ `EXTENSION-SUBSCRIPTION` §6.3 (ii) is currently unsatisfiable for a common workload

**The requirement** names §5.5 gap detection as its mechanism. **§5.5's primary detector** is a
predecessor-hash chain: a gap is detected when a **later** notification on the same path arrives
carrying a mismatched predecessor.

**Measured:** 2,000 records written to a replicated folder at once; the sender bound all 2,000; **676
reached the receiver**; transfer then stopped permanently, **with no error on either side and status
reporting healthy throughout.** The sender had discarded **2,327** notifications — a deliberate,
counted, deadlock-avoiding drop under delivery-queue saturation.

⭐ **Of those 2,327, the number the predecessor chain could have detected is ZERO.** A folder
replication writes most paths **exactly once**, so a dropped notification is the only one that chain
will ever carry and **no successor arrives to mismatch.** *The detector is structurally blind to the
write-once workload.*

**Verified independently in a reference implementation:** the subscription engine drops on a full
delivery shard and counts it, deliberately; and **the same implementation ships no repair** — its
cross-peer synchronization tool wires notification chains with no reconciliation pass of any kind.

⚠ **And §6.3's own text explains why this survived to be found by a product failure rather than by the
suite.** Its conformance paragraph states that **drop injection is implementation-internal and
unspecified**, and that *"a conformance run that cannot inject drops still MUST assert (ii) under normal
delivery."* ⇒ **Property (ii) has only ever been exercised in the mode where it cannot fail.** That is a
normative `MUST` that no check drives, and it is the class this corpus's coverage instrument reports.

⚠ **Scoped precisely:** §6.3's own scope paragraph binds a peer that **offers the mirror recipe**, and
this proposal does not establish that the tool in question claims it. **What is not in question:** the
drop is deliberate and by design, the predecessor chain cannot see it, and §6.3 (ii) names it as the
mechanism anyway.

### 8.2 What the chapter states

1. **Currency across peers is a composition property, not subscription's.** The comparison requirement
   (§6.3), the droppable-position test, and the outcome taxonomy (§6.2) are stated once, where extensions
   meet.
2. **The predecessor chain is demoted to what it is good at:** **write-many paths**, where a successor
   arrives to mismatch.
3. **The authority axis (§6.5) is stated here**, because it governs whether *any* composed convergence
   requirement applies at all.
4. **§6.3 (ii) is rewritten** to require convergence and to cite the composition chapter for the
   mechanism, rather than citing a detector that cannot deliver it.
5. **The chapter is titled for its question, not for the word *convergence*** (§4.3), and
   `SYSTEM-COMPOSITION` §5 gains one line distinguishing cascade termination from cross-peer currency.
6. **A record-design rule, as a `SHOULD`:** *every field of a shared record is MINE, THEIRS, or OURS —
   and a party MUST NOT gate its own behaviour on a THEIRS field that no channel carries back.*
   **Measured against its own defect:** a reverse leg gated on a peer's acceptance state, stored in the
   gating party's own record, written into the *receiver's* tree with nothing carrying it back — a share
   that established cleanly, reported healthy, and delivered nothing, for weeks. **It is orthogonal to
   §6.5, not a generalization of it**: §6.5 asks *who may write this subject*, the field rule asks *who
   can know this field*, and the defect above sits in a subject §6.5 classifies correctly as `owned`.
   ⭐ **And the field rule predicts §6.5's third value** — a subject carrying OURS fields is exactly a
   `shared` one.

---

## §9 Corrections to landed text

### 9.1 `EXTENSION-SUBSCRIPTION` §5.5 and §6.3 — see §8.

### 9.2 ⭐ `APP-CONVENTION-FEED` — the index narrows, and its stated rationale is false for the case it was written about

**§11 `F-1` already anticipates the first half**, stating that a general reader-loop mechanism *"is the
better home, and if one lands, this field is removed in favour of it."* **That deferral's trigger
condition is this proposal.**

⇒ **The follow record stays a declaration** — subject, label, provenance of how it was learned, when it
began. **The position becomes output and belongs to the mechanism** (§6.4).

⚠ **Sequencing note for implementers.** A separate accepted correction reshapes that field's type. **It
is superseded by this proposal and should not be built.** The cost of holding it is zero: the field is
currently constructed empty everywhere and never written.

**And a second consequence that changes what gets built rather than what it is called.** Under §6.3 a
walk obtains its delta from content addressing, so **a publication index stops being the
change-detection mechanism and carries only authored order.**

⭐⭐ **This was the one delta nobody had measured, and it has now been measured — by the party who
expected to argue against it.** One publisher, a flat write-once prefix, a reader holding the previous
walk's nodes:

| entries | trie nodes | cold walk | no-op check | **1-entry delta** |
|---|---|---|---|---|
| 1,000 | 53 | 56 requests | **2** | **5** |
| 10,000 | 1,054 | 1,057 requests | **2** | **6** |
| 50,000 | 3,342 | 3,345 requests | **2** | **7** |

**A 50× larger prefix costs two more requests.** The no-op check is **2 requests at every scale** — a
head and its signature, which is §6.3's fixed-key witness doing exactly what it says. **Control arm:** a
reader holding nothing pays **3,346** for the same one-entry delta, *which is what makes this a
measurement rather than a coincidence.*

⛔ **Therefore §4.1's rationale must be corrected in the same change**, because it is the sentence a
reader will cite when they re-add the index as a change detector in two years. It currently reads:

> *"discovering what is new under a prefix costs the whole tree… **So the index is not a convenience
> added after the fact**… a prerequisite of the reader loop rather than a later optimization."*

⇒ **The true statement is narrower and still justifies the index:** *enumerating a prefix from nothing
costs the tree; **discovering what is new, holding the previous root's nodes, costs the path.*** The
index earns its place on **authored order** and on **cold-start cost** — which is exactly what the
narrowing leaves it.

⚠ **Two honest limits carried from the measurement:** local traversal is still linear in the subject
(a moved root re-walks every node from cache — 63 ms at 50,000), and **cold start is unavoidably linear
and the index does not help there either**, because a first-time reader wants everything, which is the
one case where reading everything is the right amount of work. **Request counts transfer; wall-clock
from a local server does not.**

### 9.3 ⭐ Two landed convergence requirements have opposite polarity

- `APP-CONVENTION-FEED` §9.1: reader selection is *"per-reader by design and **MUST NOT converge**"*.
- `EXTENSION-REVISION` §7.2 treats reader divergence as a defect to design away; `EXTENSION-SUBSCRIPTION`
  §6.3 (ii) makes convergence a **`MUST`**.

**Both are correct for their subject, and nothing names the axis that separates them.** §6.5 does.

⭐ **And one rule is specified in the wrong place, discovered independently at both tiers.**
`APP-CONVENTION-FEED` §9.3 `[MUST]`s that *a view declare what produced it*. `EXTENSION-REVISION` §5.1
records the same gap at the substrate, unaware of it: *"there is **no audit signal today** — a
configuration that resolves a conflict via a non-default strategy produces a merge result
**byte-identical to a genuinely conflict-free merge**… not yet specified."* **Same rule, two tiers, two
parties who never spoke.** ⇒ **It is tier-general and belongs at the composition layer** (§12 D7).

### 9.4 `APP-CONVENTION-EMBED` — the container is load-bearing and the renderer contract is inert

**Measured across both implementations:**

- **Closed at the front door.** The mandatory display-fallback field has exactly two production
  sources and **both are display text**, one of them a first-line extraction that **structurally
  assumes line-oriented prose.** ⇒ a payload with no authored display string — a sensor reading, a
  game-state update — is **refused at ingest** and never reaches a renderer at all.
- **Bypassed at the back.** Both implementations lower the inline directive into the **base document
  format's image grammar** before any renderer sees it, after which there is no typed node, no media
  type and no dispatch. ⇒ **§6's degradation ladder is unreachable by construction in both.**
- **Zero implementations, anywhere, of the output basis, renderer capability declaration, capability
  tags, or rendition selection.**

⇒ ***The specification's degradation contract and both implementations' pipelines describe different
systems, and every passing test in both is consistent with that.***

**The principle, and it is the implementers':** **declining to render is a correct outcome, not a
degradation to be minimized.** A native front end could ship a payload type nothing else renders and a
browser would decline it — *the same event as a terminal declining a document format, and the format is
healthy precisely because both are expressible.* The alternative obliges every substrate to become a
browser engine.

⭐⭐ **The requirement that falls out, and it is §1's A4:**

> **[MUST] A renderer that will not present a payload MUST be able to state WHICH payload it declined
> and WHY.**

**Unsatisfiable today at any point in either pipeline — and the cause is §5.3.1's separation.** The lowered
form carries a **reference and no type**: an address with the identity stripped off. Everything
downstream guesses from a filename extension, and one shipping implementation does a substring match.

⇒ **[MUST] A typed payload remains typed at every point where a decision is made about it.**

⇒ **The split is therefore recognition, not invention:** the payload container is generic, already
reused without a renderer by a second convention, and belongs with the mechanism's lowering seam. The
display fallback, renditions and capability rules are a **presentation envelope** and belong to a
rendering convention.

### 9.5 The two-axis routing correction

A landed draft describes source selection on one axis — *how the delta is computed* (the publisher
computes it versus the reader walks). **An implementation ships a second axis — *where the data lives*
(a live peer versus a published origin) — and these are independent.** A live read that is a plain tree
read over a connection is *reader-walks delta* against a *live source*, **a cell the existing table has
no entry for.**

⇒ **Two axes, four cells, two named.** Any source-preference guidance must be re-derived on the
corrected axes; the current `SHOULD` and at least one implementation's default disagree, and the
disagreement cannot be adjudicated on a table that cannot express one of the positions.

### 9.6 A one-line divergence that needs no new rule

`APP-CONVENTION-EMBED` §3 already states the display fallback is **mandatory and non-empty**. **One
implementation now enforces it — and distinguishes a missing field from a present-but-empty one as two
different refusals. The other still accepts empty.** ⇒ **No ruling is required; the rule exists.** The
non-enforcing implementation should move, and **the two-refusal distinction is worth pinning** — it is
§1's principle at the smallest available scale.

### 9.7 ⭐ A live source can offer a prefix witness today, and the substrate already specifies how

**The gap, reported from the topology a product actually ships in:** §6.3's cheap witness for a prefix
assumes a **published root**, and two machines on a local network are live peers that have performed no
publication act. Enumeration is the only arm left, and the far side pays linear cost in the prefix every
pass.

> ## ⛔ 9.7.0 THIS SECTION'S RECOMMENDATION WAS WRONG, AND THE LANDED SPEC CLAIM BEHIND IT IS TOO
>
> **Revision 3 recommended reconstruction on the strength of a cost figure, and said the sidecar was
> *"a proof this consumer does not need, at a storage and per-write cost it should not pay."* The seat
> that was told this measured it:**
>
> | bindings | reconstruction | per binding |
> |---|---|---|
> | 1,000 | **34 ms** | 34 µs |
> | 10,000 | **455 ms** | 46 µs |
> | 50,000 | **2.62 s** | 52 µs |
>
> ⛔ **`EXTENSION-TREE` §3.7.1's own words are *"microseconds to low-ms. Cheap"* — for a stated typical
> range of 100–1,000 entries and "thousands". It is wrong INSIDE its own stated range**, by ~30× at
> 1,000 and by more at 10,000. **The asymptotic is right and the constant is not: each of those
> operations is a CBOR encode plus a SHA-256 plus a content-store put, not an arithmetic step** — and
> per-binding cost *grows* 34 → 46 → 52 µs, exactly as `n log n` predicts.
>
> ⇒ **The trade inverts for a witness in a LOOP.** Reconstruction makes the **source pay seconds of CPU
> per requesting reader per pass**; the sidecar pays `O(log n)` per **write** and `O(1)` per read.
> **Reconstruction is right for a one-off comparison — a conformance check, or a deployment asking
> *"do we hold the same set"* — and wrong for the thing it was recommended as.**
>
> ⚠ **Bounded honestly, by the seat that took the measurement:** that is one implementation's
> incremental builder, **not a lower bound on the algorithm**. A bulk bottom-up builder could plausibly
> close much of the gap — **and nobody has written one in any implementation**, so the number above is
> what the ecosystem has today and **the spec's claim is met by nothing that exists.**
>
> ⭐⭐ **The discipline this earns, and it is the transferable half:** ***a cost figure in a normative
> document is a measurement or it is marked as unmeasured.*** This one was prose in a landed spec,
> repeated into a proposal, and turned into a recommendation to another seat — **three hops with no
> measurement anywhere.** `D19` corrects the spec; the rule is what stops the next one.

⭐ **`EXTENSION-TREE` §3.7.1 already specifies the instrument, and it needs no publication and no stored
index.** *Given any subset of bindings under a prefix, an implementation builds a fresh trie over that
subset and obtains a deterministic root hash; two implementations building the same subset produce
byte-identical results.* Cost is **microseconds to low milliseconds** for subject sizes in this range, the
normalization precondition is already pinned by §3.1, **and it is local CPU at the source in place of
linear network round-trips at the reader.**

⇒ **A live source MAY offer a reconstructed prefix hash as a §6.3 witness — for a one-off comparison,
and NOT as a per-pass witness in a loop** (§9.7.0). What is missing is only an **operation** to ask for
one, and the shape is now ruled rather than open:

> **[MUST]** The operation is `witness(subject) → token`, and the token is **opaque** exactly as
> §6.3.1 requires. **It MUST NOT name the mechanism** — there are already three (a published root
> pointer, a maintained sidecar, an on-demand reconstruction) and a fourth at any non-entity origin,
> and naming one in the operation forces every source to have it.
>
> ⭐ **[MUST] A source MUST be able to answer `not_supported`, distinguishably from a token**, and a
> reader receiving it falls back to the enumerating arm. ***An absent witness that answers "unchanged"
> is a reader permanently convinced it is current — A2 generated locally, and strictly worse than no
> witness at all.*** *This is the drop-counter rule arriving at the witness, and it is the same failure
> if it is missed.*

⚠ **And the cheapest shape for the topology that reported the gap is neither of these:** let a live peer
**publish a root for the prefix it shares.** The witness is then a fixed-key pointer — **2 requests,
measured, at every scale** — maintained incrementally as a side effect of publishing, **and the live
and static legs become the same leg**, which is §15.6's point. *That is that seat's build order and it
is unchanged by this ruling.* ⚠ **And this closes a loop in the substrate's own record:** §3.7.2's audit
concluded that no consumer used prefix-subtree commitments and named a maintained sidecar index as the
recovery path if one ever appeared. **The audit was correct when written; this proposal creates the
consumer — and reconstruction, not the sidecar, is what it needs.** *(The sidecar's O(N) storage and
per-write cost is the price of the property this consumer does not need.)*

---

### 9.8 ⛔⛔ The only type in the corpus that could carry the 250× cannot express it — the gathered view is a THREAD view and the trace is a TIMELINE

**Found by the seat that built the gatherer, by building it: they implemented what the landed text
says, discovered it does not reach this document's own headline trace, and routed the question rather
than stretching the type.** *Whoever publishes first becomes the baseline — the fourth time that rule
has been invoked on this board, and the first time it has stopped a divergence before it happened.*

**The defect, and it is arithmetic over two landed productions:**

| | |
|---|---|
| `APP-CONVENTION-FEED` §2.2 | `reference = pinned-ref` — **a pin, to one entity, and its answer can never change** |
| §6's gathered record | `subject: reference` ⇒ **the subject is a pin**, and §6's own comment agrees: *"the root entry this view is of"* |
| §6.2's rationale | written about exactly that: *"a conversation spans publishers, and no single publisher holds all of it"* |
| ⛔ **§15.1's trace** | *"Bob follows Alice, Carol and Dave. Bob publishes his walk."* **Three timelines.** A timeline is §1.3's growing prefix — **not an entity, so it cannot be pinned, so it cannot be a `subject`** |

⛔ **And this is the clause the 250× rests on.** §6.0.1's Profile A lists the third `SOURCE` leg as *a
peer republishing this subject's walk*, and §6.0.1's own note says a developer without that leg
**builds the 1,000-request version.** ⇒ **for an author-feed subject, the one application type that
could carry the leg cannot name it.**

#### 9.8.1 ⭐ The ruling — widen the subject, and the mechanism's OWN §6.1 is what decides it

**The two workarounds were examined and both fail, one of them against a `MUST` in this document:**

| | |
|---|---|
| pin the author's **index-head entity** | ⛔ **forbidden by §6.1.** *"A subject is identified by a value that does not change when its current bytes change"* — **a head's hash changes every time the author posts.** It is **a witness masquerading as an identity**, and it makes the address underivable besides: you need the head's hash before you can ask, which costs the hop the gathered view exists to save |
| publish one gathered record **per author, with the author as `subject`** | `subject` is a `reference`; **a peer is not an entity.** Nothing in §6 admits it |

> **RULED. `subject` widens to `any-reference`.** A **pinned** subject names an entity many writers
> contribute to — the thread case, `shared`/`ownerless`, merged by union. A **live** subject names a
> prefix **one writer owns** — the timeline case, `owned`, ordered by the author's succession.
>
> **[MUST NOT]** A subject **MUST NOT** be a pin to a value that moves when the subject changes.
> *That is a witness, not an identity, and §6.1 already forbids it — this states it where it is being
> got wrong.*

⭐ **Why widening rather than a second type, and the argument is closure's.** The record shape carries
no field that differs between the two cases, and the reader does the same thing with both: verify each
entry against its author's detached signature, attribute to `entry.author`, render. **A second type
would double the consuming code path at precisely the seam §5.0 requires to be single** — closure's
*"verified, attributed and rendered by the identical code path"* is the clause a second type would
break, in the document that introduced it. *§2.2.1 already has an `either` column; this is that column,
on a site that was never examined for it.*

⚠ **What the authority axis buys, stated so nobody re-derives it:** the two cases are not a
presentation difference, they are `owned` versus `shared`/`ownerless` — **so the floor applies to a
timeline view and not to a thread view** (§6.5.2a), and *short* means a **prefix gap** in one and
**an unreached contributor** in the other. **The axis was already there; the type just could not
reach it.**

#### 9.8.2 ⭐ And the gathered view's own address is the OTHER half of the 250×, not a low-stakes detail

**Filed as low stakes by the seat that filed it. It is not**, and the reason is §9.8's arithmetic one
step further on. §6 pins **no path**, while §4.2 names the index head and its pages by hand and states
why: *"an index nobody can find is not an entry point."* §6 has the identical problem and no answer,
and §2's *"the cross-impl contract is the type tag, not the path"* makes finding one a type-filtered
query — **which a static origin cannot serve** (third appearance of that gap).

⇒ **Erin's 2-request no-op check requires that she can DESCEND Bob's gathered records from his signed
root without asking him what he gathers.** A conventional prefix gives her that by ordinary trie
descent; a type-filtered query does not, on the static publishing posture.

> **RULED. The prefix is normative: `app/feed/mirrors/`.** The leaf key is the subject's
> **coordinate**, derived by a stated rule per reference kind, so that a reader holding the subject
> computes the address rather than discovering it. **Mutable at a stable key** — a gatherer
> republishes as it reads more — which §1.3 makes monotone.

**The derivation, per kind, over the reference's IDENTIFYING fields only:**

| subject | key | why those fields |
|---|---|---|
| **pinned** | `hex(hash)` | **the seat's shipped key, adopted unchanged.** A pin is satisfiable by anyone holding the bytes (§2.2), so `peer` is a hint and not an identity |
| **live** | `hex(content_hash(absolute-path))` — the path **absolute**, `/{peer}/{path}`, canonical UTF-8 | `(peer, path)` is what a live reference names; **an absolute path is one identifying string at every layer**, which is the landed model and needs no separator convention invented for it |

⛔ **The trap this avoids, and it is why the rule is over identifying fields rather than over the
atom.** *Hash the whole reference* is the tidier-looking rule and it is broken: `at` and `via` are
**optional hints**, so two readers naming the same subject with different hints derive **different
keys** and neither can find the other's record. ⇒ ***a derivation that includes an optional field is
not a derivation.*** Likewise `seen` is excluded — §2.2 says in terms that it is an expectation, not
an authority.

*The derivability property is the one thing available without an index, and it is the whole reason to
pin a key rather than a query.*

---

## §10 What a conformance check set must discriminate

**This section states requirements, not fixtures.** Test vectors are byproducts of implementations
running against one another; this is the list of cases a check set must be able to tell apart.

| # | Must discriminate | Anchored on |
|---|---|---|
| **C1** | a subject that is current from one that is authentic-but-stale | §6.3 |
| **C2** | **all six outcomes of §6.2, pairwise** — and specifically *refused* from *absent*, *faulted* from *unheard*, and *unreadable* from both | §6.2.1 |
| **C3** | convergence **with the change stream entirely ignored** | §6.3 — the droppable-position test |
| **C4** | convergence after notification loss **on a write-once workload**, where the predecessor chain detects nothing | §8.1 |
| **C5** | a currency check that uses **no content hash of ours** — an opaque third-party witness — and is **conformant** | §6.3.1 |
| **C6** | a subject keyed by another entity's hash, correctly treated as mutable | §6.3.2 class 3 |
| **C7** | an empty answer from one source **not** ending the source ladder | §6.2.3 |
| **C8** | two sources both serving, **disagreeing**, reported as an outcome rather than resolved | §6.2.3 |
| **C9** | *stopped following* from *has not read lately* — by byte-comparing the persisted declaration across a position advance | §6.4 |
| **C10** | a reconciliation pass that did nothing because current, from one that did nothing because it could not look | §6.6 |
| **C11** | N observers of one unchanged subject producing **one** identity, not N | §6.1 |
| **C12** | an ownerless subject where two readers legitimately diverge, **not** reported as a defect | §6.5 |
| **C13** | a declined payload reported with **which** and **why**, distinct from *not offered* and from *broken* | §9.4 |
| **C14** | a truncated walk **reported as truncated**, not as complete | §6.3.4 |
| **C15** | ⭐ a bounded walk that **RESUMES** from one that re-walks the same prefix forever. *Both report truncation honestly; only one converges* | §6.3.4 |
| **C16** | ⭐⭐ a returning reader that pays the **PATH** from one that pays the **TREE**. *Same answer, same correctness, two orders of magnitude apart* | §6.3.3, §9.2 |
| **C17** | a memoized **complete** walk from a memoized **partial** one | §6.3.3 |
| **C18** | a drop counter that is **absent** from one that is **zero** | §11 Q4 |
| **C19** | ⭐⭐ an implementation where every consumer reaches the mechanism through **one entry point** from one where a consumer can issue a source read directly | §6.0 |
| **C20** | a `shared` subject converging between two writers under one identity, **not** refused as a rollback | §6.5.2 |

⭐ **C16 and C19 are the two that can fail while every functional assertion in an implementation
passes** — C16 because the answers are identical and only the cost differs, C19 because the pre-mechanism
state and the conformant one produce the same bytes. **C4 and C13 are the two that no current check set
anywhere can express.** Together those four are the measure of what this proposal is for.

---

## §11 Open questions

**Five of revision 1's eight are closed.** What closed them is recorded at §14.

| | |
|---|---|
| **Q2** | ⭐ **May a publisher state a preferred read path?** *(Open, and it is the one piece of §1.3's excluded class this mechanism could host.)* No mechanism exists. Three shapes: a field on the index head (cheapest; but a reader holding the head has already committed to one source); a well-known entity at a pinned path (readable before choosing — **circular unless it is the one thing every source must serve**); or rule it the reader's choice. ⚠ **The urgent form: a publisher who cannot serve live has no way to stop every reader trying, and experiences the failure as load they cannot attribute.** |
| **Q4** | **The drop signal's shape.** ⇒ **Working answer, from the party that needed it: a counter the subscriber can ASK for, scoped to the subscription.** A field on a notification is *structurally* unable to report the failure — the drop happens before the wire, so the carrying notifications are the ones that do not exist. A health record in the publisher's tree inverts the authorization and goes stale silently, *and a stale zero is the worst available value.* **Two properties: monotone, reset only with the subscription; and `not_supported` MUST be distinguishable from `0`** — otherwise every publisher without the counter reports itself perfectly healthy and disables the fallback that exists because the counter is missing. **To confirm with the subscription tier before it lands.** |
| **Q6** | **Are §6.3's witnesses one operation with a mode, or several?** Unexamined, and §6.3.1's re-keying onto the *(source, subject)* pair changes the question rather than answering it. |
| **Q7** | ⭐ **The name — proposed: `DATA-EXCHANGE`, spanning four documents.** Three candidates withdrawn; §11.1 and §15.5. |
| **Q8** | **Can the composition chapter and the extension land together, or must they be sequenced?** The extension can be authored first; the chapter **corrects a landed `MUST` in another extension** and is the more delicate half. |
| **Q9** | **NEW — does the live-source prefix witness (§9.7) need an operation in `EXTENSION-TREE`, and is that a tree-tier change or a mechanism-tier one?** The computation is specified; only the way to ask for it is not. |

### 11.1 ⭐ The name — `DATA-EXCHANGE`, after three candidates and one real lesson

**Three names were proposed and withdrawn. The lesson is not about words.**

| candidate | why it died |
|---|---|
| **`REPLICA`** | imports a primary, a consistency model and a bounded staleness guarantee — **all three explicitly refused** (§6.7). *"The replica extension keeps replicas in sync with the primary"* is a true-sounding sentence about a different system |
| **`ACQUISITION`** | ⛔ **the word is already spent in this corpus** — *the acquisition surface* / *the acquisition path* means **getting two peers into contact**, one tier over. The census that cleared it read `specs/` + `guides/`; the 33 occurrences are in `docs/proposals/` and `docs/research/` |
| **`CURRENT-COPY`** | ⛔ **it asserts what no participant in this system can know.** You never hold *the current* anything — you hold **the latest observed from your position in the network, given who you know and how you traversed.** The publisher may have been writing for a month with their lid closed and nobody asked. **A name that claims currency is the same defect as `REPLICA`: importing a guarantee the mechanism refuses** — committed one revision after writing the rule against it |

⇒ ⭐ **`DATA-EXCHANGE`.** It describes the act and claims nothing about the result. **And *exchange* is
accurate rather than loose** — §5.0's closure makes it bidirectional by construction: what a peer
obtains, it publishes; the consumer becomes a source. **A one-directional name would have been the
reader-scoped error surviving in the title.**

⚠ **It is not a single-capability extension and the name spans four documents** (§15.5), which is the
identity family's shape and is why forcing one noun kept failing. **Twenty-six extensions are single
nouns because twenty-six extensions are single capabilities. This is a data layer.**

⭐⭐ **The rule the three failures actually earn, and it outranks availability:**

> **A name must claim no more than the mechanism guarantees.** `replica`, `sync`, `mirror` and
> `current` each assert a property this refuses. **Prefer a name that describes the ACT over one that
> describes the RESULT** — the act is what an implementation controls; the result is what the network
> decides.

**Second rule, from the `acquisition` failure:** ***census every region a reader meets, not the two you
think of as the corpus.*** A count inherits the shape of its search and the shape is invisible in the
number — two clean regions read exactly like four.


## §12 Deltas

| | Target | Change |
|---|---|---|
| **D1** | **new** `specs/extensions/EXTENSION-DATA-EXCHANGE.md` | The wire-observable surface: subject and source declarations, the participant-set type, the witness operation, the six outcomes. Tier 2b. |
| **D1a** | **new** `specs/SYSTEM-DATA-EXCHANGE.md` | ⭐ **The closure property (§5.0)**, the authority axis, cross-peer currency, the droppable-position test. **The half no single extension owns** (§15.5). |
| **D1b** | **new** `specs/sdk/SDK-DATA-EXCHANGE.md` | ⭐ **The one entry point (§6.0) and the default profile (§6.0.1)** — the layer an application developer touches. |
| **D1c** | **new** `guides/GUIDE-DATA-EXCHANGE.md` | §15's four traces as instructions: feed · encyclopedia · forum · game. |
| **D2** | `SYSTEM-COMPOSITION.md` | New chapter *Holding another peer's data*: cross-peer currency as a composition property; the entry-point requirement; the droppable-position test; the outcome taxonomy; the authority axis (§8.2). |
| **D2a** | `SYSTEM-COMPOSITION.md` §5 | One line distinguishing **cascade termination** (that chapter's *convergence*) from **cross-peer currency** (the new one), each pointing at the other (§4.3). |
| **D3** | `EXTENSION-SUBSCRIPTION.md` §5.5 | Demote the predecessor chain to write-many paths; state its blind spot on write-once workloads. |
| **D4** | `EXTENSION-SUBSCRIPTION.md` §6.3 (ii) | Cite the composition chapter for the mechanism; stop citing a detector that cannot deliver the property. **And state that the property is currently exercised only in the mode where it cannot fail** (§8.1). |
| **D5** | `APP-CONVENTION-FEED.md` §2.4, §11 `F-1` | Remove the position field from the follow declaration; discharge `F-1` against this mechanism. |
| **D6** | `APP-CONVENTION-FEED.md` §4 | Narrow the index to authored order and cold-start cost; it is no longer the change-detection mechanism. |
| **D6a** | `APP-CONVENTION-FEED.md` §4.1 | ⛔ **Correct the rationale in the same change as D6** — *enumerating a prefix from nothing costs the tree; discovering what is new, holding the previous root's nodes, costs the path* (§9.2). **D6 without D6a leaves the false sentence standing as the reason to undo D6.** |
| **D7** | `SYSTEM-COMPOSITION.md` | Promote *a derived view must declare what produced it* to the composition layer; `APP-CONVENTION-FEED` §9.3 restates it and names its authority (§9.3). |
| **D8** | `APP-CONVENTION-EMBED.md` §3, §4, §5.3, §6 | Split the generic payload container from the presentation envelope; add the declination requirement and *a typed payload remains typed at every point where a decision is made about it* (§9.4). |
| **D9** | `SYSTEM-ARCHITECTURE.md` §13.1 | Add the new extension to Tier 2b. |
| **D10** | `specs/applications/APP-CONVENTION-*.md` | Add a `Tier:` line placing each in §13.1's model. **None currently declares one while every extension spec does** — which is why placement is re-derived in every discussion. |
| **D11** | `SYSTEM-ARCHITECTURE.md` §13.1 and `SPECIFICATION-FORMAT.md` §10.3 | Disambiguate **tier**: §13.1's is *architectural* (where a component sits), §10.3's is *governance* (which authoring standard binds a document). **Neither currently names the other.** |
| **D12** | `EXTENSION-TREE.md` §3.8 R1 *(cross-reference)* and `EXTENSION-REGISTRY.md` §6a.3a | Cite **TREE §3.8 R1** as the publisher-side instrument for §1's A1 — the registry section is a consumer of that rule, not its authority — **and state that it does not address A2** (§1.2). |
| **D13** | `EXTENSION-TREE.md` §3.7.2 | Record that the audit's *"no current consumer"* verdict now has one, and that **reconstruction (§3.7.1), not the sidecar index, is what it needs** (§9.7). |
| **D14** | `SPECIFICATION-FORMAT.md` §5.1 *(authoring standard)* | A proposal whose title is a sentence **MUST name the artifact it proposes in the first line of its header.** Four recorded instances in one arc of a reader unable to classify a name; **the cost of the rule is one line and the cost of the failure was a reader's whole first pass** (§14). |

| **D15** | ⛔ `SYSTEM-DATA-EXCHANGE` (§6.5) | **The `shared` reconciliation rule is an OPEN vocabulary naming an `EXTENSION-REVISION` strategy, not a two-value enum.** Found by §15.2: an encyclopedia page wants revision-DAG merge, and the only shipped vocabulary is last-arrival-wins / keep-both — **so the first wiki built on this silently gets last-write-wins.** |
| **D16** | `SDK-DATA-EXCHANGE` (§6.6) | **Permit ACQUIRE-WITHOUT-INTENT** — a single pass with no durable declaration and no position. Found by §15.2: a one-shot lookup is not a follow, and §6.6 assumed one. Cheap, and unwritten. |
| **D17** | ⭐ `SYSTEM-DATA-EXCHANGE` (§6.2.4) | **A source set MAY itself be an acquired subject; such a subject is always `owned` by its publisher, which is what BOUNDS THE RECURSION.** Found by §15.3, and it is the sharpest evidence that this is one mechanism: **its own source-discovery is an instance of it.** |
| **D18** | `SYSTEM-ARCHITECTURE` §13.1 / `GUIDE-NETWORKING-MODEL` §4a.1 | State that **`SOURCE`'s ordered legs are the reachability rungs plus publishing posture** (§15.6) — the two are one arc, and sequencing them as separate work is what made the game case look blocked. |

| **D19** | ⛔ `EXTENSION-TREE.md` §3.7.1 | **Correct the cost claim.** *"Microseconds to low-ms. Cheap"* is wrong inside its own stated range — measured **34 ms / 455 ms / 2.62 s** at 1k / 10k / 50k (§9.7.0). **State the measurement, the implementation it was taken on, and that no bulk builder exists anywhere.** |
| **D20** | ⭐ `SYSTEM-DATA-EXCHANGE` (§5.0.0) | **Promote four rules out of `APP-CONVENTION-FEED` to the tier where closure is claimed:** a republished entry carries its author's detached signature · a mirror may carry one but never supply one · an entry whose signature is absent renders as **unattributed** · republished bytes are **never re-encoded**. **The instrument is already corpus-wide; only the obligation is application-scoped.** ⭐⭐ **FOLDED — `specs/SYSTEM-DATA-EXCHANGE.md` v0.1, with `D24`.** This was the one item gating a gatherer build. |
| **D21** | `SDK-DATA-EXCHANGE` (§5.0.0) | ⭐ **Name the bind-an-obtained-entity operation.** Byte-preserving republication needs an operation that binds *obtained bytes* rather than *data*; **the ordinary `put(path, type, data)` shape is the one a developer reaches for and it is the broken one.** At least one implementation already ships the right operation and it took a grep to establish that. |
| **D22** | `SYSTEM-DATA-EXCHANGE` (§5.0.1) | State that **a `shared` subject's declared reconciliation rule IS the coordination point the CALM reduction predicts** — `owned` and `ownerless` are coordination-free and `shared` is not. |
| **D23** | `SYSTEM-DATA-EXCHANGE` (§6.2.1) | Add the **seventh outcome `declined`** and the coverage rule: **every party that can end an attempt owns a row — source, publisher, asker's authority, nobody, and the READER.** |

**Revision 5 — the build round's second half. `D20` is FOLDED; the rest are open.**

| | Target | Change |
|---|---|---|
| **D24** | ⛔ `SYSTEM-DATA-EXCHANGE` (§5.0-a) | **Name the LAYER closure is over.** *Verified, attributed and rendered* by the identical path; **the walk may differ**; `MUST NOT` publish an author's own set-layer object under the gatherer's namespace to satisfy it. **Revision 4's flat sentence is literally false as built, and taken literally it builds a forgery.** ⭐ **FOLDED into `SYSTEM-DATA-EXCHANGE` v0.1 with `D20`** |
| **D25** | ⛔ `APP-CONVENTION-FEED.md` §6 | **`subject` widens to `any-reference`**, plus `MUST NOT` pin a value that moves when the subject changes. **A pinned subject is the thread case; a live subject is the timeline case — and without the widening, the type that carries the 250× cannot name what §15.1 republishes** (§9.8). ⭐ **FOLDED** |
| **D26** | `APP-CONVENTION-FEED.md` §6 | **Pin the prefix `app/feed/mirrors/`** and the per-kind key derivation over **identifying fields only** (§9.8.2). **This is the other half of the 250×**, not a formatting choice: Erin's 2-request check needs descent from the gatherer's signed root, and a type-filtered query is unserveable by a static origin. ⭐ **FOLDED** |
| **D27** | `SYSTEM-DATA-EXCHANGE` (§6.2.1a) | **An eighth outcome — `unsupported`, reader CAPABILITY**, split from `declined`, reader POLICY. And the corrected coverage rule: **the taxonomy is generated by DESTINATION, not by party; a party with two destinations owns two rows.** *Stated per-party it is closed at seven and the eighth is unreachable by construction.* |
| **D28** | `SYSTEM-DATA-EXCHANGE` (§6.5) | **Scope the reconciliation refusal to the ACT, not to acquisition** — keep acquiring, hold only the colliding change, `MUST NOT` stop the subject, and **report the hold as `declined`.** The strong reading fires on *the other device is asleep* and is less recoverable than the hazard. |
| **D29** | `SYSTEM-DATA-EXCHANGE` (§6.3.4a) | ⭐ ***Repeated passes make progress* binds the SCHEDULER** — `MUST NOT` derive back-off from whether a pass transferred anything; a pass that advanced the resumption point made progress. Plus: **a truncated result carries its RESUMPTION POINT**, not only the fact of truncation. *The tell is a present-tense disclosure of an incomplete result.* |
| **D30** | ⛔ `SYSTEM-DATA-EXCHANGE` (§6.5.2b) | **The delivered-leg witness.** A sender-minted per-`(sender, subject)` counter is **author-anchored**, not source-minted — *on that leg the sender is the writer* — and it separates a genuine revert from a replay, which content cannot. Plus `MUST NOT` **synthesize a floor from a quantity minted by neither the writer nor the content**, and where no witness exists the leg reports `not_supported` rather than reading as defended. **Carrying it is a subscription-tier change: the second independent reason to open `Q4`.** |

**D10, D11 and D14 are independent of everything else here and should land regardless of this
proposal's fate.**

---

## §13 What this proposal does not establish

- **Fit was read from each implementation's framing, not tested by porting.** No implementation has
  been written against this specification.
- **Two code bases, and not every implementation.** §2.1.
- **The turn-based game and the real-time tick were reasoned from stated properties. Neither has been
  built**, here or anywhere.
- **The multi-device failure (§6.5.2) is measured on two publication trees under one keypair**, which is
  an honest stand-in for two devices and is not two devices. **The ruling that follows from it is a
  classification, and the classification has not been implemented anywhere.**
- **One cited defect is a source reading rather than an executed run**, and the party that filed it
  marked it so: the non-resuming bounded walk of §6.3.4 predicts a specific failing case — a subject over
  the cap, two passes, assert the second reaches paths the first did not — **that nobody has performed.**
- **§9.2's numbers are request counts against a local server.** The counts transfer; the wall-clock does
  not, and the trie shape measured is one flat write-once prefix.
- **Two of the landed-text corrections in §9 were verified by opening the text; two others reported by
  an implementer against a third extension were taken on report and are not cited in any requirement.**
- **No test vectors are proposed.** §10 states what a check set must discriminate; the fixtures are an
  output of implementations running against one another.

---

## §14 What revision 2 changed, and what closed each question

**Both consuming implementations reviewed revision 1 in full against their own source. Twelve findings
from one, ten of them measured; eleven from the other, with one measurement run for the purpose.
Neither disputed the boundary.** What changed:

| | change | what forced it |
|---|---|---|
| **1** | ⭐⭐ **§6.0 — the single entry point is normative**, and §10 gains C19 | Both, independently. Revision 1 had eight layers and no call site, and its check set could not distinguish a conformant implementation from the failure it quotes as its own motivation |
| **2** | ⭐⭐ **§6.5 — a third authority value, `shared`** | Four unrelated cases in neither column, from two parties, one of them a **shipped product the specification forbade from converging** |
| **3** | ⭐⭐ **§6.5.2 — a writer is a process's serialized authority, not a keypair or an identity** | A measured multi-device refusal, resolved **from landed corpus text** (identity agents are per-device peers; a device is a private-key holder) rather than by a new rule |
| **4** | ⭐ **§6.3 — the witness replaces the two-row table**: opaque, per *(source, subject)* | Two counterexamples from opposite ends — a prefix subject with an artifact-shaped check, and a live peer with no published root — **plus a third-party entity-tag, which decides whether this can point at an ordinary web origin** |
| **5** | **§5 — eight layers became five and a seam** | The document's own membership test, applied to itself: commitment adds nothing and addressing is the source's |
| **6** | **§6.2 — four outcomes became six**, with a generative rule and a terminality column | Revision 1 merged two conditions **inside the row whose own rule forbids merging** |
| **7** | **§9.2 — D6 is measured and §4.1's rationale is corrected with it** | 7 requests for a 1-entry delta over 50,000 entries, with a control arm at 3,346 |
| **8** | **§6.3.3–§6.3.4 — node retention, complete-walk-only memoization, no recursion, report truncation, make progress** | One party's own defects, three of them earned and one of them a source reading they marked as such |
| **9** | **§6.6 — declared intent is durable; derived intent is not** | Returned against its own two intent surfaces: the `MUST` fits one and is actively wrong for the other |
| **10** | **§7.1 — the submit path is ruled excluded, and read-your-own-write is answered** | *"Any of the three is fine; silence is not, because every application with a write path hits it on day one"* |
| **11** | **§1.3 / §7 — the publisher-side absence class is named and excluded** | Two of revision 1's own open questions were circling one unnamed thing |
| **12** | **§11.1 — the name was `CURRENT-COPY`, and it did not survive revision 3 either** | `replica` imports a consistency model and a bounded staleness guarantee this mechanism refuses — flagged as a caveat by one seat and rejected outright on review. **`acquisition` replaced it, was written into this document, and was withdrawn in the same session**: the census clearing it read `specs/` and `guides/`, and the corpus's existing *acquisition surface* — **getting two peers into contact, a different subject one tier over** — lives in the 33 occurrences under `docs/proposals/` and `docs/research/` that the census did not read |
| **13** | **§4.3 / D2a — the composition chapter is named for its question** | *Convergence* already titles a chapter about cascade termination in the document the new one joins |
| **14** | **D14 — a proposal names its artifact in the first line of its header** | A reader could not classify this document's own title as an extension, a process or a document name, and asked which it was before asking what it said. **Fourth instance in one arc of *the name is load-bearing and review does not see it*, the previous one caught by a lint gate rather than by a reader** |

**Closed in revision 2:** Q1 (cross-check semantics — accepted, and disagreement is an outcome rather
than a resolution: §6.2.3) · Q3 (walk bounds — freedom on the number, `MUST` on structure, reporting
**and progress**: §6.3.4) · Q5 — **reopened and reframed in revision 3, see below.**

---

### 14.1 ⛔ What revision 3 changed, and all of it is one error with four faces

**Revision 2 answered every question both seats asked and lost the spine doing it.** The error:
**it scoped the mechanism to the reader.** Four consequences, and the name was only the visible one.

| | what revision 2 did | what revision 3 does |
|---|---|---|
| **1** | ⛔ declared the mechanism **reader-scoped** and the publishing half *"out of scope"* | ⭐⭐ **§5.0 — the CLOSURE property is restored as the load-bearing requirement:** what a peer obtains it may publish, as the **same kind of object**, consumable by the identical code path. **This is the property that decides whether the network federates or centralizes**, and the systems that centralized did so because their aggregator's output was a different type from its input |
| **2** | ⛔ **excluded the submit path** | **§7.1 — wrong, and inverted.** Contributing to a subject you do not own is *publishing into your own namespace naming a coordinate*; there is no submit primitive because there needn't be one. **Only multi-peer ATOMIC commit is out** |
| **3** | ⛔ dropped the **publisher / consumer / gatherer / tracker** vocabulary entirely, and with it `coordinate → who` | **§5.1 — the roles are kept as ACTS** with one rule (never name a role as required or privileged); **§6.2.4 — `coordinate → who` is the SOURCE layer for a subject nobody owns**, which is where it always belonged. *It is the one genuinely new object in the design and it is ~32 bytes an edge* |
| **4** | ⛔ **wrote off real-time as "does not fit"** on reasoning, with nothing built | **§15.4 — reopened.** The mechanism makes no latency guarantee and never will; **whether that is fatal is the application's call, not the specification's**, and two of the three things a tick-rate application needs are already here |
| **5** | ⛔ named it `CURRENT-COPY` | **§11.1 — `DATA-EXCHANGE`.** *Current* is a claim no participant in this system can make about anything |
| **6** | — | ⭐ **§15 — four use cases traced end to end**, and a layer map saying which document owns what |
| **7** | ⛔ treated the live / hosted / static rungs as a different arc | **§15.6 — they are three SOURCE kinds in one ordered list.** That arc and this mechanism are the same work seen from two ends |

---

## §15 ⭐⭐ The traces — four applications run end to end through the model

**§13 admits that fit was read from each implementation's framing rather than tested. This section is
the test.** Four applications, traced layer by layer. **A trace that finds nothing was not run** — each
one below reports what it broke or what it needed that is not there.

### 15.1 A social feed — publisher, consumer, and a peer who republishes

| layer | what it is here |
|---|---|
| **SUBJECT** | `<author-peer> : feed/` — a growing prefix. **`owned`**, one writer |
| **SOURCE** | ordered: **the author's live peer · the author's published origin · any peer republishing the author's walk** |
| **WITNESS** | the signed root pointer at a fixed key. **Measured: 2 requests for a no-op check at every scale, 7 for a one-entry delta over 50,000** |
| **POSITION** | `(reader, subject)` — how far this reader has read. Droppable |
| **INTENT** | the follow record. **Declared** (it has a withdrawal act: unfollow) ⇒ durable |
| **LOWERING** | `app/feed/entry` → the reader's timeline model |

**The cycle closes, and this is §5.0 doing visible work.** Bob follows Alice, Carol and Dave. **Bob
publishes his walk** — signed entries in Bob's namespace, attributed to their authors. Erin follows
**Bob**, using the identical code path she uses for a person, and gets three authors' activity without
following any of them. *Erin can then republish her walk.* **No tier, no API, no discontinuity.**

⇒ **Found:** nothing new in the trace itself — **and the count attached to it was wrong.** Revision 3
said four of seven consumers were this profile; re-measured against the four one seat owns, **one is**
(§6.0.1). *The trace was right and the census behind it was not, which is the second bare count this
document has had to correct.*

### 15.2 An encyclopedia article — the cold reader who wants one page

**The case: *"I want to look up the latest on this article,"* with no prior copy and no follow
relationship.**

| layer | what it is here |
|---|---|
| **SUBJECT** | one article. **`shared`** — many editors, convergence required |
| **SOURCE** | the editors' streams, plus any peer publishing a walk of the article |
| **WITNESS** | per source; none offered ⇒ enumerate that editor's contributions to this coordinate |
| **POSITION** | irrelevant on first read — **and this is the case that proves position must be droppable** |
| **INTENT** | ⚠ **there may be none.** A one-shot lookup is not a follow |
| **LOWERING** | edits → a rendered page |

⛔ **Found — two things, and the first is a real gap.**

1. **The reconciliation rule vocabulary is too small.** §6.5 requires a `shared` subject to name its
   rule, and the only shipped vocabulary is *last-arrival-wins* / *keep-both*. **An encyclopedia page is
   neither** — it wants revision-DAG merge. ⇒ **the rule MUST be an open vocabulary naming an
   `EXTENSION-REVISION` strategy, not a closed two-value enum**, or the first wiki built on this
   silently gets last-write-wins. **New `D15`.**
2. **A one-shot read has no INTENT record, and §6.6 assumes one.** The mechanism must permit
   **acquire-without-intent** — a single pass with no durable declaration and no position. *This is
   cheap and it is not written down.* **New `D16`.**

### 15.3 A forum thread — the hard one, and the mechanism recurses on itself

**The case: N repliers, nobody owns the thread, and the reader follows none of them.**

**Step 1 — the base case needs no index.** If the reader follows a replier, the reply arrives in the
fetch they were already doing. **And every entry carries its outbound edges**, so pulling a followed
stream reveals that the thread exists at all. ⇒ **forward traversal is free and does most of the work.**

**Step 2 — reach past the follow graph** needs `coordinate → who` (§6.2.4). And here is the finding:

⭐ **The participant set is ITSELF a subject, acquired by this same mechanism — and the recursion
terminates.**

```
acquire(thread)
  └─ acquire(participant-set)        ← subject; OWNED by the peer that published this walk
       └─ [peer-id, peer-id, …]       ← ~32 bytes each; merge by set UNION across trackers
  └─ for each participant: acquire(their stream)   ← subjects; OWNED by each author
  └─ merge, order, render                          ← LOWERING, the application's
```

> **It terminates because a published walk is always an `owned` subject** — one writer, the peer that
> published it. **There is no cycle**: source discovery is one level of the same mechanism, and that
> level's sources are given.

⇒ **Found:** the recursion property is real, load-bearing and unstated. **New `D17`** — *a source set
MAY itself be an acquired subject; such a subject is always `owned` by its publisher, which is what
bounds the recursion.* **And it is the sharpest evidence that this is one mechanism rather than
several: its own source-discovery is an instance of it.**

### 15.4 ⭐ A multiplayer game with a semi-authoritative host — the case revision 2 wrote off

**Revision 2's Q5 said real-time *"fails at the collection model."* That was reasoned from stated
properties with nothing built, and it was wrong twice.**

| layer | each player's moves | the authoritative board |
|---|---|---|
| **SUBJECT** | `<player> : moves/<match>` — growing prefix | `<host> : board/<match>` — ⭐ **a fixed key whose value is replaced** |
| **authority** | **`owned`** by that player | **`owned`** by the host — **named by an identity or role, so the host can migrate** (§6.5.3) |
| **SOURCE** | each player's live peer; their origin if they closed the lid | the host's live peer |
| **WITNESS** | signed root | ⭐ **class 1, fixed-key mutable** (§6.3.2) — a pointer that moves iff the board moved |
| **POSITION** | the host's, per player | **none needed** — a tick has no backlog to resume |

**The loop:** a player publishes a move into their own
namespace → the host acquires each player as a source → the host resolves and **publishes the new
board** → every player acquires the board. **A player who contributes nothing still receives ticks;
the host still advances.** *That is the §5.0 cycle with the host as one more peer.*

⛔ **Both halves of revision 2's exclusion were wrong:**

1. **"It fails at the collection model."** ⇒ **No.** A tick is a **fixed-key mutable** subject — §6.3.2
   **class 1**, already specified. *There is no backlog to wrongly deliver, because a new value replaces
   the old one; the "durable append plus resumption" framing was the growing-prefix case being mistaken
   for the whole mechanism.*
2. **"Eventual-only is fatal for a tick."** ⇒ **Not the specification's call.** §6.7 refuses to
   *guarantee* a bound and that does not stop anything working: **a subscription notification collapses
   the compare interval to one round trip**, because §6.3 makes the stream an optimization over *when*
   to compare rather than the source of truth. **The floor is the network, which is where it is for
   every other system too.**

⇒ **The honest statement, and it replaces the exclusion:** *the mechanism is structurally adequate for
a tick-rate application; it guarantees eventual convergence and no bound; whether that is acceptable is
the application's measurement, not the specification's ruling.* **Nobody has built it, which is a
statement about evidence, not about fit.**

⚠ **What a tick-rate application would find first**, named so it is not discovered as a surprise: the
per-pass cost floor is a comparison plus a notification, not zero; and **`shared` subjects with a
contested reconciliation rule are where a game would actually hurt** — which is §15.2's `D15`, arriving
from a completely different application.

### 15.5 ⭐ The layer map — which document owns what, and what a developer touches

**This is why it is not one extension.** The precedent is exact and already in the corpus: **identity**
spans an architecture document, a composition document, an extension, an SDK spec and a guide.

| Layer | Document | Owns |
|---|---|---|
| **Extension (Tier 2b)** | **`EXTENSION-DATA-EXCHANGE`** | The wire-observable surface: subject and source **declarations**, the participant-set entity type, the witness **operation**, the six outcomes. *What a peer must implement to interoperate* |
| **System / composition** | **`SYSTEM-DATA-EXCHANGE`** | ⭐ **The closure property (§5.0)**, the authority axis, convergence-across-peers, the droppable-position test. **The properties no single extension owns** — which is the same reason the subscription defect in §8.1 is a layering defect |
| **SDK** | **`SDK-DATA-EXCHANGE`** | ⭐ **The one entry point (§6.0) and the default profile (§6.0.1).** *This is the layer an application developer actually touches, and it is the whole answer to "don't make me rebuild this"* |
| **Guide** | **`GUIDE-DATA-EXCHANGE`** | *"I want to build a forum / a wiki / a feed / a game"* → the traces above, as instructions |
| **Application conventions** | `APP-CONVENTION-FEED`, `-CHAT`, … | The **vocabulary only** — type tags, field names, lowering. **They register against the mechanism instead of containing it**, which is the whole reduction (§4.4) |

**What a developer writes, in order:** declare your **data model** (application) → declare the
**subject** and its authority class → declare the **sources** → call the **one entry point** → handle
**three results** → **lower** arriving changes into your model. *Everything else has a default.*

### 15.6 ⚠ The live / hosted / static rungs are three SOURCE kinds, not a separate arc

**Revision 2 treated the rung work as a different workstream. It is the same work seen from the other
end.** The corpus already fixes three ways to reach a peer, in a `MUST` ordering — **direct · mediated
live · store-and-forward** — and a statically hosted peer is *"not a special category"*, just a mode of
the same relay handler.

⇒ **`SOURCE`'s ordered legs ARE that ladder, plus publishing posture.** *live peer → hosted origin →
static origin → another peer's republished walk* is one ordered source list, and **§6.2.3's rule that an
empty answer must not end the ladder is what makes a publisher who cannot serve live still reachable.**

⛔ **And the live leg is NOT universally available, measured on the two surfaces one seat ships.** A
peer with **no listening socket is a real deployment class — it is the one that repo ships** — and for
it the live rung is rendezvous presence, which is *an activity a peer performs, not a state it has*
(cadence ~3 s decaying to ~30 s). **On one desktop runtime the WebRTC binding is absent from the
system web view entirely**, measured on two distributions: that peer is a rendezvous node and a
websocket peer, **never a live peer.**

⇒ **§15.4's *"the floor is the network"* is true for a peer with a listening socket and false for
these.** For a browser-hosted game the floor is the presence cadence; on that desktop runtime the live
leg does not exist at all, and §6.2.3 keeps such a peer reachable via its origin — **which is a
publication act per tick.** *The trace is not withdrawn; its floor is.*

⚠ **So the dependency runs both ways and should be stated rather than sequenced:** this mechanism needs
the live rung to exist for the live leg to be more than a name, and the rung work needs this mechanism
to be what rides on it. **They are one arc.**

---

## §16 ⭐⭐ Revision 4 — what a build round found that three review rounds did not

**One seat built the closure path. The other read the substrate underneath it. Between them they
produced the first findings in this arc that no amount of reading would have reached.**

| | finding | kind |
|---|---|---|
| **1** | ⭐⭐ **CLOSURE HOLDS.** A → B → C, B republishing byte-preserving, C consuming with the same consumer, no branch, no knowledge that B authored none of it. **Every entity hash byte-identical at both hops** | ✅ **run** |
| **2** | ⛔ **The NAIVE republish breaks it silently — 3 of 3 hashes moved, no error anywhere** — and the loss is authorship, not a field. **§5.0 had no byte-preservation clause** | ✅ **run, control arm** |
| **3** | ⭐ **Two peers published the IDENTICAL trie root under different peer-ids** — unpredicted, and it is the measurement that resolves the witness contradiction (§6.3.1a) | ✅ **run** |
| **4** | ⛔ **`Q9`'s recommendation inverts, and a landed cost claim is wrong inside its own stated range** (§9.7.0) | ✅ **run** |
| **5** | ⛔ **Closure's other precondition is authorship evidence that survives detachment** — the instrument is corpus-wide, **the obligation exists only in one application convention** | ✅ source, three ways |
| **6** | ✅ **`C15` is a measurement** — 25,000 entities, 20,000 cap, two passes, **0 new paths** | ✅ **run** |
| **7** | ⛔ **The six outcomes have no row owned by the READER**, and two live conditions are reader facts | ✅ source |
| **8** | ⛔ **The floor row was a security regression**; the floor is a property of a **leg** (§6.5.2a) | derivation |
| **9** | ⛔ **A `shared` rule stored per-peer does not converge** — live, in shipped code, exposed by the `MUST` | ✅ source + wire shape |
| **10** | ⭐ **The reconciliation rule IS the CALM coordination point** (§5.0.1) | derivation |
| **11** | ⭐ **250× read amplification**, and the default profile did not carry the leg that fixes it | arithmetic over a measurement |
| **12** | ⛔ **Two profiles, not one**; the census behind *"four of seven"* was wrong (§6.0.1) | ✅ source |

### 16.1 The three corrections that were against THIS document's own reasoning

1. ⛔ **A cost figure travelled three hops with no measurement anywhere** — prose in a landed spec,
   repeated into this proposal, turned into a recommendation to a seat that then had to measure it to
   find out. ⇒ ***a cost figure in a normative document is a measurement or it is marked unmeasured.***
2. ⛔ **A blanket `MUST` forbade the instrument another `MUST` required.** *Two `MUST`s in one document
   were unsatisfiable together and three review rounds did not catch it; writing the API signature down
   caught one and running the path caught the other.*
3. ⛔ **Two clauses adopted verbatim from an implementation were generalized around and not re-scoped**,
   so they `MUST`ed a data structure a conformant source need not have. **The seat whose sentences they
   were is the seat that caught it.**

### 16.2 ⭐ Where this belongs — the core-tier question, answered on evidence

**Asked: should this sit at the core peer level?** **No, and the reason is that the fix which requires
no core change is the one that works.**

| | |
|---|---|
| **What closure needs** | authorship evidence surviving detachment, and byte-preserving republication |
| **What supplies it today** | a detached signature at the invariant pointer — **corpus-wide, in twenty documents, stated as a general fact by a landed extension** — plus an SDK operation that binds obtained bytes. **Both exist; at least one implementation ships both** |
| **What a core change would look like** | making an entity carry a signer field — **a wire-format change to a locked core**, renumbering what must never be renumbered |
| ⇒ | **Keep it at this tier.** The two new `MUST`s are a **promotion of existing normative text** plus an SDK surface, and nothing in them touches the wire |

⚠ **The honest caveat: the constraint that a signed root may commit only to keys under its own peer is
a TREE-tier property, and it is what forces a gatherer's projection shape** — mirror record under the
gatherer's namespace, republished bodies reachable by content hash with no trie key under it. **That is
still this tier, and one implementation already has the shape.** ⇒ **revisit core placement after the
first working gatherer, not before.**

### 16.3 The build handoff — what is settled, and the order

**Settled and buildable now:**

| | |
|---|---|
| ✅ | closure, its two preconditions, and the projection shape |
| ✅ | the three witness classes, and `witness(subject) → token` with `not_supported` |
| ✅ | seven outcomes, terminality, and the reader-owned row |
| ✅ | two default profiles, three source legs |
| ✅ | the floor per leg · the `shared` rule's obtaining party · acquire-without-intent |
| ✅ | the entry point, its `MUST NOT` on bypass, and its qualifiers |

**Open and NOT blocking anyone:** `Q2` (a publisher's preferred read path) · `Q4` (the drop signal's
home, which is a subscription-tier change needing its own proposal) · `Q6` · `Q8` (sequencing of the
composition chapter against the extension).

⭐ **Build order, and both seats independently proposed the same one:** the live-peer source leg first —
**it is the same arc as this mechanism's SOURCE legs, not a separate workstream** (§15.6) — then the
gatherer, **because it is the only thing that exercises closure in a product rather than in a probe,
it is the 250× fix, and the byte-preserving republish operation exists on one seat with no shipped
surface.**

### 16.4 ⚠ What revision 4 still does not establish

- **The closure run is three entities, not three thousand.** Closure is a typing property and size
  should be irrelevant to it — *should be* is not *was measured*.
- **The identical-trie-root result was observed on one binding set.**
- **The `shared`-rule divergence is derived from a record shape and a wire view, not performed.**
- **The seventh outcome is argued from two instances in one implementation.** A second seat finding a
  reader-owned outcome is the evidence it deserves.
- **The 250× is arithmetic over a measured 2-request no-op, not a measured 500-follow reader.** Nobody
  has built one.
- **The reconstruction cost is one implementation's incremental builder, not a lower bound.**
- **The live-leg bound is measured on two surfaces** and says nothing about a peer with a listening
  socket, which is what every reference implementation is.
- **No implementation has been written against revision 4**, and the promotion in `D20` is not landed —
  **which is the one thing a gatherer build should wait for**, because building on an
  application-convention rule is the failure that promotion exists to prevent. ✅ **`D20` landed
  2026-09-12** (§17).

---

## §17 ⭐⭐ Revision 5 — both seats built against revision 4, and the sentence billed as load-bearing was false at one of its two layers

**Round 4 was *one seat builds, one seat reads*. Round 5 is *both seats build*, and it is the first
round in which the findings are about THIS document's own headline claims rather than about gaps in
them.**

| | finding | kind |
|---|---|---|
| **1** | ⛔⛔ **§5.0's *"identical code path"* is FALSE at the set layer** — measured in a built gatherer: the entry layer is literally one function, the walk is two. **And the literal reading builds a forgery** — a reader takes it at face value and looks for the author's own index shape under the gatherer | ✅ **run** |
| **2** | ⛔⛔ **The 250× has no type to ride on** — §6's gathered record pins ONE ENTITY, and §15.1's trace republishes THREE TIMELINES, which cannot be pinned (§9.8) | ✅ derivation over two landed productions, from a built gatherer |
| **3** | ⭐ **The closure gate is GREEN under the neuter that breaks closure** — fixtures from your own encoder make decode-and-re-encode lossless. **The arm that measures the `MUST` needs a field the reader does not declare** | ✅ **run, and self-caught** |
| **4** | ⛔ **The reader owns TWO destinations, not one** — policy (*I could and will not*) and capability (*I would and cannot*) send a person to different places, so the generative rule splits them (§6.2.1a) | ✅ source, both shipped |
| **5** | ⛔ **The delivered-leg replay is PERFORMED and it lands** — `status=200`, no error, the newer file gone from disk. **And the monotone quantity is author-anchored, not source-minted** (§6.5.2b) | ✅ **run, with a control arm** |
| **6** | ⛔ ***Repeated passes make progress* is met by a loop that never finishes** — the walk resumes and the scheduler backs off, because the first segment of an oversized folder legitimately recovers nothing (§6.3.4a) | ✅ **near-miss, caught in build** |
| **7** | ⭐ **A second seat produced a reader-owned outcome independently** — the conflict-rule hold is `declined` exactly. **§16.4 said this was the evidence the row deserved** | ✅ source |
| **8** | ⭐ **A `MUST` was satisfiable with facts both sides already held** — the `shared` rule's obtaining party needed no wire field, because the folder identifier already designates an owner | ✅ **built** |

### 17.1 ⛔ The two corrections against THIS document, and both are against sentences it called load-bearing

1. ⛔⛔ **The closure `MUST` was billed as *"the load-bearing sentence in the document"* and it is false
   at one of the two layers it quantifies over.** Three review rounds and one build round read it and
   agreed with it; **the round that ran a consumer against a gathered view measured it in an
   afternoon.** ⇒ ***a sentence that is true at the layer everyone is thinking about, and false at the
   layer nobody is, reviews clean forever.*** The repair is one word of scope and it costs the design
   nothing — **which is the tell that the original was imprecise rather than wrong.**
2. ⛔ **The headline trace and the only type that could carry it were never checked against each
   other.** §15.1 has said *"Bob publishes his walk"* since revision 3, and `APP-CONVENTION-FEED` §6
   has said `subject: reference` since it landed; **the two are one grep apart and nobody ran it,
   because each is correct in isolation and the contradiction exists only in the join.** ⇒ **a trace
   is not run until its types are named.** *§15's own preamble says "a trace that finds nothing was
   not run" — it found nothing here, four traces in a row, and the reason is that it was traced
   through the model and never through the corpus.*

### 17.2 ⭐ The method finding, and it is the one to carry past this arc

> **A gate whose fixtures came from your own encoder is testing your encoder against itself.**

**The seat that built the closure gate neutered their own byte-preservation and the closure gate stayed
GREEN** — because every fixture entry had been written by their encoder, so decode-and-re-encode was
lossless over all of it. **`APP-CONVENTION-FEED` §6.1 names this hazard by hand** — *"a round trip
through bytes your own encoder produced proves nothing: the input must be deliberately non-canonical"*
— **and their first control arm walked into it anyway and failed honestly.** The arm that measures the
`MUST` publishes an entry carrying **a field the reading build has never heard of**, which is the
realistic case *because a gatherer aggregates types it did not write.*

⇒ **This is `unobserved-must` at the fixture layer**, and it is worse than an uncovered requirement:
**a gate that covers a property and a neighbouring one that merely looks as though it does is worse
than one gate**, because the second is counted. *Both gates now say in their own text which is which.*

### 17.3 What revision 5 still does not establish

- **The gathered-view runs are 3 and 4 entities.** Unchanged from §16.4 and now true of two seats.
  *Closure is a typing property and size should be irrelevant — **should be** is not **was
  measured**, and this arc has punished that phrase twice.*
- **Nothing publishes a gathered view from a user-facing surface.** Both implementations are native
  gates; no verb, no window, nothing reachable by a person.
- **The 250× is still arithmetic**, now over a measured no-op **plus** a derivation about where the
  witness lives (§5.0). **Nobody has built a 500-follow reader.**
- **The timeline gathered view has not been run**, because until `D25` there was nothing to run it
  against. **The finding that produced `D25` is derived from a CDDL production, and is not a run.**
- ⛔ **Who may dispatch to the delivered leg is UNMEASURED.** The handler's missing ordering check is
  measured; that an unauthorized party can reach it is not, and the interim defence of that leg is
  therefore an assumption (§6.5.2b).
- **The `shared`-rule fix is measured in-process** — two peers, one machine, real TCP. Not across two
  hosts.
- **`D21`, `D22`, `D23`, `D27`–`D30` are ruled in a draft and not folded.** Only `D20`, `D24`, `D25`
  and `D26` have landed.
