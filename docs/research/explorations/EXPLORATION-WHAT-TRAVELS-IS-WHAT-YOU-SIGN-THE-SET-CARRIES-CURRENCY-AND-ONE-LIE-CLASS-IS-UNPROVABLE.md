# EXPLORATION — what travels is what you sign: the set carries currency, and one lie class is unprovable

**Status:** Exploration (design record). Not a proposal, not normative. **Nothing is folded here.**

**The question this answers.** *"When something leaves the tree it was published in, what has to go with
it, and what does the recipient then know?"* — and the harder half: *"if I say a peer published a bad
entry, what is my proof, and how does that proof travel?"*

**Why it is not a restatement of the verification ladder.** The ladder
(`EXPLORATION-THE-VERIFICATION-LADDER-WHAT-A-PUBLISHER-SIGNS-AND-WHAT-A-READER-CAN-CHECK`) answers
*what can a reader establish for itself.* Every rung of it is about the holder. **This document adds the
axis the ladder does not have: does the evidence survive being handed on, and does it survive time?**
Those are different properties from *can I check it*, they are not ordered the way the rungs are, and
the corpus has never separated them. Two of the three conclusions below are consequences of rulings that
are already landed and were made for a different subject.

---

## §0 The result, in three tables

### 0.1 The rungs gain a column, and the column is not a cost

| Rung | What a holder establishes | Instrument | **Survives hand-off?** | **Survives time?** |
|---|---|---|---|---|
| **0** integrity | these are the bytes named | content hash | **yes** | **yes** |
| **1** authorship | who wrote them | detached per-entity signature | **yes** | **yes** |
| **2** publication | the author put it there | inclusion proof against a signed root | **yes** | ⛔ **NO — perishes in the recipient** |
| **2′** publication | the author asserted this binding | signed binding assertion | **yes** | **yes** |
| **3** currency | this is the latest | signed root `seq` | ⛔ **no, by nature** | n/a |
| **4** completeness | this is everything | none | — | — |

> ⭐⭐ **The new row is 2′, and the new fact is that rung 2's two instruments are not interchangeable.**
> They were priced as *heavier* versus *lighter*. They differ in **kind**: a carried root is a claim
> about a moment, and a reader who has seen a later moment from the same author is obliged to refuse
> it — that refusal is the anti-rollback floor, which is the whole reason a signed root is worth
> having. **So the instrument that makes rung 3 possible is the instrument that makes rung 2
> non-durable, and no amount of care separates them.** A binding assertion has no equivalent failure,
> because it makes no claim about *now*.

**The practical form:** *a carried snapshot is neither valid nor invalid on its own terms — its fate is
decided by the recipient's history, and one bundle handed to two readers gets two answers.* The correct
outcome for the refusing reader is **declined**, never *invalid*: neither the carrier nor the author has
a defect, and reporting it as a verification failure hands a publisher a defect they do not have. The
field case that produces it is an author's own second machine.

### 0.2 Three lie classes, three treatments — and only two of them are provable

| A party can lie about… | The lie is caught by | Cost of the lie to the reader | **Can the reader PROVE it to a third party?** |
|---|---|---|---|
| **content** — *"these bytes are H"* | the hash, locally, always | one wasted fetch | **not needed.** The lie cannot land, so there is nothing to prove |
| **authorship** — *"A wrote this"* | the author's detached signature | nothing, if the signature is present | ⭐ **YES, permanently.** The signature is the proof and it is transferable and eternal |
| **location** — *"A's content is at C"* | the hash, after a fetch that may fail | one wasted fetch | ⛔ **NO, and it cannot be made provable** |

> ⭐⭐⭐ **This is the answer to *"what is my proof that a peer published a bad entry?"* — and the answer
> splits.** **What a peer SAID is provable, permanently, by anyone holding two entities.** **What a peer
> DID when serving is not provable at all**, and it does not need to be: a serving failure is
> self-limiting, because the bytes either hash or they do not. There is no signed transcript of a fetch
> and there should not be one.
>
> ⇒ **Accusations in this system are about authored statements, never about service.** *"This peer
> published a false claim"* is a sentence with evidence behind it. *"This peer served me garbage"* is
> not, and is not worth a mechanism.
>
> ⭐ **And this retro-justifies a decision that was previously made on preference.** The locator design
> declines a reputation system and substitutes the reader's own adoption decisions. The stronger reason
> is now available: **the evidence a reputation system would need does not exist and cannot be
> manufactured.** You can prove what an issuer asserted; you cannot prove that following it failed. A
> score computed from unprovable inputs is a rumour with arithmetic on it.

### 0.3 What to sign — and the discriminator was ruled twice already, for other subjects

| The object | Sign it? | The rule it follows |
|---|---|---|
| **immutable authored content** | **per entity**, detached, at the constructable pointer | mutability: the unit of addressing is the unit of verification |
| **a member of a published set**, read only through that set | **the SET** carries it | a set signature proves *these are all of them*; a member signature proves only *this one is theirs* |
| **a member of a set that will be QUOTED OUT of its set** | **both** | a per-member signature buys exactly one thing — **isolated quotation by a party not carrying the set** |
| **a mutable coordinate** (a page map, a name, a pinned release) | **a binding assertion** per coordinate | the coordinate is the claim, and a signature proves *who*, never *where* or *when* |
| **the whole namespace** | the signed root, `seq` monotone | currency, and **currency only** |

> ⭐⭐ **Both halves of this were settled for other subjects and neither has been applied here.** The
> mutability discriminator was ruled for the social tier's entries. The set-versus-member question was
> ruled for a peer's transport profiles, with the reasoning imported from a two-decade production
> deployment of exactly that choice in the DNS security extensions, where a signature covers a record
> *set* and never a record. **Nothing about either argument is specific to the subject it was made
> for**, and the second one contains the sentence this document most needed: *a per-member signature's
> one purchase is isolated quotation.*

---

## §1 The fourth question, and why it is a column rather than a rung

The rungs are ordered because each is strictly more than the one below: bytes, then author, then
publication, then currency. **Transferability is not ordered that way** — rung 0 and rung 1 are fully
transferable, rung 2 is transferable in one of its two forms, rung 3 is transferable in none. A rung 5
would be a lie about the structure.

**What the fourth question actually is:** *verification* asks whether **I** may accept this.
**Attribution-for-the-record** asks whether **I can make a third party accept it** — including a third
party who does not trust me, and including one reading it a year from now.

Three consequences, and the second is the one that bites:

1. **Verification tolerates a missing signature; the record does not.** A reader can accept bytes on the
   hash alone and render them unattributed. A *quotation* with no signature is worthless: it proves
   only that somebody produced some bytes.
2. ⛔ **A verifier must separate *"X published this once"* from *"this is X's current state"*, and the
   implemented path does not.** Offering a reader an author's older tree is refused outright by the
   monotonicity floor — correctly, for the currency question, and **destructively for the record
   question**, because the same bytes are the only evidence that the author ever published the entry.
   *Nothing in the corpus tells an implementation these are two questions*, and a tidy implementation
   answers both with one comparison.
3. **Therefore the durable publication instrument is the binding assertion, not the proof against a
   root** — and it moves from *optional, decided by whoever builds the first reader* to **required by
   the accountability case**, which is a question the reader does not get to opt out of.

---

## §2 What it costs, and the coupling nobody had stated

**A detached signature is a constant.** Sixty-four bytes of signature, two hashes, a type string and the
encoding frame: **about 207 bytes and two stored objects, whatever it signs.** Not proportional.

| Signing… | Costs |
|---|---|
| a short social post (≈190 B) | **+110%** |
| a documentation page (≈30 KB) | **+0.7%** |
| a large published figure (≈476 KB) | **+0.04%** |

> **The only case where the ratio is alarming is the case where the absolute number is 207 bytes.** So
> *"can we afford to sign more?"* is not a byte question and never was. **Bytes are not the argument
> against signing broadly.**

⭐⭐ **The real cost is the object count, and it lands somewhere surprising.** Signing every entity
**doubles the number of stored objects** in a publication — and a publication is re-emitted by
**projecting a tree**, which in both independent app-tier implementations means projecting **the whole
archive** on every publish. Measured on a live publisher over a paged, per-entry-signed set: 16,000
entries → 500 pages → **32,501 trie keys, 66,640 files, 4.1 s**; and adding one entry moves **three to
four keys** while the publisher re-emits all 32,501.

⇒ ⭐⭐⭐ **Two open questions are one decision.** *"Should more entity classes be signed?"* and *"may a
publisher re-project incrementally?"* are not independent. Under a snapshot projector, broad signing
doubles a cost that is already linear in the whole archive, forever. Under an incremental projector it
is `O(depth)` and the question evaporates. **Nothing in the corpus says a publisher may keep its prior
trie and recompute only the touched path, and every implementation built the snapshot** — so the
affordability of the answer is currently decided by an omission nobody made on purpose.

**And this is why the decision cannot be per entity.** *Will this travel?* is a prediction, and
predictions are wrong in the expensive direction: an entity published as purely local becomes a
quotation the moment somebody cites it. ⇒ **the signing obligation belongs to the TYPE, declared by the
convention that owns it**, so that *"no signature here"* is a readable property of the type rather than
an accident of the publisher's mood. A corpus where some entities of one type are signed and others are
not trains every reader to ignore the distinction, which destroys the fail-closed rule that makes the
whole ladder mean anything.

---

## §3 The carriage gap — the signature exists, is discoverable, and nothing puts it on the wire

Three separate things must be true for authorship to survive a hand-off, and they are in three different
states.

| | State |
|---|---|
| **minted** — the signature exists at all | ⛔ **almost nowhere.** One entity class in one implementation |
| **discoverable** — a holder can find it without knowing a private convention | ✅ **landed**, as a constructable path keyed on `(signer, target)`; and the receive side of envelope carriage binds it on arrival |
| **collected** — something puts it into the hand-off | ⛔ **nothing does** |

**The collection gap has a cause, not an oversight:** signatures point *to* content and content never
points *to* signatures, so every closure walker is signature-blind **by construction**. Content bundling
needs a root; signature bundling needs a set of **expected signers**, because the path is keyed on the
pair. **The shipped precedent is a helper plus a per-surface obligation** — the authority-chain
collector, which does exactly this for credentials and is the reason a credential chain is verifiable
away from its issuer. **What is owed is the same shape for content, and over-inclusion is free**
(anything the recipient already holds deduplicates by hash on arrival).

> ⚠ **And there is a landed obligation whose two arms are not equally reachable.** A republished object
> **MUST** carry authorship evidence that survives detachment from the author's signed root — *a detached
> per-entity signature, **or** an inclusion proof.* **The second arm has no implementation anywhere** (no
> verb takes *(entity, root, path)* and answers *is this in that root*), and by §0.1 it would perish in
> the recipient even if it did. **A conformant-looking implementation can satisfy that MUST by naming the
> arm nothing can execute.** The obligation is right; the disjunction needs to say which arm is real.

---

## §4 Applied: what a published claim table has to sign, and one design change falls out

A claim table — a published set of *"these peers serve this subject"* records, held by readers, merged by
readers, and **continuously re-asserted because every claim carries an expiry** — is the sharpest test of
everything above, because it is the first object in the corpus whose steady state is *rewriting*.

**Three signatures, and §0.3 says what each is for:**

| | Signed by | Why |
|---|---|---|
| **each claim** | its issuer | it will be **quoted out of its set** — an aggregator carries other issuers' claims verbatim, and that is precisely the isolated-quotation case |
| **the issuer's own page / head** | the issuer | *these are all of my claims as of this sequence* — the only thing that detects a **dropped** claim |
| **an aggregator's page** | the aggregator | *this is what I gathered* — never a completeness claim about anybody else's set |

⛔ **And then the arithmetic bites.** A signature's storage key is the **target's** hash. A re-assertion
with a new expiry is **a new claim at a new key**, so it needs **a new signature**. At an aggregate
carrying 50,000 claims under a one-hour expiry that is 50,000 new claims and 50,000 new signatures per
hour — roughly **200,000 stored objects per hour, per aggregator, forever**, every one of them garbage
within the hour, in a system with **no specified collection mechanism** (the one extension the corpus
cites for it does not exist — five citations, no document).

> ⭐⭐⭐ **The design change: the expiry belongs to the SET, not to the claim.** A claim carries **when it
> was asserted** — an immutable fact, true forever, signed once, ever. The **page** carries the validity
> window and the sequence, is re-signed each cycle, and a reader's rule becomes *a claim is current iff
> it appears in a page that is current.* Re-assertion then re-signs **one page**, not N claims.
>
> **Three independent arguments land on the same shape.** ① It is the set-versus-member ruling applied
> unchanged. ② A content-addressed claim key **cannot** be a freshness handle — a renewed claim is a
> different hash at a different key — so **the page is already the only currency unit available**, and a
> reader polling a claim key will hold expired claims forever while believing itself current. ③ Putting a
> validity window inside an immutable content-addressed entity is the category error the mutability
> discriminator exists to prevent: an expiry is a property of a *mutable pointer*, and it was written
> into the immutable half.
>
> **What this does not give up:** the claim is still *a statement about the present*, because its
> presence in a current page is what makes it one. What it gives up is the ability to hold a claim
> **out** of any set and still reason about its freshness — which §0.1 says is not a real ability.

**A second consequence, worth stating because it is free:** declaring the aggregate to be a
republication in the closure sense — rather than minting a second set-layer format for it — buys the
signature-carriage obligation and the paging obligation **already stated at that tier**, both of which
this object needs and neither of which a new format would inherit.

---

## §5 What this does not settle

- **Which entity classes get the type-level signing obligation.** §2 says the decision is per type and
  gives the affordability gate; it does not enumerate the types. Measured today: of the five
  application-tier conventions, **one** carries the detached-signature discipline and **three mention the
  signature type zero times.**
- **Whether the incremental re-projection permission is a MAY or a MUST.** §2 shows the answer decides
  the cost of everything else. It is not decided here.
- **The binding assertion's sequence field.** Monotonicity per coordinate is what separates *asserted* from
  *replayable*, and it is per-coordinate state the tree does not keep — one counter and logarithmic proofs,
  or N counters and constant-size ones. §1 raises the accountability argument for the assertion; it does
  not price the counter.
- **Whether a reader ever demands publication evidence on ordinary social content.** Still empirical, still
  answered by a client that does not exist yet, and this document must not be read as settling it.

## §6 Sources

**Opened for this document, by section.** The core protocol's signature model, invariant pointer and
four signature properties, its discovery-locality rule, and its envelope-arrival signature ingestion ·
the tree extension's published-root and inclusion-proof sections · the substitute extension's trust
contract and chain algorithm · the data-exchange closure document's author-anchored-evidence,
four-rules and growth-rule sections with its full requirement table · the registry extension's binding
entity · the five application conventions, counted for signature discipline · the verification ladder
and certificate-transparency explorations and the settled-conclusion register.

**Measured elsewhere and used here as measurement**, with the measuring party's own bound kept: the
constant signature size and object-count ratios; the projection curve and the moved-key counts; the
absence of any inclusion-proof verb across the three reference implementations; the two arms of the
carried-snapshot outcome. **A cost measured on one payload shape does not transfer to another** — the
byte figures in §2 are the measurer's payloads, and what transfers is the object count and the curve
shape, which are properties of the projection rather than of the payload.
