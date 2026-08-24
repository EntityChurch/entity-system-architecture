# PROPOSAL — the alternate-engine admission contract (one rule for every execution strategy)

**Status:** **IMPLEMENTED — folded to `EXTENSION-COMPUTE §11` (v3.22, 2026-07-23).** The fold gate — a green
**inproc** admission run with a real alternate engine (AE-5) — was met: **Axis-1** (Go) was admitted 330/330
byte-identical alternate==reference, 0 fallbacks, `engine-role==alternate` enforced (`entity-core-go` report
`2026-07-23-ae5-axis1-inproc-admission-GREEN`), and the run demonstrated AE-5's teeth (it caught an index-code
drift the transcribed engine had missed). **§5 (Axis-1's home) ruled:** Axis-1 **stays in workbench** (the
research/experiment tier) — §11.5 leaves the engine representation impl-local, the admission *run* (not the
engine) is what crosses into the conformance tier, and the workbench→core-go emission hand-back is a cheap
one-time seam not worth collapsing the research/conformance-tier distinction over. **Residual (ledger, not
blocking the fold):** admission is one impl's engine (independent second engine = future); the Q1 materialized-error
boundary is validated intra-Go but still gated cross-impl (Python+Rust materialize code-only + a materialized-error
vector three-way); and a narrowed Stage-1 `fn → closure`/non-closure seam no vector reaches is latent (same
category as Q1). The appendix text below is
retained as the rationale of record.
**(prior)** DRAFT — appendix text FINALIZED & ready-to-fold as `EXTENSION-COMPUTE §11` (2026-07-23); fold
fired on the first green **inproc** admission run (AE-5). This pass closed the boundary set over the Q1 error
path + the Q2 budget stop-point, added AE-5/AE-6 (inproc + no-fallback admission evidence — the LOCK's lesson),
pinned the boundary bytes to the spec/corpus not to any impl, and added the divergence-class coverage
(indirection, error, budget-edge, float, cross-peer) + their vectors. **Update 2026-07-23 (earlier):** the
portable corpus this fold is gated on now **exists and has run** (core-go, Go↔Rust 325/328 — `ABSORPTION-compute-corpus-first-crossimpl-run`). It surfaced that
AE-1's boundary was **under-specified on error outcomes** (~40% of the input space): a materialized
`compute/error` carried impl-dependent `message`/`at`/`expression` in its content hash. Ruled + folded
(`ARCH-RESPONSE-COMPUTE-CORPUS-FIRST-RUN` Q1, `EXTENSION-COMPUTE §2.4`): the materialized error boundary is
**`code`-only**, so `ae_boundary_equivalence` is now well-defined on error outcomes. **2026-07-23: the corpus
LOCKED three-way (330/330).** §8's corpus gate is **satisfied on the boundary signal** — the error boundary is
defined (Q1) and the three-way lock is the cross-impl evidence over the full input space incl. the ~40% error
paths. **Remaining admission step: the per-impl in-process (`inproc`) alternate-engine run** (guard 6 — the
wire/handler lock cannot attest *which engine* ran). This is now the **highest-value next compute spec write**,
blocked on the inproc run, not the corpus. (One caveat rides along: Q1's materialized-error path is still
latent cross-impl — Python+Rust must materialize code-only + a materialized-error vector must run three-way — so admission over materialized errors gates
with that sync.)
**Domain:** core / `system/compute` — a new **`EXTENSION-COMPUTE` appendix** (the conformance definition for any non-reference execution strategy).
**Scope:** unify **two independently-derived boundary-equivalence rules** — the lowering contract's **L-1** (a compiled backend) and the Axis-1 admission contract (a faster interpreter) — into **one** admission rule over *any* alternate engine, plus the three cross-engine invariants that make it hold.
**Depends on:** the determinism boundary (`GUIDE-CORE §6`; V7 §1.4/§1.7); the **frozen conformance corpus** (§10 /
`GUIDE-CONFORMANCE §7c`) + the canonical CBOR encoding + the v3.19 value model **as the authority for the boundary
bytes** (`§7c.4`(3) — no impl is privileged). The reference Stage-1 interpreter (`entity-core-go/ext/compute`) is a
*reference implementation of those semantics*, a differential fixture-builder — **not** the oracle.
**Absorbs:** `PROPOSAL-COMPUTE-LOWERING-CONTRACT` **L-1** (becomes the compiled-backend instance of this rule); the Part-B admission contract of `ANALYSIS-COMPUTE-CONSTRUCTION-AND-EXECUTION-READINESS`.
**Folds to:** `EXTENSION-COMPUTE` — a new top-level section (**§11, "Alternate-Engine Admission"**; the spec is
numbered §1–§10 with no appendices, so §11 is the correct home, not the working-title "App-E"), after §10
Conformance. Fold gated on the per-impl **inproc** admission run (§7).
**Tracked as:** the cross-cutting *corpus + admission* line (`WORKSTREAMS.md`). **This is the single highest-value spec write of the compute arc** (`HANDOFF-2026-07-21` §5).

---

## §1 Why one rule — two proposals converged on the same MUST

Two separate threads each arrived, independently, at *byte-identical materialized-boundary entities* as the conformance test:

- **Lowering (W-LOWERING).** A **compiled** handler is conformant iff it produces the same boundary bytes as interpreting its source IR (`PROPOSAL-COMPUTE-LOWERING-CONTRACT` L-1).
- **Axis-1 (the fast interpreter).** A **decode-once resolved-node** interpreter is conformant iff it produces the same boundary bytes as the reference tree-walk (`ANALYSIS-…-READINESS` Part B).

A compiled backend and a faster interpreter are **both alternate execution strategies** for the same content-addressed IR. Stating the rule twice, once per engine kind, is exactly the **cross-peer equivalence-collapse shape** the methodology warns against (`AGENTS.md`): an equivalence pinned per-instance instead of *once, generally*, springs apart at the next engine kind nobody wrote a rule for. So: **state it once, over "any engine," and let both instances inherit it.**

## §2 The admission rule (AE-1) — ready-to-fold appendix text

> **AE-1 — materialized-boundary equivalence (the admission MUST).** An **alternate execution strategy** — any
> evaluator for compute IR other than the reference interpreter (a faster interpreter, a compiled/fused
> handler, a memoizing backend, a defunctionalized walker) — is **conformant iff**, for the same IR over the
> same inputs, it produces the **byte-identical materialized boundary** the spec defines: the same canonical
> CBOR encoding **and** the same content hash at **every compute→non-compute crossing**, plus the same terminal
> outcome. The boundary set is exactly the crossings of **§2.3 N1** (no new surface — one home) together with
> the two outcome forms the corpus run pinned:
>
> 1. **the materialization crossings** — the §2.3 N1 placement sites (a `compute/scope` binding, a
>    `compute/construct` field, a `compute/apply` argument) and the four compute→non-compute crossings of §2.3
>    (the constructed/evaluated value **stored to the tree / a `result_path`**, **returned from a handler**,
>    **passed as a `compute/apply` arg**, or **sent on the wire**) — each materialized as a **bare** entity per
>    V7 §1.4, byte-identical to the hand-built entity;
> 2. **the cross-peer materialized subtree** — a `compute/closure`'s reachable scope materialized into the
>    envelope `included` map on peer handoff (**§2.3 N7**): the bytes another peer resolves MUST match;
> 3. **a materialized `compute/error`** — content-hashed over **`code` alone** (**§2.4**, Q1); `message`/`at`/
>    `expression` are in-flight diagnostics and are **not** part of the boundary. This is ~40% of the input
>    space and the surface an alternate engine is *most* likely to diverge on (error paths are re-derived, not
>    shared), so it is named explicitly;
> 4. **the terminal resource outcome** — value vs. a resource-limit `compute/error`, **and its trigger point**:
>    `budget_exhausted` (the `operations` ceiling, §5/Q2), `depth_exceeded` (the eval-depth limit, §4.1), and
>    `cascade_limit` (the reactive cascade limit, §7.2) MUST each fire at the **same logical point** on every
>    engine — an engine that completes, or that faults, where the reference does the other has diverged. Held by
>    AE-4 (metering) and its tail-call clause (depth).
>
> **The boundary bytes are fixed by the spec, not by any implementation.** The canonical CBOR encoding (V7 §1.4),
> the v3.19 value model (§2.2/§2.3), and the **frozen conformance corpus** (§10 / `GUIDE-CONFORMANCE §7c`) are
> the authority; the reference interpreter is a *reference implementation of those semantics*, **not** the
> oracle (`§7c.4`(3) — no impl is privileged). An alternate engine is admitted against the **frozen corpus's
> pinned boundary hashes**, which independent ground-up impls have confirmed (the three-way LOCK), never against
> one runtime's bytes.
>
> Equivalence is defined **only** at that boundary. It is explicitly **NOT** required — and MUST NOT be assumed
> — at:
> - **intermediate steps** (per-node content-addressing the reference interpreter does internally is an
>   interpretation artifact an alternate engine is free to drop — `GUIDE-CORE §6`);
> - **content-store contents** (an alternate engine legitimately leaves **fewer** entities in the store — Axis-1
>   materializes only boundary entities, not every intermediate; a byte-for-byte store-equality test is a
>   **false** conformance test that would reject a valid engine).
>
> An engine that changes any boundary hash — or the budget stop-point — is **non-conformant** (a
> miscompilation / mis-evaluation), not an optimization.

This is L-1 with "lowered handler" generalized to "alternate execution strategy," with the content-store-inequality
carve-out (which L-1 did not need to state but the fast-interpreter case makes essential) made explicit, and with
the boundary set closed over the two outcome forms the first corpus run pinned (the Q1 error boundary and the Q2
budget stop-point) — so "byte-identical boundary" is well-defined over the **full** input space, not just the
value path.

## §3 The cross-engine invariants (AE-2 – AE-4) — what makes AE-1 hold

Boundary-equivalence is the *observable*; three invariants are what an engine must preserve to achieve it. Each is a MUST folded into the same appendix:

> **AE-2 — impure-frontier preservation.** An alternate engine MUST NOT reorder, elide, duplicate, or
> introduce an impure operation (`tree.get` / `tree.put` / dispatch / `check_permission`), and MUST register
> the **same reactive dependency set** the reference registers (§7.1) — the impure ops, their order, and the
> reactive edges among them are the boundary of AE-1 and the reactive graph the rest of the system depends on;
> they survive every engine unchanged. A fast interpreter that resolves hashes by a different tier (reverse
> index vs. encountered-during-read) MUST still record the dependency edges the reference records, or reactive
> re-evaluation forks across engines. *(This is the general law under lowering's L-4 "no fusion across an impure
> op" — L-4 is the fusion-specific instance.)*
>
> **AE-3 — demand (laziness) equivalence.** An alternate engine MUST evaluate exactly the **demanded** subgraph
> — no more (speculative evaluation of an undemanded branch could materialize or fault where the reference does
> not) and no less. Same demand ⇒ same effects ⇒ same boundary. **Error-driven demand is part of demand:** the
> §4.1/§2049 short-circuits are demand boundaries an engine MUST replicate exactly — a `compute/if` whose
> condition evaluates to a `compute/error` short-circuits to that error with **neither branch evaluated**; a
> short-circuiting `compute/logic`, and the `if is_error(x): return x` after every sub-evaluation, likewise cut
> demand. An engine that eagerly evaluates a branch the reference short-circuits away can materialize or fault
> where the reference does not — a boundary divergence.
>
> **AE-4 — metering at the logical boundary (the §5 / Q2 ceiling).** The observable operation budget is metered
> over **logical evaluate() steps**, not host/engine steps — the **same `operations` ceiling §5.1/§4.2 (Q2)
> pins at one decrement per `evaluate()` step, `resolve()` at zero.** A faster interpreter that does one graph
> reduction in fewer host instructions still decrements `operations` **once per logical `evaluate()` step**; a
> fused/compiled handler that runs a pure interior natively MUST account **the logical step count the reference
> would charge on the same demanded path** (data-dependent loops included — N iterations × interior steps), via
> the `apply`-resource-ceiling bridge (`PROPOSAL-COMPUTE-APPLY-RESOURCE-CEILING`; `…-BUDGET-PREEMPTION §6`). A
> handler that counted itself as ≈1 op would **complete where the reference `budget_exhausted`s** — a boundary
> divergence (outcome 4 of AE-1), not a speedup. This is what keeps the budget cliff — and preemption (BP-1) —
> from forking across engines: because metering is logical, `budget_exhausted` and its stop-point are
> engine-independent, exactly as Q2 makes them impl-independent.
>
> **Depth and the tail-call trampoline are part of the logical model.** Eval-depth accounting (`depth_exceeded`,
> §4.1) is logical, not host-stack: the spec's **tail-call trampoline** — *tail calls do not consume depth*
> (§4.1, `is_tail_call`/`tail_call`, the T1–T3 amendment) — is a **spec-defined semantic** every engine MUST
> reproduce. An engine that recurses on the host stack where the reference trampolines will hit `depth_exceeded`
> (or a host stack overflow) where the reference loops indefinitely or completes — a terminal-outcome divergence.
> So AE-4's "logical step count" includes the logical **depth** the trampoline defines, making `depth_exceeded`
> engine-independent alongside `budget_exhausted`.

## §4 What inherits vs. what stays engine-specific

One rule, two instances, and the engine-specific MUSTs sit **on top** of it — this is the layering the unification buys:

| Rule | Kind | Relationship to AE-1 |
|---|---|---|
| **AE-1** boundary-equivalence | admission (all engines) | the general rule |
| **AE-2/3/4** frontier / demand / metering | admission (all engines) | the invariants under AE-1 |
| **AE-5/6** inproc evidence / no-fallback | admission *procedure* (all engines) | how AE-1 is demonstrated (the LOCK's lesson) |
| Lowering **L-1** | compiled backend | **retired into AE-1** (was the compiled instance; now a pointer) |
| Lowering **L-2** source-IR preservation | compiled backend only | **stays** — a *compiled* handler must keep its source; a fast interpreter has no separate compiled artifact, so N/A to it |
| Lowering **L-3** deopt fallback | compiled backend only | **stays** — the interpreter-over-source is the authority a *compiled* path falls back to |
| Lowering **L-4** fusion unit | fusion backend only | **stays** as the fusion-specific instance of AE-2 |
| Lowering **L-5** memoization soundness | memoizing backend only | **stays** — pure-only, byte-identical (AE-1 holds under caching) |
| Axis-1 admission | fast interpreter | **retired into AE-1** (was the interpreter instance; now a pointer) |

So `PROPOSAL-COMPUTE-LOWERING-CONTRACT` keeps L-2…L-5 (genuinely lowering-specific) and cites AE-1 in place of its L-1; the readiness Part-B contract becomes a pointer to this appendix. Net: **two admission statements collapse to one; four lowering-specific MUSTs remain where they belong.**

## §5 Interpreter-architecture guidance (RECOMMENDED — not normative)

Per Recommendation B(3): the representation stays **impl-local** (resolved-node / slots / decode cache — `GUIDE-CORE §6`), but the appendix carries a **RECOMMENDED** note so the cohort's fast interpreters converge on a shape the hard features can lean on:

> **RECOMMENDED.** An **explicit-state / resolved-node** evaluator (versus a recursive host-stack tree-walk)
> is what makes **preemption** (a capturable frame at a yield boundary — `…-BUDGET-PREEMPTION §3`) and
> **fusion** (a stable interior to specialize — L-4) tractable. Impls SHOULD converge on an explicit-state
> evaluator for their fast path. This is guidance, **not** a mandated representation — AE-1 admits any engine
> that is boundary-equivalent, however structured.

## §6 Conformance — the corpus *is* the artifact; games are insufficient

AE-1 has teeth only against a **differential corpus of random well-formed graphs**, not the game programs. The B4 lexical-slot case is the proof: Axis-1's no-lexical-context handling of `lookup`-fetched expressions is a correctness point the code itself comments *"no Life/Snake test would notice"* — the **CDN-corridor meta-lesson** (a subtle equivalence bug toy programs pass while it hides). Games are necessary but categorically insufficient. The first three-way run proved the point again: the **F-1 numeric-cast through-indirection collapse was invisible to Rust's own direct-case test and to prose review — the seeded sweep found it.**

### §6.1 Admission evidence is the **in-process** corpus run (normative — the LOCK's lesson)

> **AE-5 — admission is demonstrated in-process, never over the wire.** An alternate engine is admitted only by
> running the **frozen conformance corpus in-process with that engine as the evaluator** and matching the pinned
> boundary (AE-1). A **wire/handler-path** conformance run (a peer answering `EXECUTE` over the network) is
> **not** admission evidence: it attests boundary-equivalence of the *peer*, but **cannot attest which engine
> ran** inside it, so it cannot distinguish the alternate engine from a reference fallback. Admission requires
> the in-process (`inproc`) profile (`GUIDE-CONFORMANCE §7c.3`).
>
> **AE-6 — the alternate engine MUST run every vector; no per-vector fallback (guard 6).** Every corpus vector
> in an admission run MUST be evaluated by the **alternate** engine on each side — never silently deopted to the
> reference for the hard cases. A run that falls back compares the reference to itself: the agreement is
> circular and vacuous. (This is `§7c.4`(2)'s sixth anti-vacuity guard, made an admission MUST. Deopt at
> *production* runtime is L-3 and correctness-preserving; deopt during an *admission run* voids the evidence for
> that vector.)

### §6.2 The corpus must exercise the divergence-prone classes (coverage, normative-SHOULD)

A corpus that admits an engine SHOULD exercise the classes where alternate engines actually diverge — absence of
a class is silent under-coverage the ledger must state (`§7c.4`(6)):

- **numeric-intent through indirection** — the F-1 class: `numeric-cast` consumed directly vs. through a `let` /
  `if` / `construct` / scope binding (§2.2 rule 11). Materialize the cast result, not just the in-flight value.
- **error paths** (≥ the guard's error-code floor) materialized to a boundary — the Q1 class; the vector MUST
  compare `code` only (not `message`/`at`).
- **budget-edge** — a workload deliberately partway through its `operations` ceiling (the Q2 class); asserts the
  `budget_exhausted` stop-point, not just completion.
- **deep tail recursion** — a tail-recursive loop past the host recursion limit but within the eval-depth budget
  (§4.1 trampoline): the reference loops/completes, a non-trampolining engine faults `depth_exceeded` or
  overflows — the terminal-outcome (AE-4 depth) class.
- **float-carrying entities** — the canonical-CBOR float encoding (the known cbor2 float16-minimization Rule-4
  gap, `guides/GUIDE-EXTENSION-DEVELOPMENT`); a float boundary is where a re-encoder diverges.
- **cross-peer closure transfer** — a closure whose scope materializes into `included` (§2.3 N7), if the wire
  path exists.

> **Conformance vectors** (the appendix references the corpus, §10 / §7c):
> - `ae_boundary_equivalence` — over the differential corpus: same IR + inputs ⇒ identical boundary-hash
>   sequence on the reference interpreter and the alternate engine, **in-process** (AE-5). A skip counts as a failure.
> - `ae_error_boundary` — a materialized `compute/error` compares **`code`-only** across engines (AE-1 outcome 3 / Q1).
> - `ae_resource_stoppoint` — a resource-edge workload reaches the same terminal resource outcome at the **same
>   logical point**: `budget_exhausted` (budget edge), `depth_exceeded` (a tail-recursive loop the reference
>   trampolines — an engine that recurses faults here), and `cascade_limit` (reactive) (AE-1 outcome 4 / AE-4 / Q2).
> - `ae_no_intermediate_leak` — assert the alternate engine's **store contents may differ** (fewer entities)
>   while the **boundary hashes match** — i.e. the test does **not** over-assert store equality.
> - `ae_demand_equivalence` — an undemanded branch that would fault/materialize is **not** evaluated by the
>   alternate engine, **including an error-short-circuited `if` branch** (AE-3).
> - `ae_metering_parity` — the `operations` decrement sequence matches per logical step across engines (AE-4).
> - `ae_dependency_parity` — the reactive dependency set an engine registers matches the reference (AE-2 / §7.1).

## §7 Fold gate & honest ledger

- **Fold gate (updated 2026-07-23).** The corpus gate is **satisfied**: the portable cross-impl corpus exists
  and **LOCKED three-way** (330/330 byte-identical Go/Rust/Python, oracle-pinned `seed 20260716 / corpus-SHA
  d0fdd757`; `ABSORPTION-compute-corpus-first-crossimpl-run`). AE-1's boundary is now well-defined over the full
  input space — the value path (the lock), the error path (Q1 defined it code-only), and the budget stop-point
  (Q2). **What remains before §11 folds into the Active spec is the per-impl `inproc` admission run (AE-5) with
  an actual alternate engine (AE-6)** — the three-way LOCK is a **wire/handler-path** run and by AE-5 does *not*
  by itself admit any engine. So this appendix is **final and ready-to-fold**; the fold fires when the first
  inproc admission run is green. *(One rider: the Q1 materialized-error boundary is latent cross-impl until
  Python+Rust materialize code-only and a materialized-error vector runs three-way — so `ae_error_boundary`
  admission gates on that; the rest of the boundary is already locked.)*
- **Honest ledger.** "Rust eval == Go eval" is **no longer assumed — it is measured** over 330 vectors incl. the
  error paths (the LOCK). But that is the **reference evaluator** in each impl over the **wire path**, not an
  **alternate** engine in-process: the only alternate engine that exists is **Axis-1 — Go-only, experiment-tier,
  partially defunctionalized (tail-position continuations only)**, and **no cross-impl alternate engine has been
  admitted** (none has been run inproc against the locked corpus). Do **not** read the LOCK as AE-1 admission
  evidence — it is the *boundary* the admission run will be checked against, not the admission itself.

## §8 Non-goals

- **Not the corpus itself.** The portable vector set + cross-impl harness is the cohort's build — **done and
  LOCKED three-way** (2026-07-22/23). This proposal is the **rule** that corpus enforces, and the **admission
  procedure** (AE-5/6) an alternate engine runs against it.
- **Not a mandate to build an alternate engine — and explicitly *not* a mandate for all peers to build the same
  one.** AE-1 governs any engine a peer *chooses* to add; the reference interpreter alone remains fully
  conformant, and is the conformance floor (validated by the three-way LOCK). An alternate engine is a
  **performance opt-in**, built by an impl when *its* workload justifies it. **Do not port one impl's engine
  across the cohort:** three ports of Go's Axis-1 would be **cohort-consistent, not independent convergence**
  (the honesty doctrine, `AGENTS-STANDARD` §"Honesty & conformance") — weaker evidence, not stronger. The
  contract's strength comes from **independent** strategies (e.g. a compiled/fused Go handler *and* a Rust
  resolved-node interpreter) hitting the **same locked boundary**. And the representation stays impl-local (§5,
  RECOMMENDED not mandated), so "everyone builds Axis-1" would also over-fix an architecture we deliberately left
  free. **For AE-1's teeth, one alternate engine run inproc against the locked corpus is sufficient** (Go's
  Axis-1 — the routed next step); a second, independent engine — demand-driven — is what later upgrades the
  evidence from "the admission procedure works" to "independent strategies interoperate."
- **Not a representation spec.** Axis-1's resolved-node/slot model stays impl-local (§5).

---

*Authored in the arch workspace. Ratifiable via proposal → ratify → fold into a new `EXTENSION-COMPUTE`
appendix; the fold is gated on the corpus (§7). Unifies lowering's L-1 + the Axis-1 admission contract into
one rule for all alternate engines — the equivalence-collapse fix stated once, generally. Recurring lesson:
pin the contract at the materialized boundary; free every engine behind it.*
