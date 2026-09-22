# DISCIPLINE CHARTER — entity-system-architecture

**Canonical, undated, edited in place.** This is the assembled set: arch's lifecycle disciplines,
the anti-pattern catalog, and the doctrines this repo runs. `AGENTS.md` points here; this file does
not restate `AGENTS.md`, and neither restates `METHODOLOGY.md`.

**Tier: AUTHORING** (`METHODOLOGY.md` §9). The runtime disciplines D1–D11 describe substrates this
repo does not have. **D12 applies verbatim.** In place of the rest, the substrate is **the corpus
itself**, so this repo's disciplines are **lifecycle** disciplines.

**Doctrines are adopted by reference, not restated:** the **Audit Doctrine A0–A12**
(`METHODOLOGY.md` §7.2) and the **Foundation Audit Doctrine FA0–FA7** (§7.3). The Feature Doctrine
mostly does not apply. The **ratchet** (§2) and the **promotion ladder** (§3) are normative here.

---

## 0. Why the numbering is `L`, and not `A`

**The routed draft numbered these A1–A6. That collides**, in the same injected file, with the
**Audit Doctrine's A0–A12** — whose own A1 is *"trace before you theorize."* Two different `A1`s
one document apart is how a set stops being citable. So arch's lifecycle disciplines are **L1–L6**:
`D` is runtime, `A` is an audit step, `F` is a feature step, **`L` is lifecycle.**

`METHODOLOGY.md` §9's candidate table keeps its own numbering as the ecosystem-visible draft; the
mapping is one-to-one and stated per rule below.

## 0a. Status vocabulary, and why two of six are not ratified

**The promotion ladder is normative and it applies to this set on the day it is written**
(`METHODOLOGY.md` §3): bit us once → **anti-pattern catalog**; bit us a **second time in a
different shape** → **ratified**. Between the two it is a **candidate** — applied, but not yet
claimed to generalize.

**Ratifying all six because six were routed would be the first violation of the set.** Four are
ratified on the evidence. **L3 and L6 are candidates**, with the reason recorded, and that is a
real outcome rather than a gap.

| | Rule | Draft | Status |
|---|---|---|---|
| **L1** | No normative spec edit without a proposal | A1 | **RATIFIED** |
| **L2** | Read the filing seat's own document | A2 | **RATIFIED** |
| **L3** | A partial fold does not get a completeness marker — a version bump **or** a move to `implemented/` | A3 | **RATIFIED** 2026-08-17 — second incident, different marker |
| **L4** | A claim about a document or a tree is checked by opening it | A4 | **RATIFIED** |
| **L5** | Spec text is not our log | A5 | **RATIFIED** |
| **L6** | Resolve divergence from the table, before the fix is written | A6 | **CANDIDATE** — no arch-side incident yet |
| **L7** | Check the toolkit for the instrument before building one — and **quote the invocation with the count** | — | **CANDIDATE** — five instances, enforcement partial |
| **L8** | An artifact is not a conclusion about the thing it names — open it | — | **RATIFIED** 2026-08-17 |
| **L9** | A deferral is a build-state claim and expires like one — **including a resolved open item in a folded proposal**. **And a sentence found to be FALSE is deleted in the session that finds it; writing it up is not an alternative to deleting it** — a false sentence in a normative document is *generative*, since every later reader derives from it in good faith (2026-09-05: one uncorrected paragraph reached three homes in two specs and a guide over nineteen days, and was re-reported to the operator as fact). **Grep the corpus for the other homes in the same session** | — | **RATIFIED** 2026-08-20 — second shape, a folded proposal's open item; sixth instance 2026-08-22 (self-blocking sequencing note); **delete-don't-file clause added 2026-09-05, operator-raised** |
| **L10** | Check the *framing* of a routed finding, not only the finding | — | **CANDIDATE** — one incident |
| **L11** | Read the study that produced a design space before ruling inside it | — | **RATIFIED** 2026-08-17 |
| **L12** | A mechanism cited in a ruling must be reachable by the actor the ruling assigns it to | — | **CANDIDATE** — one incident |
| **L13** | Filing is not routing — a record that a seat owes something is not a delivery to that seat | — | **RATIFIED** 2026-08-18 — second shape, same day |
| **L14** | `ENTITY-CORE-PROTOCOL` is not ours to version; extension versions are ordinary work | — | **RATIFIED** 2026-08-18 — operator ruling |
| **L15** | A ruling goes back to the seat that filed it before it goes to anyone else | — | **CANDIDATE** — one incident, operator directive |
| **L16** | Before ruling a cross-impl semantic, search **every** tier for a seat that already implements it — **and before claiming our own corpus is silent, search every `specs/` extension, not the layer the question sounds like** | — | **RATIFIED** 2026-08-19 — second instance, opposite direction; **corpus axis added 2026-09-04** |
| **L17** | A normative MUST naming a value **or a capability** does not land without a declared site a peer can carry and a conformance check | — | **RATIFIED** 2026-08-20 — second shape, the capability-encoding axis |
| **L18** | A cohort implementation is not evidence that a cohort ruling is right — **and a citation labelled *corroboration* is verified like any other claim, or dropped** | — | **RATIFIED** 2026-09-02 — second shape, an unopened corroboration citation false about both peers it named; one save recorded |
| **L19** | Say which kind of "vector," and state its satisfaction mode — open the section that owns the surface, not §7.0's index | — | **RATIFIED** 2026-08-20 — second shape |
| **L20** | An example set cannot falsify a rule it does not span | — | **CANDIDATE** — one incident, third shape |
| **L21** | A fold is a delivery to every seat that reads the corpus — route by who **consumes** it (implements, cites, **pins**), not only who implements; name the **divergence unit** when a dormant field goes load-bearing; scope a relay by the fold's **diff**, never by what the seat shipped; and when a fold pins a **vocabulary**, grep the cohort for the names being pinned **and the names they replace** | — | **RATIFIED** 2026-09-01 — second shape, a guide consumed by a pin; third + fourth shapes 2026-09-02; **fifth shape 2026-09-03 — an app-tier convention, where the fourth shape's enforcement point is written in core-spec nouns and so never fired** |
| **L22** | A borrowed sentence is re-verified by the borrower, not the author — whoever moves it, however short the move | — | **RATIFIED** 2026-08-31 — second shape, the filing seat as mover |
| **L23** | A rule has every normative home it is stated in — enumerate by the rule's **subject**, not its tokens; and a restatement names its authority | — | **RATIFIED** 2026-08-30 — second shape; third and fourth shapes 2026-08-31 |
| **L24** | A reference is only a pin if it resolves in the history **and the layout** the receiving audience gets | — | **RATIFIED** 2026-08-23 — second shape, the build surface |
| **L25** | Read the section for its **examples**, not only the clause you came for — a tightening that closes no hole is not conservative | — | **CANDIDATE** — one incident, refuted by a peer who built it |
| **L26** | The cohort discovers by **building** — arch's failure mode is not folding what they built; do not write a constraint telling seats to hold off | — | **RATIFIED** 2026-09-02 — operator correction; `spec ledger`'s `proposal-state-mismatch` is the fold debt and it ratchets to zero |

> **Fourth recurrence, 2026-09-02 — two days after the third, and it is now BUILT rather than
> re-diagnosed.** Four rows were stale: **L9** and **L21** read CANDIDATE here while `AGENTS.md` had
> carried them as RATIFIED since 2026-08-20 and 2026-09-01; **L18** ratified today; **L26** was missing
> entirely. Found by opening this table to update one row — the same way the third was found, which is
> the tell: **every recurrence has been found by someone who happened to be editing the file for an
> unrelated reason, and never by a check.**
> **So the diagnosis was complete three times and changed nothing, because all three remedies were
> habits.** The third note's own prescription — *"the rows land the same session the rule is earned"* —
> is exactly the discipline that had just failed twice, restated as a resolution. **A rule whose
> enforcement point is "remember to do it" is the theater this repo's own ladder forbids** (§3: *a
> discipline with no enforcement point does not count*), and the ladder was not applied to the document
> that contains it.
> ***Enforcement point, and it is a gate, not a habit:*** **`spec charter`** (arch-tools) parses the
> `L{n}` status out of **both** homes — this table and `AGENTS.md`'s summary line — and diffs them. A
> rule ratified in one and candidate in the other, or present in one and absent from the other, is an
> error. **This is L23's fourth shape applied to our own two homes:** `AGENTS.md` restates this table
> without marking itself a restatement, so neither document is readable as a copy and the divergence is
> invisible from both. The gate makes the next sweep a `make check` instead of a lucky edit.

> **L17–L25 were added to this table on 2026-08-31 — the third recurrence of the drift the two notes
> below describe, and the largest.** Nine rules, **six of them RATIFIED**, were earned and written up
> in `AGENTS.md` across 2026-08-19…31 and never reached the document that calls itself the canonical
> home. The charter has read *"the set is L1–L16"* for twelve days.
> **This one was found from the outside**, which is the part worth keeping: it surfaced while ruling
> `entity-browser-rust`'s window-index proposal, whose finding is **L23's fourth shape — an unmarked
> restatement of a canonical table, invisible from the authority.** `AGENTS.md` restates this set and
> does not say it is a restatement; this table claims to be the set and is short. **We shipped the
> diagnosis of our own defect in the same session we were still committing it**, and only noticed
> because the rule being written was about exactly this. *The rule that catches an unlanded rule is
> still the one we do not apply to ourselves* — now three times running, which is no longer a lapse
> but a property of how this file is maintained. **The row is cheap and the body section is not; the
> rows land the same session the rule is earned, and the body sections stay owed below.**

> **L13 and L14 were added to this table on 2026-08-18, having been earned and written up on
> 2026-08-17/18 in `AGENTS.md` only** — the identical drift the note below describes, recurring inside
> the very session that read the note. **The gap between "the rule is written" and "the rule is in the
> canonical home" is not closed by knowing about the gap.** L13's own second shape is a variant of the
> same thing one level out: a true statement filed in a place that keeps its audience from acting on it.

> **L7–L25 have no body section below, and that is itself a finding recorded here rather than fixed
> quietly.** `[2026-08-17]` This document declares itself *"the canonical home"* of the discipline set
> and the ratchet's own law is **"if it didn't land in the charter, it didn't land."** Six rules —
> two of them **ratified** — were earned, written up in full, and landed in **`AGENTS.md` only**. The
> table above was never extended, so the charter has read "the set is L1–L6" through every session
> since. **The rule that catches an unlanded rule was the one not applied to itself.**
>
> **Their full text — incident, evidence, enforcement point, competing pressure — is in `AGENTS.md`
> and is authoritative there until folded.** Do not paraphrase these rows into a working summary:
> read the entries (D12). **The fold of L7–L25 into §1 body sections is owed work**, and is the kind
> of debt that is invisible precisely because both documents are individually coherent.
>
> **`AGENTS.md` is the authority for the rule bodies; this table is the authority for the SET.**
> Stated because L23's fourth shape is exactly the failure of an unmarked copy: a reader of either
> document must be able to tell which half they are holding. Neither said so until now.

---

## 1. The disciplines

Each carries: the invariant · why · the source incident `(repo, commit, date)` · **the enforcement
point** · **the competing legitimate pressure that will eat it.**

> **The mechanism, before the rules.** `entity-core-go`, 2026-08-12: **"a legitimate pressure ate a
> discipline, and nothing in the process noticed."** Arch's August window is that shape — cohort
> findings arrived faster than ratify-and-fold consumed them, and the discipline that got eaten was
> proposal-first. **It degraded under load; it was not absent.** A rule that only survives when
> nothing is urgent is not a rule, which is why every entry below names its competing pressure.

### L1 — No normative spec edit without a proposal `[RATIFIED]`

**Invariant.** A change to normative text is proposal-first. Wording-only hygiene may go direct.
**Cohort impl findings fix the spec in place, no rev bump** — that carve-out is real and stays.
If the change is normative and no proposal exists, write one first.

**Why.** The proposal is where the *why* lives. **A fold with no proposal has nowhere to put the
rationale, so the rationale goes into the spec text** — L5 is downstream of L1 and cannot be fixed
independently of it.

**Incident.** `EXTENSION-REVISION` v3.11, 2026-08-15: folded with **no proposal in existence** —
two new MUSTs, one a capability-authority rule, plus a rev bump, outside even the cohort-finding
carve-out. **Not one incident:** `PLAN-2026-08-15-spec-provenance-reconstruction.md` measures **50
commits touching `specs/` since 2026-08-01, 34 with no proposal, 25 of those inside six days**
(method in its §5, re-runnable). Ratified on the measurement, not on the acute case.

**Enforcement.** **The one rule in this set with no mechanical rung** — a corpus linter checks
*text*, and L1 is a property of the *change*. The gate is specified in §4 and is **not yet built**.
Until it is, L1 runs on the pre-edit checklist in `AGENTS.md` and on review.

**Pressure.** *Throughput under a cohort backlog.* A proposal feels like ceremony when the finding
is obviously correct and the fix is three lines.

### L2 — Read the filing seat's own document, never a peer's summary `[RATIFIED]`

**Invariant.** Before ruling on a routed item, open the filing seat's own spec-issue in their tree.
A sibling's summary, a routing packet's restatement, our own queue row, and our own earlier note
are all **derivative artifacts carrying their author's framing**.

**Why.** Summaries are lossy in a specific direction: **the items after the main finding are the
ones that get dropped.** A filer writing *"two smaller points that travel with it"* is attaching a
warning label.

**Incidents — three, different shapes.**

1. v3.11 ruled **two of seven** items from a formal peer spec-issue that was on disk and unread.
2. **2026-08-15, `E-R5`.** The queue carried it as *"the vector's rendezvous key MUST be
   floor-formed"* — one bullet. `entity-browser-rust`'s own routing document
   (`ROUTING-2026-08-15-the-3b31-oracle-does-not-discriminate-and-its-k-is-not-a-key.md`, read at
   browser-rust `8fdd635`) filed **two** defects, and the retraction had closed only one. The
   summary was not wrong so much as **finished-sounding**.
3. **2026-08-15, `D-§5.3`.** The queue named it an `EXTENSION-ENCRYPTION` section. It is **arc D's
   proposal §5.3** — the item already had a home, a recommendation, and a history. Ruling from the
   queue row would have re-derived all three.

**Enforcement.** A proposal-template field — **"Sources read (repo, path, commit)"** — listing the
filing seat's own document, or stating explicitly that none exists. Reviewable, and **greppable for
absence**, which is the property that matters.

**Pressure.** *A good summary is genuinely faster and usually right.* The 5-of-7 it silently drops
is the cost, and it is invisible at the moment you accept the summary.

### L3 — A partial fold does not get a completeness marker `[RATIFIED 2026-08-17]`

**Invariant.** A **completeness marker** signals to every downstream consumer that *the area has been
dealt with*. There are two, and the rule binds both: a **version bump** on the spec, and a **move to
`docs/proposals/implemented/`**. If a fold lands some deltas and not others, either complete it, or
withhold the marker and name the open items where the next reader will meet them.

**Why.** The fold being partial was survivable. **The marker is what made it invisible.**

**Incident 1 — the version bump.** `EXTENSION-REVISION` v3.11, 2026-08-15. Two of seven items from a
peer's spec-issue ruled, then a rev bump.

**Incident 2 — the `implemented/` move, and it is a different marker with a measured cost.**
`[2026-08-17]` `PROPOSAL-PUBLISHED-ROOT-PREFIX-AND-REPUBLISH` §7 declares seven deltas. **D1, D2, D3,
D5, D6 and D7 landed. D4 did not** — *"replace the `(planned)` pointer to
`PROPOSAL-PEER-MANIFEST-STATIC-HANDSHAKE` with the §3.3a citation; same at the `MANIFEST_GET` body
MUST."* The proposal was filed under `implemented/` regardless.

**What the residue cost.** `PROPOSAL-PEER-MANIFEST-STATIC-HANDSHAKE` **has never existed in any
repo** — searched by filename and by content across the whole checkout. Three normative sentences in
`EXTENSION-NETWORK` hand it obligations, including the `MANIFEST_GET` body MUST's revocation
primitive and the Amendment-10 signed-root closure. **Both app-tier seats then built a static
publishing surface against it**: one asked arch whether their reading matched it, the other shipped a
different reading — a `MANIFEST_GET` serving a transport-profile entity where the spec requires a
signed root. Neither could have found the problem by reading, because the pointer looks exactly like
a pointer to a document that exists.

**Why this ratifies rather than adding a tally mark.** The ladder's bar is *a second incident in a
different shape*. The invariant is the same — a completeness marker on an incomplete fold — and the
**marker is different**, which is what makes the generalization real rather than assumed. It also
inverts the visibility: v3.11's bump was on the artifact a consumer reads, while `implemented/` is on
the artifact only *we* read, so the residue was invisible to the cohort **and** to us, and surfaced
only when a downstream seat built on it three months later.

**Enforcement.** Two rungs, one existing and one owed.
- *Existing, reviewable:* pair with the L1 gate — a completeness marker names a proposal whose delta
  list is fully landed.
- *Owed, and mechanically checkable:* a `spec` rule that reads a proposal's §7 delta table and
  verifies each `(file, section)` row against the tree before permitting the `implemented/` move.
  The delta tables are already tabular and already name file + section; this is the same shape
  `sdksync` solved for restatements. **Filed, not built** — recorded here so the rule is not
  mistaken for enforced.
- *Landed this session, partially covering the symptom rather than the cause:* arch-tools `46c6e50`
  makes `spec address` report a document **named** in normative text that exists nowhere (new
  `absent-doc` class), and implements `SPECIFICATION-FORMAT` §11.4's second clause so a `(planned)`
  marker no longer excuses a forward reference inside a MUST. That catches *this* residue's
  signature; it does not catch an unlanded delta in general.

**Incident 3 — the declaration the fold argued against and never edited.** `[2026-08-19]`
`PROPOSAL-DEFAULT-NAME-FORMAT-DISPATCH` Addendum 2 closed the `name_format_dispatch` pattern grammar
to `*`-only, wrote *"implementations MUST NOT delegate this to a path-glob or shell-glob library,"*
shipped `REG-DISPATCH-GRAMMAR-1`, and bumped **REGISTRY 1.12 → 1.13**. It left `pattern: <POSIX
shell-glob>` standing in the §4 schema, twenty lines above — plus two more POSIX references, one in
`§4.1a` and one in `GUIDE-RESOLUTION` §6. The parent proposal's scope line, *"Not in scope: the glob
grammar itself (§4's POSIX choice stands),"* was never revisited when the grammar was closed.

**Why it is L3 and not a typo.** The version bump said *this area has been dealt with*, and the area
still contained the exact sentence that produced the wrong reading. `entity-core-go` reached for
`path.Match` — **the correct reading of what the schema declared** — and the cohort spent a cycle on
a divergence plus one pulled conformance check. **A fold that lands the argument and not the
declaration is a partial fold**, and the declaration is the half that implementers read first:
prose MUSTs are what a *reviewer* reads, and a field's type annotation is what an *implementer*
reads. The bump made the residue invisible in the direction that matters.

**The generalization this adds:** when a fold changes what something *means*, the delta table must
include **every site that declares it**, not only the sites that argue about it. Grep the changed
term, not the changed section.

**Pressure.** *A marker is how progress is signalled.* Withholding it at 6-of-7 feels like
under-reporting a fold that was, in every other respect, done.

### L4 — A claim about a document or a tree is checked by opening it `[RATIFIED]`

**Invariant.** This is one rule with two faces, and **consolidating them is deliberate** — the
failure mode is identical and only the target differs.

- **L4a — outward, build state.** Any claim about what an implementation has, has not, or no longer
  has built **decays the moment that repo commits**. `git status` + HEAD, then read the actual
  source; `git log --since` on every sibling the claim touches. Cite **`(repo, commit, date)` at
  the point of claim**, with how it was observed (source-read / peer-reported / measured). **Never
  carry a dated measurement across a commit change.** A peer's correction outranks your inference,
  immediately.
- **L4a-iii — never carry a seat's characterization of a *third* seat.** A report's table about its
  peers is the report author's reading, not the peer's tree. **Arch published py as non-conformant on
  the v3.23 ruling from go's table; py's evaluator had been on the ruled side the whole time**, and
  py's own handoff said so. **Cost: a wrong non-conformance claim, sent.** The rule is not "distrust
  the filer" — go's reading of *their own* tree was exact — it is that **a seat is authoritative
  about itself and derivative about everyone else.**
- **L4b — inward, our own corpus.** A claim that something is **unruled, unbuilt, or owed** is a
  claim about a document, and it is checked by opening that document. **Peer trees get read
  carefully; it is the local file and our own last commit that get assumed.**

**Sub-rule, because arch routes constantly.** *(from `entity-core-go`, 2026-08-15)* **An obligation
is only routable onto a surface the target actually serves.** Before routing *"MUST appear on both
surfaces,"* confirm the sibling serves the second surface — **`git grep` the listener, not just the
field.** go routed `reflection_endpoints` at both the wrapped and the §9.2 unwrapped `advertise`;
**both peers corrected it — neither serves the unwrapped surface.** The field's absence was
verified; the *surface's existence* was asserted without opening the tree.

**Incidents — the most-repeated defect in the arch record.**

- **Outward, four:** 2026-07-28, 07-31, 08-07, 08-08 — **every one caught by a peer, never by us.**
  The 08-08 case cited an 08-07 report while the commit refuting it, titled *"republish the signed
  root on every tree-root change,"* was already an **ancestor of the very commit that report
  measured**.
- **Outward, 2026-08-15, twice more.** The post-fold validation ledger filed rows `C-1`/`C-3`/`C-4`
  🔴/🟡 from a **2026-07-16** measurement; re-taking before sending found `entity-core-rust` had
  already fixed `C-1` and all three had built `C-3`/`C-4`, converged. **Two seats would have been
  named non-conformant for work they had done.** Later the same day, a ledger row about the §4.4
  resolver was written and then re-taken: **all three cohort HEADs had moved** (go `2df96f8`→
  `2f4f6a2`, rust `462f2c2`→`0b4e0cd`, py `f33526f`→`c0de5c2`), voiding the pins it was about to
  carry. Readings agreed; the pins did not.
- **Inward, five — and the fifth refines the rule.** On 2026-08-16 a routed item arrived framed as
*"the case is under-pinned"*; **our own §4.1 carried both behaviours in sequence**, and the real gap
was an undefined predicate one level below the framing. **The refinement: L4b is not only "is this
already ruled?" but "is the routed framing where the defect actually is?"** A filing seat reports
from where it *hit* the defect; the corpus shows where it *lives*, and those differ often enough to
check every time.

**Inward, four:** `F-E3` reported unruled while §9.2 already ruled it **at a verified ancestor
  commit** · `NAMESPACE-CLEANUP` listed S2 as owed when S2 was folded · the same proposal sat
  *"PARTIALLY EXECUTED"* for two weeks with **every spec edit already landed** · **2026-08-15,
  `D-§5.3`**: the queue said *"needs a ruling"*; **half of it had already been ruled, by us, in
  `EXTENSION-IDENTITY` §4.2's registered-function row at `dbb4faf`**, and nobody connected it back.

**Why prose is not enough.** It has been written into `AGENTS.md` at length and it still recurs.
**The stale claim is always plausible** — it was true recently, came from a real measurement, and
**passes every review that does not open the tree.** Plausibility is exactly why review does not
catch it.

**Enforcement.** Structural, not attentional: the `(repo, commit, date)` pin is required **inline
at the claim**, and an unpinned build-state sentence in `ROUTING-*` / `HANDOFF-*` is a lint finding
(§4, companion rung — **not yet built**). The **POST-FOLD VALIDATION LEDGER** in
`docs/status/WORKSTREAMS.md` is the running rung that exists today (consolidated to four workstreams 2026-08-17; the pre-consolidation snapshot is in `docs/archive/status/`), and its standing rule is: **if
the reading is older than the seat's HEAD, it is re-taken first or it is not sent.**

**Pressure.** *Re-opening four sibling trees to re-verify a claim you are confident in is pure
friction with no visible payoff* — until the fifth peer correction.

### L5 — Spec text is not our log `[RATIFIED]`

**Invariant.** A specification is implemented by someone who has never heard of this cohort. It
does not name `entity-core-go`, quote which seat argued what, carry `[ruled 2026-08-15]` stamps, or
cite commit hashes. **That material is rationale, and rationale lives in the proposal.**

**Why.** It is the **second-order failure of skipping L1**: no proposal → the rationale has nowhere
to go → it goes into the spec.

**Incident.** 489 findings across 30 normative specs, held in `.spec-baseline.json` across five
rules (`impl-team-ref`, `date-in-body`, `proposal-citation`, `amendment-provenance`,
`document-history-section`). Live instance, 2026-08-15: the gate **refused an Amendment 12 fold
banner** carrying impl names, dates and commit pins. **The fix was to move the sentence into the
proposal, not to touch the baseline** — and the spec got shorter.

**Enforcement — the strongest rung in the repo, and the pattern to copy.** All five rules at
**error**; existing debt held and non-gating; **anything new gates**; `--update-baseline` **only
ever lowers and refuses to raise**, so re-baselining to green is not available.

> **The generalizing sentence, carried here from the baseline's own header because it applies to
> every gate this repo will ever add: *a WARN in a green run is a claim nobody reads.***

**Pressure.** *When there is no proposal, the spec is the only place the reasoning fits.*

### L6 — Resolve divergence from the table, before the fix is written `[CANDIDATE]`

**Invariant.** When a cross-impl divergence is measured, the response is chosen from
`GUIDE-CONFORMANCE` §4's table **before** a fix is written, and the row is named. **All three
differ = spec ambiguity; tighten §§3–9 in the same pass.** Two-and-two = **spec arbitrates; do not
vote.** Arch's own half: treat an inbound *"we converged on X"* as **an ambiguity to tighten**, not
a fait accompli to ratify.

**The tell, worth memorizing.** *A fix whose justification includes a count of implementations is
the smell.*

**Incident.** `entity-core-go`'s F-1, 2026-08-12: §6a.9's `202 pending_review` measured
all-three-differ; go adopted the majority carrier.

> **Promotion evidence arrived 2026-08-16, and it is the *positive* case.** `entity-core-go` met a
> clean all-three-differ divergence (`compute/error` in a construct field), **routed it rather than
> adopting a majority**, and cited `GUIDE-CONFORMANCE` §4 by name. Arch's half then ran for the first
> time: the response was **a general pin plus four consequential edits in one pass**, not an A-or-B
> verdict — which is what "tighten §§3–9 in the same pass" means operationally. **This is compliance,
> not an incident, so it does not itself promote L6** — the ladder promotes on what we have paid for.
> **What it does establish is that the arch-side half is runnable**, which was the open question.

**Why it is a candidate.** **The only measured incident is in a peer's tree, not ours.** The routed
draft argues that arch owns the discipline because arch owns arbitration — true for the *rule*, but
the ladder promotes on **incidents we have paid for**, and arch has not yet been measured violating
its own half. Adopting it as ratified would import a peer's evidence as our own, which is the
overclaiming this charter exists to prevent. **Applied as a candidate; promoted the first time arch
ratifies a convergence it should have tightened.**

**Watch item, recorded now so the second incident is recognizable.** On 2026-08-15 arch folded
`CONTINUATION` v1.23 largely as *"writing down what the cohort had already converged on."* That was
the correct call — the convergence was **on identical spellings citing arch rulings that predate
the fold** — but it is the exact shape L6 warns about, and the distinction (codifying a convergence
vs. ratifying one that papers over an ambiguity) **is the judgment L6 exists to make explicit**.

**Pressure.** *Round-trip reduction* — the same pressure that ate it in go. **This one is arch's own
interface**: every ambiguity tightened is a round trip a peer absorbs, so the pressure is real on
both ends and cannot be wished away.

---

## 2. What is deliberately NOT in this set

**Trap 2 of the routed package: inflating the set because assembling it feels productive.** Six is
a good size; fifteen means most were promoted on one incident and nobody runs it.

- **"Check our own corpus before recording an item as owed" was NOT added as a seventh rule.** It
  is the same failure as L4 with the target reversed, and the routed package's own instruction was
  *harvest, don't author*. It is **L4b**, not L7.
- **The candidates rejected outright: none.** All six were adopted; two as candidates rather than
  ratified, with reasons above. **A rejected candidate would be recorded here with its reason** —
  an unexplained disappearance is drift.

---

## 3. Anti-pattern catalog

**Numbered, never renumbered, never deleted.** A retired entry is marked with the commit that
retired it. Each is something already paid for.

| # | Anti-pattern | Source |
|---|---|---|
| **AP-1** | **The partial fold with a version bump.** Two of seven items ruled, then a rev bump — which reads downstream as *this area has been dealt with*. The partial fold was survivable; the bump made it invisible | `EXTENSION-REVISION` v3.11, 2026-08-15 |
| **AP-22** | **The blacklist built from an implementation census.** `0.8.2.7`'s §9.1 row enumerated five 501 spellings as *"non-conformant synonyms"* of `unsupported_operation`. Four were. The fifth, **`unsupported_mode`, is normatively MUSTed by `EXTENSION-REGISTRY` §6a.9.2** — a deliberate pin (*"a cross-impl-observable answer with four plausible codes, so it is pinned rather than left to converge"*) — for a **different failure**: the handler is registered and `register` **is** implemented, and the refusal is about the *mode of a stored policy*. **Arch blacklisted a token the corpus requires, and put two seats in conflict with an extension spec.** The census was correct — rust does emit it — and that is the trap: **a census tells you what peers emit; it cannot tell you which of those the corpus mandates.** Inferring *minted* from *appears in a tree* is L8 with the census as the artifact, and rewriting the synonym set without grepping the corpus for each token is **L23** — every token in a blacklist is a rule with homes. The tell is that a blacklist is a **negative about the whole corpus** (*"nothing defines this"*), which is exactly the claim `AGENTS-STANDARD` says to prove and nobody does, because a list of tokens does not read like a claim. **Before a token is named non-conformant, grep `specs/` and `guides/` in both repos for it — a one-line check per token — and where it is defined, the test is the FAILURE NAMED, never the status shared** | `ENTITY-CORE-PROTOCOL` §9.1 at `0.8.2.7`, caught by `entity-core-py` (SA-PY-37) and routed by `entity-core-go` as B2; corrected at `0.8.2.8` |
| **AP-21** | **The token count published as a distribution — and the sweep scoped by the token instead of the slot.** `0.8.2.6` gave §3.3's 500 row a default code and justified it **in normative spec text**: *"three implementations had independently converged on `internal_error`."* The evidence was `grep -c internal_error` in three trees — **184 · 27 · 18, published as "229 sites, converged."** Censused by **slot** (every `code` emitted at status 500), `entity-core-go` carries **16 spellings** — 183 `internal_error`, **58 bare `internal`**, 60 across 14 others — and rust and py likewise, so the claim is **false about all three**, not only about the seat that caught it. **A `grep -c X` is a fact about X and contains no information about what else occupies X's slot**, which is the entire content of a convergence claim. *"N implementations converged on X"* is a positive-sounding sentence whose truth condition is a **negative**, so the *prove a negative* rule never fires: it reads as measured, and it cites a number. **It recurred inside its own remedy** — the sweep the same fold ordered retired `unknown_operation` from the **501** slot and reported the class closed, while `not_implemented` survives in all three ground-up trees and four generated peers, beside `not_supported`, `unsupported_mode`, `not_available`. **A token-scoped sweep reports done while the slot stays divergent.** Sibling of AP-15 (*measure the gate with the gate*) and of AP-2 (*the plausible stale claim*), and it inherits AP-5's aggravation: the sentence entered the spec as **rationale**, which is the disguise that got it past the *spec text is not our log* rule. **Group by the field and publish the whole distribution; scope a sweep to every value at the site that is not the ruled one.** **SECOND SHAPE 2026-09-03, next day, three axes at once:** the slot arch censused was `(status, spelling)` and **a code is `(status, FIELD, spelling)`** — rust wrote 14 codes under key `type` via a shadowed helper, so a correct-spelling fix was invisible on the wire and every census counted it closed; arch's 500 census read **3** where the tree had **19**, missing 16 sites behind a second `const STATUS_INTERNAL` alias (**OP-3 was sized off that census**); and the taxonomy proposal's *"go and py have not built these handlers"* was false since go's **v0.8.0 initial release**, at the very commit it cites as searched, because both searches used **the vocabulary of the seat already read**. *A census keyed on a NAME for a thing is blind to every other name for it — key on the VALUE at each axis, and discharge an absence by naming the bounded region searched, never by a token grep* | `ENTITY-CORE-PROTOCOL` §3.3 500 row at `0.8.2.6`, caught by `entity-core-go` spec-issue `2026-09-03-a`; struck at `0.8.2.7`. Second shape: `entity-core-rust` `f0a399b`/`91397d2`, surfaced by go's harness `c2ff3a2` |
| **AP-20** | **The capability claim with no gate — a conformance apparatus that scores refusal cannot see what its artifact cannot be extended with.** `entity-core-keystone` states in **four published places** that *"the community installs [the standard extensions] atop a generated peer"* and that *"the extension surface stops at the dispatcher interface."* **Nothing has ever tested it.** No phase gate installs a handler, no `validate-peer` category asks, and no peer has ever had an extension installed on it — so the one property the next tier depends on sits exactly in the apparatus’s blind spot. By source read, the first **five** peers answer **three** different ways (`csharp`/`typescript` expose a public bind-a-body API — `Peer.RegisterHandler` / `Peer.registerHandler`; `go`'s `Peer` exports four unrelated methods; `rust`/`haskell` expose only the §6.13(a) *wire* operation), and the remaining 41 carry no verdict because nothing counts them. **Keystone’s own vacuous-green family (§4.8, *"the oracle passes a peer that doesn’t do the thing"*) with the polarity turned outward:** there the gate cannot see a missing behaviour, here it cannot see a missing capability, because a capability is a claim about what someone *else* can do with the artifact. Sibling of AP-8. **The habit it earns is one question at publication: *which sentence here is a claim about what someone else can do with our artifact, and what measures it?*** | `entity-core-keystone` `README.md` / `AGENTS.md` / `PROJECT-RETROSPECTIVE.md` §10 / `shared/lifecycle/PROMPT-CONSTANTS.md`, found 2026-09-01 opening the `entity-system-generator` track |
| **AP-19** | **Fixing the checker to agree with the emitter you wrote in the same session.** `entity-browser-rust` emitted the signed root at `…/published-root.bin`, crossing §6.5.3's *"singular/terminal: no suffix"* `MANIFEST_GET` route with §6.5.3.1's `{path}{tree_leaf_suffix}` hash-pointer route — a 3-key wire entity where a conformant consumer expects a 2-key pointer. **Their own `--verify` said "BROKEN — not a hash pointer," and the first fix excluded the path from the sweep.** Three commits before the emitter was suspected. **A checker and the thing it checks are not symmetric evidence:** the checker encodes a reading of the spec taken deliberately, the emitter encodes one taken while solving a different problem — so on disagreement, **the emitter is the more likely defect and carries the burden of proof.** Sibling of AP-8 (*unscanned reads as zero*) with the polarity flipped: there a gate that could not look reported a pass; here a gate that looked correctly was taught not to. **Self-reported, which is the only reason it is available to anyone else** | `entity-browser-rust` `664b36b`, reported against themselves 2026-08-18 |
| **AP-18** | **The phantom document — a normative sentence that hands an obligation to a file nobody wrote.** `EXTENSION-NETWORK` §6.5.3.1's `MANIFEST_GET` MUST deferred its revocation primitive to `PROPOSAL-PEER-MANIFEST-STATIC-HANDSHAKE`; §6.5.6's closure MUST cited it `(planned)`. **It has never existed in any repo.** A pointer at a phantom is indistinguishable from a pointer at a document you have not opened, so **it reads as a deferral instead of a hole** and no amount of care by the reader recovers the difference — L4 says open the document, and there was nothing to open. Two app-tier seats built one surface against it and reached two incompatible shapes. Sibling of AP-8 (*unscanned reads as zero*) in the corpus rather than a gate: **an unwritten document reads as an unread one.** Gated since arch-tools `46c6e50` | `EXTENSION-NETWORK` §6.5.3.1 / §6.5.6, found 2026-08-17 via browser-rust's Q1 |
| **AP-2** | **The stale build-state claim.** Always plausible: true recently, from a real measurement, passing every review that does not open the tree | Four instances 07-28 → 08-08; two more 2026-08-15 |
| **AP-3** | **The obligation routed onto a surface the target does not serve.** The field's absence was verified; the *surface's existence* was assumed | `entity-core-go`, 2026-08-15; both peers corrected it |
| **AP-4** | **Majority-vote resolution of a three-way divergence.** All-three-differ is a spec ambiguity, and counting implementations is the smell | `entity-core-go` F-1, 2026-08-12 |
| **AP-5** | **Rationale in the spec because no proposal existed.** One error produces the other; the second is the one that survives review | v3.11, 2026-08-15; 489 baselined findings |
| **AP-6** | **The retraction that discharges the instance and leaves the class.** Withdrawing the bad artifact closes the reported defect and removes the evidence, and the section then *reads* as closed while the generator can reproduce it | `REGISTRY` §3b.3.1, retracted `57c1ab6`, class closed `80d3ca2` |
| **AP-7** | **The item recorded as owed that our own corpus already ruled.** Four instances; the ruling is usually in a *different document* from the one the item names | `F-E3` · `NAMESPACE-CLEANUP` S2 · `D-§5.3`, 2026-08-15 |
| **AP-8** | **Unscanned reads as zero.** A gate that could not look reporting as a pass; a count-checker whose pattern stopped matching reporting green; a ratchet wiping entries for files it never read | arch-tools `7fd538f` (exit-2), `C11`, `spec ledger` `df28037` |
| **AP-9** | **The count that drifted because it was written twice.** One number updated, the other not — three times in one file, twice inside one day | `docs/proposals/INDEX.md`; gated since arch-tools `df28037` |
| **AP-13** | **A gate whose subject changed when the ruling changed.** The vector named *"materialized-into-construct"* kept its name, ran green three-way, and **now proves the opposite behaviour** — propagation, not materialization. A gate is defined by what it *exercises*, not by what it is *called*, and a ruling that moves the behaviour **invalidates every gate phrased in terms of it.** Re-derive the gate in the same pass as the ruling | `PROPOSAL-COMPUTE-ERROR-MATERIALIZATION` §2.4, 2026-08-16 |
| **AP-14** | **One value, two in-language representations, and tests for only one.** A `compute/error` arrives **minted** or **as a value** (SA-1). Coverage of one is **0% coverage of the other**, and two implementations independently reached ~100%/0% and could not see it. **A green board proves the tested representation, never the type** | go + py, measured 2026-08-16; go's own ratchet |
| **AP-11** | **Unreachable text that two implementations implemented.** §4.1's construct branch listed `compute/error` in a materialization set the guard above made unreachable. **Dead pseudocode is not inert** — it reads as licensing, and the reader who starts at the bottom of the block implements it. Three seats, one block, two orders | `EXTENSION-COMPUTE` §4.1, ruled v3.23, 2026-08-16 |
| **AP-12** | **Citing the report of the work instead of the work.** A build-state claim pinned to the status document *announcing* a build rather than the commit that *is* it. The conclusion survives; the evidence pointer does not, and the next reader who follows it finds docs | `entity-core-go`'s §4 citing py's `a6140a8` (docs-only) for a collector that is `5fbb8f3`, 2026-08-16 |
| **AP-17** | **Counting front ends as peers.** The ledger listed five seats — go, rust, py, browser-rust, workbench-go — and reported the P2P evidence base as four-of-five covered. **It was one lineage.** `entity-browser-rust` links `entity-core-rust`'s `entity-{peer,network,signaling}` **by path**; `entity-workbench-go/entitysdk` links `entity-core-go` by `replace`. **A front end cannot validate the core it compiles against**, so every "browser-rust found X" on a P2P surface is `entity-core-rust/core/peer` reporting on itself — the tell being that their §13 item 6 remedy landed *in `entity-core-rust`*. Arch then folded that evidence as general MUSTs and cited the resulting spec back as authority: **an echo, not adjudication.** [ADR-0012]'s cohort-consistency warning, applied to ourselves and missed — and the disproof was two dependency files we had never opened. **Count code bases that can disagree, per seam. Two front ends over two *different* cores are a real N=2 at the app tier; a front end and its own core are one** | W-EVIDENCE census, 2026-08-16; raised by the operator, not by us |
| **AP-16** | **The hygiene edit that launders a stale claim.** §13 item 6 was rewritten to strip seat names and dates (L5). The edit was *scoped* as cosmetic, so the content it **preserved** was never re-validated — and that content was three claims the filing seat had **withdrawn the day before**, in a document committed to their tree at an **ancestor of the very commit we read**. The stale row then went out **twice more**: quoted back to them in a routing packet as their position, and copied into a queue row as *"the filing seat stated the precondition up front."* **Touching a line makes you its author.** A rewrite re-attests every claim it carries under a fresh commit, and it strips the one signal a reader had — that the text was old. **"I was only reformatting" is not a scope; it is how a dated claim gets a new date.** Sibling of AP-2: the stale claim is plausible because it *was* true, and here we added a fresh timestamp to it | `EXTENSION-SIGNALING` §13 item 6, laundered in `b92efe0`, caught by the seat 2026-08-16 |
| **AP-15** | **Counting a gate's findings by hand instead of asking the gate.** Q7a was planned against *"12 occurrences across 3 files."* The corpus held **11 textual occurrences on 10 lines**, and the number that governed the work was **7** — because the analyzer emits **one finding per line**, and only for lines **not already flagged by another arm of the same rule**. Two lines already named an `entity-core-*` seat, so widening the pattern changed nothing there. Three defensible numbers describe one corpus, they differ by 40%, and **the paydown plan is only correct for the one the gate actually emits.** A `grep -c` counts lines and gets reported as "occurrences"; a `grep -o` counts occurrences the gate does not use. **Sibling of AP-8 pointed inward:** there the gate could not look, here we did not ask it. **Measure the gate with the gate** | `Q7a`, planned 2026-08-16 against `5996aa4`, corrected at execution |
| **AP-10** | **A measurement quoted without the window it was taken over.** *"The gate's first run flags two commits"* was true — over an unstated 6-commit window. Over the backlog window it is **66 findings across 40 commits**. The number was right and the range was load-bearing and missing, so the reader inferred a smaller problem than exists. **This is AP-2's shape applied to a gate's own output:** plausible, correct when taken, and it survives review because nobody re-runs it wider. `ADR-0012` already forbids a bare conformance percentage for the same reason — **a provenance count without its range is a conformance number without its oracle pin** | Reported by meta, 2026-08-15, against arch's own report of `718f0d5` |

---

## 4. Enforcement — what exists, and the two gates that do not

**A discipline with no enforcement point is theater.** This table is the honest state.

> **Scoped by the operator, 2026-08-21, because arch was over-applying it.** That sentence is a
> pressure to *build the gate*, and it is **not** a licence to treat an unenforced rule as worthless
> or to drop one for lacking a gate. *"Just because we haven't figured out the way to enforce it
> doesn't mean it's not critical that we have it in our disciplines and our doctrines… at least when
> we do audits, when we review, we can say we did actually know this and we just didn't follow
> what's there."*
> **An unenforced discipline still catches some of the time** — it is read, it is cited in review,
> and it converts an unknown failure into a **known one that was skipped**, which is a far better
> position to audit from than not having written it down. **Unenforced is a cost, not a nullity**, and
> the `ENFORCED` column below records that cost rather than disqualifying the row.

| Rule | Rung | State |
|---|---|---|
| L1 | **`spec provenance --since REF`** — a normative spec edit must name a proposal or declare a carve-out | **BUILT at warn**, arch-tools `718f0d5` |
| L2 | Proposal-template "Sources read (repo, path, commit)" field | **NOT BUILT** — one template edit, and it is L1's evidence |
| L3 | Bump names a proposal with an empty open-item list | Reviewable; rides on L1's gate |
| L4a | `(repo, commit, date)` pin required inline; unpinned build-state sentence in `ROUTING-*`/`HANDOFF-*` is a lint finding | **NOT BUILT** (companion rung); the POST-FOLD VALIDATION LEDGER is the live rung |
| L4b | Open the named document before recording an item as owed | Checklist; no mechanical rung, and probably cannot have one |
| L5 | `.spec-baseline.json` — five rules at error, ratcheted, refuses to raise | **BUILT, strongest in the repo** |
| L6 | Commit-message convention naming the §4 row | Convention only |
| — | `spec ledger` — declared counts vs their directories | **BUILT**, arch-tools `df28037` |

**Two declared trailers are now part of the commit vocabulary**, and they are the only way to claim
an exemption:

```
Spec-Change: hygiene          wording-only; no normative change
Spec-Change: cohort-finding   an impl finding fixed in place, no rev bump
```

### 4b. The L1 backlog window, and what promotes the gate

**The window has a name, because "a clean run over the backlog" is otherwise a range someone
re-chooses.**

| | |
|---|---|
| **Base ref** | **`6de519d`** — the last commit before 2026-08-01, the start of the measured window |
| **Debt at ratification** | **66 findings across 40 distinct commits and 20 files, over 136 commits since `6de519d`** *(taken 2026-08-15 at arch `bbed24b`)* |
| **Command** | `make -C <arch-tools> provenance CORPUS=$(pwd) SINCE=6de519d` |
| **Promotion** | flip `normative-edit-without-proposal` to `error` in `[analyzer.provenance.rules]` |

**Promotion condition — both halves, and neither alone:**

1. the reconstruction ledger's **7 reference proposals have landed**, and
2. **a run over `6de519d..HEAD` reports 0 warnings.**

**Not before.** Turned on retroactively against 66 findings it fails on contact and is off within a
day — which is the failure the ratchet exists to avoid.

> **Why 66 and not the reconstruction plan's 34.** Different unit and different method, and the two
> must not be compared as if they were one number. The plan counted **commits** whose message named
> no proposal, measured at `cec21fa`. The gate counts **(commit, file) pairs** that *trip a trigger*
> and name no proposal **that resolves on disk**, at `bbed24b` — a stricter proposal test over a
> window that has since grown. **40 distinct commits** is the comparable figure. Neither supersedes
> the other; the gate's is the reproducible one, and it is the one the promotion condition uses.

**The trigger for the promotion itself lives with the work, not here.** `docs/status/WORKSTREAMS.md`
carries the burn-down row, so **the last reference proposal to land is the one that runs the check
and flips the gate** — a condition nobody is watching is a condition that does not fire.

### 4a. The L1 gate — specified, with a correction to the routed sketch

**The routed sketch keys on the `**Version**` header changing.** Adopted with one substantive
change, because as sketched **it would have been silent on both of arch's normative folds on the
day it was written.**

> **Measured, not argued.** `80d3ca2` (`REGISTRY` §3b.3.1, +2 MUSTs) and `40586c5` (`ENCRYPTION`
> §4.4, +4 MUSTs) each added normative requirements and each changed **zero** version headers —
> **correctly**, under arch's standing *"cohort impl findings fix the spec in place, no rev bump"*
> carve-out. **A version-triggered gate misses precisely the class arch uses most.**

**So the trigger is two-part, and either fires:**

1. the `**Version**` header of a `specs/**.md` changed relative to the base ref; **or**
2. a `specs/**.md`'s **count of normative tokens** (`MUST`, `MUST NOT`, `SHALL`, `[MUST]`)
   **changed** — added **or removed**; retiring a requirement is as normative as adding one.

> **Trigger 2 counts the whole file, not the diff's `+` lines.** The `+`-line form reports every
> reworded sentence that merely *contains* a MUST as a new requirement, so an editorial touch-up
> fires the gate — caught by the module's own self-test before it shipped. **Residue, stated
> because a gate should name what it cannot see:** a rewrite swapping one MUST for a different MUST
> nets to zero and is invisible.

Requirement, on either trigger: the change **names a proposal that exists** anywhere under
`docs/proposals/` — the gate globs the tree, so `deferred/` and `superseded/` satisfy it too, which
is correct: the question is whether the decision was written down, not what became of it — in a
commit-message trailer or the diff.

Design notes, so it does not become the gate everyone switches off:

- **Three-valued exit** — `0` clean, `1` violation, **`2` could-not-look** (shallow clone, no git
  dir, unresolvable base ref). Learned the expensive way in `7fd538f`; **a gate that cannot see must
  never read as a pass.**
- **Ratchet it, don't flip it.** Land at **warn** — turned on retroactively against the 34-commit
  backlog it fails on contact and is off within a day. Promote to error once the reconstruction
  ledger burns down. Same play that worked for the five `standards` rules.
- **A declared, greppable carve-out, not a judgement at the gate.** The hygiene class (~7 of the 34)
  is legitimately proposal-free: a commit trailer `Spec-Change: hygiene`. **An exemption nobody can
  audit is not an exemption.** The cohort-finding carve-out needs its own trailer for the same
  reason — otherwise trigger 2 fires on every in-place fix.
- **Buildable with machinery arch-tools already has:** `convergence.py` shells to git (`_git`,
  `repo_is_measurable`); `standards.py` parses the version header (`header_field`); `docs/proposals/`
  is machine-readable by directory (`INDEX.md` §0 — **state is the directory, tier is the
  subdirectory**).
- **It is arch-tools work, and arch-tools is ours** — same session, own commit, own tests.

---

## 5. Sequence

1. ~~Charter, catalog, doctrines adopted, `AGENTS.md` synced~~ — **this file**.
2. ~~**L1 gate at warn** in arch-tools~~ — **landed `718f0d5`**, `make test` green. **Its first run
   flagged two commits, both arch's own, both correctly**: `80d3ca2` states in prose that it is a
   cohort finding and now owes the trailer; `40586c5` updated a proposal that exists and never
   named it. **A gate whose first run indicts its author is the only kind worth having.**
3. **The L2 template field** — one edit to the proposal template, and it is L1's evidence.
4. **Then** the reconstruction ledger burn-down — 7 reference proposals, 3 updated. **In that
   order:** doing it after the gate exists means the backlog burns down *into a system that
   prevents its recurrence*, instead of into the conditions that produced it.

> **Trap 1, carried from the routed package because it is the sharper risk.** A reference proposal
> written *after* the fold makes it easy to **write a justification for what already shipped and
> call it a design record**. Where the fold was wrong or partial, the reference proposal is the
> place to say so.

---

## 6. Provenance and the boundary

Drafted by **meta** as candidates A1–A6 (`METHODOLOGY.md` §9) and routed as
`HANDOFF-2026-08-15-ARCH-DISCIPLINES.md` (meta `ae1f459`). **Assembled, renumbered, ladder-applied
and ratified here, in arch's tree, on arch's authority** — meta drafted and routed; it did not
ratify.

**Corrections routed back to meta** (their tree wins on their content; ours wins on ours):

1. **The A1→A6 numbering collides with the Audit Doctrine's A0–A12** in the same injected file.
2. **The A1 gate's version-header trigger misses the no-rev-bump class**, which is arch's most
   common normative edit — measured against two folds landed the same day.
3. **A3 and A6 do not meet the ladder's second-incident bar** and are carried as candidates.
