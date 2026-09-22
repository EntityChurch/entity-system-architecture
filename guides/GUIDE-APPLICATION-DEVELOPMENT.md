# GUIDE-APPLICATION-DEVELOPMENT — what an L5 application convention must be

**Version**: 2.0
**Status**: Active
**Kind**: tier-standard
**Authority**: binding
**Governs**: `specs/applications/APP-CONVENTION-*.md`

---

## 1. What this tier is

A home for **cross-impl, application-layer (L5) conventions** — standards that let
independent applications and independent front-ends converge on **one shared
format** instead of each reinventing it.

This tier exists because the spec taxonomy had no home for it. The core protocol
and its extensions define the **substrate**; the SDK specs define the
**bindings**; neither is the place for *"a content-format convention that web,
Godot, and terminal front-ends all agree to render the same way."*
`GUIDE-EXTENSION-DEVELOPMENT.md` §3.5 — *"paths are convention; the entity graph is
coherence"* — put app-layer conventions **outside** the core protocol but never
gave them a convergence-supporting home. This is that home.

**This document is the applications tier's sub-tier standard** in the sense of
`SPECIFICATION-FORMAT.md` §10.3. It states what is specific to this tier and
**nothing that is true corpus-wide** — the project tier already binds every
member and does not need repeating here.

## 2. What makes a convention a convention

An `APP-CONVENTION-*` spec:

### 2.1 Defines FORMAT — and only format

The cross-impl contract is the **entity-type vocabulary**: the `{type, data}`
shapes, their CDDL schemas, the dispatch keys, the on-wire/on-disk bytes. Two
conformant implementations produce byte-compatible entities and interpret each
other's.

### 2.2 Defines no kernel or SDK machinery

*Format is the contract; rendering, overlay UX, and front-end wiring are
per-front-end and live in the application, never here.* A convention adds **no**
kernel features and **no** required SDK surface.

**These two are the whole of what is distinctive about this tier.** Everything
else a convention must do, it must do because it is a specification — see §3.

## 3. What binds a convention from the project tier

**These are not restated here, and that is deliberate** — see
`SPECIFICATION-FORMAT.md` §10.3, under which a sub-tier standard restates nothing
that is true corpus-wide, and where it does restate, it names its authority. They
are listed so an author of this tier can find them, and each is a pointer, never
a second home.

| The rule | Its authority |
|---|---|
| A capability has a valid floor; capability adds, never assumed | `SPECIFICATION-FORMAT.md` §8.6 |
| A published vocabulary is a compatibility contract | `SPECIFICATION-FORMAT.md` §8.7 |
| Grow by handler or renderer, not by entity type | `SPECIFICATION-FORMAT.md` §8.8 |
| A disposition property lives on the entity, never its container | `SPECIFICATION-FORMAT.md` §8.9 |
| Never lock a hash width — reference content by the self-describing `content-hash` | `SPECIFICATION-FORMAT.md` §8.4.5 |
| A conformance requirement is one row with a stable id | `SPECIFICATION-FORMAT.md` §8.5a |
| Improvise no protocol — a gap files an ask, and the answer is a proposal | `AGENTS-STANDARD` (proposal-first); `GUIDE-EXTENSION-DEVELOPMENT.md` §7 |
| **A spec names what a check must discriminate and ships no artifact** | `GUIDE-EXTENSION-DEVELOPMENT.md` §7 · `GUIDE-CONFORMANCE.md` §5.1a, §7.0, §7c.6 |
| **Removal from a tree is UNPUBLICATION, never erasure — a convention MUST NOT let an application present it as deletion** | `APP-CONVENTION-FEED.md` §7.5 |
| **A type that requires no authorization has no revocation lever at all — for it, withdrawal is unlisting and the convention says so** | `APP-CONVENTION-SHARE.md` §2.5 |
| **Where your type carves an exception out of a rule you otherwise satisfy, the exception needs its own refusal and its own named check — following the general rule launders exactly the shape the exception exists to exclude** | `APP-CONVENTION-SHARE.md` §2.5 + `SHARE-8` · `APP-CONVENTION-SEMANTIC-CONTENT-SITE.md` §4 + §9 |

> **The last two rows were added 2026-09-09, and why they were missing is the more useful half.**
> **Both are tier-wide honesty rules and both were living inside a single member convention**, where
> the author of a *sixth* convention would never meet them: an author reads this document and their
> own draft, not a peer's §7.5. **The tell is that they were re-derived rather than cited** — the
> publication half was worked out from first principles in `APP-CONVENTION-SHARE` §2.5 while
> `APP-CONVENTION-FEED` §7.5 had carried the general rule, with a `[MUST NOT]`, for days.
>
> **They are pointers and not a second home**, per §10.3 — the authority is the section named, and a
> convention that needs to say more about withdrawal says it about *its own types* and cites these.
> **The two are different levers and a convention usually owes both**: §7.5 governs what a party who
> already holds the bytes can do (nothing reaches into another peer's store, ever), and §2.5 governs
> whether *future* retrieval can be stopped — which depends entirely on whether the type has an
> authorization step to withdraw.
>
> ***The general form, which is where this comes from:*** a published claim set is **monotone**, so
> *un-publishing* is a non-monotone operation and the architecture has no mechanism for it. Where a
> type carries an authorization step, the capability layer supplies a legitimate closed world and a
> withdrawal is real. **Where a type is defined as needing none, there is no closed world to find, and
> a control labelled *Delete* is a promise the architecture cannot keep.**

> **The carve-out row was added 2026-09-10, on its third instance, and the third is what generalized
> it.** The first two were about **unknown fields**: `GUIDE-ENTITY-WORKBENCH-APP` §5.4 rule 3, and
> `ENTITY-CORE-PROTOCOL.md` §2.6's MUST-ignore against `SHARE-8`'s refusal of a publication carrying an
> `audience`. The seat that hit both wrote *"if a third appears it is probably a tier-wide note rather
> than three local ones."*
>
> **The third is not about unknown fields at all**: `app/site-asset`'s `payload` **imports a union
> wider than the field admits** — `APP-CONVENTION-EMBED` §3 has three arms and the asset takes two — so
> a decoder that reuses the imported type accepts the excluded arm and reports nothing. **An imported
> type that is wider than the importing field is the same laundering with no unknown field anywhere in
> it**, which is why the note is stated about *exceptions* rather than about *tolerance rules*.
>
> **Both halves are load-bearing.** The refusal keeps the shape out; the named check is what stops the
> refusal being the one path nobody exercises. And keep the refusal **attributed** — an excluded arm
> that is correctly formed is the *publisher's* schema violation, not a malformed byte and not a gap in
> the reader.

> **The next row is the one this tier got wrong, and it is worth the sentence.**
> This document's predecessor said *"each convention ships example entities +
> expected hashes"* **and made that the ratification gate**, so five conventions
> were recorded *not ratifiable* on an artifact their author does not produce.
> **A specification names the cases a check must discriminate and what each
> asserts. The implementations and the conformance oracle produce the fixtures,
> the bytes, the harness and the run.** The set is pinned **after** two
> implementations trade the format, not before — two implementations exchanging
> it are a better oracle than a set written in advance by a party with nothing to
> run it against.

## 4. Scope boundary

`specs/applications/` houses **format / content-vocabulary conventions**
(`APP-CONVENTION-*`).

**Peer-composition charters** — operational wiring recipes; the named
compositions: operational peer, observer, service pool, hub-and-spoke, recovery
cluster, bridge, compute pool — are a **different genus**. They are deployment
recipes, not cross-impl wire contracts, and they stay in `guides/`.

## 5. Members

**Read the Status column against §3's last row.** *Authored* means the format is
written and its required checks are named. *Exercised* means independent
implementations have traded the format and run those checks. **A convention
ratifies on the second, and the second is not this document's to perform** —
every member below is complete as a specification and is waiting on an exchange,
not on a missing section.

| Spec | Role | Status |
|---|---|---|
| `APP-CONVENTION-REFERENCE` | **Foundational — the reference atom (*"this points at that"*) and its string form.** The single home for the shared `content-hash` / `peer-id` / `tree-path` atoms; **imported by the other members rather than restated in them** | Draft v0.1 — **authored; not yet exercised.** Eleven required checks named in `APP-CONVENTION-REFERENCE.md` §6.2, of which `REF-V3`, `REF-V7` and `REF-V9` would not be written from the prose alone |
| `APP-CONVENTION-EMBED` | Foundational — the generic rich-content typed node + two-level registry + output shape | Draft v0.2.3 — spine locked 3-way; **authored; not yet exercised.** Required checks in `APP-CONVENTION-EMBED.md` §9 |
| `APP-CONVENTION-SEMANTIC-CONTENT-SITE` | First consumer — content sites built on Embed (document / content / compute anatomy) | Draft **v0.5.1** — spine locked 3-way; `pages` cut; ordering floor pinned, semantic feeds open; v1 = manifest/page/nav/`.list`; **v0.5 adds §11's `sites` URL-projection prefix.** ⭐ **PARTLY EXERCISED — the only member that is.** **3 of §9's 7 cases addressed**, by two implementations independently: the **entity round-trip** (each decodes and re-encodes the other's `app/site-manifest` / `app/site-page` bytes byte-identically), **`G-PIN-4`** (one fixture, two publishers, agreeing structural root, across two *different cores*), and **`F-5`** (nav depth, both bounds falsified). **The other 4 are blocked on something other than effort** — `G-PIN-3` on a signable pin artifact, and the lowering / passive-refuse vectors on an `Embed` entity node that exists in neither implementation, which is `APP-CONVENTION-EMBED`'s sequencing question and not this convention's. Required checks in §9, jointly with EMBED's |
| `APP-CONVENTION-SHARE` | The share record + audience binding — a share is a titled grant; the audience is the `grantee`, never the `peers` scope | Draft **v0.2.1** — adds `app/share/publication`, the audience-less type, with D1–D4 and `SHARE-7`/`-8`/`-9`; **v0.2.1 disambiguates §2.5's *"no grant"*** (the audience model, not the policy table) and adds `SHARE-10`. **Authored; not yet exercised**, and it leaves the follow `strategy` vocabulary open. **Ten** required checks named in `APP-CONVENTION-SHARE.md` §8; `SHARE-4` and `SHARE-6` are the two that fail loudly under the intuitive-but-wrong reading, and `SHARE-10` is the one that fails quietly |
| `APP-CONVENTION-FEED` | **A thing someone posted** — the entry, the key-addressed index, the bounded collection, and the mirror. The vocabulary that makes *following someone across independent hosts* a format two implementations can both produce and both read | Draft v0.1 — **authored; not yet exercised.** Eleven required checks named in `APP-CONVENTION-FEED.md` §11.2, of which `FEED-3`/`-5`/`-6`/`-9` are load-bearing |

## 6. Document history

**v2.0** — reclassed and rehomed. Four of the nine disciplines were general rules
that happened to be discovered here, so the 26 extension specs were governed by
none of them; they are now `SPECIFICATION-FORMAT` §8.6–§8.9 and bind every spec
that mints a vocabulary or a container. Two more were restatements of authorities
that already existed and are now pointers. **Two remain, and they are what makes
an application convention an application convention.** The document moved from
`specs/applications/CHARTER.md` to `guides/`, matching the name, place and job of
`GUIDE-EXTENSION-DEVELOPMENT.md` — the word *charter* named a genus with one member.

**v1.1** — corrected #5, which had required each convention to ship example
entities and expected hashes and made that the ratification gate.
