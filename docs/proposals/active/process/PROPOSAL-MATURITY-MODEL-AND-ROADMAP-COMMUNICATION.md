# PROPOSAL — maturity is multi-dimensional, and it should be measured, not maintained

**Status:** DRAFT (2026-08-13)
**Target:** `docs/status/STATUS-RELEASE-SURFACE-AND-MATURITY-CANONICAL.md` (replaced by a
generated artifact) · `ROADMAP-{EXTENSIONS,SDK,APPLICATIONS}.md` (keep, re-point) ·
`specs/SPECIFICATION-FORMAT.md` (the `Status:` header's meaning) · a new
`spec convergence` + a `spec maturity` aggregator in `entity-system-arch-tools` · discharges **ADR-0004**'s open
follow-up.
**Scope:** how we classify, track, and publish where a surface is. **No spec-behaviour
change, no wire change, no requirement moves.**
**Sibling of:** `PROPOSAL-CONFORMANCE-ORACLE-CONTRACT` — that one specifies how conformance
is *measured*; this one specifies what we *say* about the result. §4 of that proposal (the
verdict document) is this proposal's primary input.

---

## 0. The argument

We have **three** maturity vocabularies. One is hand-maintained and stale, one was ratified
and never implemented, and one is documented as not meaning what it says. None of them
carries the axes we actually need — generators and community — and the one that is canonical
uses a **single cumulative scalar** for something that is genuinely multi-dimensional, which
is why its own notation has already broken.

This proposes separating the two things that got merged: an **adopter-facing tier** (three
rungs, already ratified in ADR-0004, still unimplemented) and a set of **independently
measured convergence dimensions** that the tier is *derived from*. And it proposes deriving
the dimensions from the artifacts that already carry them, because the evidence below is that
maintaining them by hand does not work — not through carelessness, but because a hand-kept
number is stale the moment anything moves, and eight repos move daily.

---

## 1. What exists today

| Vocabulary | Where | State |
|---|---|---|
| **M0–M6 ladder** + 🟢🟡🔴 trajectory | `STATUS-RELEASE-SURFACE-AND-MATURITY-CANONICAL.md` | canonical, hand-maintained, **last touched 2026-08-05** |
| **`experimental / preview / stable`** | **ADR-0004** (Accepted 2026-06-16) | ratified ecosystem-wide, **never implemented** |
| **Spec `Status:` header** | every spec (`Draft`/`Active`/`Superseded`/`Normative`) | the maturity doc states outright it is **not a maturity signal** |

The ladder is `M0 Proposed → M1 Ratified → M2 Landed → M3 Implemented → M4 Peer-converged
(3-way) → M5 Validated (vectors) → M6 Keystone-converged`, cumulative.

**ADR-0004's own follow-up is the unfinished half of this:** *"define what each tier means
for the spec and the keystone (entry/exit criteria); wire the labels into the conformance
presentation."* That was never done. The M-ladder was built instead, and **nothing maps the
two vocabularies to each other** — so the ratified, adopter-facing tier has no criteria and
appears nowhere, while the internal ladder is what gets published.

---

## 2. Evidence — measured 2026-08-13, live

### 2a. The ladder's top rung is wrong in both directions at once

`STATUS-RELEASE-SURFACE-AND-MATURITY-CANONICAL.md` states the convergence floor as:
*"exercised by the keystone cohort (**15** generated peers `--profile core` 0-FAIL)."*

Measured directly from `entity-core-keystone` `9292a3a` by parsing every
`protocol-generator/*/status/CONFORMANCE-REPORT.json`:

| | Measured |
|---|---:|
| conformance reports present | **44** |
| reports carrying a `summary` object | **43** |
| **0-FAIL** | **42** |
| **0-FAIL *and* 0-SKIP** | **0** |
| skipped per peer | **95 – 101** |

**Understated:** 15 vs 42 — and keystone's 40-peer remediation landed **2026-07-28**, *eight
days before* the maturity doc was written. **It was stale on the day it was authored.**

**Overstated at the same time:** `ADR-0012` says **a skip counts as a failure**, and
**every** peer skips 95–101 checks. Under our own published rule, the honest statement is
that **zero** generated peers are clean, not 15 and not 42. M6 "Keystone-converged" reads as
a binary that has been achieved; the measurement says it is a spectrum nobody is at the top
of.

*(Both halves matter. A number that is too low undersells the work; a number measured against
the wrong bar is the thing `ADR-0012` exists to prevent.)*

### 2b. One verdict document is a different shape

Of 44 reports, **`io`'s has no `summary` key at all** — its top level is
`codec_impl_info, measurement_note, multisig_accept_unit_test, oracle, origination_core,
peer`. Any census must either special-case it or silently drop a peer. **This is
`PROPOSAL-CONFORMANCE-ORACLE-CONTRACT` §4 (the verdict document has no specified shape)
producing a concrete cost**, and it is the reason §4 should land before this proposal's tool
is built.

### 2c. The scalar has already broken

Four of the ladder's entries cannot be expressed as one level and use arrows or ranges:
`EXTENSION-REGISTRY` **M2→M3**, `EXTENSION-DISCOVERY` M2→M3, `EXTENSION-ENCRYPTION` M2→M3,
`EXTENSION-SIGNALING` **M3→M4**, plus `EXTENSION-TRANSACTION` M0–M1 and
`EXTENSION-DURABILITY` M0–M2.

**That is not sloppy notation, it is the model failing.** A cumulative scalar forces
lock-step on dimensions that genuinely advance independently: a surface can be
vector-validated but unimplemented by generators, or generator-converged while its spec is
still `Draft`. `EXTENSION-SIGNALING` is the clean example — `Version: 1.0`, `Status: Draft`,
ladder `M3→M4`, with the note *"S5 gate not run."* Three vocabularies, three different
answers, none of them wrong.

### 2d. Every level in the table is unverified against its current spec

The maturity doc is dated 2026-08-05. Spec headers, read live:

| Extension | Header | Spec last moved | Ladder says |
|---|---|---|---|
| `EXTENSION-ENCRYPTION` | 1.0 · **Active** | 2026-08-10 | M2→M3 |
| `EXTENSION-SIGNALING` | 1.0 · **Draft** | 2026-08-10 | M3→M4 |
| `EXTENSION-REGISTRY` | 1.2 · Active | 2026-08-12 | M2→M3 |
| `EXTENSION-NETWORK` | 1.6 · Active | 2026-08-12 | M4 |
| `EXTENSION-DISCOVERY` | 1.0 · Active | 2026-08-11 | M2→M3 |

**All five moved after the ladder was written.** The doc's own instruction — *"verify a
specific level against the spec header + the cohort convergence report before publishing a
number"* — is correct, and is an instruction to redo the measurement by hand every time
anyone cites it. That is the definition of a number that will be published stale.

Across the corpus: **26 extension specs, 22 `Active` / 4 `Draft`** — and the `Draft`/`Active`
split does not correlate with the ladder at all.

---

## 3. The design

### 3a. The pipeline is the spine, and each stage is reported by its owner

**Refined 2026-08-13 and partly built.** The sequence this project actually runs is:

```
spec  ->  core reference peers  ->  generators  ->  community
```

**Architecture measures the first stage and only the first stage.** Its dimension is the
one it owns: **how much a spec is still changing** — convergence and rate of change, not
whether someone implemented it. Every later stage is owned by the repo doing the work,
**reports its own state**, and is aggregated here. Arch never asserts a stage it does not
own; that is the failure mode §2a documents.

The stages are **data, not code** (`[maturity.stages]` in the toolkit config), so the
sequence grows — a second generator, a new language family, a community register — without a
schema change. A stage with no published source reports **`unreported`**, deliberately
distinct from being absent.

**`community` is usage, not existence.** The question is not *"does an implementation exist
in language X"* — keystone can generate 45 of those. It is **whether the spec has been
validated or confirmed in use** by something outside this project. That is why it is a
separate stage from `generators` rather than more of the same.

**Built and landed** (`entity-system-arch-tools` `7e3e4e2`): `spec convergence` measures the
spec stage from git and prints the pipeline with the unreported stages visible.

### 3a-i. What the first run found

85 documents — **40 volatile · 5 active · 40 settling**. And the hand-set flag was not
slightly stale, it was **inverted for most of the set it describes**:

> **Nine of the fifteen specs the canonical doc flags `M5` 🟢 stable — "the v1 locked set" —
> changed within the last three days.** `CONTENT`, `TYPE`, `REVISION`, `SUBSCRIPTION`,
> `CONTINUATION`, `INBOX`, `IDENTITY`, `QUORUM`, `ROLE`.

`EXTENSION-SIGNALING` carries 19 commits in the 90-day window with **all 19 inside the last
fortnight**, against a ladder entry of `M3→M4` 🟡 and a spec header of `Draft`.

**This is the argument for the whole proposal in one measurement.** The flag was not wrong
because someone was careless; it was wrong because it is hand-set, and nine specs moved
after it was written.

### 3b. Two vocabularies, one derived from the other

**Adopter-facing (published): ADR-0004's `experimental / preview / stable`.** Three rungs,
per surface, broadly attested (Rust, browsers, K8s all use this shape). This is what users,
researchers, and developers read. **It is already ratified — this proposal supplies the
entry/exit criteria it has been missing since June.**

**Internal (measured): independent convergence dimensions.** No scalar, no cumulation.

| Dimension | Question it answers | Mechanical source |
|---|---|---|
| `spec` | proposed / ratified / **folded** | proposal lifecycle + spec header + `spec standards` |
| `vectors` | is there conformance coverage, and does it resolve to a normative home? | conformance register (`-json`) + `spec corpus` |
| `cohort` | Go / Rust / Python convergence | cohort gate verdicts, oracle-pinned |
| `generators` | how many keystone-generated peers, at what verdict | keystone's report census (§2a) |
| `community` | independent implementations outside the cohort | **nothing measures this — see §3c** |

**Every dimension reports a value *and its measurement pin* `(repo, commit, date)`.** A
dimension with a stale pin reports as **unmeasured**, not as its last value. That single rule
is what the current doc lacks and what §2a is a consequence of.

### 3c. The tier is derived, and the criteria are published

Illustrative, to be settled at ratification — the point is that the mapping is *stated* and
*checkable*, not that these exact thresholds are right:

| Tier | Entry criteria (all must hold) |
|---|---|
| **experimental** | spec folded or ratified. No convergence claim of any kind is published. |
| **preview** | spec folded · vectors exist and resolve to a normative home · cohort 3-way at 0-FAIL, oracle-pinned |
| **stable** | all of preview · generators ≥ N peers · **0-FAIL *and* 0-SKIP** per ADR-0012 · no unresolved-citation findings against its categories |

**No-permabeta, per ADR-0004:** a surface sitting in `preview` past a stated window is a
tracked item with an owner, not a resting state.

### 3d. `community` gets published as zero

Nothing in the ecosystem measures independent adoption, and the honest current value is
**zero independent implementations**. `AGENTS-STANDARD` already draws the distinction this
dimension exists to carry:

> a cohort of implementers all passing one author's vectors is **cohort-consistent, not
> independent convergence**

**Publish the zero.** A maturity page that omits the axis reads as though the question was
never asked. This proposal does *not* attempt to define what counts as a community
implementation beyond "not produced by this project's cohort or generators" — that needs a
decision, and it is named here rather than assumed.

### 3e. Derive it — the rest of the pipeline

The dimensions above are already machine-readable, in four places that no tool reads
together: spec headers, the conformance register's `-json`, `spec corpus`, and keystone's
report census. **A generated maturity artifact replaces the hand-maintained table**, with
the same discipline as `spec corpus`:

- reports **could-not-look** distinctly from **measured-and-empty**;
- a dimension whose pin is older than the tree it describes reports **unmeasured**;
- publishes counts with their denominators and never a bare percentage (`ADR-0012`);
- a skip is reported beside a fail, never folded into it.

**Dependency, stated plainly:** the census in §2a needed a hand-written parser and hit §2b on
the way. **`PROPOSAL-CONFORMANCE-ORACLE-CONTRACT` §4 should land first**, so this tool
consumes a specified verdict document rather than reverse-engineering 44 of them.

### 3f. What happens to the existing documents

- `STATUS-RELEASE-SURFACE-AND-MATURITY-CANONICAL.md` — **becomes generated.** Its ladder-to-
  dimension mapping is preserved as history; the file stops being hand-edited.
- `ROADMAP-{EXTENSIONS,SDK,APPLICATIONS}.md` — **keep.** They are the *forward, stage/phase*
  view — what is planned and in what order — which is a genuinely different question from
  where a surface is now, and it is not derivable. They re-point at the generated artifact
  for state, exactly as they already do for M-levels.
- **Spec `Status:` header** — `SPECIFICATION-FORMAT` should say what it means (the spec
  document's own lifecycle: `Draft` = not folded, `Active` = folded and current, `Superseded`)
  and say explicitly that it is **not** a maturity or convergence signal. Today that
  disclaimer lives only in the maturity doc, which is the wrong home for a statement about a
  header the format spec defines.

---

## 4. Why this is worth doing now

The user-facing reason is the one that matters: **we are nine days from a release with a
research-preview label and no published, defensible statement of what is safe to build on.**
ADR-0004 exists precisely so adopters can depend on the stable subset while the edges churn,
and it has been ratified and unimplemented for two months.

The internal reason is that this is the same defect as everything else this cycle. A
hand-maintained number that nothing checks goes stale and then gets cited as authority — the
corpus version stamp, the blast-radius table, the build-state pins, the `194`, and now the
maturity ladder. **The pattern is established well enough by now to stop re-deriving it: if a
number can be measured, measure it; if it cannot, pin it and expire it.**

---

## 5. What this does not do

- **Does not change any spec's content, version, or conformance requirements.**
- **Does not re-tier anything by fiat.** The first generated run reports what is measurable;
  where a tier assignment needs judgement it is a decision with an owner, not a default.
- **Does not delete the roadmaps** — they answer a question the measurement cannot.
- **Does not define what a community implementation is** (§3d) — flagged, not assumed.

---

## 6. Open questions

1. **Does `generators` belong in `stable`'s criteria at all?** A generated peer is produced
   from the spec by keystone; a surface can be perfectly stable for adopters with no
   generator coverage. **Recommended: yes, but as its own published dimension rather than a
   gate** — it measures how mechanically derivable a surface is, which is real signal, but
   gating `stable` on it conflates "safe to depend on" with "we generated it."
2. **What window does no-permabeta use?** ADR-0004 requires progression without naming a
   period. **Recommended: one release cycle in `preview` before it becomes a tracked item**,
   because that is the shortest window that cannot be gamed by a quiet re-stamp.
3. **Who owns the tier call when the dimensions disagree** — e.g. cohort-green, generators
   absent, spec `Draft`? **Recommended: architecture, published with the dimension values
   beside it**, so a reader can disagree with the judgement without re-deriving the facts.
