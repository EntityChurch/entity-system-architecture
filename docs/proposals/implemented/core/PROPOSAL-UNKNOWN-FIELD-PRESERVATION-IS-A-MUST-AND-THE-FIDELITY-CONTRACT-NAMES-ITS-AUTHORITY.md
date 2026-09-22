# PROPOSAL — unknown-field preservation is a MUST, and the fidelity contract names its authority

**Status:** IMPLEMENTED — folded at `ENTITY-CORE-PROTOCOL` **0.8.2.10** / `ENTITY-CBOR-ENCODING` **v1.6**.
All of D0–D7 landed; the fold-time L23 re-run (§2.3) added D7 and three enumeration rows, and found a
**fourth** stale `§2.7` citation the enumeration named two of. §6's open items 2 and 3 are **not**
closed by this fold and are carried on the board rather than in this file.
**Tier:** core — a normative delta to `entity-core-protocol`, arch's repo.
**Target:** `specs/ENTITY-CORE-PROTOCOL.md` §1.8, §9.1 · `specs/ENTITY-CBOR-ENCODING.md` §5.4 ·
`specs/ENTITY-NATIVE-TYPE-SYSTEM.md` §8-evolution, §9.10 — plus a `0.8.2.x` fourth-component bump.

> **What this proposes, in one sentence.** One rule — *a peer preserves content it did not model* —
> is stated in **thirteen places at two different strengths**, and the document that declares itself
> the canonical home carries the weaker one; this raises both weak statements to MUST, corrects two
> dangling cross-references, and makes every restatement name its authority so the next sweep is a
> grep.

**No wire change. No new field. No new behaviour required of any conformant peer** — every
implementation already does the strong form, because the weak form breaks content addressing and the
conformance corpus would catch it. **This is a spec-text defect, and its cost is to the next
implementer rather than to the current cohort.**

**Found by:** `EXPLORATION-EVOLVABLE-SCHEMAS-AND-THE-VOCABULARY-PROBLEM` §3, during the 2026-09-04
landscape read. The external prior art is why it was noticed at all: **many Thrift implementations
historically *dropped* unknown fields rather than preserving them, silently corrupting
read-modify-write proxies**, and proto3 retained them from 3.5 onward specifically to close that.
Reading `ENTITY-CBOR-ENCODING` §5.4 against that history is what surfaced the split.

**Measured at** `entity-core-protocol` **`221d8c3`** (2026-09-03), by opening every site listed in §2.

---

## §1 The defect

### §1.1 One rule, two strengths, and the canonical home has the weak one

| Site | Text | Strength |
|---|---|---|
| `ENTITY-CORE-PROTOCOL` **§2.10** | *"Unknown fields **MUST** be preserved on storage and forwarding."* | **MUST** |
| `ENTITY-NATIVE-TYPE-SYSTEM` **§2.4** | *"Unknown fields **MUST** be preserved."* | **MUST** |
| `ENTITY-CORE-PROTOCOL` **§1.8** step 5 | *"**Preserve unknown fields**: **SHOULD** preserve fields not understood."* | **SHOULD** |
| `ENTITY-CBOR-ENCODING` **§5.4** step 5 | *"**SHOULD** preserve unknown fields"* | **SHOULD** |

**§5.4 declares itself the authority:** *"This section is the canonical home of the entity-fidelity
contract… an implementation or validator citing any older location should cite
`ENTITY-CBOR-ENCODING.md` §5.4."*

**So the document that claims authority over the rule carries the weaker form, and the correctness
argument for the stronger form lives in a different document.** §2.10's argument is the one that
settles the strength: *"content hashing covers all of `{type, data}` including unknown fields… A peer
that strips or rejects unknown fields produces different hashes for the same entity, **breaking
content addressing for all downstream peers**."* **That is a correctness claim about the network, and
a correctness claim about the network does not belong under a SHOULD.**

### §1.2 §5.4 disagrees with itself, four lines apart

The numbered list says SHOULD. The governing paragraph immediately below states the property
unconditionally — *"all received content — known fields, unknown fields, CBOR tags, null-vs-absent —
survives the round-trip"* — and makes losslessness **precondition (b)** of the re-encode mechanism:
*"its parsed representation is lossless at every nesting level, so the re-encode reproduces unknown
content."*

**Precondition (b) is where the rule actually binds**, and it binds at MUST strength while the item
it restates says SHOULD.

### §1.3 Why it is not vacuous — and precisely how far the exposure goes

Stated at its real width, because the tempting overstatement is *"we have a live interop bug"* and we
do not.

- **Under mechanism A** (store original bytes, forward original) **the SHOULD is close to vacuous.**
  Unknown fields survive because the bytes survive, whether or not the parser noticed them.
- **Under mechanism B** (lossless parse + canonical re-encode) **it is load-bearing**, and clause (b)
  is the only thing binding it. An implementer reading the numbered list — which is what implementers
  read; §5.4's own neighbouring note says so about a different rule — could choose mechanism B and
  treat losslessness as advisory. **That peer accepts an entity, re-encodes it without a field it did
  not model, and publishes a different content hash for what it believes is the same entity.**

**No cohort seat is known to do this**, and this proposal makes no claim about any implementation's
tree — it is a claim about spec text only.

### §1.4 The conformance floor lists the rule twice, at two strengths

`ENTITY-CORE-PROTOCOL` §9.1 *MUST Implement* carries **two separate rows**:

```
- Entity fidelity (§1.8)
...
- Unknown field preservation (§2.10)
```

**§9.1 pulls §1.8 into the conformance floor without resolving whether step 5's SHOULD is inside or
outside that floor**, and separately lists §2.10, which contains the MUST. Reading §9.1 alone, the
rule is a MUST; reading §1.8 alone, it is a SHOULD. **This is the site that makes the split
consequential**, because §9.1 is what a conformance author reads.

### §1.5 Two dangling cross-references

`ENTITY-NATIVE-TYPE-SYSTEM` cites **`ENTITY-CORE-PROTOCOL.md` §2.7** for *"the open type model"* at
two sites — line 858 (the schema-evolution section) and line 1355 (`delegation_caveats`). **§2.7 is
*Type Name Type*.** Open Types is **§2.10**. Verified by opening §2.7.

**The evolution-rules site is the load-bearing one**: it is the section telling an implementer what a
compatible schema change is, and its pointer to the mechanism that makes field-addition safe goes to
the wrong section.

---

## §2 The home enumeration — done by subject, whole-document, both repos

**L23's ratified enforcement point requires this before the delta is written**, by the rule's
**subject** (*what obligation exists about content a peer did not model*) rather than by its tokens,
across every `specs/` file in both repos plus every conformance/MUST list. **Thirteen sites.**

### §2.1 `entity-core-protocol` — normative, in scope for this proposal

| # | Site | Role | Action |
|---|---|---|---|
| 1 | `ENTITY-CORE-PROTOCOL` §1.8 step 5 | the fidelity list | **SHOULD → MUST**; name §5.4 as authority |
| 2 | `ENTITY-CORE-PROTOCOL` §2.10 | the rule + its rationale | **unchanged** — this is the correct statement |
| 3 | `ENTITY-CORE-PROTOCOL` §9.1 | conformance floor, two rows | **disambiguate** — one row, citing §2.10, with §1.8 named as the mechanism |
| 4 | `ENTITY-CBOR-ENCODING` §5.4 step 5 | the declared canonical home | **SHOULD → MUST** |
| 5 | `ENTITY-CBOR-ENCODING` §5.4 paragraph | property vs mechanism, precondition (b) | **unchanged** — already correct and already testable |
| 6 | `ENTITY-CBOR-ENCODING` §7.4 | CDDL `* tstr => any ; Unknown fields preserved` | **unchanged** — encoding-level, consistent |
| 7 | `ENTITY-NATIVE-TYPE-SYSTEM` §2.4 | the rule | **unchanged**; add the §2.10 authority pointer |
| 8 | `ENTITY-NATIVE-TYPE-SYSTEM` §1.x (l.91) | *"a peer that strips unknown fields is broken; a peer that doesn't validate types is merely Level 0"* | **unchanged** — the best sentence in either repo on this |
| 9 | `ENTITY-NATIVE-TYPE-SYSTEM` §15.2 | security carve-out — handlers validate only known fields | **unchanged** — this is the MUST-ignore invariant and it is correct |
| 10 | `ENTITY-NATIVE-TYPE-SYSTEM` §8-evolution (l.858) | *"open type model (`ENTITY-CORE-PROTOCOL.md` §2.7)"* | **§2.7 → §2.10** |
| 11 | `ENTITY-NATIVE-TYPE-SYSTEM` §9.10 (l.1355) | *"open type semantics (`ENTITY-CORE-PROTOCOL.md` §2.7)"* | **§2.7 → §2.10** |

### §2.2 `entity-system-architecture` — restatements, handled

| # | Site | Status |
|---|---|---|
| 12 | `specs/ENTITY-SYSTEM-REFERENCE.md` §14 items 6–7 | **FIXED 2026-09-04, direct (wording-only hygiene).** Was an unmarked restatement of both halves with **no authority named** — L23's fourth shape exactly. Now names §5.4 and §2.10 |
| 13 | `specs/SPECIFICATION-FORMAT.md` §-authoring | *"Open Types make adding a field cheap and make removing one impossible"* — the authoring consequence | **unchanged**, consistent |

**Also checked and consistent, no action:** `specs/extensions/EXTENSION-COMPUTE.md` (two sites, both
cite *V7 §2.10* — the correct section) · `specs/extensions/EXTENSION-CONTENT.md` (cites §1.8 for
receipt validation, which is the right half) · `guides/GUIDE-CONFORMANCE.md` (already repointed to
§5.4).

**Honest limit on this enumeration.** It was produced by a subject-keyed grep over both repos'
`specs/` and `guides/` plus §9.1, read at `221d8c3`. **It is not certified exhaustive** — a
restatement using none of *unknown / field / fidelity / open type / strip / preserve* would not have
been found, and L23's second shape is precisely that such a site exists and is load-bearing. **Whoever
folds this should re-run the sweep and say so.**

### §2.3 The re-run, at fold time — **three more sites, and two of them are L23's second shape exactly**

**Run at `entity-core-protocol` `87e1218` on axes the first pass names as uncovered** — *MUST-ignore ·
lossless · round-trip · verbatim · re-encode · unrecognized · normalize · discard · additional/extra
field* — whole-document, both repos.

| # | Site | Text | Why the first pass missed it | Action |
|---|---|---|---|---|
| **14** | `ENTITY-CBOR-ENCODING` **§4.6** *Format Evolution* | *"Unknown **formats** SHOULD be preserved when forwarding entities"* | **the noun is `format`, not `field`** | **SHOULD → MUST** (D7) |
| **15** | `ENTITY-CBOR-ENCODING` **§9.3** *Format handling* | *"Unknown **format codes** SHOULD be preserved when forwarding"* | same | **SHOULD → MUST** (D7) |
| **16** | `ENTITY-CBOR-ENCODING` **Appendix E** | *"Re-encode-and-compare implementations **MUST** additionally verify that round-tripping each `encode_equal` vector's `canonical` bytes … produces byte-identical output"* | discussed in §4 as the *gate*, never listed as a **home** | **unchanged** — already MUST; enumerated so the sweep is complete |

**Sites 14 and 15 are the defect this proposal exists to fix, surviving the fix.** They state the
rule's own subject — *what obligation exists about content a peer did not model, when forwarding* — at
**SHOULD**, in the **same document** as the canonical home, 100 and 380 lines from it. Had the fold
shipped as drafted, `ENTITY-CBOR-ENCODING` would carry the field arm at MUST and the format arm at
SHOULD, and the next reader would find a fresh contradiction where this one was closed. **This is
exactly L23's ratified enforcement point earning its keep: enumerate by the rule's SUBJECT, not by its
tokens — the token here is a different noun for the same obligation.**

**The correctness argument is the same one and is if anything sharper.** A `content_hash`'s format byte
is what tells a consumer *how to hash*; §4.5's digest width *follows* it. A peer that normalizes an
unknown format code on forward emits a reference that is unparseable or wrong, and — unlike a dropped
field, which yields a valid entity with a different hash — **a rewritten format code yields a
reference that resolves to nothing.** §5.4's precondition (b) already binds this at MUST for mechanism
B (*"lossless at every nesting level"*), so sites 14 and 15 are **weaker restatements of a rule the
canonical home already states more strongly** — which is precisely the D2/D5 shape, one noun over.

**And the format axis is a live evolution surface by ruling**, not a hypothetical: `SPECIFICATION-FORMAT`
§8.4.6 declines to make the floor mandatory for peers specifically so that *"`content_hash_format`
negotiation"* remains *"a live wire surface."* **A live negotiation surface whose unknown values may be
silently dropped on forward is not a negotiation surface.**

---

## §2a The case the spec does not name — and it is why a bare MUST would be wrong

`[operator, 2026-09-04, paraphrased: "publishing a different hash when you change a type is
reasonable — it's more about compatibility. If you want deduplication, we do have entity fidelity. If
you make a change in general, incoming types, you preserve, so you hash the same. Once you transform
it for whatever reason you justify, you get a new hash, you lose the dedup — but you're signing off
in a different form, a different identity."]`

**This is the framing the whole section was missing, and it changes the delta.** The spec today
describes **two** mechanisms for one act. There are **three acts**, and only the first two are in the
text:

| # | Act | What the hash does | Obligation |
|---|---|---|---|
| **A** | **Relay** — store original bytes, forward original | **unchanged** | preservation is **mandatory**; this is the pass-through path |
| **B** | **Re-encode** — lossless parse, canonical re-encode, same entity | **unchanged, and that is the point** | preservation is **mandatory**, and it is precondition (b) that makes it so |
| **C** | **Transform** — deliberately author a derived entity | **new hash, correctly** | **not a fidelity violation at all.** You are not forwarding someone's entity; you are publishing your own, under your own name, and you own the result |

**The defect is not that peers might drop fields. It is that the spec has no name for C**, so a flat
*"MUST preserve unknown fields"* reads as *"you may never transform an entity"*, which is false and
would be a worse rule than the SHOULD it replaces. **Conversely the SHOULD, read on the relay path,
licenses exactly the silent corruption the contract exists to prevent.** One strength cannot serve
both, because they are different acts — and the spec collapses them by describing everything as
*"forwarding."*

**What separates C from a fidelity violation is authorship, not bytes.** A transformed entity is a
new content hash; whoever publishes it signs it; **the original remains intact, still addressable by
its own hash, still carrying its own author's signature.** Nothing is lost except dedup against the
original — which is a *cost the transformer chose*, not damage done to anyone else's data. **That is
the sentence the corpus does not have anywhere**, and it is the one that makes the MUST safe.

**Two consequences worth stating, because they are the actual design content:**

1. **Dedup is the thing preservation buys, and it is measurable.** A peer that strips a field it did
   not model, then forwards, produces a *second* content hash for one entity: every downstream store
   now holds both, every mirror of it fails to match, and the original author's signature no longer
   verifies against the forwarded bytes. **That is the harm, stated concretely** — and §2.10's
   *"breaking content addressing for all downstream peers"* is the compressed form of it.
2. **The line between B and C is intent, and intent is not observable on the wire.** Which is
   precisely why B needs precondition (b) as a **testable** obligation and C needs a **signature**:
   a re-encoder claims *"this is still your entity"* and must prove it byte-for-byte; a transformer
   claims *"this is my entity, derived from yours"* and proves it by signing. **The two claims are
   distinguishable only by what the publisher does, so the spec must give each one a form.**

**`APP-CONVENTION-FEED` already ran into this and solved it at the application tier without naming
it.** A mirror *republishes bytes unmodified with the original author's signature* (act A), and a
quote or a revision is *a new entry referencing the old* (act C) — never an edited copy. **The
convention has both acts and the core spec has neither by name.**

---

## §3 The proposed delta

**D0 — `ENTITY-CBOR-ENCODING` §5.4 gains the three-act frame (§2a), and it lands *before* D1**, because
D1 is unsafe without it. A short paragraph naming **relay**, **re-encode** and **transform**, stating
that §5.4 binds the first two and that the third is not a fidelity question:

> **Three acts, and this section binds two of them.** *Relaying* an entity (store original, forward
> original) and *re-encoding* one (lossless parse, canonical re-encode) are both claims that **this is
> still the sender's entity**, and both MUST preserve every byte of meaning including fields the
> implementation does not model. *Transforming* an entity — deliberately authoring a derived one — is
> neither: it produces a **new content hash**, the publisher **signs it as their own**, the original
> remains intact and independently addressable, and the only thing lost is dedup against the original,
> which is a cost the transformer chose. **A transform is not a fidelity violation. Silently emitting
> a transform while claiming a relay is.**

**D1 — `ENTITY-CBOR-ENCODING` §5.4, item 5.** `SHOULD preserve unknown fields` → **`MUST preserve
unknown fields`**, scoped by D0 to the relay and re-encode acts. Rationale sentence appended: *content
hashing covers all of `{type, data}`, so a stripped field is a **different entity** — a second content
hash for one thing, which costs dedup at every downstream store and breaks the original author's
signature against the forwarded bytes (`ENTITY-CORE-PROTOCOL` §2.10).*

**D2 — `ENTITY-CORE-PROTOCOL` §1.8, item 5.** `SHOULD preserve fields not understood` → **`MUST
preserve fields not understood`**, and the section gains a one-line authority pointer:
*`ENTITY-CBOR-ENCODING` §5.4 is the canonical home of this contract; the list here is the short
form.* **(L23's fourth shape: a restatement names its authority.)**

**D3 — `ENTITY-CORE-PROTOCOL` §9.1.** Collapse the two rows to one, so the floor states the rule once
at one strength:

```
- Entity fidelity, including unknown-field preservation (§2.10; mechanism and its two
  conformant forms in ENTITY-CBOR-ENCODING §5.4, gated by Appendix E)
```

**D4 — `ENTITY-NATIVE-TYPE-SYSTEM`, two sites.** `ENTITY-CORE-PROTOCOL.md §2.7` → **`§2.10`** at the
schema-evolution site and the `delegation_caveats` site.

**D5 — `ENTITY-NATIVE-TYPE-SYSTEM` §2.4.** Append the authority pointer to §2.10, same reason as D2.

**D7 — `ENTITY-CBOR-ENCODING` §4.6 and §9.3 (added at fold time, per §2.3).** *"Unknown formats SHOULD
be preserved when forwarding entities"* and *"Unknown format codes SHOULD be preserved when
forwarding"* → **MUST**, each naming §5.4 as the authority, same reason as D2 and D5. **Scoped by D0
identically:** the obligation is on the relay and re-encode acts; a transform authors a new entity
under its own format and its own signature, and is not a fidelity violation.

**D6 — version.** One **fourth-component** bump on `ENTITY-CORE-PROTOCOL` (`0.8.2.x`). The first
three components are the operator's and are not touched (**L14**).

> **Note on D7's blast radius, so the fold text can state it.** D7 does not widen the audience or
> change the flag-day answer: `content_hash_format` negotiation already exists, the floor
> (`ecfv1-sha256`, code `0x00`) is REQUIRED of every peer and is unaffected, and no cohort seat is
> known to emit a non-floor format on the wire today. **What changes is what a conformant peer may do
> with one when it meets it** — which is a rule about the future of the negotiation surface, and the
> cheapest moment to state it is before anyone ships a second format.

---

## §3a The use case that makes this concrete — carrying somebody else's bytes

**Added after `EXPLORATION-THE-BRIDGE-…` §6.2, which is a stronger argument than the abstract dedup
one and was absent from the first draft.**

**The operator has named being a bridge to other systems a primary goal.** The cheapest and most
honest bridge mode is **carry**: hold a foreign object — an ATProto record, an SSB message, a Nostr
event — as an entity whose payload is **the foreign bytes, verbatim, with its native signature
preserved inside them**. Our content hash pins the bytes, so anyone who implements that system's
crypto can verify that system's signature, **now or in ten years**, and we can say *"these are exactly
the bytes X published"* without claiming our verifier checked it.

> **That mode requires holding content we did not model, exactly, forever, through arbitrary
> forwarding — and *everything we did not model* is precisely what a foreign object consists of.**

**So the fidelity contract is not only about our own entities: it is the property that makes this
substrate a credible archive of anyone else's.** Most systems cannot make that promise, because their
schemas are closed and their encoders normalize on ingest.

**Under the current SHOULD, a conformant peer may drop it.** A peer that relays a carried foreign
object and silently strips what it did not model produces bytes that **no longer verify under the
foreign signature** — the archive's whole value, gone, with a green conformance run. **Under the
proposed MUST it cannot.** And act **C** (§2a) is what keeps the rule from over-reaching: a bridge
that *translates* rather than carries is authoring its own entity and signs it, which is correct and
is explicitly not a fidelity violation.

---

## §4 What conformance already covers, and what it does not

**Covered.** `ENTITY-CBOR-ENCODING` Appendix E already makes precondition (b) testable:
re-encode-and-compare implementations MUST round-trip every `encode_equal` vector's canonical bytes
byte-identically. **That gate is the strong form, already.** So this proposal aligns the prose with a
gate that is already running — which is why no seat's behaviour changes.

**Also covered at the application tier, and it is the wrong way round.**
`PROPOSAL-APP-CONVENTION-FEED` **FEED-6** is *"an entry carrying an unknown field, mirrored, round-trips
byte-identical."* **A convention gates at MUST what the core spec states at SHOULD.** D1–D3 fix the
direction.

**Not covered, and named rather than fixed here.** There is no vector for *"a peer using mechanism B
drops a field it did not model, then publishes."* Appendix E tests the round-trip in isolation, not
the store-and-forward path with an unmodeled field. **Whether that is worth a vector is a question for
the seats**, not a blocker on this text change — and the honest note is that a spec-text alignment
with no new check is exactly the shape `spec census`'s `unobserved-must` row exists to surface.

---

## §5 Routing — who this reaches, and it is scoped by the diff

**L21's fourth and fifth shapes bind here**, so the relay is scoped by *what this fold changes*, not by
what each seat has shipped.

| Seat | Why they are in the audience |
|---|---|
| **`entity-core-go`** (core tier lead) | one consolidated packet; rust's and py's worklists as a relayable section inside it. **The diff is D1–D5 and the worklist is "confirm your mechanism choice and, if B, that clause (b) holds at every nesting level"** |
| **`entity-core-keystone`** | **pins arch documents by hash** in `spec-data/*/MANIFEST.md` — L21's second shape. `ENTITY-CBOR-ENCODING`, `ENTITY-CORE-PROTOCOL` and `ENTITY-NATIVE-TYPE-SYSTEM` all move, so three pinned inputs move. **Route in the same session as the fold** |
| **`entity-browser-rust`, `entity-workbench-go`** | app tier; FEED-6 is theirs and this makes the core rule agree with it. **Informational** — nothing they built changes |

**Is this a flag day (L21's third shape)?** **No.** The field was already read by everything — content
addressing enforces it on every hash comparison — so there is no dormant-field-goes-load-bearing
transition and no divergence unit. **Say so in the fold text**, since the absence of a flag day is
itself the information the seats need.

---

## §6 Open items

1. **The sweep in §2 is not certified exhaustive** (see the honest limit). Re-run before folding.
2. **Is a store-and-forward-with-unmodeled-field vector wanted?** §4. Arch's to ask, the seats' to
   answer.
3. **`ENTITY-NATIVE-TYPE-SYSTEM` §8's evolution rules are the natural home for a defaults question**
   the corpus does not answer: the field's own literature says **defaults are what make add/remove
   compatible**, and we have none — because a producer-materialized default changes the content hash
   while a consumer-materialized one does not. **That asymmetry is real, has no paragraph anywhere,
   and is out of scope here.** Filed so it is not lost:
   `EXPLORATION-EVOLVABLE-SCHEMAS-AND-THE-VOCABULARY-PROBLEM` §5.3.
