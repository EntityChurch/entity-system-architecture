# PROPOSAL — a publish commits to the evidence, or the subject is not attributable; and one landed rule cannot tell two absences apart

**Proposes:** a **publishing** obligation in `APP-CONVENTION-FEED` §1.1 — a publisher that offers entries
as attributable MUST commit, in the published root's key set, to the detached signatures it holds for
them — plus the reader-side `MUST NOT` that makes the existing attribution rule falsifiable. **Nothing is
renamed and no wire format changes.** The addition is one obligation on a party the convention currently
does not address at all.

**Status:** **DRAFT 2026-09-16 · revision 2.** *(revision 1: 2026-09-15.)*

> ### ⭐ What revision 2 changes, and it is the part revision 1 got wrong
>
> Revision 1 closed §3.3 with *"the change is to the publish **scope**, not to the grant"* — a publisher
> publishes over its own peer root instead of over `app/feed/`. **An implementation built exactly that and
> measured what else it committed to: the committed key set went from 4 keys to 386, and the static emit
> from 7 entities to 400 across 379 paths — putting a peer operator's filesystem path, another peer's
> local network address and an unshared document's body into the directory that gets uploaded to a CDN.**
>
> **The objection revision 1 answered was about the GRANT, and that answer holds.** The one it did not
> reach is about the **static road, which has no grant at all** — there the disclosure control *is* the
> published prefix, which is the thing revision 1 moved.
>
> ⛔ **But the three answers the finding offered — a prefix SET, an implicit commitment to derived keys,
> or accepting the disclosure — are all answers to a question that rests on a false premise.** §3.4 is the
> ruling: **the prefix is a BOUND on the publication, and the binding set is a SEPARATE INPUT.** Widening
> the prefix does not widen the content, and the landed substrate says so in terms. **Both halves of the
> fix are already shipped — by different implementations** (§3.4.1).
**Depends:** `APP-CONVENTION-FEED.md` §1.1, §1.1.1, §4.2, §6.0a, §11.1 (`FEED-R2`, `FEED-R4`) ·
`ENTITY-CORE-PROTOCOL.md` §3.5 (the invariant pointer and its four properties; the discovery-locality
rule) · `EXTENSION-TREE.md` §3.3a (the declared published prefix) ·
`APP-CONVENTION-SEMANTIC-CONTENT-SITE.md` §7 (the adjacent grant obligation)
**Kind**: proposal · **Authority**: non-binding until folded · **Governed-by**: `SPECIFICATION-FORMAT.md`
**Audience**: application-convention authors and the implementers of feed publishers and readers.

---

## §0 Summary

| | |
|---|---|
| **The gap** | **`FEED-R2` binds the COMPOSER at authoring time. Nothing binds the PUBLISHER.** So whether an entry's signature is reachable from a published artifact is currently an implementation choice, and two conformant implementations make it differently |
| ⭐ **Why it is a defect and not a documentation gap** | **`FEED-R4` is unfalsifiable as written.** It says present an entry whose signature is absent as *unattributed*. **Absent has two causes** — *the author never signed* and *the publisher did not commit it* — and a reader cannot distinguish them. ⇒ a fully conformant reader makes **a false statement about an author**, manufactured by the publisher's own scoping choice |
| **What is new** | one `[MUST]` on the publisher · one `[MUST NOT]` on the reader · one sentence naming the mirror case, which is genuinely different |
| ⛔ **What is NOT needed** | **no wire change, no widened grant, and — revision 2 — no widened CONTENT.** The signature path is already derivable from what the index pins; the prefix that contains both the entries and the signatures is the publisher's own peer namespace, not the universal tree; and **widening that prefix does not widen what is published**, because `prefix` is a bound and the binding set is a separate input (§3.4) |
| **Cost** | one subsection in §1.1 plus §1.1.3a, seven conformance rows, three required checks, and one split sentence in §4.3 rule 6. **Reversible; no format moves** |

---

## §1 The three obligations, and the corpus names two

A feed entry is attributable to its author end-to-end only if **three** things hold. They bind three
different parties, and only two are written down.

| | obligation | party | where it is specified |
|---|---|---|---|
| **1** | the detached signature is **minted**, at authoring time, per entry once | whatever **composes** the entry | ✅ §1.1 `FEED-R2`, scoped explicitly by §1.1.1 |
| **2** | the signature is **reachable** by a reader that is authorized to verify the subject | whoever writes the **grant** | ✅ specified one convention over — `APP-CONVENTION-SEMANTIC-CONTENT-SITE` §7's two sentences, and that shape is not site-specific |
| **3** | ⛔ the signature is **present in the published artifact** — the publish commits to it | whoever **publishes** | **unspecified** |

**Obligation 3 is the one this proposal adds**, and the ordering matters: §7's grant rule fixes *who may
fetch the evidence*; this fixes *whether the evidence is in the artifact at all*. A grant cannot make
reachable a key the published root does not name.

### 1.1 The measurement, and it is a divergence rather than a bug

**Two independent implementations publish feeds. Both are conformant with every landed rule. They
disagree on this, and the disagreement is invisible from either side alone.**

- **One commits the signature.** Its root projector records every binding written **under the
  publishing peer's own namespace**, and its publisher signs an entry it authored, so
  `system/signature/{hex(entry_hash)}` lands in the committed key set. An ordinary walk resolves it.
  Measured against a real published tree: every entry carries the attributed verdict; the falsifier
  that never consults the per-entry signature takes the count to zero.
- **One does not.** Its publish commits over the `app/feed/` prefix. The signature key is not in that
  prefix, so it is absent from the committed key set — measured directly against the walk result of a
  real signed root. A **live** reader obtains it anyway, over a separate grant resource; a **static**
  reader has no second channel, so every entry arrives unattributable.

⇒ **Neither implementation is wrong against the text, because the text does not say.** That is the
signature of a missing normative sentence rather than of an implementation defect, and it is the case
the convention's own cross-peer discipline exists for: **a property that holds locally and for the
issuer, and springs apart at a consumer seam.**

---

## §2 ⭐ Why this is the defect and not a nice-to-have: `FEED-R4` cannot see its own trigger

`FEED-R4` is a `MUST`: *present an entry whose signature is absent as unattributed, rather than
attributing it.* It exists for a real and permanent condition — §1.1.1 states that entries authored
before an implementation adopted `FEED-R2` **can never be attributed**, and that this is a boundary in
the data rather than a migration window.

**But with obligation 3 unspecified, "absent" is ambiguous at the reader, and the two causes are not
alike:**

| the reader observes | cause A | cause B |
|---|---|---|
| no signature at the constructable path | **the author never signed it** — §1.1.1's permanent boundary. Presenting it as unattributed is correct and is the whole point of the rule | **the publisher did not commit it.** The author signed. Presenting it as unattributed is a **false statement about the author** |

⇒ **A rule that cannot distinguish its own trigger from a third party's scoping choice is satisfiable by
a correct reader publishing a falsehood.** This is the same failure the adjacent convention's §7
`MUST NOT` was written for — *an implementation MUST NOT report an authorization failure as an absent
published root, because that is a false statement about the publisher manufactured by the reader's own
grant.* **Here the manufacturing party is the publisher rather than the reader, and the falsehood is
about the author rather than the publisher.** Same shape, one step earlier in the chain.

**Adding obligation 3 is what makes `FEED-R4` mean something:** once a publisher is obliged to commit
every signature it holds, *absent* recovers its single intended meaning.

---

## §3 The two alternatives, and why each contradicts landed text

Three answers are available. Two are wrong against sections that already exist, which is what makes this
a derivation rather than a preference.

### 3.1 ⛔ "Attribution is live-only; a static feed is explicitly unattributable"

**Rejected — it inverts §1.1's stated reason for choosing this mechanism.** §1.1 selects the detached
signature over the inclusion proof precisely because *an entry that travels alone should verify alone,
in `O(1)` extra objects — without its author's tree, without their origin, and without a root that may be
many publishes stale.* Making attribution conditional on the live transport makes **the form chosen for
travelling the one that does not travel.**

It also contradicts §1.1.1 directly: the obligation is the composer's, **at authoring time**, which is
transport-independent by construction. A convention cannot bind a party at authoring time and then make
the result conditional on a delivery choice made later by someone else.

### 3.2 ⛔ "Carry the signature hash in the index page rows"

**Rejected — it is a wire change that buys nothing, and it duplicates a derivable value.**

The index already pins each entry. The signature's location is `hex(entry_hash)` under the author's
namespace — **a core-general constructable path** (`ENTITY-CORE-PROTOCOL` §3.5), which that section's own
discovery-locality rule names as strategy (B) and which requires no extension knowledge. **A reader
holding an index row can already compute where the signature is.** What it cannot do is conjure bytes
that the publish did not commit.

⇒ adding the hash to the row stores a second copy of a derived value, which can desynchronize from the
value it is derived from and which this convention has already ruled against in the neighbouring case:
`FEED-R26` forbids a `subject` pin to a value that moves, and `FEED-R27` forbids optional hint fields in a
key derivation. **The wire is not missing anything. The publish is.**

### 3.3 ✅ The publisher commits to the evidence

The remaining answer is the correct one, and it is already shipped by one implementation.

⭐ **And the objection against it does not survive measurement.** The objection is that *no prefix
contains both the entries and the signatures except the universal tree, and a public grant over the
universal tree is a wildcard in a different spelling.* **The prefix that contains both is the publisher's
own peer namespace** — `{peer}/app/feed/…` and `{peer}/system/signature/…` are both subpaths of
`{peer}/`. That is not the universal tree, it is one peer's subtree, and it is exactly the scope
`ENTITY-CORE-PROTOCOL` §3.5 designs for when it places each signer's signatures **in that signer's own
namespace**.

⇒ **the change is to the publish SCOPE, not to the grant.** A publisher publishing over its own peer
root, rather than over `app/feed/`, commits to both with no wildcard anywhere and no grant widened. A
reader's grant remains two named resources — the same two the adjacent convention's §7 already requires
for verifying a site.

> ⚠ **Revision 2: that paragraph is true and incomplete, and the incompleteness is expensive.** It says
> *publish over the peer root* without saying what a publish over the peer root **contains** — and the
> reference helper available to a publisher answers that question by scanning everything the peer holds
> under the prefix. §3.4 supplies the missing half.

---

## §3.4 ⭐ THE RULING — the prefix is a bound, the binding set is an input, and the corpus already says so

**The finding asks: does a published root commit to ONE prefix or to a SET of them?** It offers three
answers — **(A)** a root commits to a prefix set; **(B)** a prefix implicitly also commits to the derived
`system/signature/` keys of what it binds; **(C)** accept the disclosure and make it an explicit act of
consent by the peer operator.

**All three answer a question that does not arise, because the premise under it is false against landed
text.** The premise is *widening the prefix widens what is published.* It is not.

### 3.4.1 What the substrate actually says

`EXTENSION-TREE` §3.3a is unambiguous and states this three times, in three different registers:

- **The bound.** *"`prefix` bounds the publication's scope; it is NOT a completeness claim. It says
  nothing under this trie lies outside `prefix`. It does not say every binding the publisher holds under
  `prefix` is in the trie, and a consumer MUST NOT read it that way."*
- **The permission, in terms.** *"A publisher MAY declare `/{peer_id}/` and publish a small subset; that
  is a narrow claim honestly made, not a false one."*
- **The negative-scoping `[MUST]`.** A negative from a published root means *"this root does not bind
  K"* — **never** *"the publisher does not bind K"* — and *"a publisher MAY serve different subsets to
  different audiences from the same prefix."*

**And the opposite rule was tried and withdrawn in full.** §3.3a carries the record: a completeness
`[MUST]` — *"a publisher MUST publish completely under `prefix`"* — landed in v4.1 and was revoked on
four independent grounds, one of them structural (*"an anchor cannot sit inside the tree it anchors"*).
⇒ **The one rule that would make the premise true is the one rule the substrate has already refuted.**

⇒ **The answer is (D), and it is not a compromise between the three:** the publish moves to the peer-root
**prefix**, so the signature keys are expressible as relative keys under a single declared prefix — which
is `A-36` unchanged and is what the attribution fix requires — **and the binding set stays exactly the
feed's own entries, the index, and the signatures held for them.** The disclosure is not a cost of the
ruling. It is a cost of deriving the content set from the prefix.

### 3.4.2 Both halves are already shipped, by different implementations — which is what makes this a missing sentence

**This is the second measured divergence in this document and it is sharper than §1.1's**, because
neither side is doing anything unusual:

| | how it selects what to publish | consequence |
|---|---|---|
| **one implementation** | **accumulates.** Its projector records each entity *the projection just wrote*, keyed under the publishing peer, and builds the trie from that accumulation. Its own comment states the invariant: a root signed by this key must commit only to keys under this peer, *"or the declared §3.3a prefix is a lie the consumer reconstructs confidently"* | ✅ peer-root prefix, curated content. **Attribution works and nothing private travels** |
| **the other** | **scans.** It calls the core helper that collects *all bindings under the prefix from the location index* and builds the trie from those | ⛔ peer-root prefix ⇒ everything the peer holds. 4 keys → 386 |

**Neither is non-conformant, because the convention never said which — and the scanning one is not a
lapse.** It is the only thing the reference publish helper does: the core tier exposes a
prefix-scanning trie builder on the publish path, and the accumulating form is reachable only by calling
the lower-level builder that takes an explicit binding list directly. **The helper that is easy to reach
implements the reading this ruling rejects.** ⇒ *the seat that hit the disclosure hit it by using the
substrate as shipped, which is the strongest possible case that the sentence is missing rather than that
an implementation is wrong.*

### 3.4.3 Why (A), (B) and (C) are each rejected, on landed text

| | | why not |
|---|---|---|
| **(A)** | a root commits to a prefix **SET** | ⛔ **Contradicts the landed schema and breaks reconstruction.** `published-root.prefix` is a single `system/tree/path`, REQUIRED, and §3.3's reconstruction is `absolute_prefix + relative_key` — **one operand.** A set makes a `relative_key` ambiguous across members, which is the defect `prefix` was added to close. It is also a schema change to a type three core peers implement, **to buy a property (D) already has** |
| **(B)** | a prefix implicitly commits to the derived `system/signature/` keys of what it binds | ⛔ **Implicit commitment is not commitment.** The trie's key set is the signed artifact; a key not in it is not committed to, and a consumer reaching it would be **trusting a path outside the hash chain** — precisely the threat model the walk-from-signed-root exists to defeat. It also breaks the byte-identical cross-implementation invariant: the trie would no longer be a function of the binding set |
| **(C)** | accept the scope; make the disclosure an explicit consent step | ⛔ **Accepts a cost the rules do not impose.** And it is the worst of the three in one specific way: **a consumer is already forbidden to read a prefix as a completeness claim**, so a publisher scanning the prefix volunteers data *no reader is entitled to infer from the prefix anyway.* Pure loss. ⚠ **A consent surface is still right for a publisher that genuinely wants a wide set — it is just not the answer to this** |

⚠ **`APP-CONVENTION-SHARE` §366 reconciles cleanly and was checked before this was written.** It says
`published-root` *"stays singular and is the verification anchor, not a share catalog."* **That is the
same field under different pressure and it points the same way**: (A) would make the anchor a catalog of
prefixes, and (D) leaves it singular. Nothing in §366 constrains what the trie under that anchor
contains.

---

## §4 The proposed text

### 4.1 New — `APP-CONVENTION-FEED` §1.1.3, *The publisher commits to the evidence*

> **[MUST]** A publisher that publishes entries as attributable **MUST** commit, in the published root's
> key set, to the `system/signature` entity of every entry it publishes **for which it holds one**. A
> publish whose committed key set omits a held signature **MUST NOT** be presented as an attributable
> feed.
>
> **[MUST NOT]** A publisher **MUST NOT** rely on a channel outside the published artifact to supply
> evidence for a subject it offers as independently verifiable. A live reader obtaining the signature
> over a separate resource is a **property of that transport**, never a discharge of this obligation.
>
> **This is a publishing obligation and it does not move §1.1.1's authoring one.** A publisher that does
> not hold a signature — because the author never minted one (§1.1.1's permanent boundary) — publishes
> the entry and **MUST NOT** synthesize, omit or substitute anything for it; the reader applies
> `FEED-R4`.

#### 4.1a ⭐ New — the clause revision 2 adds, and without it §4.1 is a disclosure instruction

> **[MUST]** The published root's `prefix` and its **binding set** are **two separate decisions**, and a
> publisher **MUST NOT** derive the second from the first. `prefix` bounds the publication
> (`EXTENSION-TREE` §3.3a); **it is not an instruction to publish everything under it**, and that section
> forbids a consumer from reading it as one.
>
> **[MUST]** A feed publish's binding set **MUST** be the **narrowest set that satisfies §1.1.3** — the
> entries offered, the index head and pages that address them, and the `system/signature` entities held
> for those entries — **and MUST NOT** include a binding merely because it falls under the declared
> prefix.
>
> **Stated as the thing an implementer gets wrong:** widening `prefix` from `app/feed/` to the peer root
> is **required** by §1.1.3, and widening the *content* to match is **forbidden** by this clause. They
> are not the same operation, and a publish helper that takes only a prefix cannot express the
> difference.

### 4.2 New — the mirror case, which is genuinely different and must be said

> **A mirror cannot satisfy the above by committing, and MUST carry instead.** A mirrored entry's
> signature lives in **its author's** namespace, which the mirroring publisher's root cannot commit to —
> a root signed by one key commits only to keys under that peer, or the declared prefix
> (`EXTENSION-TREE` §3.3a) is false. ⇒ **[MUST]** a mirror **MUST** carry each mirrored entry's signature
> as carried content alongside the entry, and **MUST NOT** publish a mirrored entry as attributable by
> reference to a root it does not control.

*(This is not new engineering: one implementation already carries a mirror's signature as a body for
exactly this reason, and states it. The proposal is recording the rule the practice already follows.)*

### 4.3 Amended — `FEED-R4`'s reader-side companion

> **[MUST NOT]** A reader **MUST NOT** present an entry as *authored without a signature* where the
> signature is merely **not present in the artifact it read**. Where a reader can distinguish the two, it
> **MUST**; where it cannot, *unattributed* is the required presentation and the reader **MUST NOT**
> characterize the cause.

### 4.3a Does §1.1.3 want a ROAD qualifier? — no, and the correction that prompted the question is accepted

**The question, as filed:** the obligation is trivially met on the live road — a reader's grant names the
signature resource separately, measured — and expensive on the static one, and the proposed text names
neither road.

✅ **The correction inside it is accepted and is a defect in revision 1's reasoning.** §3.2 rejected the
carry-the-hash-in-the-row candidate partly on the ground that *"a reader can already compute where the
signature is."* **That is true of the PATH and it is not an answer about reachability** — a path outside
the committed prefix is unfetchable on a trie-addressed corridor however exactly it is computed.
**That candidate stays rejected, and it is now rejected on `FEED-R26` alone** — a derived value stored a
second time can desynchronize — with the reachability clause withdrawn from the argument.

⛔ **But the obligation itself takes no road qualifier, and qualifying it would be the §3.1 error one
level down.** §1.1 chose the detached signature so that *an entry that travels alone verifies alone*. A
publishing obligation that binds only on the road where it is expensive is a rule that switches off
exactly where the artifact it protects is least recoverable — and the static road is the one with no
second channel. **The obligation is uniform; what differs by road is the cost of discharging it, and
after §3.4 that cost is one binding per entry on both.**

⚠ **One road distinction IS real and belongs in the text — but it is about the live root, not the
obligation.** A peer's continuously-tracked root on the live road is scanned from the prefix by design and
is reached only through a grant, which is where disclosure is controlled. **§4.1a's narrowest-set rule
binds the STATIC publish — the artifact whose bytes are emitted to a directory — and does not oblige a
peer to narrow its live tracked root.** Saying so is the difference between a clause implementers can
satisfy and one that reads as forbidding the live tier's normal operation.

### 4.3b The same clause has a second under-specification, filed independently by the other implementation

**`FEED-R13` / §4.3 rule 6 says:** *"The index is an optimization and MUST NOT be treated as the
authority. A reader that cannot fetch it falls back to enumerating the prefix — slower, same answer."*

**Two obligations are wedged into one sentence and only one of them is implementable on both roads.**

| | | |
|---|---|---|
| ✅ **the floor** | an entry absent from the index is still valid, and a reader encountering one by reference **MUST NOT** reject it for being unlisted | a `MUST NOT` on the reader. **Implementable everywhere**, and it is what the rule's own rationale is about — closing the publisher's lie-by-omission |
| ⛔ **the fallback** | *"falls back to enumerating the prefix — slower, same answer"* | **names a prefix the convention never defines** (§2: *"the cross-impl contract is the type tag, not the path"*), so it must mean a type-filtered query — **and a static origin is a file layout with no query endpoint.** There the index is not a slow path, it is the only path |

⇒ **Two corrections, and the second is forced by §3.4 whether or not the first is accepted.**

> **Proposed, `APP-CONVENTION-FEED` §4.3 rule 6.** Split the sentence. The `[MUST NOT]` floor stands
> unchanged and unqualified. The fallback becomes **a capability of a transport that has one, not an
> obligation of the convention**: *a reader that cannot fetch the index MAY recover the set by
> type-filtered query where its transport offers one; **a reader with no such facility MUST report the
> index as unreachable and MUST NOT present a short view as a complete one.***

⭐ **And *"same answer"* is false independently of the road, which is what makes this more than
wording.** `EXTENSION-TREE` §3.3a's negative-scoping `[MUST]` says a negative from a published root is
scoped to *that root* and never to the publisher — and **§4.1a makes the published binding set narrower
than the peer's holdings by design.** ⇒ enumeration over a published artifact returns *what this root
committed to*, which is **not** the same answer as the index and must not be described as one.
**A reader that treats a short enumeration as complete makes exactly the false statement §2 of this
proposal exists to stop, one mechanism over.**

> ⚠ **This clause is separable and is offered as separable.** It is filed here because §4.1a is what
> makes *"same answer"* provably wrong, and ruling them apart would leave the two halves of one sentence
> ruled by two documents. **If it is preferred as its own proposal, nothing above depends on it.**

### 4.4 Conformance rows

| id | requirement | level | §ref |
|---|---|---|---|
| `FEED-R34` | Commit, in the published root's key set, to the `system/signature` of every published entry for which the publisher holds one | MUST | §1.1.3 |
| `FEED-R35` | Present a publish omitting a held signature as an attributable feed | MUST NOT | §1.1.3 |
| `FEED-R36` | Carry a mirrored entry's signature as carried content rather than by reference to a root the mirror does not control | MUST | §1.1.3 |
| `FEED-R37` | Characterize an absent-from-artifact signature as an unsigned entry | MUST NOT | §1.1.3, §6.1 r3 |
| `FEED-R38` | Derive a publish's binding set from its declared `prefix` | MUST NOT | §1.1.3a |
| `FEED-R39` | Publish the narrowest binding set that satisfies `FEED-R34` — entries, index, held signatures | MUST | §1.1.3a |
| `FEED-R40` | Present a short enumeration of a published artifact as a complete view of the feed | MUST NOT | §4.3 r6 |

### 4.5 The required check, and it is one that already nearly exists

| check | what it must discriminate |
|---|---|
| `FEED-14` | a feed published over a prefix that **excludes** the signature location, read by a **static** reader with no second channel ⇒ the reader reaches zero attributed entries, **and the publisher's own publish is non-conformant before the reader is consulted.** ⭐ The anti-vacuity arm is the same feed published over the peer root: the same reader reaches every entry attributed. **A single-prefix fixture passes against both rules and measures neither** |
| `FEED-15` ⭐ | **the two decisions are independent.** A peer holding bindings outside the feed publishes at `prefix: "/{peer_id}/"` ⇒ every entry is attributed **and** the committed key set contains **no** binding outside the entries, the index and their signatures. ⚠ **The anti-vacuity arm is the one that makes this a check rather than a formality: the same peer, seeded with private bindings under the same prefix, must produce the same committed key set** — a fixture whose peer holds nothing else passes `FEED-R38` by having nothing to leak, and measures nothing |
| `FEED-16` | **a reader with no type-filtered query, against an artifact whose index is unreachable** ⇒ reports the index as unreachable; **MUST NOT** report a complete feed and **MUST NOT** report an empty one. The anti-vacuity arm is the same reader against a reachable index, which must report the full set — *the cheapest way to satisfy a `MUST NOT` is to delete the honest answer* |

---

## §5 Scope — what this does NOT do

1. **It does not generalize, yet.** The same three-obligation structure applies to any convention whose
   artifacts are published with evidence outside the subject's own prefix — the adjacent site convention
   is the obvious second instance, since a site's verifying root signature also sits outside the site
   subgraph by design. **That generalization is named here as a candidate and is deliberately not
   proposed**: it is ruled on one convention where it is measured, and a second instance is what would
   earn the promotion. A convention-independent restatement belongs wherever a publisher's obligations
   live for **any** convention, and that document does not exist.
2. **It does not touch grants.** §7's grant obligation one convention over is unchanged and is not
   widened. This proposal's whole content is the publish scope.
3. **It does not touch the wire.** No entity gains a field; no key derivation changes.
4. **It does not resolve the ordering question.** §6.0a's *gather order is not post order* cost is
   untouched and unrelated.

---

## §6 Open questions

| | |
|---|---|
| **1** | **Should `FEED-R34` say "holds one" or "the author minted one"?** *Holds* is what a publisher can check; *minted* is the property that matters and is unobservable to a publisher that did not author. **Leaning `holds`** — a MUST an obliged party cannot evaluate is satisfiable by assertion |
| **2** | **Is the mirror rule (§4.2) a restatement or a change?** One implementation already does it, for the stated reason. If both do, this is recognition; if not, it is a change and wants its own round |
| **3** | ⚠ **Does the publisher-side MUST belong in this convention at all, or in the publishing mechanism beneath it?** The obligation is about what a published root commits to, which is substrate. **Leaning: state it here where it is measured, and let the second instance force the generalization** — a rule hoisted to a layer on one instance is a taxonomy invented in advance. ⭐ **Revision 2 sharpens this and does not close it:** §4.1a is *more* substrate-shaped than §1.1.3, because *"the binding set is not derived from the prefix"* is true of every publisher of anything. **It is stated here because here is where it is measured**, and §3.4.1 shows the substrate section it belongs to already implies it — `EXTENSION-TREE` §3.3a says the prefix is not a completeness claim, and stops one sentence short of saying a publisher must not treat it as one |
| **4** | ⭐ **NEW — is the reference publish helper the right place to fix this instead?** §3.4.2's scanning implementation is conformant-as-shipped and hit the disclosure by using the substrate as given: the easy-to-reach helper takes a prefix, and the explicit-binding-list builder is a layer down. **A normative sentence that every publisher must satisfy, against a helper whose only mode violates it, is a rule with a known non-compliance path.** This is a finding for the core tier and is not this proposal's to rule — **it is named here so the clause does not land looking cost-free** |

---

## §7 What this revision establishes, and what it does not

✅ **Establishes:** that three obligations exist and the corpus names two (§1) · that the divergence is
measured on two conformant implementations and is a missing sentence rather than a bug (§1.1) ·
⭐ **that the existing attribution `MUST` is unfalsifiable without this one, and is satisfiable by a
correct reader publishing a falsehood about an author** (§2) · that the two alternative answers each
contradict landed text, with the sections named (§3) · ⭐ **that the objection about prefix scope does not
hold — the containing prefix is the publisher's own peer namespace, not the universal tree** (§3.3) ·
the proposed text, four conformance rows and the discriminating check (§4).

**Revision 2 adds to *establishes*:** ⭐ **that the prefix/content premise under the whole prefix-or-set
question is false against `EXTENSION-TREE` §3.3a, which states it three ways and records the withdrawal
of the one rule that would make it true** (§3.4.1) · that the divergence is measured on two conformant
publishers, **one accumulating and one scanning**, with the scanning one using the reference helper as
shipped (§3.4.2) · that (A), (B) and (C) each fail on landed text, and that `APP-CONVENTION-SHARE` §366
points the same way (§3.4.3) · the narrowest-set `[MUST]` and its two conformance rows (§4.1a) ·
⭐ **that `FEED-R13` rule 6's *"same answer"* is false independently of the road, and is made provably
false by §4.1a** (§4.3b) · that the road qualifier is declined, with the reachability half of §3.2's
argument withdrawn as the filing correctly demanded (§4.3a).

⛔ **Does NOT establish:** the generalization past this one convention (§5.1) · whether the mirror rule is
a change or a restatement (§6.2) · the correct layer for the publisher obligation long-term (§6.3) ·
⛔ **whether the reference publish helper should grow an explicit-binding-set mode** (§6.4) — a core-tier
question, named and not ruled · ⛔ **that the narrowest-set rule is free for the scanning publisher**: it
is one call lower in the same library, but that is a reading of the substrate and **not a measurement of
their build**, and the seat that owns it re-measures before anyone records it as cheap.

⚠ **And it is deliberately not folded.** It is a normative addition to a convention that two
implementations are actively building against, and the cheapest moment to be corrected is before the
text lands. **The shape is what is offered to build against; the fold follows the first cross-publisher
run.**
