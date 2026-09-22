# PROPOSAL — the conformance coverage-failure taxonomy (§2.4a / §5.2a / §5.2b / §5.2c)

**Status:** **DRAFT — reference proposal, written after the fold. See §0.** All four rules are
already in `guides/GUIDE-CONFORMANCE.md`.
**Target:** `guides/GUIDE-CONFORMANCE.md` §2.4a, §5.2a, §5.2b (+ §5.2b.1, §5.2b.2), §5.2c.
**Tier:** `process/` — validated by core-go (oracle half) · arch-tools (gate half).
**Scope:** how a conformance suite fails to measure what it claims. **Not** any wire format, entity
type, or operation. No implementation owes a behavioural change from this document; what they owe is
an audit of their own checks.
**Source:** the 2026-08-09…12 cohort round. Every rule below was extracted from a defect an
implementation found and routed; none was invented here.
**Cohort review:** NOT HELD as a package. Each rule was ruled in-cycle against the packet that
produced it.

---

## 0. Process deviation — recorded, not hidden

**These rules were folded directly into `GUIDE-CONFORMANCE` with no proposal.** They are normative
(`[MUST]` in three cases) and they bind every implementation's conformance suite, so they are not
wording-only hygiene and `AGENTS-STANDARD`'s proposal-first rule applied.

Written now as the design record the folds should have had, per the reconstruction ledger
(`docs/status/PLAN-2026-08-15-spec-provenance-reconstruction.md`, arc B). **Not back-dated, not prior
authorization.** If cohort review rejects a rule, that rule comes out of the guide.

**One aggravating detail specific to this arc, and it is the reason it is sequenced first:** these
rules are **cited as law by other documents and by implementations' own check comments**, and they
live in `guides/`, which is **outside the linter's analysis scope** (W-CORPUS **C1**). So the
corpus's most-cited process rules sit in the one directory no gate reads — recorded on 2026-08-11 as
a cost, deferred twice since.

## 1. The gap

A conformance suite's job is to make divergence visible. Four times in one cycle it did the
opposite: the suite was **green**, and the surface it claimed to cover was **not measured**. Each
was found by an implementation building against the surface, never by review of the checks.

The rules exist because *"run the conformance suite"* had come to mean *"the scoreboard is green,"*
and those are different claims whenever a check is absent, unreachable, inverted, or lucky.

## 2. The four shapes, and why they are one family

`GUIDE-CONFORMANCE` §2.4a states the taxonomy; this is the reasoning behind it. The family axis is
**what the suite's green means**:

| Rule | Shape | The green means |
|---|---|---|
| **§5.2a** | a rule with **no peer-observable surface** | nothing — no probe can exist |
| **§5.2b** | a surface the suite **cannot reach** | the harness could not turn the knob |
| **§2.4a** | a surface the suite reaches and **scores backwards** | a *correctly guarded* peer fails |
| **§5.2c** | a surface reached **only by accident** | it passed the times it happened to run |

This is ADR-0012's *"conformance-green ≠ correct if the test asserts the wrong thing"* in its
authoring-time form. Stating them as one family is deliberate: each was found separately and each
looked like an isolated harness bug, and it was only the fourth that made the shared root visible —
**what gets implemented and what gets scored are both driven by the vector list, not the prose.**

## 3. §2.4a — the negative half `[MUST]`

**Every check of a guarded operation MUST assert the negative half.**

**Source:** `entity-core-go`, 2026-08-11 (`docs/validation/spec-issues/2026-08-11-revoke-names-an-operator-with-no-proof-shape.md`
and the packet folded at `4dd07f5`).

**What it cost, and this is the argument.** **`entity-core-go` and `entity-core-rust`** shipped
`revoke`/`renew` with **no verification at all** — any peer able to reach a registry could permanently
revoke any binding in it, and revocation is monotonic, so there is no undo. Go's own conformance check
**certified the hole**: `registry_issuer` asserted `200`/`202` with no proof attached, so a
*correctly implemented* peer **failed** it — and one did. **`entity-core-py` verified the proof on
both ops and was scored red for it.**

> **Corrected 2026-08-15 — and the correction is itself an instance of this taxonomy.** §3 previously
> read *"all three implementations shipped `revoke`/`renew` with no verification at all,"* which is
> false: py's `_handle_revoke_request` / `_handle_renew_request` gate on `_verify_proof_by` at
> `4cdf805`, dated **before** the finding was filed. The sentence came from go's spec-issue and
> **propagated into three arch documents** — this one, `PROPOSAL-REGISTRY-PEER-ISSUED-REGISTRATION`
> §2, and **`GUIDE-CONFORMANCE` §2.4a's own body**, where it sat six lines above a table row correctly
> crediting *"core-py refused to converge."* **Three documents, each internally contradictory, none
> caught by review.** All three are now corrected.
>
> **The fact being wrong made the rule weaker, not stronger.** *"All three had the same hole"* reads
> as a story about three careless implementers. What actually happened is worse and is the exact thing
> §2.4a forbids: **the suite scored the one correct implementation as the failure.** Positive-only
> coverage does not merely under-measure — **it inverts, and it points at the peer that got it right.**

**Positive-only coverage of a guarded operation is not partial coverage — it is coverage of the
unguarded behaviour.** §6a.9.1 already had the rule in one place
(`policy_manual_publishes_nothing`) and nobody's registry failed there; the sections where nobody
wrote it are exactly where the hole appeared.

**§2.4a's state-probing half `[MUST, added 08-11-b]`** — the three conjuncts are *refused* **and**
*no state change* **and** *no publication*, and asserting status+code satisfies only the first. **A
peer that answers `403` and performs the write anyway passes a status-only assertion completely.**

**Two corollaries folded with it,** both of which cut against ordinary triage:

- **A sibling's refusal to converge is signal, not friction.** `entity-core-py` refused to converge
  and reported instead; that refusal is the only reason the hole was found. Reaching 16/16 would
  have meant writing unauthenticated revocation into python. *Treat a lone red as a finding to
  investigate before treating it as a defect to fix.*
- **A round-trip check whose two sides share an encoder measures the encoder against itself**, and
  must be gated by a mutation it is required to catch.

## 4. §5.2b — a surface the suite cannot reach

**Source:** 2026-08-10, extended 2026-08-12 by
`entity-core-go/docs/validation/spec-issues/2026-08-12-b-the-5-4a-negative-vector-is-not-wire-drivable.md`.

§5.2a is a rule with no observable surface. §5.2b is its opposite and worse: the surface is
**built, correct, flagged and documented**, and the harness cannot turn it on — so the absence of
coverage is invisible and the scoreboard reads *covered*.

**§5.2b.1 — the second sub-shape, *no knob exists*.** The three originals were all *"the harness
cannot turn a knob that exists."* The 08-12 instance was *"no knob exists at the wire"*: arch had
pinned a `[MUST]` whose required state — a peer unbound yet still `connected` — is reachable only
from paths the spec's own scope *deliberately excludes*. **The scope pin and the obstacle had one
cause, and the authoring defect was arch's:** §5.4a invoked the *not-validated-until-a-cross-impl-
vector* meta-rule and then pinned a vector that could not exist.

New MUST: **before a vector is pinned MUST, the state it requires must be shown constructible by a
conformance client**; where it is not, the spec states its satisfaction mode at the point of the MUST.

**Rejecting the proxy is usually right `[MUST]`** — and this is the sharpest thing in the arc,
adopted verbatim from core-go's argument. The tempting fix is a nearby reachable state. A proxy that
cannot fail the way the real case fails **MUST NOT** be recorded as covering it: in the §5.4a
instance a live idle counterpart stays *bound*, so it catches a timer-driven escalation and never the
scope-pin defect. **It reads as coverage while missing the case — strictly worse than a declared
exclusion, because §5.2b's whole failure mode is a scoreboard reading *covered*.**
**A declared exclusion is an honest zero; a proxy is a false one.**

**§5.2b.1's exclusion discipline, from `entity-core-py`:** a declared exclusion's mutation MUST be
**executed and dated, not described** — *an honest zero nobody has checked is still zero.* Their
argument is the evidence: in a prior cycle their control **and its mutation** both ran against a
malformed probe, so neither could have failed.

**§5.2b.2 — audit the extractor, not only the checks `[MUST]`.** The harness's extractor harvested a
`code` only when `status >= 400`, encoding *"codes ride failures."* §6a.9 pins a code on a **2xx**
row, so `pending_review` silently became `""` and **the assertion could not have passed against any
peer, however conformant.** The near-miss is the valuable part: `>= 300` still excludes `202` — the
status class was never the right discriminator, the result's *type* is. It is filed as a MUST because
**the symptom presents as a sibling bug and sends the hunt into the wrong tree.**

Two scope corrections folded with it, both from the cohort: audit against **the extractor's shape,
not the sibling's specific bug** (three audits found three different mechanisms), and **enumerate the
checks that BYPASS the common extractor** — py's point that *an audit of the checks cannot reach a
layer the checks bypass*. Plus: **audit doctrine-referenced instruments first**, since an instrument
named in a doctrine decides where every future investigation starts.

## 5. §5.2c — a surface reached only by accident

**Source:** `entity-core-rust`, via the Q-3 round (folded `81e73ae`). **The cycle's most valuable
finding, and it came from the implementation that looked *worse* on the scoreboard.**

Go's §5.4a defect read as a **1-in-4 flake** because two paths raced there. Rust had **no race — the
same spec defect was unreachable 100% of the time**: deterministic, silent, and green.

> **The flakiness go had was the lucky version: a check that fails sometimes gets investigated.**

Two corollaries, both of which invert ordinary practice:

- **Flake rate measures the local implementation's accidental structure, not the defect's
  severity.** Ranking work by flake rate ranks it by luck.
- **A green sibling is not evidence the flaking impl is uniquely broken.** *"Two green, one flaky"*
  is equally consistent with three defective implementations and one accidental instrument — which
  is what this was.

**Gated as a MUST:** before any flake is quieted, widened, or retried, state the mechanism that makes
it only sometimes-reachable — *a jitter theory predicts a spread, not 6 s or never* — and whether the
state is deterministic elsewhere in the cohort. **A flake is closed by explaining it, never by making
it stop.**

**Validated since, by an instance nobody had to argue about.** The `encoding.hash_wire_format` row
was closed as "a flake" **twice** — python 2026-08-08, rust 2026-08-14 — with identical row pairs and
identical non-reproduction. It was never a peer defect: a byte-scan treated any `0x58 0x21`/`0x58
0x31` hit as a hash, so high-entropy content collided by chance for *any* implementation. Named and
fixed 2026-08-15. That is this rule paying out on exactly the shape it describes.

## 6. Open questions for the cohort

1. **§2.4a's state-probing half is the expensive one.** It requires every guarded-write check to read
   back the surface. Is that affordable across all three suites, or does it want a scoped list of
   operations where the write is the security boundary?
2. **§5.2b.2's bypass enumeration has no mechanism.** py named the limit — an audit of the checks
   cannot reach a layer the checks bypass — and the answer today is diligence. Is there an
   instrument, or is this honestly a review item?
3. **Are these four the complete taxonomy?** They were found in eight days, which is not evidence the
   space is exhausted. A fifth shape would be worth more than a refinement of these.
4. **A fifth shape may already be here, and this review is where it showed up `[2026-08-15]`.** The
   four rules all describe a *suite* failing to measure a *peer*. The propagation traced in §3 is a
   different axis: **a claim about a peer, made in one seat's document, copied into three of arch's
   and into normative guide text, contradicting itself in each — and surviving every review.** Nothing
   in the taxonomy covers *"the record of what was measured is wrong,"* as distinct from *"the
   measurement is wrong."* It is the same root — the artifact everyone cites is the artifact nobody
   re-derives — and it is the shape a gate could plausibly catch (a build-state claim about a sibling
   with no `(repo, commit, date)` pin is mechanically detectable). **Proposed as §5.2d, pending cohort
   view.**

## 7. Fold plan

**Already folded** — this document changes no guide text. Ratification here means the rules stand as
written and the reasoning is on disk under review. If review rejects one, it comes out of
`GUIDE-CONFORMANCE` and the citing documents are swept.

**Dependency worth stating: C1.** Until `guides/` enters the linter's analysis scope, nothing
mechanically checks this file — the rules that generalize the cycle's most expensive defects are the
least gated text in the corpus. **This review found a factual error in that file** (§3), which is C1
paying out exactly as predicted: no gate reads it, and no reviewer re-derived it.

**One guide edit landed with this review** — `GUIDE-CONFORMANCE` §2.4a's "all three implementations"
sentence, corrected to two-of-three plus the inversion it actually demonstrates. It changes no rule
and adds none.

## 8. Review — the diligence pass `[2026-08-15]`

**Both axes run**, per `PLAN-2026-08-15-spec-provenance-reconstruction.md` §8. This arc is
process-tier, so Axis B here means *is the rule right*, not *is the build state right*.

**Sources opened:** `entity-core-go` `docs/validation/spec-issues/2026-08-11-revoke-names-an-operator-*.md`
· `.../2026-08-12-b-the-5-4a-negative-vector-is-not-wire-drivable.md` ·
`docs/validation/reports/2026-08-11-2-4a-audit-the-deny-helpers-assert-status-only.md`.
`entity-core-rust` `docs/status/ROUTING-2026-08-12-your-f1-was-ours-too-*.md` §1.
`entity-core-py` `docs/status/HANDOFF-2026-08-12-b-the-escalation-round-and-two-extractors-python.md`
· `HANDOFF-2026-08-10-*.md` §5. Source-verified in py's tree.

### What the review changed

| # | Axis | Finding | Disposition |
|---|---|---|---|
| **B1** | A | §3 carried the **same false "all three" claim** corrected in arc E — and so does **`GUIDE-CONFORMANCE` §2.4a itself**, six lines above a row that contradicts it | **§3 rewritten · guide corrected** |
| **B2** | B | The corrected fact **strengthens** §2.4a: positive-only coverage does not under-measure, it **inverts** and scores the correct peer as the defect | **§3 argument sharpened** |
| **B3** | B | A candidate **fifth shape** — the *record* of a measurement being wrong, as distinct from the measurement — surfaced by this review's own propagation trace | **New §6.4** |

### What the review confirmed as correct

- **§5's attribution to `entity-core-rust` is exact**, including the block quote. rust's routing says
  *"ours had no race … structurally unreachable, 100% of the time … the flakiness you had was the
  lucky version of it: a check that fails sometimes gets investigated."* **Credited, not paraphrased.**
- **§5's provenance pin was checked rather than trusted** — "via the Q-3 round (folded `81e73ae`)"
  reads odd for a §5.4a finding, and it is right: `81e73ae` does touch `guides/GUIDE-CONFORMANCE.md`,
  bundling the §5.2c rule with the Q-3 ruling. **Not a finding.**
- **§4's exclusion discipline is py's**, and their document says what the proposal says it says — the
  mutation is *executed, not named*.
- **§4's `[MUST]` against proxies** is adopted from go's argument, and arch's own commit records
  adopting it *verbatim* while conceding *"the authoring defect is ours."* **Consensus is not
  invented here** — the proposal states arch was the party at fault.
- **§5's "validated since" hash_wire_format paragraph** matches go's own cohort report: sighted
  python 2026-08-08 and rust 2026-08-14, closed as "flake" both times, mechanism named and fixed
  2026-08-15.

### Not covered by this pass

- **No cohort review as a package**, unchanged from §0. B3 is new and needs a seat's view.
- **§6.1's affordability question** — whether the state-probing half is affordable across three
  suites — was not put to any seat, and go's own §2.4a audit report suggests it is structural (two
  shared helpers and one gate), which would make it cheap. **Not verified; not claimed.**
