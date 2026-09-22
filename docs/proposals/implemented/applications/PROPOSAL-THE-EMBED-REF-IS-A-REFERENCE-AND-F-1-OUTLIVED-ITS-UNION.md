# PROPOSAL — the embed `ref` is a reference, `F-1` outlived the union it disambiguated, and three smaller corrections the first joint run surfaced

**Status:** IMPLEMENTED 2026-09-09 — all four deltas folded the same session. `R1`/`R1b` into
`APP-CONVENTION-SEMANTIC-CONTENT-SITE` §3.2 + §9 (`F-1` withdrawn), `R2` into §9 (`G-PIN-4`'s
comparand), `R3` into §4.1 (walk scoping + the refused manifest), `R4` into
`APP-CONVENTION-SHARE` §2.5. **Three questions are recorded here as deliberately unruled and are
tracked on the open-items ledger, not by this document** — §1.6's relative `..`, §3.2's
partial-decode alternative, and §2.2's `app/site-root` minting sequence.
**Target:**
`specs/applications/APP-CONVENTION-SEMANTIC-CONTENT-SITE.md` §3.2 (**R1** — withdraw `F-1`), §9
(**R2** — name the comparand; **R1b** — the vector that follows), §4.1 (**R3** — the walk contract
and the refused manifest) ·
`specs/applications/APP-CONVENTION-SHARE.md` §2.5 (**R4** — what withdrawing a publication does)
**Provenance:** the first application-tier cross-implementation publish run, and the three findings
its implementers filed on the way through. Every claim below was re-checked against the specification
text rather than taken from the reports.

---

## §0 Why these four are one proposal

They arrived from one activity — **two independent application-tier implementations publishing the
same fixture and comparing bytes**, which had never happened before. Three of the four are one
sentence each. The first is a withdrawal, and it is the reason the set is worth reading in order:
**each of the four is a place where a document was correct when written and stopped being correct
without anything moving in it.**

That is the shared shape, and it is more useful than any of the individual fixes:

| | The rule | What changed under it |
|---|---|---|
| **R1** | `F-1`'s two-form `ref` discrimination | the union it discriminates was retired in a sibling document |
| **R2** | `G-PIN-4`'s *"identical site root"* | nothing — the phrase was always ambiguous, and no one had run it |
| **R3** | §4.1's *"stop cleanly"* | implementations acquired a decoder ceiling below the depth the rule contemplates |
| **R4** | §2.5's publication type | a type whose definition removes the lever the surrounding prose assumes |

---

## §1 `R1` — `F-1` is withdrawn. A `ref` is a reference string and resolves as one

### 1.1 The rule as it stands

`APP-CONVENTION-SEMANTIC-CONTENT-SITE` §3.2 carries:

> **Normative rule: a `ref` value with a leading `/` is a `path` (V7 §1.4 absolute entity path);
> otherwise it is a `content-hash` in its canonical string form (multibase/hex).**

with the justification stated immediately before it:

> *at the wire layer CBOR types disambiguate `path` (tstr) from `content-hash` (bstr); in the
> directive string they don't.*

**Two forms, exhaustive by construction.**

### 1.2 The justification is no longer true

`APP-CONVENTION-EMBED` §2's normative notes:

> **`child-payload.ref` is an `entity-ref`** (`APP-CONVENTION-REFERENCE` §2.1). It previously read
> `(path / content-hash)` — an **untagged** union discriminated by CBOR major type, which is the one
> place this document's own tagged discipline was not applied.

**The union `F-1` exists to project into string space was retired.** `child-payload.ref` is a
reference atom, and `APP-CONVENTION-REFERENCE` §2.1's `REF-R1` says there is no untagged atom at all.
`F-1` is a disambiguation rule for a distinction the wire layer no longer draws.

Nothing announced this, because the two rules share no vocabulary: one is a CDDL member's type, the
other is a *"directive string discrimination"*. **A rule survives the retirement of its own subject
when the retirement is written in different words.**

### 1.3 And the surviving rule contradicts the convention that owns the subject

`APP-CONVENTION-REFERENCE` §3.4 is the normative home for what an unqualified string in a document
body denotes:

> **The base is the referring entity's own location.** A link with no scheme is resolved against it:
> a leading `/` is **root-absolute within the current site**; anything else is **directory-relative to
> the current page**.

Set against `F-1`:

| form | `F-1` says | `APP-CONVENTION-REFERENCE` §3.4 says |
|---|---|---|
| `/a/b` | a **V7 §1.4 absolute entity path** | **root-absolute within the current site** |
| `assets/x.svg` | a (malformed) **content-hash** | **directory-relative to the current page** — *the common case* |

**The first row is a security difference and not a wording difference.** A V7 absolute entity path in
an authored body addresses the whole tree; root-absolute-within-the-site addresses the site subgraph.
One escapes the subgraph an author is publishing and the other cannot. **The second row is every
`ref` that any implementation actually emits** — a relative asset reference — which `F-1` classifies
as a content-hash that will not parse as one.

**The reporting implementation's parser refuses the leading-`/` form** as part of a subgraph
confinement predicate that also refuses `://`, a leading `//`, `data:` and any `..` segment. Measured
against `F-1` that reads as non-conformance. Measured against `APP-CONVENTION-REFERENCE` §3.4 it is
**exactly what the convention prescribes** — *"a string that cannot be parsed as any of these forms
is treated as leaving the system — an external link — rather than guessed at. Refusal is the correct
behaviour."*

### 1.4 The tell, and it is inside the document being corrected

`APP-CONVENTION-SEMANTIC-CONTENT-SITE` §3.1 already defers to the reference convention, **one
paragraph above `F-1`**:

> **One exception, and it is a scoping rather than an addition: a LINK in a page body has an
> entity-native meaning.** It is a reference string and it resolves by `APP-CONVENTION-REFERENCE`
> §3.4 — directory-relative to the current page, root-absolute on a leading `/`, `site:` for a
> same-peer cross-site target, `entity+ref://` when fully qualified.

**A link in a page body and a `ref` in an embed directive are both unqualified strings in a page body
naming something in the tree.** §3.1 routes one to the reference convention and §3.2 gives the other
its own incompatible two-form rule, ten lines apart, in the same section of the same specification.

### 1.5 The edit

**Replace `F-1`'s bullet in §3.2 with:**

> - **`ref` is a reference string and resolves by `APP-CONVENTION-REFERENCE` §3.4**, identically to a
>   link in a page body (§3.1): a leading `/` is root-absolute **within the current site**; `site:` is
>   the same-peer cross-site form; `entity+ref://` is fully qualified; **anything else is
>   directory-relative to the current page.** A string parsing as none of these is treated as leaving
>   the system and is refused, not guessed at.
> - **A content-addressed embed target is spelled as a reference, never as a bare digest.** Use
>   `APP-CONVENTION-REFERENCE` §3.1's pinned form (`entity+ref://{peer}/?hash={hash}`) when the target
>   is on another peer, or a **`pointer` payload** (`APP-CONVENTION-EMBED` §2) when it is a blob in the
>   resolving peer's own store. **A bare hex digest in a `ref` position is not a distinguishable form
>   and MUST NOT be emitted** — it is indistinguishable from a directory-relative reference to a page
>   of that name.
> - *(`F-1`'s former path-vs-content-hash rule is **withdrawn**. It disambiguated the untagged
>   `(path / content-hash)` union that `APP-CONVENTION-EMBED` §2 retired in favour of a tagged
>   reference atom; with no untagged union at the wire layer there is no ambiguity for the string form
>   to inherit. Every form above is syntactically decidable, which is the property `F-1` was there to
>   provide.)*

**Why a withdrawal and not a third arm.** A third arm keeps two normative homes for one subject and
makes `assets` a reserved first segment of every site subgraph. Deferring to the reference convention
removes a home, costs no reserved word, and makes the shipped behaviour conformant as it stands.

### 1.6 One residual divergence this does NOT close — name it, do not paper over it

`APP-CONVENTION-REFERENCE` §3.3 scopes its `..` refusal to the **absolute** form and says so
deliberately: *"§3.4's relative form is directory-relative, so `..` is both meaningful and expected
there. Relative resolution consumes the dot segments; what it produces is an absolute reference in
which none survive."*

**The reporting implementation refuses `..` outright, before resolution.** That is stricter than the
reference convention, and it is not obviously wrong — refuse-early and resolve-then-confine reach the
same place for every reference that stays inside the subgraph, and differ only for one that would
leave it.

**This proposal does not rule it.** It is a genuine convergence question between two implementations
and belongs to the triage that follows a cross-implementation run, not to a specification edit made
before the second implementation has been measured on it. **Recorded so the withdrawal above is not
read as having settled it.**

---

## §2 `R2` — `G-PIN-4` names an artifact that cannot be compared

§9 currently reads:

> A **reproducible-publish** test (`G-PIN-4`) — one fixture, two publishers, identical site root.

**"Site root" reads as the signed artifact, and the signed artifact carries a wall clock.** The
published-root entity carries a `published_at` field. Measured, by publishing one fixture twice under
one pinned identity seed:

| | run A | run B |
|---|---|---|
| signed-root head | `008615f3b44c09c7…` | `00a7b337ea9fd493…` |
| **structural root hash** | **`00bc252f2a685c4a…`** | **`00bc252f2a685c4a…`** |
| content blobs | 15 | 15, 13 shared |

**Everything structural is identical and the head differs every run.** A check comparing "the signed
root" therefore fails one hundred percent of the time, and across two implementations it fails
**wearing a real divergence's clothes** — which is the expensive failure, because the first response
to it is to go looking for an encoding bug that is not there.

`EXTENSION-TREE` §3.2 already states the correct property in the section §2 of this convention cites
— determinism rule 3, *"No timestamp — a snapshot is pure structural data."* **The two documents do
not disagree; §9 simply does not say which of the two artifacts it means, and they are one word apart
in English.**

**Edit — §9's `G-PIN-4` bullet:**

> - A **reproducible-publish** test (`G-PIN-4`) — one fixture, two publishers, an identical
>   **`tree:snapshot` root over the site subgraph** (`EXTENSION-TREE` §3.2). **Not the published-root
>   head**, which carries `published_at` and is not comparable across runs or across implementations.

### 2.1 `R1b` — and `F-1`'s vector changes with `F-1`

§9's lowering-vector bullet currently pins the withdrawn rule:

> including a `ref` of **each form** (leading-`/` path and bare content-hash) to pin the `F-1`
> discrimination rule.

**Edit:** replace with — *including a `ref` of each resolvable form (`APP-CONVENTION-REFERENCE` §3.4:
root-absolute, `site:`, fully-qualified, and directory-relative) plus one unparseable string, to pin
that resolution and refusal agree across implementations.*

### 2.2 A note on `G-PIN-3`, which this makes reachable and which is otherwise not

The same clock is why the signed-root head cannot satisfy `G-PIN-3`'s *"expected signature bytes,
byte-identical across at least two independent L5 implementations."* **`app/site-root` is the only
artifact in this convention that can**: §2 specifies it as `{root, seq, site_id, ? passthrough_of}`
with no timestamp, names one signing surface (*"the canonical ECF encoding of the pin entity's
`(type, data)`"*), and Ed25519 is deterministic — so a fixture pin under a known identity has exactly
one correct signature.

**This is recorded as the answer to a question this corpus has raised twice** — whether
`app/site-root` should be withdrawn because no implementation ships it. **It should not.** It is not
a mechanism waiting for a use case; it is the artifact that makes `G-PIN-3` runnable at all, and the
published root is not a substitute for it. **No implementation is being asked to mint it on this
proposal's say-so** — that sequencing is a separate decision.

---

## §3 `R3` — §4.1's walking contract, and the manifest that is refused rather than truncated

§4.1 reads:

> `nav` is a tree and MAY contain authored cycles or pathological depth. A renderer walking nav
> **MUST**: maintain a **visited-set** (cycle detection), enforce a **max depth (recommend 32)**, and
> on either limit **stop cleanly** (render what it has; never infinite-loop / stack-overflow).

Building the vector for it raised two questions the rule does not answer.

### 3.1 The visited-set binds the WALK, not the artifact — and a value type may discharge it

An implementation whose `nav` children are an **owned value** rather than a reference cannot
represent a cycle at all: the decoded form has no back-reference, and the serialization it decodes
from has none either. A visited-set there guards a state the type system forbids.

**That is conformance, and §4.1 should say so**, because the alternative reading — that the
visited-set is a MUST on the artifact — obliges every implementation to carry dead code and makes the
structural answer look like a gap.

**Edit — append to §4.1:**

> **The visited-set obligation is on the WALK, and an implementation whose decoded `nav`
> representation cannot express a cycle discharges it structurally.** A representation in which
> children are owned values decoded from a serialization with no back-reference has no cycle to
> detect. **What MUST hold in every case is the outcome** — no unbounded recursion and no stack
> exhaustion on any authored input. An implementation claiming the structural discharge states the
> property of its representation that provides it.

### 3.2 The depth ceiling: `stop cleanly` and `refuse the manifest` are different outcomes and are not distinguishable today

Measured across authored depths 32 · 100 · 120 · 125 · 126 · 127 · 128 · 129 · 130 · 200 · 10 000 ·
1 000 000: an authored nav depth **≤126 decodes fully; at ≥127 the manifest decode fails and the
implementation substitutes an empty default** — discarding `site_id` and `title`, not merely the deep
tail, and silently.

**Two things are wrong and only the second is §4.1's.**

**The first is the implementation's** and is not ruled here: a decode failure that yields a default is
indistinguishable from a site that genuinely has no title. That is a collapsed value where two facts
needed separate words.

**The second is that §4.1 does not say whether it binds at all here.** *"Stop cleanly (render what it
has)"* was written about a **walk** that reaches a depth limit. A manifest that a decoder refuses
before the walk begins has nothing to walk, so the rule is silent precisely where an implementation
needs it.

**Edit — append to §4.1:**

> **A manifest a decoder refuses is a different outcome from a walk that stops at the depth limit, and
> the two MUST be distinguishable.** *"Stop cleanly (render what it has)"* binds the walk. Where a
> manifest cannot be decoded at all — including because its authored nesting exceeds a decoder's own
> limit — the renderer **MUST** surface a refusal distinct from a successfully-decoded empty or
> untitled manifest. **Substituting a default is not conformant**, because it reports a fact about the
> site that is not true.
>
> **Implementations SHOULD NOT rely on a dependency's nesting limit to provide §4.1's bound.** Where
> the property holds only because a decoder happens to stop first, a change in that dependency moves
> it silently; the check for this rule should fail on such a change rather than track the number.

> **Deliberately not ruled: whether a refusing decoder should instead decode partially to the depth
> limit.** A partial decode is a larger change than a distinguishable refusal, it interacts with
> canonical round-tripping, and no second implementation has been measured on it. The distinguishable
> refusal is what is needed to stop a wrong fact reaching a reader, and it is what this proposal asks
> for.

---

## §4 `R4` — what withdrawing an `app/share/publication` does, and what it cannot do

`APP-CONVENTION-SHARE` §2.5 defines a type that **requires no authorization to retrieve** — that is
its purpose, and the surrounding convention is correct about it.

**The consequence is not stated anywhere, and it should be, because it inverts what a reasonable
implementer will build.**

| | `app/share/record` | `app/share/publication` |
|---|---|---|
| what authorizes retrieval | a minted token per audience entry | **nothing, by definition** |
| what a withdrawal acts on | **the policy write** — emptying the member's entry stops the token validating; the convention already names this as complete withdrawal | **only the entity** — there is no grant, so there is nothing to revoke |
| what *unshare* can therefore mean | *stop authorizing* — a real lever at the capability layer | **unlisting, and only unlisting** |

For a record, the content layer's lack of a forget verb is a **hygiene** problem: bytes linger, but
the capability layer still decides who may have them, so the user-visible promise remains keepable.

**For a publication it is the entire access path.** Retrieval is served by hash without consulting a
binding, and the type tag is the index key — so deleting the publication removes it from a
type-filtered query and changes nothing about who can still pull the bytes. **The only mechanism the
type has is discovery, and discovery is the half that binds nobody.**

**This is not an implementation gap. It is what the definition entails**, and it is why it belongs in
the specification rather than in a defect report.

**Edit — append to §2.5:**

> **Withdrawal unlists; it does not retract.** A publication requires no authorization to retrieve, so
> there is no grant to revoke and withdrawal has nothing to act on but the entity itself. Deleting or
> unpublishing an `app/share/publication` removes it from type-filtered discovery. **Any party already
> holding the content hash may still retrieve the bytes**, and no mechanism in this convention or in
> the content layer changes that.
>
> **[SHOULD]** An interface offering this action names it for what it does — *stop listing*, *stop
> offering* — and **SHOULD NOT** present it as deletion, retraction or recall. **The failure is
> silent**: nothing errors, and the party holding the false belief is the person who published.

**Ranked behind it and not proposed here:** whether the content layer should carry a forget/unbind
operation, and if so whether forgetting a blob forgets its chunks or whether reclamation needs a walk
and a reference count because two blobs may share one. **That is a mechanism question for the content
extension**, it is not resolved by the sentence above, and the sentence above is worth landing
whatever its answer, because it is the difference between a keepable promise and a broken one.

---

## §5 What this proposal does not touch

- **No version bump is proposed for either specification.** Both are DRAFT and pre-ratification; the
  fold that lands these should carry them under the existing version lines unless the ratification
  pass renumbers.
- **No test vectors, fixtures, bytes or harnesses are authored here.** §9 names cases; the fixtures
  and the runs belong to the implementations and the conformance oracle. `R2` and `R1b` change *what
  a case asserts*, which is this document's to say.
- **`§1.6`'s `..` question, `§3.2`'s partial-decode question and `§2.2`'s minting question are
  recorded as open and deliberately unruled.** Each needs a second implementation measured on it
  before a rule is worth writing.
