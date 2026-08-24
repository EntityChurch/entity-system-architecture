# PROPOSAL — compute budget: de-confound the reduction budget from the cost ceiling; preempt instead of only failing

**Status:** DRAFT — 2026-07-21. **Update 2026-07-23:** the compute corpus's first cross-impl run confirmed the
cost **ceiling** stop-point was *not* cross-impl deterministic (impl-defined `resolve()` cost leaked into it).
Ruled + folded (`ARCH-RESPONSE-COMPUTE-CORPUS-FIRST-RUN` Q2, `EXTENSION-COMPUTE §4.2`): the observable
`operations` ceiling charges `evaluate()` steps **only**, so `budget_exhausted` is now a cross-impl-deterministic,
gated outcome. This proposal's BP-1 (result determinism under the *fairness* reduction budget) is a **separate,
unchanged** MUST — the Q2 ruling supplies the complementary *ceiling* stop-point determinism BP-1 assumed.
**Depends on:** `EXTENSION-COMPUTE` §5 (Budget & Resource Model — the amendment target), §3.19c
(suspended-state retention + resume-safety — the preemption substrate), §7.4 (reactive budget — the
existing non-terminal-exhaustion precedent); `PROPOSAL-CONTINUATION-STANDING-MODEL` (the standing
continuation a yield produces). **No wire change** (§7): the fairness reduction budget is peer-local; the
cost ceiling already travels as `bounds.budget` (ENTITY-CORE-PROTOCOL §5.9).
**Scope:** an **`EXTENSION-COMPUTE` §5 amendment** — split the single op-count into **two bounds with two
behaviors** (a fairness *reduction budget* that yields a continuation; a cost *ceiling* that hard-fails), and
wire reduction-budget exhaustion to the existing continuation substrate. Cross-impl-observable surface:
**result determinism under preemption** (a MUST) + the ceiling's unchanged hard-fail. The reduction-budget
value is peer-local (like the cascade limit).
**Domain:** core / `system/compute` (folds to `entity-core-protocol` + `EXTENSION-COMPUTE §5` at ratify).
**Research (design record):** `docs/research/explorations/EXPLORATION-COMPUTE-EXECUTION-BUDGET-AND-BOUNDARY-LOWERING.md`
(the reframe, the comparative review — BEAM/Wasmtime/EVM/eBPF/Unison, verified — and the three-reliefs
synthesis this proposal makes normative for the budget axis).

---

## §1 Problem — one number is doing three jobs

`init_budget` returns `operations = min(request_budget, bounds_budget, compute_ops_limit)`
(`EXTENSION-COMPUTE §5.2`), and every term defaults to `peer_default_max_ops = 100,000`, a **§9.3
"Recommended Limits" default** — not a spec law. That single `operations` counter is silently serving three
requirements that have different natural magnitudes and different correct behaviors on breach:

| Requirement | Guards | Natural magnitude | Correct breach behavior |
|---|---|---|---|
| **termination** (I-T) | a deterministic evaluator MUST halt; two peers MUST agree it halts | finite | hard stop |
| **fairness** (I-F) | one computation MUST NOT starve reactive / inbox / cross-peer work | **small** (a reduction budget) | **preempt & resume** |
| **accounting** (I-A) | price & refuse untrusted / metered work per capability | **policy-sized** (per grant) | hard-fail |

Conflated into one hard-failing counter, 100k is simultaneously **too big** to be a fairness reduction budget (a
100k-op tick blocks everything else for its duration) and **too coarse** to be a cost policy (a peer-wide
default, not a per-grant price). Today the *only* escape from a heavy tick is **hard failure** (a
`budget_exhausted` error value) — which is why the compute track concluded "past the budget you *must*
shard" (`PROPOSAL-APP-CONVENTION-COMPUTE-PROGRAM §4`). That conclusion holds **only** for the hard-cliff
point; de-confounding the number reopens it (§6). *(This is the same shape as W-CONTINUATION's `ttl` default
(64) colliding with the `chain_depth` ceiling (64) — two magnitudes wearing one number.)*

The comparative evidence (exploration §3, verified): **hard-fail is forced only by an adversarial-consensus
VM (EVM) or a kernel with no room for a meter (eBPF).** Entity is neither, and it already owns the machinery
the preempt-and-resume systems (BEAM; Unison's effect-handler-holds-the-continuation) rely on — its
first-class continuations. So entity is *built for* preemption but does not use it for the budget.

## §2 The design — two bounds, two behaviors

Split the one counter into two, decrementing together but breaching differently. Both count the **same
unit** — one `evaluate()` step (see the terminology note below):

```
budget := {
  operations:       uint   ; total steps remaining for this INVOCATION (the EXISTING field, §5.1) — the cost/termination CEILING
  reduction_budget: uint   ; steps remaining until the next FAIRNESS preemption; resets on each reschedule   ← NEW
  depth:            uint    ; unchanged (§5.4)
}
```

- **`operations` — the cost/termination ceiling (I-A + I-T), *unchanged*.** The existing field (`§5.1`),
  initialized exactly as today — `min(request_budget, bounds_budget, compute_ops_limit)` (`§5.2`) — and it
  **persists across preemptions** (it counts the *whole* invocation's work). Exhaustion → **hard
  `budget_exhausted`** error value, byte-for-byte as today (§4.1). Nothing that relies on it changes. Being
  finite, it is what guarantees **termination** — a genuine non-terminator still dies.
- **`reduction_budget` — the fairness bound (I-F), *new*.** A **small** peer-local count (BEAM's per-schedule
  `CONTEXT_REDS = 4000` reductions is the reference shape). It decrements with `operations`; on reaching zero
  the evaluator **suspends the computation into a standing continuation and reschedules it** (§3) — it does
  **NOT** write an error and does **NOT** consume the ceiling to do so. On reschedule, `reduction_budget`
  **resets** to the peer default; `operations` keeps its value. The computation therefore **completes across
  multiple scheduling turns**, with other work interleaved between them.

Two counters, two breach behaviors: **the reduction budget preempts; the ceiling fails.** A
heavy-but-legitimate computation (big `map`, deep recursion — over the reduction budget, under the ceiling)
now **finishes across reschedules instead of failing**; a genuine over-budget or non-terminating computation
still hits the ceiling and fails.

> **Terminology (why "reduction budget," not "quantum").** The classic OS-scheduling term for this is *time
> quantum* / *time slice* — but (a) "quantum" collides with quantum computing, and (b) "time slice" implies
> a **wall-clock** bound, which is exactly the anti-pattern for a deterministic evaluator (a wall-clock bound
> forks peers; we meter by *steps*, not time — the WASM-epoch lesson, exploration §3). The accepted term for
> a **step-counted** preemption budget is BEAM's **reduction budget** — and "reduction" is doubly apt here
> because entity-compute *literally* reduces a content-addressed expression graph (each `evaluate()` is a
> reduction step). So the unit is an **operation** ≡ a **reduction** ≡ one `evaluate()` step (we keep the
> existing spec noun "operation"/`operations` for continuity, and use "reduction budget" for the new
> per-slice fairness count, matching BEAM). Not a coinage — the established term for exactly this.

**Precedent it generalizes (not invents):** `§7.4` already says budget exhaustion during *reactive*
re-evaluation "writes a `compute/error` … but does **NOT** freeze the subgraph — budget exhaustion may be
transient." The system already treats exhaustion as recoverable in one place; this makes recoverable-via-
preemption the general fairness model and reserves hard failure for the true ceiling.

## §3 Preemption mechanism — the continuation substrate already exists

Yield reuses the landed suspended-state machinery, not a new primitive:

- **The yield point is a boundary, and the boundary set is pinned — not arbitrary mid-node.** A peer MAY
  preempt only at: a **`compute/apply` dispatch boundary**, a **`map`/`filter`/`fold` element boundary**,
  and a **tail-call trampoline iteration** (`§4.1`). These are the points where the evaluator's live state
  is already a clean, capturable frame. (BEAM likewise preempts at reduction boundaries, not mid-
  instruction.) Full arbitrary mid-expression suspension requires the evaluator in explicit-state /
  defunctionalized form — the **Axis-1 decode-once resolved-node walker** is the natural host; a Stage-1
  tree-walk preempts only at the pinned boundaries above, which is sufficient (the heavy shapes are
  iterations and recursions, which hit those boundaries constantly).
  - **Dependency honesty (surveyed 2026-07-21).** Axis-1 today is **single-impl (Go/`entity-workbench-go/
    entitysdk/axis1`), experiment-tier, and only *partially* defunctionalized** — only tail positions return
    capturable continuations; operand and `map`/`fold` evaluation still recurse on the host stack, so it
    **cannot yield mid-operand yet either.** v1's pinned-boundary preemption is fine on Axis-1's
    already-capturable tail frames, but the "full mid-expression suspension" this paragraph gestures at rests
    on an **unbuilt cohort-wide substrate** (no Rust/Py Axis-1, no portable conformance corpus). Do not let
    this proposal imply that substrate exists; the v1 boundary set is what it actually depends on. See
    `ANALYSIS-COMPUTE-CONSTRUCTION-AND-EXECUTION-READINESS` (Part B).
- **The suspended computation is a standing continuation** with the retention + resume-safety guarantees of
  `§3.19c`: its captured scope stays GC-reachable across the pause (and across a peer restart), and a resume
  with a missing binding yields a clean `scope_unreachable` error value, **never a silent substitution**.
  A yield produces exactly such a continuation and hands it to the peer's scheduler for resumption.
- **v1 scope — pure segments only (clean cut, keeps the proposal tight).** Preemption in v1 applies to a
  **pure computation segment** (between impure boundaries — the §6 partition). A computation that has
  performed an effect (`tree.put`/`dispatch`) and would preempt *before its next effect* is the common heavy
  case and is fully covered. Preemption that must **span an impure boundary** (interleaving another
  computation's reads with this one's partial effects) is governed by single-writer-ownership + the reactive
  model and is **deferred** (§8) — v1 yields *at* an effect boundary (a natural, already-observable point),
  not in a way that exposes new intermediate effect states.

## §4 Determinism under preemption — the load-bearing MUST

Preemption is a **local scheduling decision**; it MUST NOT be observable in the result.

> **Normative (BP-1).** The result of evaluating an expression MUST be **independent of the fairness
> reduction budget** — of whether, where, or how many times the evaluator preempted. Two conformant peers
> evaluating the same expression over the same inputs MUST produce the **byte-identical materialized-boundary
> entities** (the same per-tick state hash, the same `construct`/`apply`-arg hashes) regardless of their
> reduction-budget values. Preemption points are invisible to the materialized boundary — the same
> equivalence that makes compilation safe (`GUIDE-CORE §6`): equivalence is pinned at what crosses into the
> tree, never at intermediate steps.

This is a **cross-peer seam** (the methodology's equivalence-collapse shape: an equivalence that holds
locally must not spring apart across a peer boundary), so it is a MUST with a vector:

- **Conformance vector `bp_preemption_invariance`:** run the same `clock-driven` step for N ticks with a
  small reduction budget `b₁ ≪ operations` (forcing many preemptions) and `b₂ = operations` (forcing none);
  assert the per-tick state-hash sequences are **identical**. A skip counts as a failure (honesty rule).
- **Vector `bp_ceiling_unchanged`:** a computation that exhausts the **ceiling** still yields the exact
  `budget_exhausted` error value as pre-amendment (the ceiling path is behavior-preserving).
- **Vector `bp_resume_missing_binding`:** a preempted computation whose captured binding is GC-collected
  before resume yields `scope_unreachable` (§3.19c), never a partial/substituted result.

## §5 What is normative vs peer-local

Clean split of the cross-impl contract from local policy (mirrors how `RECOMMENDED_MAX_CASCADE_DEPTH = 16`
is a peer-local safety knob while cascade *behavior* is normative):

| Element | Status |
|---|---|
| **Result determinism under preemption** (BP-1) | **MUST** — cross-peer observable |
| **Ceiling exhaustion → `budget_exhausted`** (behavior + code + value) | **MUST** — unchanged from today |
| **Reduction-budget exhaustion → yield a standing continuation** (not an error, not ceiling-charged) | **MUST** (the mechanism); the reduction-budget *value* is peer-local |
| **`peer_default_reduction_budget`** (the reduction-budget size) | **RECOMMENDED default, peer-local** — not on the wire, not conformance-pinned (like the cascade limit); §9.3 gains the row |
| **Preemption boundary set** (§3) | **MUST** be a subset of {apply, collection-element, trampoline}; a peer MAY preempt at fewer (down to none — a peer that never preempts is conformant, it just isn't fair) |

A peer that implements **no** preemption is still conformant (it behaves as today: heavy ticks either fit the
ceiling or hard-fail). Preemption is a **liveness/fairness upgrade**, never a correctness precondition — the
same "the floor always works; the upgrade is optional" discipline as the compile gradient (`GUIDE-CORE §6`).

## §6 Relationship to sharding and lowering — reopen the "only answer" ruling

`PROPOSAL-APP-CONVENTION-COMPUTE-PROGRAM §4` states *"sharding is not the preferred budget answer — it is the
only one."* That is true only under the hard-cliff model this proposal removes. **Amendment (applied to that
proposal's §4 as a revision on 2026-07-21, not edited silently here):** there are **three composable reliefs**,
chosen by which pressure applies —

- **Sharding** (spatial / throughput) — split a `map` into k independent fresh-ceiling shards; uses k
  cores. *Landed.* Still the answer for **parallelism**.
- **Preemption** (temporal / fairness — **this proposal**) — yield at the reduction budget, resume; the whole
  computation completes across reschedules. The answer for **a big single tick that shouldn't block the peer.**
- **Lowering** (vertical / per-op cost — separate Stage-3 track, `GUIDE-CORE §6`) — compile a pure interior
  to a native handler (≈1 op instead of N). The answer for **per-op cost**, and the fix for boundary hashing.

They compose (shard a map, lower each shard's interior, preempt if a shard exceeds the reduction budget). The
§4 claim should read: *sharding answers throughput; preemption answers fairness; lowering answers per-op cost
— do not force one to serve all three.*

**Interaction with lowering's op-collapse (named, not solved here).** When a pure interior is lowered to a
native handler it costs ≈1 interpreter op, so the operation ceiling no longer bounds the work *inside* it —
that is what `PROPOSAL-COMPUTE-APPLY-RESOURCE-CEILING` (landed, F1–F5) bounds. This proposal and that one are
the **two halves of the mature bound**: the reduction-budget/ceiling over the interpreter here; the
apply-resource-ceiling over native handlers there. No conflict; noted so the two are read together.

## §7 Scope & non-goals

- **No wire change.** The reduction budget is peer-local (never serialized); the ceiling already travels as
  `bounds.budget` (ENTITY-CORE-PROTOCOL §5.9). Cross-peer dispatch is unchanged — each peer enforces its own
  budgets (`§5.3`).
- **Not lowering.** Stage-3 native lowering (the boundary-hashing fix) is a **separate proposal** on the
  compute-standardization track; this proposal is only the budget/preemption split. They share the
  exploration but ratify independently.
- **Not a new capability field.** The ceiling stays on the existing `constraints["system/compute"]`
  (`§5.5`); the reduction budget is a peer config value, not a capability constraint (fairness is a local
  concern, not a granted authority).
- **Depth is unchanged** (`§5.4`); this proposal touches operation-budget only.

## §8 Open questions

1. **Reduction-budget default value.** BEAM's 4000 reductions is the reference; entity's operation is
   finer-grained than a BEAM reduction, so the right number needs measurement (target: sub-millisecond
   preemption latency without material throughput loss — the Wasmtime-fuel ~overhead question). Route to the
   cohort with a measurement ask; ship the *mechanism* with a conservative default.
2. **Preemption across an impure boundary (v1-deferred, §3).** The interleaving of a preempted computation's
   partial effects with another computation's reads is governed by single-writer-ownership + the reactive
   model; pin the exact rule (does a yielded computation's already-materialized `put` become visible at the
   yield point? — lean yes, it is already a materialized boundary) before extending preemption past pure
   segments. Ties to the keystone single-writer-ownership hand-off.
3. **Scheduler policy — starvation & priority.** Once computations yield and reschedule, the peer needs a
   run-queue policy (FIFO? priority for reactive/emission work over batch?). BEAM has one; entity's is
   undesigned. Name it; do not design it here (it is a peer-local scheduler concern, N-class), but flag that
   an unfair scheduler negates the fairness win.
4. **Ceiling accounting across resume + cross-peer.** Confirm the ceiling persists correctly across a resume
   that crosses a peer boundary (a continuation resumed on a *different* peer — `PROPOSAL-CONTINUATION-STANDING-MODEL`);
   the receiving peer re-initializes budgets from its own bounds (`§5.3`), so a cross-peer-resumed computation
   gets the receiver's ceiling — state that explicitly.
5. **`request_budget` and the reduction budget.** A caller that passes `params.budget` sets the ceiling; can
   a caller also request a reduction budget, or is it strictly peer-local? Lean strictly peer-local (fairness
   is the host's concern, not the caller's) — confirm.

## §9 Ready-to-fold spec text (paste into `EXTENSION-COMPUTE §5` at fold)

> **Fold gate.** This block is the *mechanical* fold, not a licence to edit the landed spec now. W-BUDGET is
> `📄 DRAFT / unbuilt`; per the draft→build→validate→fold discipline, `EXTENSION-COMPUTE` is edited only after
> the cohort builds the split + the ceiling/preemption vectors (§4) run green three-way. Until then this is
> the pre-authored diff so the fold is paste-and-verify, not re-derivation. Line references are to
> `EXTENSION-COMPUTE.md` as of the survey (`§5.1`–`§5.5`).

**§5.1 Budget Structure — replace the struct + the decrement note.**

```
budget := {
  operations:       uint   ; Remaining operation count for the whole INVOCATION — the cost/termination ceiling
  reduction_budget: uint   ; Peer-local: steps remaining until the next fairness preemption; resets each reschedule
  depth:            uint    ; Remaining recursion depth
}
```

Every call to `evaluate()` decrements **both** `operations` and `reduction_budget` by 1. Recursive calls
decrement `depth` by 1 (restored on return). The two operation counters count the **same unit** (one
`evaluate()` step) but breach differently: `operations` reaching 0 is a **hard `budget_exhausted`** error
value (unchanged, §4.1) and bounds termination; `reduction_budget` reaching 0 **suspends the computation into
a standing continuation and reschedules it** (§5.6) — it writes no error and does not consume `operations`.
`reduction_budget` is **never serialized** (peer-local; not carried on the wire, not a capability constraint).

**§5.2 Budget Initialization — add one line; `operations` init is unchanged.**

```
  return {
    operations:       min(request_budget, bounds_budget, compute_ops_limit)   ; unchanged
    reduction_budget: peer_reduction_budget_default                            ; NEW — peer config, not from wire/params
    depth:            compute_depth_limit
  }
```

`peer_reduction_budget_default` is a peer configuration value (see §8 open-Q1 for the reference magnitude);
it is not read from `params`, `bounds`, or `constraints["system/compute"]`. A caller MUST NOT be able to set
it (fairness is the host's concern — §7, open-Q5).

**§5.3 Budget Consumption — one added sentence.** In-process and cross-peer dispatch are unchanged.
`reduction_budget` is shared through the in-process `ctx` exactly as `operations` is, and is **not** part of
any `budget_consumed` accounting (it is per-peer-schedule, meaningless to another peer). A cross-peer-resumed
continuation re-initializes both `operations` and `reduction_budget` from the receiving peer's own bounds
(open-Q4).

**§5.5 Capability Constraints — no change.** `reduction_budget` is deliberately **not** a
`constraints["system/compute"]` field: it is a local fairness quantum, not a granted authority (§7).

**New §5.6 Preemption (fairness) — insert after §5.3.** Carries §3 (mechanism: pinned boundary set, standing
continuation via §3.19c, v1 pure-segment scope) and **§4 verbatim as the normative BP-1 MUST** plus its three
vectors (`bp_preemption_invariance`, `bp_ceiling_unchanged`, `bp_resume_missing_binding`).

---

*Authored in the arch workspace; ratifiable via proposal → ratify → fold into `EXTENSION-COMPUTE §5` +
the `entity-core-protocol` budget section. Tracked as **W-BUDGET** (`docs/status/WORKSTREAMS.md`). Carries a
revision to `PROPOSAL-APP-CONVENTION-COMPUTE-PROGRAM §4`'s "sharding is the only answer" ruling (§6).*
