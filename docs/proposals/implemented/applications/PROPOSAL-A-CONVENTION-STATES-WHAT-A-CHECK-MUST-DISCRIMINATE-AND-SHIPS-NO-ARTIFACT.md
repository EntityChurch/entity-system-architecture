# PROPOSAL — a convention states what a check must discriminate; it ships no artifact, and it is not the party that runs one

**Status:** IMPLEMENTED (2026-09-09) — **FOLDED.** `specs/applications/CHARTER.md` **v1.1** carries the
rewritten #5 and #7's re-attributed enforcement point, and the Members table gains a reading note plus a
re-cut status line per member. All five members' conformance sections are re-titled **"Required checks —
what an implementation must discriminate"** with the ownership stated in the preamble; `EMBED` and
`SEMANTIC-CONTENT-SITE` also lose *"cut joint conformance vectors"* from their header `Next:` lines.
**No table row, id, level or case was edited** — §4's claim held on inspection.
**Two corrections found while folding:** the charter recorded `APP-CONVENTION-REFERENCE` as owing **ten**
checks where §6.2 names **eleven**, and `#7`'s enforcement point (landed the previous day) was resting on
the artifact this proposal reassigns, so it was rewritten in the same pass.
**Target:** `specs/applications/CHARTER.md` discipline **#5** (rewritten) and **#7** (its enforcement
point re-attributed) · the `Members` table · and the conformance section of all five members —
`APP-CONVENTION-EMBED` §9 · `APP-CONVENTION-SEMANTIC-CONTENT-SITE` §9 ·
`APP-CONVENTION-SHARE` §8 · `APP-CONVENTION-REFERENCE` §6.2 · `APP-CONVENTION-FEED` §11.2.
**Class:** a correction. It restores one tier to a division of labour the rest of the corpus already
states normatively; it introduces nothing.
**Provenance:** the same rule stated in three earlier places, none of which reached this domain's
charter — see §2.

---

## 1. The defect

`CHARTER.md` discipline **#5** currently reads:

> **Ships conformance vectors.** Because a convention is a byte-level cross-impl contract, it is not
> validated until vectors exercise it (the PRIMER meta-rule). Each convention ships example entities +
> expected hashes.

Two separate errors are packed into it, and the second is the expensive one.

**① It assigns the artifact to the wrong party.** *"Each convention ships example entities + expected
hashes"* directs the author of a specification to produce fixtures and to compute digests by hand.
That is the one thing the conformance guide forbids in the plainest language it uses anywhere
(`GUIDE-CONFORMANCE` §5.1a, a `[MUST]`): **architecture sets fields, the encoder settles bytes.** A
specification says which assertion a check makes and over what input; the bytes are a build artifact
with a build owner, and that owner is not the specification.

**② It makes that misassigned artifact the ratification gate.** Because #5 is phrased as something the
convention *ships*, a convention that has not shipped one is recorded as **not ratifiable** — and five
of five members now carry that line. The domain reads as blocked on a backlog, when what is actually
true is far simpler and not a backlog at all: **the conventions are authored and no implementation has
exercised them yet.** Those are different states with different owners, and only one of them is work
anybody here can discharge.

**The compounding effect is a sequencing inversion.** #5 asks for the check set to be fixed *before*
any implementation exists to disagree with it. The corpus's own experience runs the other way: two
implementations exchanging a format discover the divergences that matter, and several of the sharpest
requirements this tier holds were found that way and could not have been guessed in advance. **A check
set is pinned after the exchange works, not before it** — pinning early does not accelerate the
exchange, it just fixes somebody's guess as the standard.

## 2. This is not a new rule. It is a rule that never reached this domain.

Three landed statements already say it, and the charter was written past all three.

| Where | What it says |
|---|---|
| `GUIDE-CONFORMANCE` **§5.1a** `[MUST]` | *Architecture edits the source: vector ids, descriptions, kinds, inputs, and which assertion a vector makes. Architecture does NOT hand-derive, hand-edit, or hand-inspect encoded bytes.* |
| `GUIDE-CONFORMANCE` **§7.0** | Four different things are called a "vector," and the table's **Authored by** column assigns each one. Application-tier format checks are not on it at all — which is the gap this proposal closes. |
| `GUIDE-CONFORMANCE` **§7c.5 / §7c.6** | The split, stated twice for the compute corpus: *architecture authors the guidance, the shape and the discipline; the implementations build, emit and cross-bless.* And: **the expected outcome is architecture's; the boundary hashes are not** — *"a hash computed by hand would make architecture the oracle."* |

There is also a worked precedent at the extension tier. A specification once shipped a literal key, two
endpoint strings and two digests, presented as *"the oracle in the spec."* It was **retracted the same
day**, and the retraction's reasoning is exactly this proposal's: *a digest typed into prose is
unreviewable by construction — nobody can check it by reading, which is how a wrong one survives to be
banked as a false green by three implementations at once.* The section was rewritten to state the
**properties** the check must have and to author none of its bytes. (`PROPOSAL-REGISTRY-PEER-ISSUED-REGISTRATION`
§7.)

**So the finding worth recording is not "#5 is wrong."** It is that **a rule already ruled, already
published, and already worked through on a live retraction did not reach the one tier that was newest**
— and that tier then built five specifications on top of it and made it a gate. A ruling that lands in
one document is not a ruling that has landed. It has to be swept into every home the rule is stated
in, and this domain's charter was a home nobody enumerated.

## 3. The rewrite

Discipline **#5** becomes:

> 5. **States what a check must discriminate — and ships no artifact.** A convention is a byte-level
>    cross-impl contract, and it is not *validated* until conformance checks exercise it. The
>    convention's own obligation is to name the cases that must be discriminated, what each one
>    asserts, and what fails if it is missing. **It does not author the fixtures, compute the digests,
>    build the harness, or run the suite** — those are the implementations' and the conformance
>    oracle's, per `GUIDE-CONFORMANCE` §5.1a and §7.0, and a fixture authored by one implementation
>    cannot adjudicate its own operand. **A convention states properties; an implementation produces
>    bytes.**
>
>    **And the set is pinned after the exchange works, not before it.** Two implementations trading the
>    format are a better oracle than a set written in advance by a party with nothing to run it
>    against. A convention names the cases it can already see are load-bearing; the set is closed when
>    the exchange has been made and its divergences are known.

Discipline **#7**'s enforcement point loses its unowned artifact and gains an owner:

> ***Enforcement point:*** the type tags a convention pins are greppable — and a shape change fails
> **cross-impl comparison**, which is what makes it discoverable at all. The comparison is the
> implementations', not this document's; what the convention owes is that the case is **named** in its
> conformance section, so the comparison exists to be run.

## 4. What does not change

- **The property tables stay, exactly as written.** All five members' conformance sections enumerate
  cases, what each drives, and what fails without it — none of them authors a byte or a digest, so
  every one of them is already on the right side of §5.1a. They are the deliverable, not a placeholder
  for one.
- **The requirement inventories stay.** `<PREFIX>-R<n>` rows and their levels are unaffected.
- **The meta-rule stays.** A normative claim about bytes is not validated until a cross-implementation
  check exercises it, hostilely. That is why the cases are named. It was never a claim about who runs
  them.
- **The core tier is untouched.** §7.0's assignment of the encoding fixture corpus, §7d's host-seam
  reference handler and §7c's compute corpus keep their existing owners; each already scopes
  architecture to fields, inputs and expected outcomes, which is the same boundary this proposal
  applies one tier up.

## 5. The consequence for the five members

Every member's status line stops reporting an artifact the author does not produce, and starts
reporting the state that is actually true. **"Authored; not yet exercised"** is a complete and honest
description of a specification that no implementation has yet built against — and it is a state that
resolves by somebody building, not by somebody writing.

The conformance sections are re-titled from *"Vectors — OWED, NOT SHIPPED"* to **"Required checks —
what an implementation must discriminate,"** and their preamble states the owner. No row moves, no id
changes, no table is edited.

## 6. Open

- **Ratification.** A convention ratifies when independent implementations have exchanged its format
  and the named cases have been exercised. Nothing here shortens that; it relocates who is waiting on
  whom. Whether a convention may ratify on *one* exchanged pair, or needs the full set, is the
  ratification bar and is not settled by this proposal.
