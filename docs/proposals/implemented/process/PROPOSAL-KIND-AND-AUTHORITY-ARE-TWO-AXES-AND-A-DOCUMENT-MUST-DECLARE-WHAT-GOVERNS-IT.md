# PROPOSAL — kind and authority are two axes, not one; and every spec declares what governs it

**Status:** IMPLEMENTED (2026-09-09) — **ruled and folded the same day.** §7's three
questions are answered in §8; all five moves in §5 landed.
**Target:** `specs/SPECIFICATION-FORMAT.md` §10 (the document-kind table, rewritten on two axes) · §5.3
(a `Governed-by` header field) · §8 (three disciplines promoted in from the applications charter) ·
`specs/applications/CHARTER.md` (renamed, reduced, reclassed) · a consuming gate in the toolkit.
**Rides with:** `PROPOSAL-DOCUMENT-CLASS-HEADER-FIELD`, which already targets §5.3 to add a class field
and diagnoses the same root cause from the other end. **These two should land together or not at all.**
**Scope:** where authoring rules live and how a reader finds the one that binds them. **No wire change,
no protocol behaviour, no requirement retired.** Three rules gain scope; none loses force.

---

## 0. The question, and the answer

*Why does the applications tier have a charter when no other tier does, and what is a charter anyway?*

**Because the corpus has no taxonomy with a slot for the kind of document it is.** `SPECIFICATION-FORMAT`
§10 is the only place the corpus classifies its own documents, and it is a **two-row table** — *normative
spec* vs *architectural document* — with `Authority: Binding` welded to the first row and
`Authority: Informational` to the second. **Empirically there are six kinds and authority does not track
kind at all.** A tier-authoring standard is neither a spec nor a rationale document, so when one was
needed there was nothing to call it, and it got a name no other document in the corpus uses.

**The generalizable defect: kind and authority are independent axes, and this corpus has been treating
them as one.** Two documents classed *guide* are normative and gated (§1.3). One classed
*informational* carries nine numbered disciplines that gate five specifications (§1.2). Neither
situation is wrong — both are correct documents doing necessary work — but **the table says they cannot
exist**, so every one of them is an unlabelled exception, and a reader has no way to learn which
document binds the thing in front of them.

---

## 1. What is actually there — measured

### 1.1 The taxonomy the corpus states

`SPECIFICATION-FORMAT` §10, in full:

| Aspect | Normative Spec | Architectural Document |
|---|---|---|
| Audience | Implementors | Designers, evaluators |
| Content | Type definitions, algorithms, constraints | Rationale, tradeoffs, vision |
| **Authority** | **Binding** | **Informational** |
| Format | This format | Prose, diagrams, free-form |
| Stability | Versioned, breaking changes tracked | Evolves freely |

That is the whole of it. **§11.3 leans on this line to decide citation legitimacy**, so it is not
decorative — it is load-bearing for a gate.

### 1.2 The document the taxonomy has no row for

`specs/applications/CHARTER.md` — the only document in the corpus named `CHARTER` on the published
surface. It carries nine numbered disciplines, several stated as `MUST`, and **five specifications
declare conformance to it by number** (*"charter #5"*, *"charter class: FORMAT-only"*). It is declared
canonical and it publishes.

Its class had to be ruled by the linter, and the ruling is the finding stated from the outside:

> *A domain CHARTER declares what a domain is for and which workstream owns it — **process, not
> protocol.** It defaulted to canonical-spec, which scored its own "Owning workstream" line as spec
> contamination.*

**So the toolkit classes it `arch-doc`, whose §10 row says `Authority: Informational`** — while it is the
binding authority for five specifications. Nothing is wrong with the document. **The row is wrong.**

### 1.3 And the inverse, twice

`SPECIFICATION-FORMAT` itself and `STYLE-NAMING-CONVENTIONS` are both classed **`guide`** by the same map.
Both are normative authoring standards; both are enforced by gates; one of them is *this document's
target*. **A document cannot be the corpus's normative authoring standard and informational at the same
time**, and the only reason nobody has tripped over it is that no rule reads the class as authority — the
gate reads it for citation disposition only.

### 1.4 The governing layer that does exist, and what it is called

Stripping out the ~30 subject guides (which teach one surface to a downstream developer — `GUIDE-QUORUM`,
`GUIDE-REVISION`, `GUIDE-IDENTITY`…), the documents that **govern authoring** are:

| Governs | Document | Called | Where it lives |
|---|---|---|---|
| how any spec is written | `SPECIFICATION-FORMAT` | a *format* | `specs/` |
| how anything is named | `STYLE-NAMING-CONVENTIONS` | a *style* | `specs/` |
| whether a thing should be an extension at all | `GUIDE-EXTENSION-BUILDER` | a *guide* | `guides/` |
| how an extension is authored, and the stage process | `GUIDE-EXTENSION-DEVELOPMENT` | a *guide* | `guides/` |
| how conformance is verified, and who authors what | `GUIDE-CONFORMANCE` | a *guide* | `guides/` |
| how an implementation is built | `GUIDE-IMPL-DISCIPLINE` | a *guide* | `guides/` |
| **what an application convention must be** | **`applications/CHARTER.md`** | **a *charter*** | **`specs/`** |
| what a core spec must be | — | — | — |
| what an SDK spec must be | — | — | — |
| what a domain spec must be | — | — | — |

**Two readings, and both are true.** The extension tier has *two* governing documents and the
applications tier has one that merges both jobs plus format rules. And three tiers have none — which is
not urgent, because the extension tier's `GUIDE-EXTENSION-DEVELOPMENT` §7 process is what they actually
run.

**The one structural anomaly is the last column.** Every other governing document sits in `guides/`;
this one sits inside the tier's own spec directory, which is why it reads as a member of the tier rather
than as the rules for it.

### 1.5 The discovery failure, which is the one that costs

**Nothing points a new author at the document that binds them.** A spec's header carries `Version`,
`Status`, `Depends`, `Encoding` — and no field naming what governs it. So the governing layer is
discoverable only by someone who already knows it exists, and the measured consequence is not
hypothetical: **the applications charter's discipline #5 contradicted `SPECIFICATION-FORMAT` §8.5 and
`GUIDE-EXTENSION-DEVELOPMENT` §7 for as long as it existed**, was folded into five specifications, and
became the ratification gate for all of them, because nothing connected the tier's rules to the corpus's.

---

## 2. The nine disciplines, placed by scope

Read against the rest of the corpus, the applications charter's nine sort into four groups. **Three
genuinely belong to the tier. Three are general rules that exist only here — so the 26 extension specs
are governed by none of them. Three are restatements**, one of which correctly names its authority and
two of which did not.

| # | Discipline | Scope, measured | Disposition |
|---|---|---|---|
| **1** | Defines FORMAT, and only format | defines what a member of *this* tier is | **stays** |
| **2** | No kernel or SDK machinery | the same boundary, stated as a prohibition | **stays** |
| **3** | Improvise no protocol — file an ask, then a proposal | proposal-first, already an ecosystem rule | **→ pointer** |
| **4** | Has a valid floor — capability *adds*, never *assumed* | binds any spec written against a levelled substrate | **→ promote** |
| **5** | States what a check must discriminate; ships no artifact | `SPECIFICATION-FORMAT` §8.5 already requires a conformance item to name its class *"because the four classes have different authors"* | **→ pointer** *(corrected 2026-09-09; it said the opposite)* |
| **6** | Never lock a hash width | already `SPECIFICATION-FORMAT` §8.4.5, **and #6 correctly names itself its applications-layer form** | **stays as-is — the worked model for how a tier restates a corpus rule** |
| **7** | The compatibility contract — new fields optional, no renames, no type changes, breaking ⇒ new tag | **grepped `specs/` and `guides/`: stated nowhere else.** The wire half exists (unknown fields MUST-ignore, the locked core never renumbered); nothing covers a *type vocabulary*, which every extension mints | **→ promote** |
| **8** | Growth by body type or renderer, not by entity type | argued from one tier's extension point, but *"a new entity type only if a conformant consumer must behave differently"* is a general anti-explosion rule | **→ promote (arguable — see §7)** |
| **9** | A disposition property lives on the entity, never on its container | **stated nowhere else.** Extensions mint containers constantly; *containers do not travel, entities do* is not an application-tier fact | **→ promote** |

**The pattern is one thing, and it is why this proposal is about the taxonomy and not about nine rules:**
a general rule gets filed under whichever tier happened to discover it, because there is no home for
"true of every spec" that an author of one tier would think to look in.

---

## 3. The delta — §10 rewritten on two axes

**Axis 1 — kind** (what the document is for). Six values, all already in the corpus:

| Kind | Answers | Example |
|---|---|---|
| `normative-spec` | what must an implementation do | `EXTENSION-TREE` |
| `authoring-standard` | what must a *document* be | `SPECIFICATION-FORMAT`, `STYLE-NAMING-CONVENTIONS` |
| `tier-standard` | what must a member of this family be | the applications charter, `GUIDE-EXTENSION-DEVELOPMENT` |
| `process` | how work moves between parties | `GUIDE-CONFORMANCE` |
| `subject-guide` | how do I use this surface | `GUIDE-QUORUM`, `GUIDE-REVISION` |
| `architecture` | why is it this way | `SYSTEM-ARCHITECTURE` |

**Axis 2 — authority.** `binding` · `informative`. **Declared, never inferred from the kind.** A
subject-guide is normally informative and `GUIDE-EXTENSION-DEVELOPMENT` §3.3 is binding and gated; both
must be sayable.

**§10's Authority row is deleted** and becomes this field. Every other row it states stays true.

## 4. The enforcement point — `Governed-by`, and it is the whole answer to "how does it stay clean"

**A rulebook nobody opens is not a rulebook.** The failure this proposal exists for is not that the
applications charter is misfiled — it is that an author working in `specs/applications/` had no reason to
know it existed, and the same is true of every tier.

**Add to `SPECIFICATION-FORMAT` §5.3 a `Governed-by` header field**, naming the documents whose rules bind
this one — its tier standard, plus the authoring standards, by path.

```
**Governed-by**: `guides/GUIDE-APPLICATION-DEVELOPMENT.md` · `specs/SPECIFICATION-FORMAT.md`
```

**The gate is mechanical and reuses shipped machinery.** Every declared spec resolves its `Governed-by`
targets (the check `spec address` already performs on citations), and every document of kind
`tier-standard` is named by at least one document — **a tier standard nobody declares is a rulebook with
no readers, and that is now a finding rather than a silence.** Two conditions, held apart the way the
inventory and dependency gates already are: a malformed or dangling declaration fires immediately; the
adoption backlog is held by a ratchet that only ever rises.

**Why a header field and not a convention:** the same argument the class-field proposal already makes and
measured three times — *a fact the corpus depends on lives only in a tool's config, keyed by a filename
pattern, and is re-derived, differently, by whoever needs it next.*

## 5. The moves, which want a ruling before they land

1. **`specs/applications/CHARTER.md` → `guides/GUIDE-APPLICATION-DEVELOPMENT.md`**, kind `tier-standard`,
   authority `binding` — matching the name, place and job of `GUIDE-EXTENSION-DEVELOPMENT`. The word
   *charter* leaves the published surface, where it names a genus with one member.
2. **#4, #7, #9 promoted into `SPECIFICATION-FORMAT` §8**, where they bind every spec that mints a type
   vocabulary or a container. **The tier keeps a one-line restatement naming the new authority**, which
   is #6's shape and the only shape that survives a sweep.
3. **#3 and #5 reduced to pointers** at authorities that already exist.
4. **#1, #2 and (pending §7) #8 stay** as what makes an application convention an application convention.
5. **§10 rewritten; `Governed-by` added; the gate built.**

**Nothing is retired and no rule loses force.** Three gain scope they should always have had.

## 6. What this does not do

- **It does not create tier standards for core, SDK or domains.** Three tiers have none; two of them
  have not needed one yet, and inventing three documents to make a table symmetrical is how the corpus
  got here. **A tier standard is written when a tier has members that disagree**, not before.
- **It does not touch any subject guide**, which is ~30 of the 35 files in `guides/` and is not the
  layer with the problem.
- **It does not merge `GUIDE-EXTENSION-BUILDER` into `GUIDE-EXTENSION-DEVELOPMENT`.** They look like
  duplicates and are not: *should this be an extension at all* and *how do I author one* are different
  questions with different readers.

## 7. Open — for the ruling

1. **Is #8 tier-specific or general?** It is argued from one tier's open extension point, but its rule
   is about entity-type proliferation, which is everyone's. **Promoting it is the aggressive read**;
   leaving it is defensible and costs a second discovery elsewhere.
2. **Does `Governed-by` list the authoring standards, or only the tier standard?** Listing everything is
   honest and verbose; listing only the tier standard is short and makes the tier standard responsible
   for naming the rest. **Recommended: only the tier standard**, which keeps one hop and one home.
3. **What binds a document with no tier** — the four top-level `SYSTEM-*` / `ARCHITECTURE-*` specs?
   Today: the authoring standards and nothing else, which may be correct and should be said rather than
   left blank.


---

## 8. The ruling, and the fold `[2026-09-09]`

**The ruling is a two-level tier model, and it answers all three of §7 at once.** A rule that applies
to every document goes in the shared, corpus-wide standard. A rule specific to one family goes in that
family's own standard. And a document that sits at the top, under no family, is not unclassified — the
top level is itself a tier, the project tier, with sub-tiers beneath it.

**The corpus had been trying to express exactly this with one level and a table that welded authority
to kind.** Every one of §7's questions dissolves under the two-level reading.

| §7 | Answer |
|---|---|
| **1. Is #8 tier-specific or general?** | **General, so it goes in the shared standard.** Its rule is about entity-type proliferation, which is everyone's. The aggressive read was the right one, and the conservative read's cost is precisely the defect this proposal documents: a general rule filed under the tier that discovered it. **Four promotions, not three:** #4, #7, #8, #9 |
| **2. Does `Governed-by` list the authoring standards or only the tier standard?** | **Only the sub-tier standard — and for a stronger reason than "one hop."** The project tier binds *unconditionally*; it is the root, not a dependency. Declaring it in every header would be a fact repeated 40 times, which is a fact that goes stale 40 ways. **`Governed-by` names the nearest governing standard, and its ABSENCE is the positive statement that the project tier governs directly** |
| **3. What binds a document with no tier?** | **It is a member of the project tier — that is not a gap, it is a position.** The four top-level `SYSTEM-*` / `ARCHITECTURE-*` specs are governed by the authoring standards and by nothing else, and `SPECIFICATION-FORMAT` §10.3 now says so |

### 8.1 What landed

1. **`SPECIFICATION-FORMAT` §10 rewritten** as §10.1 kind (six values) · §10.2 authority
   (`binding`/`informative`, **declared, never derived**) · §10.3 **the tier model** · §10.4 rationale.
   The old Authority row is gone from the table and is a header field.
2. **§5.3 gains `Kind`, `Authority` and `Governed-by`** — which is where `PROPOSAL-DOCUMENT-CLASS-HEADER-FIELD`
   lands too, as promised. **The two landed together.**
3. **§8 retitled *Spec Conventions*** and gains **§8.6** (a capability has a valid floor) · **§8.7** (a
   published vocabulary is a compatibility contract) · **§8.8** (grow by handler or renderer, not by
   entity type) · **§8.9** (a disposition property lives on the entity, never its container). **No
   existing §8 number moved**, so every citation in the corpus still resolves.
4. **`specs/applications/CHARTER.md` → `guides/GUIDE-APPLICATION-DEVELOPMENT.md` v2.0**, kind
   `tier-standard`, authority `binding`. Nine disciplines → **two** that are genuinely tier-specific,
   plus a pointer table naming the authority for each of the other seven. **The word *charter* is off
   the published surface**, where it named a genus with one member.
5. **All five members re-headed** with `Kind` / `Authority` / `Governed-by`, and **every `charter #N`
   citation in the corpus rewritten to its new authority** — 5 specs, ~25 citations. `SYSTEM-ARCHITECTURE`'s
   tree diagram and `CANONICAL-DOCS.toml` follow the move.

### 8.2 What did NOT land, and why

**The `Governed-by` gate is not built.** §4 specifies it and it is real work: resolve every declared
spec's target, plus *a `tier-standard` no document declares is a rulebook with no readers.* **It is
filed rather than built** because the ruling was the blocker and the gate is not — and because the
corpus currently has exactly one sub-tier standard with a `Governs` field, so a gate written today
would be calibrated against one example. **That is the mistake this session already made once**, in
`spec expiry`: a mechanism built against a single case, which then fired on zero rows.

**No requirement was retired and none lost force.** Four gained the scope they should always have had.
