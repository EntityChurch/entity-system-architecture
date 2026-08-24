# PROPOSAL — `app/program`: the hostable-compute-program interface (a runtime contract)

**Status:** DRAFT — 2026-07-13.
**Depends on:** `EXTENSION-COMPUTE` (eval + the expression language, unchanged); `system/tree`,
`system/inbox`, `system/subscription` (the port mechanisms — all existing); `EXTENSION-CLOCK` (candidate
tick driver). **No compute-extension change, no wire change** (§6).
**Scope:** an **L5 applications-domain convention** — the standard **descriptor + port model + tick
contract** that makes a *pure compute program* **hostable across runtimes** (browser, workbench-go,
Godot). It standardizes the **seams**, never the I/O. It is the **inspectable-workload sibling** of the
WASM actuation path (`EXPLORATION-RUN-ENVIRONMENTS-AND-WORKLOAD-ACTUATION` — that runs opaque WASM; this
runs transferable compute).
**Domain:** `applications/` (see `CHARTER.md`; charter class = convention/pattern — confirm at fold).
**Research (design record):** `docs/research/explorations/EXPLORATION-COMPUTE-PROGRAM-RUNTIME-CONTRACT.md`
(three worked probes — Life / Snake / Tetris — validate this field set and the "zero compute change"
verdict; the cost calibration; the port taxonomy). **Empirical validation:**
`docs/research/reviews/ABSORPTION-compute-program-poc-life-snake.md` (workbench-go built Life + Snake
descriptor-faithfully, ran them on the *unchanged* Stage-1 evaluator, and shipped both as Avalonia panels;
the §2/§3.5 refinements below are folded from it).

---

## §1 Concept — program / descriptor / runtime

A compute expression is a *transferable, content-addressed program* (GUIDE-CORE-COMPUTATIONAL-ARCHITECTURE:
compute is the system's IR). What it lacks, to be *run as an interactive program*, is a standard way to
say **where its input enters and its output leaves** — so a host can pick it up and drive it. This
convention adds exactly that, in three layers:

1. **The program** — a pure compute subgraph: an initial state and a `step: (state, input) → state'`. It
   reads declared input paths and derives declared output paths; it knows **nothing** about devices.
2. **The interface descriptor** (this proposal) — a small manifest entity declaring the program's **ports**
   (input paths a host writes, output paths a host reads) and its **tick contract**. The shared semantic:
   the *same* program is hostable anywhere because every host reads the same descriptor and binds the same
   ports.
3. **The runtime** — a per-host harness (host code, not specified in its guts) that reads the descriptor,
   binds ports to real I/O, and clocks the tick. *Not a VM — a harness.*

The convention standardizes the **port entity**, never the I/O: a renderer gets an output-state entity and
draws it however it likes; an input loop does a `tree:put` at a declared input path. This is the same
"one contract, per-host provider" discipline as `system/device` and the actuation runtime driver.

## §2 The descriptor entity

```
; kebab type name; snake_case keys (STYLE-NAMING-CONVENTIONS).
app/program/interface := {
  state_path:    {type_ref: "system/tree/path"}                 ; where the current state entity lives
  initial_state: {type_ref: "system/hash"}                      ; state₀
  step:          {type_ref: "system/hash"}                      ; expression: (state, input…) → state'
  input_ports:   {array_of: {type_ref: "app/program/port"}}     ; paths the HOST writes
  output_ports:  {array_of: {type_ref: "app/program/port"}}     ; paths / projections the HOST reads
  tick:          {type_ref: "app/program/tick"}
  shard:         {type_ref: "app/program/shard", optional: true}   ; sharding declaration (§4/§4a); absent = unsharded
  imports:       {array_of: {type_ref: "app/program/import"}, optional: true}  ; capability handlers it calls (§5)
}

app/program/shard := {
  ; --- policy (who/how) ---
  form:          {type_ref: "primitive/string"}                    ; "static-k" (Option B, the floor) | "range" (Option A, dynamic k)
  orchestration: {type_ref: "primitive/string"}                    ; "host-managed" | "continuation-managed"  (§4a)
  k:             {type_ref: "primitive/uint", optional: true}       ; frozen shard count (static-k form only)
  n_field:       {type_ref: "primitive/string", optional: true}    ; state field giving N (default: state array length; §4 "§1.3 owner")
  stitch_owner:  {type_ref: "primitive/string", optional: true}    ; "compute-gather" (generic-host default, §4a) | "host" (program-specific host only)
  ; --- artifacts the host actually evaluates (Finding A, workbench 2026-07-18) ---
  ; A host cannot eval a policy. The static-k floor is k pre-authored shard expressions + a stitch expression;
  ; the host needs their addressable paths — same class as the base descriptor's step/initial_state.
  shards:        {array_of: {type_ref: "system/hash"}, optional: true}  ; static-k: the k shard expressions (fresh budget each). Absent for form=range (uses one step hash + shard-range port).
  fragment_base: {type_ref: "system/tree/path", optional: true}    ; where the host WRITES fragment j before the stitch reads it ({fragment_base}/frag{j})
  stitch:        {type_ref: "system/hash"}                          ; the stitch (gather) expression → the whole state entity. Required for any sharded program.
}

app/program/port := {
  name:     {type_ref: "primitive/string"}                      ; stable port name (host binds by name)
  path:     {type_ref: "system/tree/path"}                       ; the tree path (or inbox) the port lives at
  type_ref: {type_ref: "system/type/name"}                       ; the entity type carried
  kind:     {type_ref: "primitive/string"}                       ; "snapshot" | "stream"   (§3)
  role:     {type_ref: "primitive/string", optional: true}       ; console binding hint: "display"|"input"|"audio"|… (§7)
  initial:  {type_ref: "system/hash", optional: true}            ; seed value the runtime writes before tick 0 (§3, F-E1)
}

app/program/tick := {
  mode:       {type_ref: "primitive/string"}                     ; "clock-driven" | "event-driven"   (§4)
  rate_hint:  {type_ref: "primitive/uint", optional: true}       ; advisory ticks/sec (clock-driven)
  op_cost:    {type_ref: "primitive/uint", optional: true}       ; declared per-ELEMENT op cost (§4); runtime derives k = ceil(N·op_cost / max_ops)
}
```

`state_path`, `initial_state`, `step` are load-bearing; `input_ports`/`output_ports`/`tick` are the seam.
`imports` is the WIT-world analog — the capability handlers the program's drop-downs (§5) require, matched
against the peer's offered capabilities at admission.

**`step` when `shard` is present (RULING 2026-07-18, Finding §3).** A sharded program has no single unsharded
step the host evals — it runs the shard family + `shard.stitch`. So **`step` is OPTIONAL when `shard` is
present**; if supplied, it is the **reference pre-shard step** (the monolithic `(state)→state'` the shard
family is equivalent to) carried for **transfer/verification**, not for evaluation — the host evals
`shard.shards` + `shard.stitch` and ignores `step`. Workbench's interim "`step = stitch`, host ignores it"
is superseded by this: don't overload `step` with the stitch; leave it absent, or use it for its real value
(the verifiable pre-shard identity).

## §3 Ports — two kinds; effects are *declared*, not *performed*

The pure-program model is `(state, input_ports) → (state', output_ports)`. Input ports are what drivers
write **in**; output ports are what drivers read **out** and actuate. The load-bearing rule:

> **The program never performs an effect. It emits effect *intent* as output-port state; a driver
> actuates it.** "Show this frame" / "play these samples" / "send this packet" are values the step writes
> to an output port. This keeps the step pure and deterministic, which is what preserves save-state,
> deterministic replay, and lockstep. (An impure `compute/apply` drop-down remains available for service
> work that doesn't need replay — §5 — but an interactive program should route effects through ports.)

**Two port kinds — both are existing mechanisms:**

| Kind | Mechanism | Semantics | Use |
|---|---|---|---|
| `snapshot` | a **tree path** | last-write-wins; stale values may be dropped | display frame, pointer position, small state |
| `stream` | an **inbox / subscription** | ordered, lossless, backpressured | input events you can't drop, audio samples, packets |

`snapshot` is the default; `stream` is required when dropping a value is incorrect (a rotation the player
pressed, an audio sample). This is WASI's io-stream distinction; the inbox is our stream mechanism.

**Port initialization is the runtime's job (F-E1, POC).** The step reads its input ports on tick 0, and a
`lookup/tree` on an unseeded path is a `compute/error` — so a program with input ports cannot take its
first tick until every one is seeded. Normative: **the runtime MUST seed every input port before the
first `step`** — writing the port's declared `initial` value if present, else a runtime default for the
port's `type_ref`. A program that declares no input ports (Life) needs no seeding and the runtime does
none — F-E1 is *per-port*, which is the same evidence that ports are a per-program property, not a runtime
tax.

## §4 The tick contract — host-owned time

Pure compute has no time; **time is a host concern.** The discrete `state → state'` loop is clocked by
the runtime, not by the program.

- **`clock-driven`** (a game that advances on its own — Life, Snake, Tetris): the runtime evaluates `step`
  each tick (`system/compute:eval` at `step`'s path) and writes the result to `state_path`.
- **`event-driven`** (a program that advances only on input — an editor): the runtime installs `step`
  reactively, keyed on the input ports; a write to an input port triggers one step.

**Normative caveat (the cascade rule).** The tick loop **MUST NOT** be a `step` that both reads and writes
`state_path` under a reactive install — the write re-triggers the dependency and cascades to
`cascade_limit`. `clock-driven` uses **host-clocked explicit eval** (no reactive install on state).
**Reactivity is for *projections*** — an output port `p = f(state)` MAY be a reactive install (recomputes
when state changes, never feeds back). Tick = host-clocked; projections = reactive.

**Budget is part of the tick contract, and the Axis-1 results settle how (POC + Axis-1 absorption).** `step`
is evaluated under a per-eval op-cost ceiling (`EXTENSION-COMPUTE §9.3`; core-go's `DefaultMaxOps` is 100k),
and a step that exceeds it **cannot tick at all**. The Axis-1 measurement established the key fact — with a refinement folded in below: **the
budget cliff is orthogonal to a *faster interpreter*.** Op count measures the *IR* (one decrement per
logical graph step), so an engine that preserves the graph structure and only decodes it faster
(Stage-1 → Axis-1) does **not** move the cliff — arith Life fails at 32×32 on both alike.

**AMENDMENT (2026-07-21 — `PROPOSAL-COMPUTE-BUDGET-PREEMPTION §6`).** This section previously concluded
*"sharding is not the preferred budget answer — it is the only one."* That held only under the hard-cliff,
fixed-graph model. Two later results reopened it: **preemption** removes the hard cliff (yield at a reduction
budget and resume, so a big tick completes across reschedules instead of failing outright), and **lowering**
*does* move per-op cost — a fusion pass that collapses a pure interior to a native handler turns N graph-steps
into ≈1 op, and since the count measures the IR, rewriting the IR is exactly what moves the cliff. So there
are **three composable reliefs**, chosen by which pressure applies:

- **Sharding** (spatial / throughput) — split a `map` into k independent fresh-ceiling shards; uses k cores.
  The relief for **parallelism**.
- **Preemption** (temporal / fairness — `PROPOSAL-COMPUTE-BUDGET-PREEMPTION`) — yield at the reduction budget,
  resume; the whole computation completes across reschedules. The relief for **a big single tick that must not
  block the peer.**
- **Lowering** (vertical / per-op cost — `PROPOSAL-COMPUTE-LOWERING-CONTRACT`, `GUIDE-CORE §6`) — compile a
  pure interior to a native handler (≈1 op instead of N). The relief for **per-op cost**, and the
  boundary-hashing fix.

They compose (shard a map, lower each shard's interior, preempt if a shard still exceeds the reduction
budget); **do not force one to serve all three.** Sharding remains the throughput relief, and it is the one
measured to work today:

- **Shard = same program.** Split the `map` into k contiguous index ranges, eval each (each gets a **fresh**
  budget — the budget boundary is the handler invocation, not the expression tree), stitch the k partial
  arrays, write one grid. Verified **boundary-hash-identical to the unsharded step at every k** (incl. a
  ragged k that doesn't divide N) — so it preserves the replay/lockstep property, not just the pixels.
- **It's cheap and it parallelizes** — +6% wall at k=4, reaches 64×64, and independent shards over read-only
  state run concurrently and **deterministically** (parallel and serial produce identical state hashes),
  taking 32×32 from `budget_exhausted` to realtime.

**CORRECTION (2026-07-17 — a generic host cannot shard by itself).** An earlier draft said the *runtime
derives* `k = ceil(N·op_cost/max_ops)` and shards any step. **That overreached** (caught by workbench-go
reading this against the IR it would build on): a shard is a **distinct baked expression** — the index set
is a `c.Literal(...)` array and the size is baked as `c.Literal(W/H)` throughout the step, so **`N` is an
authoring-time constant, not a runtime quantity**, and a *harness* (this proposal, §1: "not a VM — a
harness") has no `N` to derive `k` from and no licence to rewrite a `map` inside an opaque IR (that is a
compiler pass). The Axis-1 sharding was **author-side** (Go built the k expressions), not a generic host
power. So sharding is **not** a base-mount-contract behaviour. Corrected model:

- **Sharding is an opt-in *program contract*, not a host power.** A shardable step declares a **shard-range
  input port** and maps over `compute/range(i1−i0)` (offsetting inside the lambda) — one step hash, k evals
  with k port values. The host, knowing `N` (from the state array's length, or a descriptor-named field),
  writes k ranges and drives k evals + stitch. This keeps "the runtime derives k" *true* (it now has a
  runtime `N`), needs **no IR rewriting** and **no new host power**, and rides the existing port/seeding
  mechanism (§3). It *uses* `compute/range` — reclassified **load-bearing for the dynamic-k form**
  (`PROPOSAL-COMPUTE-COLLECTION-PRIMITIVES` §2). **This is the general (Option A) form.**
- **RULING (2026-07-17 — the Option-B floor lands now; it is NOT a grudging fallback).** The earlier draft
  filed "the descriptor declaring a pre-authored shard family (k hashes), k frozen at authoring" as a
  *fallback only if `range` slips*. Workbench-go then proved the whole floor end to end
  (`entitysdk/continuation_shard_test.go`, both engines under `-race`;
  `COMPUTE-PARALLELIZATION-TWO-MODELS-2026-07-17.md` §8): the static-k shard family shards *and stitches*
  today with **zero core-go dependency** — the stitch is a `map`-over-`[0..N)` **conditional gather** in
  pure compute (shard boundaries baked as literals since k is static), hash-identical to the unsharded grid.
  So the ruling is: **the static-k Option-B floor is the sharding form that lands now** (it unblocks
  realtime parallel Life/Snake/Asteroids immediately), and `range` (Option A) is the **dynamic-k
  convergence upgrade** — the same contract with `N` runtime-derived and, later, an O(N) concat stitch — not
  a gate on the floor. Neither the floor nor its stitch is gated on `range` or on `array-concat`.
- **The stitch join is determinism-critical → `fire-partial` is prohibited (R4, 2026-07-22).** A sharded
  program's `shard.stitch` produces the whole-state boundary entity — its output is a boundary hash consumed
  for equivalence (Axis-1 §1.4/§4.3), so the stitch join is **determinism-critical by construction.** Its
  completion policy (`PROPOSAL-CONTINUATION-STANDING-MODEL` §4) **MUST NOT be `fire-partial`** (a partial
  stitch is a mixed/incomplete boundary hash — the seam-collapse class); the descriptor MUST NOT declare
  `on_incomplete: fire-partial` for a stitch join. On an incomplete round the stitch uses `abandon` —
  **round-identity-guarded** (STANDING-MODEL §4.1, so a straggler shard cannot bleed into the next tick's
  grid) — emitting a `lost` marker, never a partial grid.
- **Not on the phase-1 critical path.** The three product programs (Snake 8×8, Life 16×16, Asteroids
  ~24 slots) are all **under budget**, so **none needs sharding to mount** — the base mount contract is
  plain `eval → put`. Sharding lands as the opt-in extension when `range` does.
- **Boundary ownership under a sharded step — the descriptor MUST name the owner per orchestration model
  (RULING, Q4).** The stitched output-port hash is produced at a different layer in each model (§4a), and
  under continuation-managed sharding the owner moved again — from the host (host-managed) to the **join's
  target dispatch**. All three are **provably identical** (the state-hash equivalence test is the licence;
  workbench pinned parallel == serial == unsharded across both models under `-race`), but *which entity
  authors the boundary bytes* differs, so the descriptor names it: a `shard` block declaring
  `orchestration ∈ {host-managed, continuation-managed}` and, for continuation-managed, the `stitch_owner`
  (a Go/app handler, or the compute gather step once program-owned). A sharding host **MUST** reproduce the
  byte-identical boundary entity whichever owner is named. (The **program-owned** stitch — the gather step
  as the join target, or `concat` once blessed — is the end state that returns the boundary to the
  program's own compute; §4a and `PROPOSAL-COMPUTE-COLLECTION-PRIMITIVES` §3.)

Authoring `op_cost` still needs a way to *measure* eval cost — an open core-go item (§9). (Measured for
arith Life: **153 ops/cell**; the cliff is exactly `N · op_cost ≤ max_ops`.)

## §4a Two orchestration models — the parallel substrate (RULING 2026-07-17: keep both)

Sharding (§4) is *what* to split; **who holds the fork, the barrier, and the stitch** is a second, separable
choice. Workbench-go built both ends and proved them byte-identical
(`COMPUTE-PARALLELIZATION-TWO-MODELS-2026-07-17.md`, `entitysdk/continuation_shard_test.go`, both engines,
`-race`). The ruling is **support both** — they are the two ends of this track's own axis (capability floor
vs. transferability), not competing designs:

| | **host-managed** (the floor) | **continuation-managed** (the ceiling) |
|---|---|---|
| Fork | host goroutines / SDK dispatches | bounded async-dispatch pool + k `deliver_to` dispatches |
| Barrier | host `WaitGroup`/channel | `system/continuation/join` (accumulate-under-lock, fire-on-complete) |
| Stitch | **compute-gather** in a generic host (`host` code only for a program-specific host — Finding B) | the join's `Target` dispatch — a stitch owner (§4 boundary rule) |
| Lives in | host code (does **not** travel with the program) | the tree (**protocol-only — travels to any runtime with the extension**) |
| Peer needs | compute + a host that can spawn work | **+ `ext/continuation` + the async pool** |
| Failure | **synchronous, bounded, debuggable** | routes to `on_error`; a lost slot advance can **wedge the join** (§4b) |

- **Host-managed is the floor** — works on any peer, synchronous/bounded failure, no extension dependency;
  but the *orchestration* (the fork/barrier loop) is host code, **re-implemented at each runtime**. This is
  what a program past the budget cliff shards through today; workbench built it into the generic host and
  proved parallel == serial == unsharded at 64×64 (`COMPUTE-SHARDING-INTO-HOST-2026-07-18` §0/§5).
- **Continuation-managed is the ceiling** — fork, barrier, and the whole tick are **pure protocol**, so the
  orchestration *itself* travels, which is the transferable-compute thesis recurring one level up. Every
  piece is a **landed primitive** (async pool, `system/continuation/join`, `deliver_to`); it is a wiring
  task, not a design-from-scratch.
- **The program is byte-identical across both** (`state₀ + step`, same shard split, same fresh budget per
  shard), and so is the determinism bar — `Received` is keyed by **slot name**, not arrival order, so
  completion order cannot reach the output. The **generic host selects the substrate by peer capability**
  (§5): the author ships `state₀ + step` and an op-cost class and never sees the choice.

**Finding B (RULING 2026-07-18) — in a *generic* host the stitch is *always* program-owned compute, in
both models.** §4's boundary rule offered `stitch_owner = host` (Go concat) as the host-managed default.
The build showed that is only available to a **program-specific** host: a generic host that names no program
**cannot** concatenate k fragments in Go, because concat needs the entity's shape (`{width, height, cells}`
for Life) — which is program knowledge the host is forbidden. So the generic host drives even the
host-managed stitch as **a compute eval of the program-owned gather** (`stitch_owner = compute-gather`):

```
for j in 0..k:  put(fragment_base/frag{j}, eval(shards[j]))   # fresh budget each; parallel
stitch:         put(state_path, eval(stitch))                 # program-owned gather — compute, not Go
```

Two consequences, both folded: (1) **the program-owned stitch is reached on the *floor*, not only the
ceiling** — §4a's earlier framing ("program-owned = the ceiling's end state") was too narrow; it is the
generic host's *only* viable stitch in either model. `stitch_owner = compute-gather` is therefore the
generic-host **default**; `stitch_owner = host` is reserved for a program-specific host. (2) **A shard
expression MUST self-wrap its output into a fragment *entity*** (`app/…/fragment{cells}`), not a bare array,
so the host writes it verbatim with no decode — an **authoring rule** that keeps the host blind. A shard's
output type is a real design point, not incidental.

**Finding C (RULING 2026-07-18) — the floor is a data-parallel MAP primitive, NOT a scan.** "Sharding" in
the static-k floor is a partition of an independent `map` — each shard is a pure function of read-only state,
which is exactly *why* it shards k-independently and deterministically. A **`fold`/scan** (a left-to-right
carry chain) **cannot** be split into k independent shards + a gather: the k partials must be threaded by
the carry, which is serial by construction (workbench built the negative — `TestChain_MapParallelizes_FoldDoesNot`;
map cells byte-identical under parallel eval, fold carry corrupted). **Scope fence: the sharding floor (and
the continuation-managed ceiling as framed) covers embarrassingly-parallel maps only.** Parallelizing a
reduction/scan needs a multi-phase up-sweep/down-sweep (Blelloch) orchestration — a **separate parallel
primitive**, out of scope for this convention until a program needs it (none of Life/Snake/Asteroids/Doom's
sim does; Doom's frame is a map + a native rasterizer, not a scan). Routed as a named out-of-scope item, not
a gap.

**Finding D (RULING 2026-07-18 — a MUST, cross-impl correctness) — host-managed shards MUST be mutually
independent; the host CANNOT check it.** The parallel == serial guarantee holds **only** for independent
shards. A shard that reads another shard's fragment declares a *serial dependency the floor does not enforce*,
and **parallel eval will silently corrupt it — with no fault** (workbench's carry chain ran 15 clean
iterations of a wrong computation). The generic host cannot detect this: knowing a shard reads a sibling's
fragment would be program knowledge, the one thing it is forbidden. So the guard is an **authoring MUST**:

> **Under `orchestration = host-managed` (static-k floor), the k shards MUST be mutually independent** —
> each a pure function of read-only state, reading **no** sibling's fragment. A shardable step with a
> cross-shard carry is **not** floor-shardable; it is a scan (Finding C) and MUST NOT be declared
> `host-managed`. The `parallel == serial == unsharded` equivalence is guaranteed **only** under this
> independence, and the host has **no way to verify it** — it is the author's contract.

This is the exact **cross-peer seam equivalence-collapse** shape the methodology guards: the equivalence
holds for independent shards and springs apart, silently, for dependent ones. Stated once, generally, with
the collapse called out.

## §4b The continuation-managed failure story is a separate open (routed to a continuation proposal)

Continuation-managed's failure mode is strictly worse than host-managed's, and this is the load-bearing
reason to keep the floor. The barrier has **no timeout**: a partially-filled join is never reaped, so a
shard whose slot never advances leaves the join stuck — and a *standing* per-tick join is then wedged for
every subsequent tick. Workbench pinned this as a test (`TestContFork_FailureAsymmetry` — wedged at 5/6
slots after 2 s). The build refined the shape into **two distinct mechanisms, neither of which exists
today**:

- **delivered-error** — a shard that returns a *delivered* non-2xx still fills its slot with an error
  payload (`processAsyncDelivery` delivers regardless of status), so the failure surfaces **at the stitch**;
  the stitch owner must be able to **reject a bad slot**.
- **undelivered** — a dropped trigger, a pool refusal (429 under saturation), or a Go error before delivery
  leaves the slot unfilled and the join **wedges**; this needs a **partial-completion / timeout / abandon**
  policy on the join.

**RULING (Q3):** this is the continuation extension's to solve, and it is drafted as the completion
facet of **`PROPOSAL-CONTINUATION-STANDING-MODEL` §4** — *not* folded here, because it is a
continuation-extension change, not an L5 convention. (It landed there rather than in a standalone
join proposal because the compute join-completion concern converged with the network authority concern
on one primitive — the standing continuation; see that proposal's two-lineage header.) Until it lands, **realtime parallelism uses host-managed** (bounded failure,
no dependency on the join-error story); continuation-managed sharding is for **batch / non-realtime** work,
or realtime only once the policy lands. The convention **MUST NOT** make realtime parallelism depend on that
story being finished.

## §5 The runtime contract & admission

A conforming **runtime** (per host — browser, workbench-go, Godot):

1. **Reads the descriptor** at a program's root.
2. **Binds** each `input_port` to a real input source (writes the port), each `output_port` to a real sink
   (reads/subscribes the port); `role` hints the console binding (§7).
3. **Clocks the tick** per §4 and hosts `system/compute:eval` (dropping to native for hot projections —
   render/audio — via the `compute/apply` seam, GUIDE-CORE §5).
4. **Provides the `imports`** — the capability handlers the program's drop-downs dispatch to.
5. **Owns run-state** — start / stop / restart / reseed are **runtime verbs, outside the program** (POC
   §3.5). The tick contract says *how* to clock; *who* starts and pauses a program is the runtime's
   surface, not the descriptor's. A runtime MAY also halt a `clock-driven` program on its own by
   observing its output **snapshot** port stopped changing (`stateₙ == stateₙ₋₁` — a fixed point burns an
   eval per tick to reproduce bytes the host already holds); this is free for snapshot ports and
   meaningless for stream ports — the two-kind split (§3) seen from the run-state side. This is distinct
   from an *in-program* halt (Snake's death is program state; a still-life stop is the host noticing).

**Admission** = match the program's `imports` against the **peer's offered capabilities** — precisely the
`system/device/host/offered` field (`PROPOSAL-SYSTEM-DEVICE §7`). A program whose imports the peer cannot
supply is not admitted (or runs with that import unlinked → deny-by-default). This is the WASM-world
"WIT-world vs policy allowlist" check, entity-native.

**`offered` is one shared contract, and its accuracy is load-bearing *here*, now
(`ANALYSIS-SUBSTRATE-CONVERGENCE-COMPUTE-DEVICE-SCHEDULER` §2).** The generic host is a layer-3 *actuator* on
the sense→schedule→actuate substrate; `system/device/host/offered` is the layer-1 sensor's **hostability
contract**, and this admission step is its first landed reader — the future scheduler and any later actuator
(WASM-component hosting) read the *same* field. Because admission binds a program on it **today**, `offered`
**MUST** advertise only **currently-dispatchable** handlers (`discover_handlers`-backed), never aspirational
hostability — the device native-provider review's §8.1 constraint, pulled from "forward scheduler risk" to a
requirement of *this* convention. The contract's canonical home is the device proposal; this convention
consumes it, and MUST NOT restate or fork its shape.

The runtime is **N-class** (host-local); only the *contract* is standard. Thin contract, fat per-host
implementations.

## §6 What is convention vs. a compute change (the verdict)

**This proposal requires zero changes to `EXTENSION-COMPUTE` and no wire change.** The step is expressible
in existing primitives; the tick is a host `eval`+`tree:put` loop (or a reactive install); ports are
`tree`/`inbox`/`subscription`. Three worked programs (Life/Snake/Tetris, in the research doc) confirm the
coverage — including Tetris's line-clear, the textbook dynamic-array operation, which lowers to `filter` +
index arithmetic on a fixed board.

**The one ergonomic finding — routed elsewhere, not part of this proposal.** All three programs carry a
static `[0…N-1]` index array and `index` back into their data. An **indexed `map`/`fold`** (closure
receives the index) or `compute/range` would remove that recurring boilerplate. On the operator's
SDK-vs-extension question:

- **As SDK builder sugar** — a `map_indexed(...)` that *lowers* to the carry-index-array pattern: pure
  ergonomics, ships today, **no** performance gain (same IR).
- **As a compute builtin** — a native indexed iterator beside `map`/`filter`/`fold`: ergonomics **and**
  performance (native iteration, no index-array materialized through the interpreter), and cleaner IR.
  Cross-impl-observable → **MUST-given-COMPUTE** (GUIDE-CORE §7).

Recommendation: it belongs on the **compute-standardization / lowering-toolkit track**, not here. This
convention neither needs nor blocks it.

## §7 Console-port entity types (named; the hard one deferred)

The `role` field binds a port to a console driver. The **console tier** — the only genuinely new
standardized surface — is small and mostly deferred to a follow-on once this descriptor lands:

- **`display`** (snapshot, output) — a `frame` entity `{width, height, format, pixels}` with a small pinned
  format set. Easy: the renderer blits the latest; stale frames drop. Doom's framebuffer is exactly this —
  a constructed entity field read by the host (`compute/construct` materializes bare).
- **`input`** (stream for keys/buttons, snapshot for pointer) — a small `input-event` entity.
- **`audio`** (stream, output) — **the one hard console port**; a realtime PCM stream with ordering,
  backpressure, and timing against the host clock. **Deferred** (§9) — no toy needs it yet.

Everything **outside** the console tier is **not new**: network/storage/assets/devices/random are ports
over **existing** entity types or drop-downs to existing extensions (NETWORK/RELAY/INBOX, DURABILITY/tree,
the content store via `lookup/hash`, the device proposal). Assets are free — content-addressed (a Doom WAD
is content-store entities, no filesystem port).

**UPDATE (Asteroids probe + generic-host design, 2026-07-16).** A third program surfaced the descriptor's
missing content, now being designed in **`docs/research/explorations/EXPLORATION-GENERIC-HOST-AND-DESCRIPTOR.md`**
— to fold here once a generic host validates it against all three programs. The concrete additions:
- **`display-list`** is the **missing middle** of this taxonomy — a third output shape between `raw-state`
  and `framebuffer`: world-space drawables + kind tags, **O(actors), resolution-independent** (measured 7×
  cheaper than an 8×8 framebuffer; a pure `(state)→frame` cannot scatter, so a framebuffer is O(pixels×actors)
  and belongs to a **native** rasterizer, not compute). Doom's renderer is a display list (spans/walls), not
  per-pixel compute.
- **A port gains `shape`** (the driver-binding discriminant — `role` alone can't pick between framebuffer
  and display-list) **and `scene`** (per-shape metadata a driver can't infer, e.g. `wrap:true` for seam
  tiling). The `(role, shape)` set is the **driver ABI** that makes programs mount across runtimes.
- **`output_ports` is genuinely plural/heterogeneous** (Asteroids emits `state` + `display-list`); the **input
  row splits** into held-state snapshots (`direction`/`key-set`/`pointer`) vs. discrete `event-stream`; and
  **`rate_hint` moves fully into the descriptor** (the host must not hard-code the clock for a transferable
  program).

## §8 Non-goals / boundaries

- **Not a VM.** The program is compute; the runtime is a harness. If you want a bytecode VM, that is the
  WASM actuation path (the sibling), not this.
- **Rendering / audio-DSP are native drop-downs**, not compute — the §5 `compute/apply` seam. The sim is
  compute; the pixels/samples are native.
- **Performance is gradient-dependent — but the gradient is shorter than expected.** Simple programs run
  realtime on the Stage-1 interpreter; the Axis-1 general interpreter (decode-once + live frames) then buys
  **34–163× on the whole tick** and moves the bottleneck *off* the evaluator, so a per-program compiler
  (rung 3) is not the near-term answer (exploration §12). Scale past the budget cliff is **sharding**
  (§4), not compiling. This convention is about *portability and the contract*, not a realtime guarantee.
- **The console tier is an optional profile.** A headless *service* program (the entity-native peer itself)
  uses only capability ports — no display/audio.

## §9 Open / deferred

- **Console-port entity types** (`frame`, `input-event`) — a follow-on once the descriptor lands; small.
- **Audio stream port** — the hard design (per-tick buffer vs. decoupled inbox the audio driver pulls);
  defer until a program needs sound.
- **Empirical cost numbers** — ✅ **done** (POC + Axis-1 absorptions): Life across grid sizes on Stage-1
  *and* the Axis-1 general interpreter (34–163×/tick), the budget cliff proven orthogonal to the engine,
  and sharding measured (boundary-equivalent, parallel-deterministic, reaches 64×64). The compile ladder
  and its measured rung-2 terminus are exploration §12.
- **Op-cost measurement surface** — the `op_cost` clause (§4) asks authors to declare a per-element op
  cost, but core-go today exposes **no way to measure eval cost** except by exhausting the budget
  (bisection). A `dry_run` op, or an op-count on the eval response, is needed before `op_cost` is
  authorable — routed to core-go; it gates the clause.
- **Sharding fairness** — a runtime that shards evaluates k sub-evals synchronously; TTL fuses recursion
  and budget bounds each, but *fairness* (not starving concurrent reactive/emission work) is a runtime
  scheduling property the contract should name if in-compute sharding (self-dispatch) is ever blessed
  (Axis-1 absorption §4c).
- **Indexed `map`/`fold` / `range` + array-concat** — the compute collection-primitive candidates (§6);
  concat is what an *in-compute* sharded step needs to rejoin fragments. On the compute track, batched as
  one proposal.
- **Descriptor discovery** — how a runtime *finds* a program's descriptor (a well-known path vs. a registry
  entry); left to the follow-on.

## §10 Worked examples

The research doc carries full bottom-up sketches; the descriptors:

```
; Life — output + tick, no input
app/program/interface { state_path:"app/life/state", initial_state:<glider>, step:<grid→grid'>,
  input_ports:[], output_ports:[{name:"grid", path:"app/life/state", type_ref:"app/life/grid",
  kind:"snapshot", role:"display"}], tick:{mode:"clock-driven", rate_hint:10} }

; Snake — adds the input port
app/program/interface { state_path:"app/snake/state", initial_state:<seed>, step:<step>,
  input_ports:[{name:"dir", path:"app/snake/input/dir", type_ref:"primitive/int",
  kind:"snapshot", role:"input"}],
  output_ports:[{name:"grid", path:"app/snake/state", type_ref:"app/snake/state",
  kind:"snapshot", role:"display"}], tick:{mode:"clock-driven", rate_hint:8} }
```
