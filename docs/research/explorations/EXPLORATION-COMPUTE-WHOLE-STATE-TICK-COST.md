# EXPLORATION — the whole-state O(N)-per-tick floor: is a tick always a whole-state rewrite?

**Status:** exploration — 2026-07-18. Opens the last unanswered lever on the compute floor.
**Origin:** workbench-go `COMPUTE-SHARDING-INTO-HOST-2026-07-18` §9 + §10 row 2 — after sharding (space
axis), the chain negative (time axis), and the Axis-1 engine (~33× the constant) were all built and
measured, the **dominant remaining cost is fixed, k-independent, and O(N) per tick**, and workbench correctly
routed it here as *"a model question, not a parallelism or engine question."*
**Reads:** `PROPOSAL-APP-CONVENTION-COMPUTE-PROGRAM` §4/§4a (the sharding floor this sits on top of);
`PROPOSAL-COMPUTE-COLLECTION-PRIMITIVES` §3 (concat — the *other*, ~20% lever, distinct from this);
the load-bearing invariants in `AGENTS.md` (tree = path→hash; content-store dedup) — **this exploration is
those two invariants applied to program state.**

---

## §1 The finding, stated precisely

The measured floor (workbench §9, `BenchmarkAxis1HostShardKSweep`): even at **k=1** — no sharding overhead
at all — the Axis-1 engine is ~33× faster than Stage-1, so the tick is dominated by **fixed whole-grid work
independent of k**, not by the sharding round-trip. Every generation runs **three O(N) passes**, each with
its own dispatch + whole-grid CBOR + a store put:

1. **the shard map** — computes the N next-cells (the actual simulation);
2. **the gather stitch** — re-materializes all N cells into the state entity (`put(state, …)`);
3. **the display projection** — re-maps all N cells to glyphs (`refreshPorts`).

Axis-1 shrank the *constant* ~33×, but the **shape stays Ω(N) per tick** because a tick is
`put(state_path, f(state))` over an **immutable** entity: there is no in-place mutation, so the whole state
entity is rewritten every generation even if two cells changed. **The question routed to arch:** is a tick
*always* a whole-state rewrite, or is there a diff/incremental-state shape?

## §2 The reframe — the cost is the *monolithic-entity representation*, not the immutable model

The tempting read is "immutable entities are too expensive for realtime large-N." **That is the wrong
diagnosis.** Immutability does not force a whole-state rewrite — it forces a whole-state rewrite **only when
the state is a single monolithic entity.** The substrate already carries the escape, in two of the five
load-bearing invariants:

- **`tree = path → hash`** (invariant 1). A tree is not one blob; it is a map from paths to content hashes.
  If program state is a **subtree** (`state_path/` with per-chunk child entities) rather than one
  `{width, height, cells[N]}` entity, then a tick re-`put`s **only the chunks that changed** — the unchanged
  paths keep their existing hashes untouched.
- **content-store dedup** (invariant 2). Even for the chunks the step *does* recompute, a chunk whose bytes
  didn't change **hashes to the same value and its put is a content-store no-op.** A quiescent region of a
  Life grid (a still-life, empty space) dedups for free — the store already holds those bytes.

So the model already supports an incremental tick. The compute-program convention just **defaults to the
monolithic shape** (`state_path` = one entity, §2 of the proposal), which is the shape that forces all three
O(N) passes. The lever is a **representation profile**, not a new primitive.

## §3 Decomposing the three passes — which are reducible, and by what

| pass | what it is | reducible? | by what |
|---|---|---|---|
| **1. shard map** | the *computation* — next-state from current | **only for sparse/dirty programs** | dirty-region compute (an *algorithm* property, not the model); a dense global stencil (Life) recomputes all N by definition |
| **2. gather stitch → state put** | the *store write* | **yes** | **subtree state** (tree=path→hash) + **dedup** → O(changed chunks), not O(N) |
| **3. display projection** | the *projection* to an output port | **yes** | **display-list** (O(actors), already in the taxonomy) or a **diff/dirty-rect frame** port → O(changed) |

The honest split: **passes 2 and 3 are representation-reducible today with mechanisms already in the
substrate; pass 1 is reducible only when the *algorithm* is sparse.** No representation makes a dense
cellular automaton skip recomputing cells whose neighbourhood might have changed — that is inherent to the
computation, and the answer for it is engine speed (Axis-1's ~33×) + sharding (the space axis), which caps
how large a *dense* automaton runs realtime. That cap is real but it is **not** the Doom-class case.

## §4 Why this does not block the actual target (Doom-class programs)

The whole-state floor bites **dense cellular** programs (Life at 64×64: every cell a live function of its
neighbourhood, most of the grid churning). The programs the track is actually aimed at are **sparse/actor**
shaped:

- **The sim state is small and sparse.** Doom's simulation is entities — player, monsters, doors,
  projectiles: O(actors) ≈ tens–hundreds, not O(pixels). A tick mutates the handful that moved. Under a
  subtree/per-entity state (§2), pass 2 is O(changed actors).
- **The framebuffer is native, not compute.** Doom's frame is produced by a **rasterizer** (a
  `display-list`→pixels *driver*, §7 of the proposal — a native drop-down), **not** by a per-pixel compute
  map. Pass 3 for a Doom-class program is the display-list (O(actors)), and the O(pixels) rasterization is
  native work the convention already routes off compute.
- So a Doom-class tick is **already O(actors-changed)** once state is a subtree — the whole-state O(N) floor
  is **not** on its critical path. It is on the critical path only for large *dense* automata, which are not
  the goal.

**This is the load-bearing correction to the roadmap:** "can entity-compute run Doom realtime" does **not**
hinge on beating the whole-state O(N) floor — it hinges on the sparse-state representation + the native
rasterizer seam, both of which exist. The dense-automaton O(N) wall is a separate, narrower limit.

## §5 The design lever this surfaces — an incremental-state profile

The concrete addition to the compute-program convention (a **candidate**, to sharpen then propose):

- **`state_path` as a subtree root**, not one entity — an opt-in **incremental-state profile** beside the
  monolithic default. The step writes per-chunk / per-entity children; unchanged children keep their hashes;
  dedup collapses recomputed-but-identical children.
- **The tick becomes `put` over the changed sub-paths**, so store cost is O(changed), and the state's
  content hash (the determinism oracle) is the **tree hash** — which is *still* a single deterministic value
  over the whole state, so the boundary-equivalence discipline is unbroken (the tree hash of an unchanged
  region is unchanged; the oracle still catches any real divergence).
- **The step must be expressible as a chunk/entity-local function** for this to pay — which is exactly the
  sparse-update shape, and exactly what a `map` over independent shards already is. (A dense global stencil
  can still *use* the subtree — quiescent chunks dedup — but recomputes all chunks, so pass 1 stays O(N)
  for it.)

Open sub-questions before this is proposal-ready:
- **O1 — chunk granularity.** Per-cell is too fine (hash overhead per cell); per-row / per-tile / per-entity
  is the real unit. Is granularity author-declared, or a host default keyed on the state type?
- **O2 — does this interact with sharding's `fragment_base`?** A sharded tick already writes k fragment
  entities (§4a). Subtree-state and shard-fragments are both "state as several entities" — are they the same
  mechanism seen twice (a shard fragment *is* a state subtree chunk), or two layers? Suspect the former;
  worth collapsing if so (the anti-"patch instances" discipline).
- **O3 — the determinism oracle under a subtree.** Confirm the tree-hash-of-state is as strong a boundary
  oracle as the flat-entity hash the cohort tests today — i.e. that a cross-impl divergence in any chunk
  still surfaces in the root tree hash (it must, by tree = path→hash, but pin it as a conformance assertion
  before folding — the CDN-corridor meta-rule: not validated until a cross-impl test exercises it).

## §6 What this exploration concludes (the routable position)

1. **A tick is NOT necessarily a whole-state rewrite.** The whole-state O(N) cost is a property of the
   **monolithic-entity state default**, not of the immutable-entity model. `tree = path→hash` + dedup already
   make an incremental (subtree) tick O(changed) — the two invariants applied to program state.
2. **Passes 2 (store write) and 3 (projection) are representation-reducible now** — subtree state; display-list
   / diff ports. **Pass 1 (compute) is reducible only for sparse/dirty algorithms**; a dense stencil's O(N)
   is inherent and answered by engine speed + sharding, capping large *dense* automata (not the target).
3. **Doom-class programs do not hit this floor** — sparse sim state + a native rasterizer put their tick at
   O(actors-changed). The whole-state wall is a narrower dense-automaton limit, not the Doom gate.
4. **Next:** sharpen §5 (O1–O3) into a `PROPOSAL-COMPUTE-INCREMENTAL-STATE` (an L5-convention profile, likely
   no wire/compute change — it rides tree + dedup), and route O2/O3 to workbench-go as the next build probe:
   **re-run the §9 k-sweep with a subtree state whose step writes per-tile**, and measure pass-2 dropping from
   O(N) to O(changed). That is the empirical test of this whole reframe — and it is the useful next negative:
   *does subtree state actually collapse the fixed floor, or does per-chunk hashing overhead eat the win?*
