# PROPOSAL — compute collection primitives (keyed grouping, indexed iteration, concat, indexed update)

**Status:** **IMPLEMENTED — FOLDED 2026-08-20**, `EXTENSION-COMPUTE` 3.23 → 3.24. Ruled 2026-08-17,
all four adopted (§7).
**Fold state: LANDED.** The four args types + observable semantics + the `assoc` no-implicit-lowering
MUST + the not-ruled fairness carve-out are in §3.5's builtin section. **§4's fairness clause remains
NOT ruled** and still gates an in-compute sharded step — folding the primitive did not bless the
pattern.

> **Two corrections the fold produced, both worth keeping.**
> **(1) The L9 pin fired and did its job.** The deferral was pinned to core-go's
> `ext/compute/builtins.go` at `6fc0af3` with *"re-check when either moves; if a `concat` builtin
> appears before this folds, the fold is **late**, not pending."* It moved (`61b431c`, 2026-08-16).
> **Re-checked: the builtins are still absent** — the set is `{arithmetic, compare, logic, field,
> construct, map, filter, fold, store}`. So the fold was **pending**, and the pin is the reason that
> was a measurement rather than an assumption. **This is L9 working in the forward direction**, four
> days after it ratified on the backward one.
> **(2) §7's binding note said *"core-go ships the builtin"* — an assignment, and arch read it as a
> build state.** It propagated into `WORKSTREAMS` T5 lever 1 and into a routing packet drafted for
> `entity-workbench-go`, where it would have told the seat blocked on this primitive that it already
> existed. **Caught by running the pin, corrected in all three places before the packet was sent.**
> The rule this repo opens with — *verify build state before you assert it, read the live worktree* —
> and the failure shape it names: plausible, recently true, and invisible to any review that does not
> open the tree.
> **(3) The naming gate caught `group_by`.** `STYLE-NAMING-CONVENTIONS` reserves snake_case for
> data-structure keys; an entity-type path segment is **kebab**. Folded as **`group-by`**, and the
> args type as `system/compute/group-by-args`. The proposal's own text used the snake form throughout
> — **implementers should build `group-by`.** Was DRAFT — 2026-07-16 (rev. after the Asteroids probe). Evaluative — this
makes the *case* and ranks the candidates; it does not assert all must land.
**Operator policy (2026-07-16):** *"It's our extension. We started small and add as needed — if there's
real value, get it in and improve the ergonomics; don't restrict it artificially."* So this proposal now
carries a **recommendation to adopt**, not just an evaluation — gated on real value and on the one hard
tradeoff below (§5: an indexed update buys scatter *at the cost of sharding*).
**Scope:** candidate additions to `EXTENSION-COMPUTE`'s builtin set, all **cross-impl-observable** (they
produce boundary bytes) → **MUST-given-COMPUTE** if adopted (GUIDE-CORE §7). A **compute amendment**, on the
compute-standardization / lowering-toolkit track.
**Motivation sources:** `EXPLORATION-COMPUTE-PROGRAM-RUNTIME-CONTRACT.md` §6.3 (indexed iteration) + §12.7/§4
(sharding); `docs/research/reviews/ABSORPTION-compute-axis1-results.md` §4a/§4c (concat + budget/fairness);
**`docs/research/reviews/ABSORPTION-compute-asteroids-actor-probe.md` §3/§5 (the scatter/parallelism duality
and the `group_by` finding — the strongest new evidence)**; verified against
`entity-core-go/ext/compute/builtins.go` (builtin set: `map/filter/fold/construct/store` + eval core —
**no keyed grouping, no concat, no indexed iterator, no indexed update**).

---

## §1 Two gaps the programs keep hitting

Every worked program (Life, Snake, Tetris) and the sharding prototype ran into one of two missing
collection operations. Neither is a *capability* gap — both are expressible today via workarounds — and, as
a 2026-07-17 build settled (§2/§3), **neither gates parallelism**: the static-k shard floor shards and
stitches with existing primitives. Both remain worth adopting for reach and ergonomics.

1. **Indexed iteration (`range`).** All three programs carry a static `[0…N-1]` index array and `index` back
   into their data, because `map`/`filter`/`fold` pass the *element* but not its *index*. Boilerplate and a
   double-indirection today; load-bearing for the **dynamic-k** shard form (§2).
2. **Array concat.** There is no way to join k arrays into one. `fold` can't (the accumulator step needs the
   append that doesn't exist), `map` yields k arrays, `construct` changes the entity shape (forbidden by
   boundary equivalence). Once thought the gate on a program-owned sharded stitch; a build disproved that (a
   concat-free gather stitches in pure compute, §3), so concat is an **optimization** (O(N·k)→O(N), dynamic
   k), not a blocker.

## §2 Primitive A — indexed iteration (the ergonomic one)

**The gap:** a closure over a collection cannot see its own index without a carried index array.

**Two forms** (the exploration's SDK-vs-extension split, §6):
- **`map_indexed` / `fold_indexed`** — the closure receives `(element, index)`. Minimal, familiar.
- **`compute/range(n)`** — a first-class integer range you `map` over, replacing the materialized
  `[0…N-1]` array. Composes with existing `map`/`fold`; arguably the cleaner primitive.

**Verdict (REVISED 2026-07-17 — LOAD-BEARING for the *dynamic-k* shard form; NOT a gate on parallelism).**
This was first filed as "ergonomic, not load-bearing"; a mid-day revision over-corrected to "the tick
contract cannot shard without it." **Both were wrong at the edges, and the second error was flushed by a
build.** The precise picture, after workbench-go proved the sharding floor end to end
(`COMPUTE-PARALLELIZATION-TWO-MODELS-2026-07-17.md` §8, both engines under `-race`):

- A shard is currently a *distinct baked expression* — the index set is a `c.Literal(...)` array and the
  grid size is baked as `c.Literal(W/H)` throughout the IR (verified: `program_life.go:482-485`,
  `axis1_shard_test.go`), so with a **frozen k** (the **static-k / Option-B floor**) `N` is an authoring-time
  constant and **the program shards and stitches with no `range` and no host IR-rewrite at all** — proven.
  So parallelism does **not** gate on `range`; the floor lands today (`PROPOSAL-APP-CONVENTION-COMPUTE-PROGRAM`
  §4 RULING).
- `compute/range(n)` is what makes `N` a **runtime quantity** and unlocks the **dynamic-k / Option-A** form:
  a shardable step reads its range from an input port and maps over `range(i1−i0)`, one step hash, k evals —
  "runtime derives k" as a contract instead of a frozen count. This is the **convergence upgrade**, the
  cleaner and more general form, not the enabler. **Single-argument `range(n)` (offset inside the lambda) is
  sufficient** — no wider primitive needed.

**So: load-bearing for dynamic-k, and it also retires the hand-rolled `[0…N-1]` array everywhere.
Cross-impl-observable → MUST-given-COMPUTE.** *Recommendation: adopt — the highest-value ergonomic-plus-
capability primitive here — but it is **no longer on the critical path for parallelism** (the floor ships
without it), so it need not be sequenced ahead of `group_by` on urgency; sequence by cohort readiness.*

## §3 Primitive B — array concat (an OPTIMIZATION, not a blocker — a build disproved the "load-bearing" framing)

**The gap:** `concat(a, b, …) → array` (or `flatten(array-of-arrays) → array`) — join k arrays into one
flat array, deterministically, order-preserving.

**REVISED 2026-07-17 — concat is an optimization, decided by build.** An earlier draft filed this "the
load-bearing one" on the reasoning that the *program-owned* stitch (rejoining k fragments in pure compute
rather than in the host) was impossible without it. **That gate does not hold.** Workbench-go built the
program-owned stitch **without concat** (`TestComputeStitch_ConcatFreeGather`, both engines, `-race`,
hash-identical to the unsharded grid): the flat `cells` array is `map` over `[0..N)` where each global index
selects from whichever shard fragment covers it — a **k-way conditional gather** (`if`/`compare`/`index`/
arithmetic), shard boundaries baked as literals since k is static, fragments read once via `lookup/tree`.
So the transferable, program-owned stitch is **buildable today with existing primitives**, and **nothing on
the parallelization track is blocked on core-go**.

**What concat still buys (why keep it on the list).** The gather is O(N·k) compares and requires **static
k** (boundaries are baked literals). `concat` turns the rejoin into an **O(N) append** and **drops the
static-k constraint** — the dynamic-k form (with `range`, §2) wants it to stay O(N). And it lets an
**in-compute sharded step** via `compute/apply` fresh budgets (Axis-1: the budget boundary is the handler
invocation) be a single self-contained expression. So:

- concat is a **real, recommendable improvement** — O(N·k)→O(N), dynamic k — **not a dependency**;
- the equivalence oracle is unchanged: the concatenated array must be **byte-identical** to the unsharded
  `map`'s array, exactly the licence the concat-free gather already proves.

**MEASURED 2026-07-18 — worth ~20% at k=16, and it is PRIMITIVE-BLOCKED, not just blessing-gated.** Workbench
lifted sharding into the generic host and swept k at fixed N/op-cost on both engines
(`COMPUTE-SHARDING-INTO-HOST-2026-07-18` §9, `BenchmarkAxis1HostShardKSweep`): the O(N·k) gather's k-dependent
term is **~20% of the tick at k=16** (system 1.2×, axis1 1.6× across a 16× k-range) — real at large k, smaller
below. So concat is now an optimization **with a number**, not a hunch. Two corrections this forces:
- **It is smaller than a mid-cycle read implied.** An earlier workbench "Finding E" leaned on the store
  round-trip as the next big lever; the k-sweep corrected it — even at k=1 (no gather) axis1 is ~33× faster
  than system, so the tick is dominated by **fixed whole-state O(N) work independent of k** (three O(N)
  passes: shard map, gather stitch, display projection), not by the sharding round-trip. Concat attacks the
  ~20% k-term, **not** that fixed mass. The fixed mass is a *model* question routed separately
  (`EXPLORATION-COMPUTE-WHOLE-STATE-TICK-COST`).
- **There is no builtin to author it on.** The compute builtins are `{arithmetic, compare, logic, field,
  construct, map, filter, fold, store}` (`ext/compute/builtins.go`) — **no `concat`/`append`/`flatten`.** An
  O(N) concat stitch therefore **cannot be authored on the existing vocabulary**; adopting it is a **core-go
  builtin + spec change**, not merely an arch blessing. This proposal is the decision point; core-go ships it
  if adopted.

**Semantics to pin (if adopted):**
- **Order-preserving, deterministic** — `concat(a,b)` is `a`'s elements then `b`'s, always. No sorting, no
  dedup (this is array concat, not set union).
- **One level** — `concat` joins arrays of elements; it does not recursively flatten. (A separate
  `flatten` is a distinct decision; start with binary/variadic `concat`.)
- **Type discipline** — element types must match; a mixed concat is a `type_mismatch` error-as-value, like
  the other builtins.
- **Empty/singleton** — `concat()` = empty array; `concat(a)` = `a`. No special cases at the boundary.

## §3b Primitive C — keyed grouping (`group_by` / `partition_by`) — the strongest candidate

**The gap (Asteroids, verified).** Binning actors into spatial cells — the operation under any spatial
index — currently costs a **gather**: for each of B bins, `filter` the whole actor set asking "are you in
bin b?", i.e. **O(B·N)** (measured at a flat `5·B·N`). There is no one-pass keyed grouping. This is the
*general* shape of "index a collection," and it recurs anywhere a program partitions its state.

**The primitive:** `group_by(collection, key_fn) → [(key, members)]` — one pass, **O(N log N)**, producing
groups the existing combinators consume natively (`map`/`filter`/`fold` over `[(key, members)]`). It
**retires the `B·N` product** for every partitioning workload at once.

**Why it is the strongest candidate:**
- **It composes** — the output is just a collection of (key, member-list) pairs; no new consumption
  machinery, no slicing (which is why bare `sort` was withdrawn — without `slice`/`take`/`drop`, sorting
  leaves you re-`filter`ing each run, back at O(B·N)).
- **It preserves sharding** (§5) — grouping is still a read-only fold over the input; the parallel-gather
  structure the track's scaling story depends on is intact.
- **It generalizes past games** — "partition a collection by a key" is the shape of a spatial index, a
  histogram, a bucketed aggregation, a router. This is the one candidate whose value is obviously not
  Asteroids-specific.

*Recommendation: **adopt.** It is ergonomics-and-complexity both, composes cleanly, keeps sharding, and has
the broadest reach.* (`partition_by` — split into two by a predicate — is the boolean-keyed special case;
`group_by` subsumes it.)

## §3c Primitive D — indexed update (`assoc`) — real capability, real tradeoff

**The gap:** `assoc(array, i, value) → array'` — replace one element, producing a new array. This is the
missing **indexed update** that makes *scatter* directly expressible: a `fold` with an `assoc` accumulator
step bumps `acc[k]` in O(1) instead of rebuilding the whole accumulator (§Asteroids). It is the primitive a
framebuffer or a mutable spatial index "wants."

**The tradeoff that makes this the careful one (Asteroids §3 duality — the sharp finding).** A gather is
O(product) *because every output cell is independent* — which is exactly *why* it shards, in parallel,
deterministically (the track's whole scaling result). A scatter via `assoc`-fold is O(sum) but
**sequential by construction** (step i+1 consumes step i's accumulator). So:

> **`assoc` buys cheap scatter at the cost of sharding.** You cannot have both from one structure — the
> expensive-but-parallel gather and the cheap-but-sequential scatter are two faces of one coin.

So `assoc` is a genuine capability add (it makes a whole class directly expressible), but it is **not free
even conceptually** — a program written with `assoc` opts *out* of the parallel-shard scaling path for that
step. *Recommendation: **adopt, but frame it as a program-author choice, not a default** — and document the
duality at the point of use, so an author reaches for `assoc` knowingly (small fixed-size accumulator) vs.
`group_by`/gather (large, shardable). Do not lower `map`/`fold` onto `assoc` implicitly.*

## §4 The budget-escape & fairness analysis (why in-compute sharding is *safe* but needs a fairness clause)

The operator asked whether the fresh-budget-per-apply "escape" is intentional and whether it locks up the
system. Verified in `entity-core-go` (`core/protocol/dispatch.go::decrementBounds`,
`ext/compute/handler.go`):

- **It is intentional and bounded by *termination*.** Recursion is **fused by TTL** (`decrementBounds`
  decrements TTL per dispatch hop; TTL=0 → "TTL exhausted"); each invocation is **budget-bounded** (100k,
  never raised in core-go); and an **explicit anti-runaway guard** gives a resource-less dispatched apply
  an *empty* `ResourceTarget` so it cannot inherit the parent's resource and point back at the compute
  expression itself (the F4 resource-ceiling fix). A step that dispatches to its own handler **cannot run
  forever**: TTL bounds depth, budget bounds each step, self-reference is prevented.
- **It is *not* bounded by fairness.** The dispatch is **synchronous** (`hctx.Execute` blocks the parent
  eval), so a large-but-bounded fan-out/recursion **occupies the serve loop** and could delay concurrent
  reactive / emission (phase-2) processing. That is a runtime *scheduling* property, not a termination one
  — and it is the operator's exact concern ("as long as it doesn't lock it all up").

**Consequence for this proposal:** array concat is safe to *specify*, but blessing an **in-compute
sharded step** (fan-out via self-dispatch + concat rejoin) requires the tick contract to state the fairness
posture explicitly — either (a) a bound on synchronous fan-out width / total nested work, or (b) a runtime
guarantee that nested eval yields (a core-go scheduling change). **This is the real cost of "restore the
program-owned boundary," and it is why the decision is not obvious.**

## §5 The recommendation (ranked — given the "add as needed if real value" policy)

Under the operator's stated policy, the question is no longer "should we add anything" but "which of these
carry real, non-Asteroids-specific value, and what does each cost." Ranked:

| candidate | value | cost / caveat | recommend |
|---|---|---|---|
| **`range` (§2)** | unlocks the **dynamic-k** shard form (`N` runtime-derived); removes the hand-rolled `[0…N-1]` array everywhere | tiny; single-arg is enough; **not on the parallelism critical path** (static-k floor ships without it) | **adopt — highest value; sequence by cohort readiness, not urgency** |
| **`group_by` (§3b)** | retires the O(B·N) partition product; the general "index a collection" shape (spatial index, histogram, router); **preserves sharding** | none structural | **adopt** |
| **`assoc` / indexed update (§3c)** | makes *scatter* directly expressible (a whole class) | **trades away sharding** for that step (the §3c duality); author must choose knowingly | **adopt, but as an explicit author choice — never an implicit lowering** |
| **`concat` (§3)** | O(N) program-owned stitch + **dynamic k** (the gather is O(N·k), static-k); **measured ~20% of the tick at k=16** (§3) | **primitive-blocked** — no `concat`/`append`/`flatten` builtin exists, so it is a **core-go + spec change**, not just a blessing; needs a fairness clause (§4) for self-dispatch | **adopt — as a measured optimization for large k**, not a blocker. Ships when the compute-standardization track opens; the fixed whole-state O(N) mass (a model question) dominates it |

**The through-line the duality gives us (§3c):** `group_by` and `range` are pure wins — they make existing
gather/iteration cheaper or cleaner *without* changing the scaling model. `assoc` is a real capability but
it is the one that **moves a program off the parallel-shard path**, so it is added *as a tool an author
reaches for deliberately*, not as a default the compiler picks. That is the honest shape of "improve the
ergonomics without restricting artificially": add the primitives that compose and preserve sharding freely;
add the one that trades sharding with its tradeoff documented at the point of use.

**Sequencing:** `group_by` + `range` are ready to land on the compute-standardization track whenever it
opens (both MUST-given-COMPUTE, both need cohort agreement on semantics + a corpus entry — which ties to the
compute-corpus ask). `assoc` and `concat` follow as their forcing cases mature (the generic-host work and
the Doom display-list vocabulary are the likely next evidence).

## §6 Non-goals / boundaries

- **Not a general list library.** `sort`, dedup, reverse, `slice`/`take`/`drop` are separate decisions —
  though note `slice` is the missing piece that would make a bare `sort` useful (§3b), so if sorted access
  is ever wanted, `sort` + `slice` travel together, not `sort` alone.
- **No wire change, no new operation semantics beyond the builtins** — these are collection builtins beside
  `map`/`filter`/`fold`, same evaluation model, same error-as-value discipline.
- **Does not settle the fairness clause** — §4 identifies it as the gating cost of in-compute sharding and
  routes it to the tick contract + core-go; it is not resolved here.

---

## §7 RULING — 2026-08-17

**All four primitives are adopted, exactly as §5 ranks them.** The proposal's own recommendation is
accepted without amendment; nothing below re-argues it.

| Primitive | Ruling | Binding note |
|---|---|---|
| **`compute/range(n)`** (§2) | **ADOPT.** Single-argument form; offset inside the lambda | MUST-given-COMPUTE |
| **`group_by`** (§3b) | **ADOPT** | MUST-given-COMPUTE. `partition_by` is subsumed, not separately adopted |
| **`assoc`** (§3c) | **ADOPT as an explicit author choice** | **MUST NOT be an implicit lowering target** — `map`/`fold` are never lowered onto `assoc`. The §3c sharding duality is documented at the point of use |
| **`concat`** (§3) | **ADOPT** with §3's pinned semantics — order-preserving, one level, matched element types (`type_mismatch` error-as-value), `concat()` = empty, `concat(a)` = `a` | MUST-given-COMPUTE. **This is `entity-workbench-go` lever 1** and it is primitive-blocked: **core-go is the seat that ships the builtin** — an assignment, not a build state. *(Read as a state once; see the fold note.)* |

**What is NOT ruled, and stays the gate it already was.** §4's **fairness clause** is unresolved.
Adopting `concat` does **not** bless an **in-compute sharded step** (self-dispatch fan-out + concat
rejoin) — that still requires the tick contract to state a fairness posture: either a bound on synchronous
fan-out width, or a nested-eval yield guarantee (a core-go scheduling change). Termination was never the
question; occupancy of the serve loop is. **Adopting the primitive and blessing the pattern are two
decisions, and only the first is made here.**

**Fold is deferred, and the deferral is pinned.** The compute track is **held by operator decision**
(`WORKSTREAMS.md` T5) and this ruling does not lift it — it removes arch from the critical path so that
lifting it later is a no-op rather than a fresh round of arch work.

> **Per L9 (candidate — a deferral is a build-state claim and expires like one), the pin:**
> the fold blocks on `(EXTENSION-COMPUTE.md, §builtin set — `{arithmetic, compare, logic, field,
> construct, map, filter, fold, store}`, entity-system-architecture@6aab719)` and on core-go's builtin
> table `(entity-core-go, ext/compute/builtins.go, 6fc0af3)`. **Re-check when either moves.** If a
> `concat`/`append`/`flatten` builtin appears in either before this folds, the fold is late, not pending.

**The other four levers stay owed and are unchanged** — lever 2's subtree-state descriptor shape, lever 3's
join-failure policy + stitch-home, lever 4's scan/up-sweep scope, lever 5's host-blindness breach. They are
arch's, they are not discharged by this ruling, and the hold does not make them un-owed.
