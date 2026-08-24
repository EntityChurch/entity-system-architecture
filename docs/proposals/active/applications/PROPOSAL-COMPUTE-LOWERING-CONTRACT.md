# PROPOSAL — the lowering contract: what a compiled compute interior MUST preserve

**Status:** DRAFT — 2026-07-21.
**Depends on:** `GUIDE-CORE-COMPUTATIONAL-ARCHITECTURE` §6 (the compile gradient — this proposal
**formalizes its principles into normative MUSTs + vectors**; Stage-3 native compilation is marked
"Unbuilt" there); `EXTENSION-COMPUTE` §2.3 (`construct` materialization — the boundary shape), §6 (purity
model — the fusion partition); `PROPOSAL-COMPUTE-BUDGET-PREEMPTION` (the op-collapse interaction, §8);
`PROPOSAL-COMPUTE-APPLY-RESOURCE-CEILING` (landed — bounds the native handler lowering produces).
**No wire change; no new expression types.** Lowering is a **per-peer backend** (N-class, `GUIDE-CORE §3`);
the cross-impl-observable surface is **not** the compiler but the **entities that cross the materialized
boundary**, which are byte-identical whether interpreted or compiled.
**Scope:** pin the **normative contract** a conformant lowering MUST honor (boundary-equivalence,
source-IR preservation, deopt fallback, memoization soundness) so the cohort can build Stage-3 native
lowering **safely** — and so "compiled" can never mean "diverged." The *strategy* (§6) is RECOMMENDED design
guidance, not normative.
**Domain:** `system/compute` backend / compute-standardization track (folds to `GUIDE-CORE §6` +
`EXTENSION-COMPUTE` conformance).
**Research (design record):** `docs/research/explorations/EXPLORATION-COMPUTE-EXECUTION-BUDGET-AND-BOUNDARY-LOWERING.md`
§6/§6a/§6b/§7 (the design, the verified comparative — Truffle/Futamura/JAX-XLA/Salsa/Nix-CA/deforestation —
and the deopt safety-net).

---

## §1 Why now — the sustained-parallel-load gate

The measured hot cost of interpreted compute is **not arithmetic — it is decode + hash of intermediate
nodes**: ~54% per-node `cbor.Unmarshal` + scope-load ≈ **~75% of a tick** (Life 16×16,
`ABSORPTION-compute-program-poc-life-snake`). Under **sustained parallel compute load** — many programs,
many ticks, many peers — that ~75% is the difference between a peer that keeps up and one that falls behind
and **wedges** (the machine-boundary survivability concern: a peer that cannot sustain its load is the
Profile-3 failure the substrate work is trying to make cheap, not fatal). **Lowering is the *vertical*
relief** (the third of the three composable reliefs, `PROPOSAL-COMPUTE-BUDGET-PREEMPTION §6`): it attacks
the per-op cost directly, where sharding (spatial) and preemption (temporal) do not. It is therefore
**load-bearing for the sustained-parallel-load acceptance gate** (§9), not a mere optimization.

The mechanism is already specified in principle (`GUIDE-CORE §6`) and unbuilt: **a compiled interior
computes the same boundary entities in registers, materializing (and hashing) only where a value re-crosses
into the tree** — so the ~75% intermediate decode/hash **evaporates** for the compiled region. What is
missing, and what this proposal supplies, is the **contract** that makes building it safe: the guarantees a
lowering MUST preserve so that "faster" never becomes "different."

## §2 The contract — three MUSTs that make compilation safe

These formalize `GUIDE-CORE §6`'s principles ("determinism pinned at the materialized boundary"; "compiled
is a cache beside the source"; the deopt fallback) into normative, vector-checked requirements.

> **L-1 — boundary-equivalence (the determinism MUST).** A lowered handler MUST produce **byte-identical
> materialized-boundary entities** to interpreting its source compute IR — the same canonical encoding + the
> same content hash for every value that crosses into the tree (a state write, a stored `construct`, an
> `apply` argument, an output-port value). Equivalence is defined **only** at the materialized boundary,
> **never** at intermediate steps: the per-node content-addressing the interpreter uses internally is an
> interpretation artifact the backend is free to drop (`GUIDE-CORE §6`). A lowering that changes any boundary
> hash is a miscompilation, not an optimization.

**Unified (2026-07-21).** L-1 is now the **compiled-backend instance** of the general
`PROPOSAL-COMPUTE-ALT-ENGINE-ADMISSION` **AE-1** (one boundary-equivalence rule for *every* alternate engine —
fast interpreter or compiled handler). L-1 stays here as the lowering-facing statement and cites AE-1 as its
source; L-2–L-5 below remain lowering-specific (source preservation, deopt, fusion unit, memoization) and sit
on top of AE-1. See that proposal §4 for the inherit-vs-specific split.

> **L-2 — source-IR preservation (the genome/reversibility MUST).** A compiled handler MUST exist **in
> addition to**, never instead of, its source compute entities in the tree. "Compiled" is a **cache beside
> the source**, not a replacement. A backend MUST NOT emit a compiled-only handler: such a handler cannot
> travel as transferable IR (Class T), cannot be re-lowered, cannot be inspected, and breaks self-hosting
> (the genome reads source IR). This is the invariant `GUIDE-CORE §6` states; here it is a MUST.

> **L-3 — deopt fallback (the safety-net MUST).** The interpreter over the source IR is **authoritative**. A
> lowered handler MUST fall back to interpreting the source IR whenever it cannot honor L-1 — an input
> outside the specialization's guarded assumptions, a speculation invalidated, or any condition the compiled
> path does not cover. The fallback is a **correctness-preserving deopt** (Truffle's
> `transferToInterpreterAndInvalidate`, exploration §6b), never a miscompile or an error. Because L-2 keeps
> the source in the tree, the fallback target always exists.

Together: **L-1** says the compiled path is *observably identical*; **L-2** says the source is *always
present*; **L-3** says the interpreter is *always the authority*. That triad is what lets a peer compile
aggressively without any risk of divergence — the compiled path is a fast cache that is continuously
falsifiable against the authoritative interpreter.

## §3 The fusion unit — a pure interior between impure boundaries

*What* may be lowered/fused is pinned by the purity partition, so a backend never fuses across an effect:

> **L-4 — the fusion unit.** A backend MAY fuse a **pure compute subgraph** — a region containing **no**
> impure operation (`tree.get` / `tree.put` / dispatch / `check_permission`) — into a single native handler.
> The impure operations, and the reactive edges among them, are the **boundary** and MUST remain observable
> (they are the materialized-boundary entities of L-1 and the reactive graph's preserved edges). A backend
> MUST NOT fuse across an impure operation: doing so would hide an effect or a reactive edge that the rest of
> the system depends on.

This is `GUIDE-CORE §6`'s "four-way coincidence": purity / compilation / reactivity / visibility all anchor
on the same impure ops, and the lookup-type partition (`scope` pure / `tree` impure-reactive / `hash`
pure-fixed) **is** the fusion frontier. Inside a fused interior the intermediates have **no addressable
existence** (XLA's fusion invariant — nothing materialized, nothing hashed); only inputs and the final
boundary entity cross the seam. This is deforestation applied to **hashed entities** — the highest-value
fusion here because each intermediate is a CBOR-encode + SHA-256, the exact ~75%.

## §4 Memoization & early-cutoff — the content-address makes it free

Because the IR is content-addressed, a fused interior is memoizable with **no separate proof layer**
(`GUIDE-CORE §3`: identical structure ⇒ identical hash ⇒ valid reuse). This is where the sustained-load win
compounds across ticks:

> **L-5 — memoization soundness.** A backend MAY cache a fused interior's result keyed by the **content hash
> of its inputs**, and MAY **skip re-evaluation** when the input hash is unchanged (reusing the cached output
> hash — Salsa red-green *early-cutoff* / Nix-CA skip-on-identical-output). Such caching is sound **only over
> the pure fragment** (L-4): an impure operation MUST NOT be memoized (it may observe or change the world).
> A memoized result MUST be byte-identical to recomputing it (L-1 holds under caching too).

The payoff for sustained parallel load is **sparse-update early-cutoff**: when most of a program's state is
unchanged tick-to-tick (Doom actors: O(actors) change, the world is otherwise identical —
`EXPLORATION-COMPUTE-WHOLE-STATE-TICK-COST`), only the changed subtrees recompute; the unchanged majority
costs a **hash comparison**, not an eval. This lifts the content-store dedup invariant (#2) from storage to
computation, and it is what turns "many ticks per second across many programs" from O(whole-state) into
O(delta) — the actual lever for sustaining load.

## §5 Normative vs impl-local

| Element | Status |
|---|---|
| **L-1 boundary-equivalence** | **MUST** — cross-impl observable (the compiled path is falsifiable against the interpreter) |
| **L-2 source-IR preservation** | **MUST** — a compiled-only handler is non-conformant |
| **L-3 deopt fallback** | **MUST** — the interpreter over source is authoritative |
| **L-4 fusion unit** (no fusion across an impure op) | **MUST** — the boundary stays observable |
| **L-5 memoization soundness** (pure-only; byte-identical) | **MUST** *when a backend memoizes*; memoizing at all is OPTIONAL |
| **Whether a peer compiles at all; the strategy; the cache policy** | **impl-local, N-class** — an interpret-only peer is fully conformant |

The discipline is `GUIDE-CORE §6`'s: **the interpreted floor always works; compilation is an optional
upside; the build track MUST NEVER make the upside a precondition of the floor.** A peer that only
interprets passes conformance; it is just slower. So this proposal constrains lowering *if you do it* — it
does not mandate doing it.

## §6 Strategy (RECOMMENDED design guidance — not normative)

The verified comparative (exploration §7) points at one composed strategy; a backend SHOULD follow it but
MAY choose otherwise so long as §2–§5 hold:

1. **Lower the tree-walker by partial evaluation (Truffle / first Futamura projection).** Hold the
   content-addressed step IR *constant* and partially-evaluate `evaluate()` against it so dispatch
   constant-folds into straight-line native code. The immutable, hashed IR is an ideal PE target. **Guard
   every speculation; deopt to the interpreter (L-3) on an invalidated guard.**
2. **Fuse the pure interior (XLA-fusion / deforestation)** so intermediates never materialize (§3) — this is
   the direct ~75% attack.
3. **Cache keyed by content hash; early-cutoff on unchanged input (Salsa/Nix-CA)** — the trace-compile-cache
   loop of JAX/`torch.compile`, except the cache key *is* the content hash, computed for free (§4).
4. **Sequence: Axis-1 first, measure, then Stage-3 where it still matters.** The **Axis-1 decode-once
   resolved-node interpreter** already buys ~20–33×/tick (measured 34–163×/tick, 69–304× on eval — the
   `LoadScope` cost collapses O(N²)→O(N)) by killing the per-node decode + scope-load (the bulk of the ~75%)
   **without** compiling. The honest gating question (`GUIDE-CORE §6`: the gradient is optional): measure
   whether Axis-1 alone clears the sustained-load gate (§9); build Stage-3 native fusion **only** for the
   interiors that still dominate after Axis-1. Do not compile speculatively.
   - **Dependency honesty (surveyed 2026-07-21).** "Axis-1 first" assumes Axis-1 exists cohort-wide; it does
     **not** — it is **single-impl (Go/`entity-workbench-go`), experiment-tier, with a Go-internal
     equivalence oracle but no portable go/rust/py conformance corpus.** So the measure-first sequence is
     today a **Go-only** measurement, and the sustained-load gate on Rust/browser depends on Axis-1 (or an
     equivalent fast interpreter) existing there first. Track that prerequisite explicitly; see
     `ANALYSIS-COMPUTE-CONSTRUCTION-AND-EXECUTION-READINESS` (Part B) and the execution-strategy admission
     contract it recommends.

## §7 Conformance vectors

- **`lc_boundary_equivalence`** — for each probe program (Life / Snake / Asteroids), run N ticks interpreted
  and N ticks with the interior lowered; assert **identical per-tick state-hash sequences** and identical
  output-port boundary hashes (L-1). A skip counts as a failure.
- **`lc_deopt_fallback`** — feed a lowered handler an input outside its guarded specialization; assert it
  produces the **same** boundary entity as pure interpretation (via deopt), never an error or a wrong value
  (L-3).
- **`lc_source_present`** — assert a compiled handler's source compute entities are resolvable in the tree
  (a `tree:get` of the source IR path succeeds) and the handler still travels as Class T (L-2).
- **`lc_memoization_identity`** — with memoization on, assert a cache hit yields byte-identical results to a
  cache miss; assert **no** impure operation is served from cache (L-5).

## §8 Relationship to the budget and to sharding

- **Op-collapse → the apply-resource-ceiling (the co-evolution, from `…-BUDGET-PREEMPTION §6`).** A lowered
  interior costs ≈**1 interpreter op** (the `apply` node-visit) instead of N, so the op-`operations` ceiling
  no longer bounds the work *inside* it. That work is bounded by `PROPOSAL-COMPUTE-APPLY-RESOURCE-CEILING`
  (landed, F1–F5). **The two are the two halves of the mature bound** — the reduction-budget/ceiling over
  the interpreter; the apply-resource-ceiling over native handlers — and lowering is the mechanism that
  *moves* work across that line. Read them together.
- **Composes with sharding and preemption.** Shard a `map` (spatial), lower each shard's pure interior
  (vertical), and preempt at the reduction budget if a shard is still heavy (temporal). The three reliefs
  are orthogonal; lowering is the one that raises the *per-core* ceiling the other two distribute and
  time-slice.

## §9 The acceptance gate & open questions

1. **Define the sustained-parallel-load test's pass criteria.** This proposal is motivated by it; the test
   itself needs a pinned definition — how many concurrent programs × ticks/s × peers, on what reference
   hardware, with what latency/throughput bar and no wedging (the survivability signal). **Route to the
   cohort to define**; lowering's contribution (per-op cost ↓ + early-cutoff for sparse updates) is measured
   against it. Without a defined gate, "essential for performance" is an assertion, not a target.
2. **Axis-1 vs Stage-3 — the measure-before-build fork (§6.4).** Does Axis-1 alone clear the gate? Build
   Stage-3 native fusion only for the residual hot interiors. Do not compile before Axis-1 is measured.
3. **Cache invalidation & memory (L-5).** The output-hash cache is unbounded without eviction; ties to the
   **deferred GC collector** (`EXTENSION-COMPUTE §3.19c`, the keystone crash-mid-flight collector). A
   sustained-load peer *will* hit memory pressure — the eviction policy is part of "sustaining" load. Route
   with the GC work.
4. **Deopt cost under adversarial input.** A workload that constantly invalidates guards pays the deopt tax
   every tick (Truffle's `recompile_limit`/eager-fallback shape). Pin a bound: after K deopts, a handler
   SHOULD fall back to interpret-only for that program rather than thrash — a peer-local policy, named so it
   isn't discovered under load.
5. **Fusion-region discovery.** How does a backend *find* the maximal pure interiors (the L-4 units)? A
   static pass over the IR partitioning by lookup-type (`scope`/`hash` pure vs `tree` impure) — likely the
   Axis-1 resolved-node graph already exposes this. Impl question, routed with the Stage-3 build.

## §10 Scope & non-goals

- **Not a mandate to compile.** Interpret-only is conformant (§5). This constrains lowering *if done*.
- **Not a new expression/type/wire surface.** Zero `EXTENSION-COMPUTE` expression change; the contract is
  over the *backend*, observable only at the already-pinned materialized boundary.
- **Not the budget model.** The reduction-budget/ceiling split is `PROPOSAL-COMPUTE-BUDGET-PREEMPTION`; this
  is the per-op-cost relief. Siblings, ratify independently.
- **Not the lowering *toolkit* (frontend→IR).** `GUIDE-CORE §8`'s lowering toolkit is language-frontends
  *producing* IR; this proposal is the *backend* compiling IR to native. Different "lowering," kept distinct.

---

*Authored in the arch workspace; ratifiable via proposal → ratify → fold into `GUIDE-CORE §6` (promoting its
principles to the pinned L-1..L-5 MUSTs) + the `EXTENSION-COMPUTE` conformance set (the `lc_*` vectors).
Tracked as **W-LOWERING** (`docs/status/WORKSTREAMS.md`). Sibling of W-BUDGET (the vertical relief of the
three); the two co-evolve at the interpreter/native-handler bound (§8).*
