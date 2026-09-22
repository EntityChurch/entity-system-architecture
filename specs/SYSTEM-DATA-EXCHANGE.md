# System Data Exchange — closure, and what a peer may republish

**Version**: 0.3
**Status**: DRAFT — the closure chapter and the growth rule. The subject, source, witness, position,
intent and authority layers are specified but not yet folded; they arrive in later revisions of this
document.
**Depends**: ENTITY-CORE-PROTOCOL.md (v7.20+) §1.2, §1.4, §3.5 · EXTENSION-TREE.md (v4.8+) §1, §3.3a ·
EXTENSION-SIGNALING.md (signature binding at the invariant pointer) · APP-CONVENTION-FEED.md (v0.2+) §1.1, §6
**Encoding**: ENTITY-CBOR-ENCODING.md (ECF)
**Tier**: system composition — an emergent property no single extension owns

---

> **Path notation.** Paths use peer-relative notation (without leading `/{peer_id}/`). Every path in
> the entity tree is absolute at rest, rooted at a peer identity. See `ENTITY-CORE-PROTOCOL.md` §1.4.

This document specifies one property: **what a peer obtains from other peers, it may publish, and the
result is the same kind of object it was built from.** The property is called **closure**, it is what
allows an aggregating peer to be aggregated in turn, and it belongs here rather than in any one
extension because no single extension can state it — it is a fact about the relationship between what
a peer reads and what a peer writes.

**Who this binds.** Any peer that republishes content authored by another peer: a gathered view of a
conversation, a republished timeline, a cached set of foreign artifacts, a participant index. **The
rules below were previously stated only in an application convention.** They are promoted here because
the property they protect is claimed at this tier, and a second convention shipping a gathered view
would otherwise inherit the property without inheriting the rules that make it true.

---

## 1. The closure property

### 1.1 The statement

> **[MUST]** What a peer obtains through a data-exchange mechanism, it **MAY** publish. A published
> result is the **same kind of object** as the sources it was built from — signed entries in the
> publishing peer's own namespace — and **MUST** be **verified, attributed and rendered by the
> identical code path**.

> **[MUST]** The closure is over the **aggregator's output**. A gatherer's result **MUST** be
> consumable by another gatherer with no new type and no new code path. **The fixed point is
> `gathered → gathered`.**

### 1.2 Which layer is closed — and the distinction is load-bearing

**Closure binds the entry layer. It does not bind the walk.**

| layer | closed? |
|---|---|
| **entry** — decode, obtain the author's detached signature, verify, attribute, render | **YES.** One code path. An implementation **MUST NOT** have a second, weaker path for republished entries |
| **set** — how a reader finds the entries | **NO.** An author's own set and a gatherer's gathered set are different shapes, and they are allowed to be |

> **[MUST NOT]** An implementation **MUST NOT** publish, under its own namespace, an author's own
> set-layer object over content that author did not place there.

**Why the prohibition is here rather than implied.** Stated without the layer distinction, *"consumable
by the identical code path"* reads as a requirement that a gathered set have the same shape as an
authored one — which would oblige a gatherer to publish the author's index shape under the gatherer's
own namespace. That is the gatherer asserting a claim only the author can make, and because the
authorship instrument of §2 signs **entries** and not sets, nothing would authenticate it. *A reader
following the unqualified sentence builds a forgery and every byte of it verifies.*

⇒ **The property that federation actually depends on is that the OUTPUT carries the same authorship
evidence the INPUT did** — which is the entry layer, and which is exactly what §2 pins.

### 1.3 Why closure is the design and not a convenience

If a peer's **output** is a different type from its **input**, aggregation cannot be aggregated: one
party ends up holding what everyone needs, and there is exactly one of them.

| an aggregator whose output is | can the next layer consume it? |
|---|---|
| an API | **no** |
| a query response | **no** |
| a private database | **no** |
| **signed entries, exactly like a source's** | **yes — and that is the difference** |

⇒ **A mechanism closed under its own output has no privileged tier to centralize into.** A peer
subscribes to an aggregating peer the same way it subscribes to a person; an aggregator can aggregate
aggregators; one code path verifies and renders both, because at the entry layer they are one type.

**And the degenerate case is the same mechanism at zero infrastructure.** A reader that trusts nobody
performs the gathering itself over the peers it already follows. The spectrum from *no infrastructure*
to *a well-resourced public index* has no discontinuity and no protocol change in it — only how much
work a peer chose to do.

### 1.4 Closure is a capability, not a permission

> **[MUST]** Closure states what the mechanism **can** carry. **Whether a given object may be
> republished is that object's own disposition**, and the mechanism neither grants nor withholds it.

*Stated because §1.1 will be read alone. Read alone, an unqualified "what a peer obtains, it may
publish" licenses republishing a message from a closed conversation.*

---

## 2. The two preconditions

**Closure does not hold by itself. It holds under two conditions, and an implementation that omits
either produces a result that passes every integrity check while having lost the property.**

### 2.1 Byte preservation

> **[MUST]** A republished entity **MUST** be bound **byte-identically** to the form in which it was
> obtained. An implementation **MUST NOT** re-encode it — **including by decoding it through a type it
> does not fully declare.**

**The failure is silent and it is not a lost field.** An entity's address is the hash of its bytes. A
detached signature binds at a pointer derived from that hash. Move the hash and **the signature no
longer names the entity**, so a conformant renderer must present every republished entry as
**unattributed**.

⇒ **A gatherer that re-encodes produces a complete, verifiable, correctly-walked publication in which
nobody wrote anything.** That is the centralization failure in a different costume: the output stopped
being the same kind of object, because it lost its authorship.

⚠ **The realistic path to this defect is not carelessness — it is the ordinary shape of the code.**
Republishing by decoding a body into a local structure and binding it back is what a developer writes
first, and it is lossless *only* over types the implementation fully declares. **A gatherer aggregates
types it did not write**, so the case that breaks it is the normal case, and it raises no error at any
layer.

**[MUST]** An implementation **MUST** provide, and republication **MUST** use, an operation that binds
**obtained bytes** rather than **data**. *The ordinary `put(path, type, data)` shape is the one a
developer reaches for and it is the broken one.*

#### 2.1.1 Testing it — the fixture requirement, which is where this is actually lost

> **[MUST]** A check exercising §2.1 **MUST** use input the implementation's own encoder would not
> emit.

**A round trip through bytes your own encoder produced proves nothing**, because decode-and-re-encode
is lossless over exactly that input. **A closure gate built from self-generated fixtures stays green
with byte preservation removed** — and it is then worse than no gate, because it is counted. The input
that measures the requirement is legal content the implementation does not fully declare: an entry
carrying a field this build has never heard of, which the core protocol obliges it to ignore and
preserve.

### 2.2 Author-anchored evidence

> **[MUST]** An object republished through this mechanism **MUST** carry evidence of authorship that
> **survives detachment from the author's signed root** — a detached per-entity signature at the
> invariant pointer `/{signer_peer_id}/system/signature/{target_hash_hex}`, or an inclusion proof.

> **[MUST]** A republished object with **neither** carries integrity without authorship and **MUST
> NOT** be presented as attributed.

**Why detachment is the property that matters.** A signature over an author's root proves what the
author published *as of that root*, and it is unavailable to a reader holding the entry from someone
else. A signature at the invariant pointer travels with the entity: a canonical peer id is a commitment
to its public key, so such a signature verifies **with no key distribution and no second fetch**. That
is why it survives being carried by a third party, and it is why closure can rest on it.

#### 2.2.1 Where it is bound, and where a third party looks

**Both rules below restate `ENTITY-CORE-PROTOCOL` §1.4 — the URI and path model, the local view, and
its cross-peer worked example — at the tier that needs them. They are stated here, and named as
restatements, because an implementation reading only this section derived the pointer under the wrong
peer and concluded the corpus was silent.**
`ENTITY-CORE-PROTOCOL.md` §1.4 is the normative home for both rules below.

> **[MUST]** A republishing peer **MUST** bind the evidence at the pointer's own absolute path —
> `/{signer_peer_id}/system/signature/{target_hash_hex}` — **in its own local view**, unchanged. The
> path is rooted at the **signer**, never at the republisher: it is the same path the evidence
> occupies on the author's own peer, and it is the same path on every peer that carries it.

> **[MUST]** A reader obtaining a republished object from a third party **MUST** resolve that pointer
> **against the peer serving the object**, by naming that peer's tree handler and the signer-rooted
> path as the resource. A reader **MUST NOT** derive the pointer under the serving peer's identifier,
> and an implementation **MUST NOT** re-qualify the already-absolute pointer to any other namespace.

**Neither rule introduces an operation, a field or a route.** The peer being asked and the resource
being asked about are already separate: the first is the handler URI of the request, the second is an
absolute path rooted at the namespace it belongs to. Asking a peer what it holds under another peer's
namespace is the ordinary read, and the answer is that peer's own view of it — authoritative only for
the key holder, which is what the signature settles.

**The re-qualification prohibition is not a new constraint either.** Re-qualifying a path that is
already absolute is the prepend-local defect `ENTITY-CORE-PROTOCOL` §1.4 names as the most-recurring
cross-implementation bug class, and it is guarded by the `universal_address_space` conformance
category. It is restated here because a signature-locating helper is exactly the *"handler-internal
function that takes a path"* that rule already covers, and because its two failures here — deriving
under the peer being read from, and prepending the serving peer — look like two bugs and are one.

### 2.3 The four rules a republishing peer follows

**Promoted from `APP-CONVENTION-FEED` §6.1, where they were application-scoped. The instrument they
depend on is corpus-wide; the obligation was not.**

1. **[MUST]** **Republish the original bytes** — not a re-encoding, not a re-normalization, not a
   re-serialization through a local model (§2.1).
2. **[MUST]** **A republished entry travels with its author's detached signature** (§2.2). A
   republishing peer **MUST** carry one whenever the entry it republishes has one obtainable, and
   **MUST NOT** supply one: it does not hold the author's key and cannot author in the author's name.

   > **The two halves are not in tension — *carry what the author signed; never sign in their name*.**
   > Carrying costs a republisher nothing it has not already done: it obtained the entry through a
   > verifying consumer, so it resolved the signature in order to verify it, and binding it per §2.2.1
   > is binding bytes already in hand. **The conditional is load-bearing and is not a softening** —
   > rule 3 and the `DX-C5` check require an entry whose signature is genuinely unobtainable to be
   > *carried and rendered unattributed*, never dropped, so an unconditional obligation here would
   > oblige exactly the drop those forbid.
3. **[MUST]** **Attribution follows the entry's own author field, verified against that signature,
   always.** A renderer that attributes a republished entry to the republishing peer is
   **non-conformant**. One holding an entry whose signature is absent **MUST** present it as
   **unattributed** rather than attributing it to anyone.
4. **[MUST NOT]** A republishing peer **MUST NOT** claim completeness, and a republication format
   **MUST NOT** provide a field in which to claim it. **The verifiable property is narrower and more
   useful: a republished set can OMIT but never SUBSTITUTE.**

> **Rule 2 and rule 3 answer *"is republishing someone else's content legitimate?"* structurally
> rather than as a norm.** You do not hold their key; you cannot author in their name; you cannot alter
> a byte without the signature failing. All you can do is carry what they already chose to publish —
> which is what publishing has always meant, and which this makes **verifiable** rather than merely
> customary.

### 2.4 What omission costs, and why it is invisible

**A convention that ships a gathered view, inherits closure, and does not carry §2.3's rules renders
correctly-verified entries attributed to whoever handed them over. No gate sees it, because the bytes
check out.** *This is the reason the rules live at this tier: the failure is not in the convention that
forgot them, it is in the next convention that never knew about them.*

### 2.5 The growth rule — a gathered set grows, and the format has to survive that

> **[MUST]** A republication format whose membership **grows with participation** **MUST NOT** carry its
> members as an unbounded collection in a single entity. It **MUST** be a **bounded head plus
> key-addressed pages**. Pages **MUST NOT** be renumbered, merged or compacted, and a reader **MUST NOT**
> assume any page size.

> **[MUST]** A specification introducing a member collection **MUST** state which side of the test below
> it falls on.

**The test, and it is one question.** *Does this collection grow with how many parties participate, or
with something the format's author controls?* If the first, it is unbounded and it is paged. An author's
own curated menu is bounded by the author; a gathered view of a growing set is bounded by nothing.

**Why this is at this tier and not in the convention that needs it.** **The cost is the reader's, and it
is invisible from the format that causes it.** A flat list is correct, verifiable, cheap to write, and
passes every check at the size its author tested — it becomes a defect only in somebody else's fetch, at
a size no fixture has. So the party positioned to notice is never the party that wrote it.

⇒ **This is §2.4's failure one axis over.** There, a convention inherits closure and loses **authorship**;
here it inherits closure and loses **readability at scale**. Both are silent, both pass every integrity
check, and both are properties of the relationship between a writer and a reader that no single
convention can state about itself.

> **A page size is deliberately not specified.** The right value depends on member size and publication
> cadence, which vary by orders of magnitude between publishers. **What is normative is the shape, not
> the arithmetic.**

> **The reason the paging shape and not a cap.** A cap on the collection makes a large view
> *unpublishable*; paging makes it *incrementally readable*, which is the property a source leg actually
> needs — a reader reads down from the head to the position it already holds and stops. **A format that
> can only be read whole cannot be read cheaply by the second reader, which is the reader republication
> exists to serve.**

---

## 3. Cross-Implementation Conformance

**Requirement id prefix:** `DX`. **Requirement ids are `DX-Rn`; check ids are `DX-Cn`.**

### 3.1 Requirements

| id | Requirement | Level | § |
|---|---|---|---|
| `DX-R1` | A published aggregate is signed entries in the publishing peer's own namespace | MUST | §1.1 |
| `DX-R2` | Republished entries are verified, attributed and rendered by the same code path as directly-obtained ones | MUST | §1.1, §1.2 |
| `DX-R3` | A gatherer's output is consumable by another gatherer with no new type | MUST | §1.1 |
| `DX-R4` | Publish an author's own set-layer object over content that author did not place there | MUST NOT | §1.2 |
| `DX-R5` | Treat closure as authorization to republish an object whose disposition withholds it | MUST NOT | §1.4 |
| `DX-R6` | Bind a republished entity byte-identically to the form obtained | MUST | §2.1 |
| `DX-R7` | Re-encode a republished entity, including by decoding through a partially-declared type | MUST NOT | §2.1 |
| `DX-R8` | Provide and use an operation binding obtained bytes rather than data | MUST | §2.1 |
| `DX-R9` | Exercise §2.1 with input the implementation's own encoder would not emit | MUST | §2.1.1 |
| `DX-R10` | Carry authorship evidence that survives detachment from the author's signed root | MUST | §2.2 |
| `DX-R11` | Present a republished object with no surviving authorship evidence as attributed | MUST NOT | §2.2, §2.3 |
| `DX-R12` | Supply a detached signature for a republished entry the peer did not author | MUST NOT | §2.3 |
| `DX-R13` | Attribute a republished entry to the republishing peer | MUST NOT | §2.3 |
| `DX-R14` | Provide a field in which a republication claims completeness | MUST NOT | §2.3 |
| `DX-R15` | Carry a participation-grown member collection as an unbounded collection in a single entity | MUST NOT | §2.5 |
| `DX-R16` | Carry such a collection as a bounded head plus key-addressed pages, never renumbered, merged or compacted | MUST | §2.5 |
| `DX-R17` | State which side of §2.5's test a member collection falls on | MUST | §2.5 |
| `DX-R18` | Bind a republished entry's authorship evidence at the signer-rooted invariant pointer in the republisher's own local view, and carry it whenever it is obtainable | MUST | §2.2.1, §2.3 |
| `DX-R19` | Derive or re-qualify that pointer under the serving peer's identifier rather than the signer's | MUST NOT | §2.2.1 |

### 3.2 Required checks — what an implementation must discriminate

| id | The check | Drives | What fails without it |
|---|---|---|---|
| `DX-C1` | Publish at A → republish at B → consume at C. **Every entity hash byte-identical at both hops**, every entry attributed to **A**, and C's consumer is the same code path it uses for a direct read, **with C's reads directed at B — a check in which C can reach A does not measure this** | `DX-R1`, `DX-R2`, `DX-R6`, `DX-R10`, `DX-R13` | closure is asserted and never run |
| `DX-C2` | The same path, with republication **re-encoding through a type that does not declare one of A's fields**. The hashes **MUST** move and the result **MUST NOT** render attributed | `DX-R6`, `DX-R7`, `DX-R8`, `DX-R11` | ⭐ **the naive republish ships silently** — a complete, verifiable publication in which nobody wrote anything |
| `DX-C2a` | **Anti-vacuity guard on `DX-C2`:** assert the fixture is not what this encoder emits — normalizing it **moves the hash** — before asserting anything about republication | `DX-R9` | ⭐⭐ **`DX-C2` passes with byte preservation removed**, and is then worse than no check because it is counted |
| `DX-C3` | B republishes B′, which republished A. Authorship survives **both** hops and B′'s output needed no new type at B | `DX-R3` | the fixed point — aggregating an aggregator |
| `DX-C4` | A republication that **substitutes** a body is refused; one that **omits** an entry is short and valid | `DX-R14` | omit-but-never-substitute, **in both directions** — a one-direction check passes a format that cannot be short |
| `DX-C5` | An entry whose detached signature is unobtainable is **carried** and rendered **unattributed** — not dropped, and not attributed to the republisher | `DX-R11`, `DX-R12` | **integrity without authorship**, which is invisible to every check that only compares hashes |
| `DX-C6` | The gathering peer's tree is inspected: it holds **no binding presenting as the author's own set-layer object** over that author's content | `DX-R4` | a gatherer forging the author's index — and every byte of it verifies |
| `DX-C7` | An object whose disposition withholds republication is obtained and **is not republished** | `DX-R5` | closure read as a permission |
| `DX-C9` | The third hop of `DX-C1`, **with the consumer reading only from the serving peer and never contacting the author** — assert the serving peer *holds* the evidence at the signer-rooted path, and that the consumer *resolves it there*. Run a **control arm** reading the same entries directly from the author | `DX-R10`, `DX-R18`, `DX-R19` | ⭐⭐ **a mirror that carries integrity without authorship** — every hash matches, every reference is right, the view is addressable and complete, and nothing in it is attributable |
| `DX-C8` | A republished view is grown **past one page** and read by a second party, which fetches the head and **one page** and stops. The head's size **MUST NOT** be a function of the member count, and a sealed page's bytes **MUST NOT** move when the view is extended | `DX-R15`, `DX-R16` | ⭐ **a view that is correct, verifiable and unreadable** — the cheap leg becomes the expensive one, and no single-publisher fixture can see it |

⚠ **`DX-C8` needs a view larger than one page, which is the whole difficulty** — a check built at the
size its author tested passes against a flat list and measures nothing. **The anti-vacuity arm is the
second page**: assert the view spans more than one before asserting anything about reading it.

⚠ **`DX-C9`'s anti-vacuity arm is the control read, for the same reason `DX-C2a` is a row of its own.**
A third-hop measurement with no direct-read arm is a statement about the harness rather than about
mirrors. **A consumer that can still reach the author passes `DX-C1` while measuring nothing**, which
is how a third hop goes unrun while the check is counted as covered.

⚠ **`DX-C2` is the check this document exists for, and it is the one most likely to be built wrong** —
which is why `DX-C2a` is a separate row rather than a note inside it. It has a documented history of
passing while measuring nothing, and the fixture obligation is normative (`DX-R9`).

**No test vectors are declared here.** Vectors are a byproduct of implementations running against one
another; this section states what a check set must discriminate, which is the input to that
convergence and not a substitute for it.

---

## 4. Open — specified, not yet folded

The data-exchange model specifies five further layers — **subject**, **source** (with its outcome
taxonomy), **witness**, **position** and **intent** — plus a three-value authority axis and a single
normative entry point. **None of them is in this document yet**, and a reader should not infer from
their absence that they are unspecified. This revision folds the closure chapter only, because it is
the precondition every other layer rests on and the one an implementation needs before it republishes
anything.

---

## Document History

**v0.3:** adds **§2.2.1** — where a republisher binds authorship evidence, and where a third party
resolves it. Both rules restate the core protocol's URI and path model at this tier and name it as
their authority; neither introduces an operation, a field or a route. §2.3 rule 2's `MAY carry`
becomes `MUST carry when obtainable`, resolving a disagreement between that clause and the bolded
obligation above it: a republisher declining to carry evidence it already resolved emits a view every
reader is then obliged to render unattributed — complete, verifiable, and authored by nobody. Adds
`DX-R18`, `DX-R19` and check `DX-C9`, and pins `DX-C1`'s third hop to a consumer that cannot reach the
author, because a hop measured against a reachable author measures nothing. Additive: no landed
requirement is renumbered.

**v0.2:** adds **§2.5, the growth rule** — a republication format whose membership grows with
participation carries its members as a bounded head plus key-addressed pages, never as one unbounded
collection. Promoted here for the same reason §2.3's four rules were: the cost is the **reader's** and
is invisible from the format that causes it, so a convention inherits closure without inheriting what
makes a closed object still readable when it is large. The corpus had stated the rule twice, in two
application conventions neither citing the other, and a third convention made the mistake anyway.
