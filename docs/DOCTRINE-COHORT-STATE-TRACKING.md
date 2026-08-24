# DOCTRINE — how arch aggregates and tracks cohort state

**Status:** governing doctrine for `docs/status/` (WORKSTREAMS, STATUS, handoffs). Written 2026-07-28
after arch shipped, three times in one session, directives built on state that was **weeks stale and
opposite to what the peers had already reported and proven green on the wire.** This doctrine exists so
that specific failure cannot recur.

## What arch's job actually is (the sentence I violated)

**Arch does not run the wire. Arch aggregates what the peers report and tracks it accurately.** The peers
build and validate; they report results. Arch's product is a *coherent, provenance-pinned ledger* of those
reports plus the arch-owned design decisions. When arch's ledger and a peer's fresh report disagree,
**the peer is right and the ledger is wrong — every time.** Build-state is peer-owned; arch records it,
never originates or overrides it.

> **Correction 2026-08-16 — this sentence used to read "the peers (go / rust / py / browser)", and that
> framing caused a second failure of its own.** There are **three lineages, not four peers**:
> `entity-browser-rust` links `entity-core-rust`'s `entity-{peer,network,signaling}` **by path**, and
> `entity-workbench-go/entitysdk` links `entity-core-go`'s `core`+`ext` by `replace`. **A front end
> cannot validate the core it compiles against**, so counting front ends as peers inflated the apparent
> evidence base on every P2P surface from **one lineage to four seats**. Canonical map:
> `docs/status/CENSUS-2026-08-16-…` / `WORKSTREAMS` **W-EVIDENCE**.

## Adjudication requires two readings — the N rule `[added 2026-08-16]`

**Arch's authority is adjudication between independent readings.** It is real when peers read one spec,
build separately, and diverge — arch then holds context no single peer has, and the compute thread is the
worked example (three implementations, three readings of one `is_error` block, one ruling, 331 agree / 0
divergences). **It is not a job at all when there is one reading.**

| Independent implementations | Arch's move |
|---|---|
| **≥ 2, divergent** | **Adjudicate.** Rule it. |
| **≥ 2, agreeing** | Pin the agreement before it drifts. |
| **1** | **Document the finding, credit it, ask the others. Attach no MUST** — it ratifies one team's taste. |
| **0** | **Design freely, and say the section is unbuilt.** A version bump implies a validation that has not happened. |

**Count code bases that can disagree, per seam — not repositories.** A front end over a core is not a
second implementation of that core; **but two front ends over two different cores ARE two implementations
of the app tier.** The count is per-seam and must be re-taken, not remembered.

**The failure this prevents:** folding one lineage's measurement as a general MUST, then citing the
resulting spec text back to that same lineage as authority. **That is not adjudication; it is an echo.**

## The failure (2026-07-28), stated plainly

- The cohort had NETWORK liveness + reconnect + the continuation substrate **built and validated green
  three-way since 2026-07-15…07-19** (reports `2026-07-15-a12-liveness-harness`,
  `2026-07-19-reconnect-disconnect-crossimpl`; re-proven live on the wire 2026-07-28).
- The arch tracker asserted the **opposite** — "reactive half UNBUILT everywhere," "rust/py must
  reconcile" — as present-tense fact.
- I read that prose, believed it over core-go's fresh reports, and issued handoffs telling the cohort to
  **start building work they had finished two weeks earlier**, plus a §A2 "unblock" that was a
  declaration-home formality over fields already built and green.

## Root-cause failure modes (doctrinal defects, not one-off slips)

- **RC1 — Wrong source of truth.** I ranked arch's own prose above the peers' fresh reports. Core-go told
  me §4 was not zero-impl and §3's DROP/CONFIRM were already done; the tracker disagreed; I wrote from the
  tracker.
- **RC2 — Present-tense claims with no fresh provenance.** The tracker's own header says states are "not
  re-verified against live sibling code; a state without a pin is design-status, not a fresh observation" —
  and then prose asserted "unbuilt everywhere" as current fact anyway.
- **RC3 — Corrections by re-reading, not re-proving.** The flip fold-ready→"unbuilt" (`78a5d8b`) was made by
  re-reading a **2026-07-14** proven-negative (proposal §B) and presenting it as a 07-28 correction. A
  proven-negative is a timestamped snapshot; re-asserting it two weeks later without re-proving is a
  fabrication of currency.
- **RC4 — Conflation of design-state and build-state.** Arch-owned decisions (drafts, rulings, fold-staging)
  and peer-owned facts (built, validated three-way) were merged into one narrative, so build-facts rotted
  invisibly behind authoritative-sounding prose.
- **RC5 — Directives issued without ingesting the target peer's latest report.** I handed core-go "start
  §A1" without first reading what core-go had most recently reported.
- **RC6 — Prose as the tracking unit.** Narrative can't be diffed and carries no per-cell provenance — which
  is exactly what let the rot hide and compound across sessions.

## The corrected doctrine (the rules, enforceable)

- **D1 — Source of truth.** Build/validation state is **peer-reported**. Arch records it; arch never
  originates or overrides it. Ledger-vs-fresh-report conflict → the report wins, correct the ledger *now*.
- **D2 — Provenance is mandatory.** Every build-state cell is `{state, pin (commit or report id),
  date_observed}`. No pin + date → it is **not a fact**; mark it `UNCONFIRMED` / design-status, never
  present-tense "built"/"unbuilt".
- **D2a — Name which half of the surface `[added 2026-08-14]`.** A build-state cell for a protocol
  surface says **producer**, **consumer**, or **both**; a cell that cannot say which is `UNCONFIRMED`.
  `PROPOSAL-REGISTRY-SERVICE-ADVERTISEMENT` was tracked as *"built three-way, consumed by name, cited as
  law"* and folded on that basis three weeks late — while the **entity and the `resolve` field existed in
  no tree.** What all three had built was the *selection function over a caller-supplied pool*; the three
  name-searches that confirmed "built" had hit the type name **in doc comments** describing where a pool
  would one day come from. *Prove a negative* was followed and still produced a false positive, because
  the search proved presence of a **string**, not of a **capability**. **The shape to watch for is a
  consumer with no producer:** it is the half with the interesting algorithm, so it is the half that gets
  built, reviewed and reported, and it passes every unit test its author writes while being unreachable in
  production.

- **D3 — Freshness decays; it is never manufactured.** A cell past a staleness horizon without
  re-confirmation is flagged `STALE`, not restated as current. A proven-negative expires the moment it ages
  or anything plausibly superseded it.
- **D4 — A status flip requires fresh evidence.** Never flip a state from a re-read of an older doc. Flipping
  requires either a peer report or a **freshly re-run** proven-negative (exhaustive named search **plus**
  `git log --since <last-known-good>`). This is the AGENTS-STANDARD rule "prove a negative before you claim
  it," applied to corrections.
- **D5 — Separate the two registers.** A **Build ledger** (peer-owned facts, pinned) and a **Design ledger**
  (arch-owned decisions). Never let one masquerade as the other in prose.
- **D6 — Ingest before you direct.** No handoff or directive ships until arch has reconciled it against the
  target peer's **latest** report. **Tripwire:** if a directive would tell a peer to do work they have
  already reported done, STOP and re-audit — that is the canonical signature of this exact failure.
- **D7 — A ruling pins to source, not to a summary — even the peer's own, and even the correction.** A normative
  pin (a MUST that creates or cancels cohort work) requires the behavior confirmed against **impl source**, not a
  narrative summary — and a *retraction* is a ruling too, so it gets the same source check. The 2026-07-28 O5 reap
  pin needed **three** passes because arch skipped source twice: pass-1 ("deadline-abandon MUST be observable
  without an advance") rested on a summary claiming Go auto-abandons a quiescent join — false; pass-2 ("touch-driven,
  all three match, no build") fixed the timer error but asserted convergence **without reading the siblings** — also
  false (Go sweeps *all* tracked joins on touch; Rust/Py reap only the touched one). Only pass-3, after arch read all
  three at source, was correct (converge on sweep-all — a real build). **Rule:** before any pin — including one that
  says "already converged, no work needed," which is the seductive one — read the source in every impl the claim
  covers, and name the `file:line` it rests on. "No build needed" is a load-bearing claim; source-check it as hard as
  "build this."

- **D8 — Build state has one home, and a normative document is not it.** *(Added 2026-07-31, after D1–D7 failed
  to prevent a recurrence — see below.)* A build-state claim MUST NOT be stated in a **spec**, an **amendment
  header**, or a **proposal**. Those documents are read as normative, revised rarely, and mirrored downstream,
  while the fact inside them expires in **hours**. When a spec genuinely needs to reference build state — a
  conformance gate saying what has and has not been exercised — it **points at the dated ledger** and does not
  restate it. Any surviving instance carries its date and an explicit *"re-read the peers' reports before citing
  this."*

  **Why this rule exists and D1–D7 were not enough.** On 2026-07-31, hours after writing a handoff whose §1 was
  entirely about violating this doctrine, arch violated it again — twice, in the same document set: *"the §10.3
  seam is not present in any implementation"* and *"`observe-address` is unbuilt in both."* Go had built the
  seam, the punch, and the reflect responder; Rust had built its seam counterpart. Both errors were caught by
  **peer packets contradicting an arch document**, not by arch.

  D1–D7 govern how a build-state claim is *sourced*. They say nothing about **where it is written**, and the
  placement is what made the recurrence invisible: a sentence inside an Amendment header does not look like a
  ledger cell, so it does not attract the provenance discipline D2 demands, and nothing prompts a re-check when
  it ages. **The fix for a recurring error is rarely more care at the same site.** Here it is moving the fact to
  a home whose format forces a date and whose readers expect it to expire.

- **D9 — A conformance exclusion is a build-state claim, and it expires like one.** *(Added 2026-08-14.)* A
  per-type skip, a declared exclusion, a "known gap in peer X" note in a harness — each asserts something
  about a sibling's tree at a moment in time. Two of ours **outlived the gap they described** and then
  reported `WARN` or `skip` rather than failing, so the stale claim was invisible in a green run. Therefore:
  every exclusion carries `{reason, pin, date, the condition that retires it}`, and when the pin moves the
  exclusion is **void until re-taken** — exactly D3 and D4 applied to the instrument instead of the ledger.
  **An expired exclusion MUST fail, not warn.** A `WARN` in a green run is a claim nobody reads; that is how
  five stale claims survived a month. *(Same finding, arrived at independently by `entity-core-go` from the
  `failing_since` WARN and by this session from the exclusion side — which is what makes it a class.)*

- **D10 — A rule that has recurred twice buys a mechanism or gets retired.** *(Added 2026-08-14, and it is the
  rule this doctrine most needed.)* D8 exists because D1–D7 did not prevent a recurrence. On **2026-08-13** the
  same failure happened a **fifth** time — `EXTENSION-REGISTRY`'s completeness banner asserted §6a.9.3 was
  *"unbuilt in all three"* and was stale inside twenty-four hours, in a **spec header**, which is the precise
  placement D8 was written to forbid. More care at the same site has now been tried five times.

  **What actually held, measured across this cycle:** every invariant with a gate behind it held; every
  invariant carried as prose recurred. `EXTENSION-REGISTRY` §6a.9 wrote *"a `MUST` may not name a referent the
  corpus does not define"* as a paragraph beside one instance — and a second live instance was sitting in
  `EXTENSION-CONTINUATION` §3.6 at that moment, undetected, having already cost two cycles of
  cross-implementation misrouting. §6a.9's box about *"an operation whose declared return type does not
  enumerate every branch of its own pseudocode"* was reproduced **one subsection later, by the section written
  to fix it.**

  So: when a failure shape recurs a second time, the response is **not** a stronger sentence. It is (a) a
  mechanical check if the shape is checkable, (b) a change of *placement* so the fact lives where its format
  forces provenance, or (c) an honest retirement of the rule. **First payment on D10:** the
  `undefined-wire-referent` and `enum-value-not-declared` rules in `entity-system-arch-tools`
  (`spec coherence`, arch-tools `e57e731`), which turn both of the above into gate failures and found a third
  instance — `EXTENSION-REVISION`'s merge-strategy vocabulary disagreeing with itself in three places — on
  their first run.

  **The corollary for reading this document:** a rule here with no mechanism behind it is a rule that has not
  yet been paid for. Say which ones those are rather than trusting them equally.

## The mechanism

- The **Build ledger** is a normalized table: one row per delta, columns `go | rust | py`, each cell =
  `state · pin · date`. A blank cell means unknown, not "unbuilt."
- Each workstream carries a **"last cohort report ingested"** watermark (report id + date). A handoff cites
  the watermark it was written against, so a reader can see its freshness.
- Design decisions (folds, rulings, ownership calls) live in the Design ledger and cross-reference the Build
  ledger — they do not restate build-state.

## The ratchet

This failure produced this doctrine — and the tripwire in D6 is now a named detector, so "arch told a peer
to redo done work" trips an alarm instead of shipping. The build ledger's per-cell provenance makes RC2/RC3
structurally hard: you cannot write a build-state without a pin and a date, and an aged pin shows as stale
on its face. *A failure must make the process stronger, not weaker.*

**Second turn of the ratchet (2026-07-31): the recurrence taught more than the original failure.** This doctrine
was violated by the very session that wrote a handoff about violating it, which is the strongest available
evidence that **a rule aimed at the author's care does not survive the author's confidence.** D8 replaces care
with placement: build-state facts live where the format demands a date and the reader expects decay, and
normative documents point at them rather than restating them. The detector for D8 is cheap and mechanical —
grep the spec corpus for present-tense build claims (`unbuilt`, `not present in any`, `built in all three`) and
require each surviving hit to carry a date and a re-read instruction.

**The load-bearing observation, worth more than either rule:** both recurrences were caught by a **peer packet
contradicting an arch document**. Arch has never caught one of these itself. That is not a criticism of arch's
diligence — it is structural, since arch cannot observe build state directly and every check it can run is the
same check that produced the error. **So the cohort's blunt contradictions are the primary detector, and they
should be invited rather than merely tolerated.** A packet that tells arch it is wrong is the mechanism working
at its best, and the correct response is to correct the record loudly and thank the sender.
