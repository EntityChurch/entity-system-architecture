# PROPOSAL — the observable operation-budget ceiling charges `evaluate()` steps only

**Status:** IMPLEMENTED — 2026-07-23 (ratified into `EXTENSION-COMPUTE §4.2`, v3.21). **Ratifiable unit for the
Q2 ruling. Validated three-way** by the compute corpus LOCK (330/330 byte-identical Go/Rust/Python, oracle-pinned
`seed 20260716 / corpus-SHA d0fdd757`): no `budget_exhausted` divergence, all three charge `evaluate()` steps
only, `worked/budget/exhausted-deterministic` agrees (`entity-core-go` report `2026-07-23-compute-corpus-three-way-LOCKED`).
**Domain:** core / `system/compute` — `EXTENSION-COMPUTE §4.2` (resolution-cost guidance) / §5.1 (budget model).
**Source of the finding:** the compute corpus's first cross-impl run (core-go, 2026-07-22) —
`ABSORPTION-compute-corpus-first-crossimpl-run`, `ARCH-RESPONSE-COMPUTE-CORPUS-FIRST-RUN §Q2`; core-go
spec-issue `2026-07-22-budget-exhaustion-cannot-be-cross-impl-deterministic`.
**Fold target:** `EXTENSION-COMPUTE §4.2`, version → **v3.21**.
**Relates to:** `PROPOSAL-COMPUTE-BUDGET-PREEMPTION` (BP-1 — the *fairness* reduction budget; this proposal
supplies the complementary *ceiling* stop-point determinism BP-1 assumed).
**Fold gate (CDN-corridor meta-rule):** budget-edge outcome determinism — **not validated until the cohort
re-runs budget-edge vectors three-way**. In-place §4.2 edit landed **provisionally** (stamped `v3.21 — pending
cohort validation`).

## The defect

`§10.1`/`§4.1` charge one decrement per `evaluate()` step (deterministic). But `§4.2` let scan-based impls
*"deduct one or more operations from the budget per `resolve()` call … The deduction factor is
**implementation-defined**."* So the `operations` ceiling — whose exhaustion is the hard `budget_exhausted`
(`PROPOSAL-COMPUTE-BUDGET-PREEMPTION` line 244) — is reached at **different points** across impls. For any
budget-tight vector, one impl returns a value where another returns `budget_exhausted`, and neither is wrong.

The spec **claimed the opposite, falsely** — §4.2: *"Cross-implementation determinism is not affected … result
values do not [differ]."* True only for computations that **complete** before any impl's ceiling; **false at the
budget edge** — which is exactly where `GUIDE-CONFORMANCE §7c.5` says the corpus gates W-BUDGET preemption
determinism (BP-1). A corpus that avoids the edge cannot gate BP-1 at all.

## The rule (normative)

The **conformance-observable** operation budget (the `operations` ceiling of §5.1/§10.1) is decremented **one
per `evaluate()` step and nothing else**. `resolve()` calls cost **zero** in the observable budget. Given the
same `(IR, inputs, budget)`, two conformant impls MUST reach `budget_exhausted` at the **same** `evaluate()`-step
count.

An implementation MAY impose an **additional, implementation-private** resource guard against scan-based DoS (a
scan-depth or wall-clock bound), but it **MUST NOT alter the observable budget outcome**: if it trips, it
surfaces as a **distinct resource-limit condition** (a transport/resource failure, out of band), **never** as
the in-band `compute/error{budget_exhausted}` value at a shifted stop-point.

## Why pin, not carve out

Excluding budget-edge vectors from the equality gate (the spec-issue's "Outside" option) concedes that a core
outcome — value-vs-`budget_exhausted` — is impl-dependent. For a determinism protocol that is a contradiction,
and it leaves BP-1 (which §7c.5 gates) ungateable. Pinning the observable ceiling makes `budget_exhausted` a
deterministic outcome the corpus **can** gate, and it does not remove DoS protection — it relocates it to a
private guard with a distinct, out-of-band outcome. The spec already leans here: reverse-index impls "MAY treat
`resolve()` as zero-cost," and content-store-direct scope resolution (§1613, normative) is O(1). The only thing
breaking determinism was the "scan-based SHOULD deduct implementation-defined **into the same budget**" hatch;
this closes it without banning the guard.

`budget_exhausted` is therefore now a **gated outcome** — budget-edge vectors stay in the equality gate.

## BP-1 is separate and unchanged

core-go asked (Q2.3) whether BP-1 needs its own determinism statement. It already has one: BP-1 is
*fairness-schedule independence* of the result (reduction budget), a different obligation from *ceiling
stop-point determinism*. This proposal supplies the second; BP-1's `bp_result_determinism` MUST is untouched.

## Fold delta (landed in place, provisional)

1. `§4.2` — resolution-cost note rewritten: observable ceiling = `evaluate()` steps; resolve() = 0; scan-DoS
   guard is out-of-band; the false determinism claim corrected. ✅ landed (`fold` commit).
2. `GUIDE-CONFORMANCE §7c.5` — `budget_exhausted` is a gated, cross-impl-deterministic outcome; budget-edge
   vectors stay. ✅ landed.

## Route

- **Cohort:** budget accounting charges evaluate-steps only in the observable ceiling; move any per-resolve DoS
  charge to a private, out-of-band guard; re-run budget-edge vectors three-way.
- **On three-way green:** flip `EXTENSION-COMPUTE` header to v3.21 (Source += this proposal, implemented).
