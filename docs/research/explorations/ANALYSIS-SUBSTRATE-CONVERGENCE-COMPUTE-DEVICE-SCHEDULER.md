# ANALYSIS — do the generic host, compute, the scheduler, and system-device align? (and the GPU question)

**Phase:** analysis / reconciliation (not a proposal). 2026-07-19.
**Question (operator):** the compute-program spike is done and system-device is in cohort review — **are the
generic host, `EXTENSION-COMPUTE`, the (future) scheduler, and `system/device` all going in the right
direction, the way continuation and network converged?** And: **is a "generic GPU host provider" feasible —
and does system-device depend on it?** (If not, park it.)
**Method:** place each track on the sense→schedule→actuate map the execution-substrate exploration already
drew, find the seam they share, and pressure-test it — the same way the continuation/network review found the
standing continuation.
**Reads:** `EXPLORATION-EXECUTION-SUBSTRATE-ORCHESTRATION` (the 5-layer family, the map); `PROPOSAL-SYSTEM-DEVICE`
§7 (`offered`); `PROPOSAL-APP-CONVENTION-COMPUTE-PROGRAM` §5 (admission); `entity-core-go`'s device native-provider
review §8 (the five scheduler flags); `EXPLORATION-COMPUTE-WHOLE-STATE-TICK-COST` (the sparse-state / native-render split).

---

## §0 Verdict

**They align — cleanly, and on the architecture every orchestrator uses: sense → schedule → actuate, no
conflation.** Device is layer-1 sensing; the generic host is a layer-3 *actuator* (the deterministic-compute
end of the actuation spectrum); the scheduler is the layer-2 seam between them, still future. Three findings:

1. **There is one real convergence to name — `offered`.** `system/device/host/offered` (device, the *sensor*
   side) and the generic host's **admission match** (`imports` → offered, the *actuator* side) are **the same
   contract read from two ends.** This is the continuation/network pattern exactly: two tracks converged on one
   primitive. Name it once, with one accuracy MUST, before a third consumer (the scheduler) reads it.
2. **The compute-program's budget story *is* the scheduler's resource story.** `op_cost · N` vs. the peer's
   `available_bytes`/cpu is precisely a K8s requests-vs-allocatable placement. The compute track and the device
   track are the same axis seen at two layers — which is *why* they must not fuse (sense stays dumb) but *must*
   share the `offered`/metrics vocabulary.
3. **The generic GPU compute host is not feasible near-term — and it is *unnecessary*, and device does NOT
   depend on it.** The determinism boundary (§4) is also the CPU/GPU boundary: entity-compute stays CPU
   tree-walked; GPU work already lives on the **native** side of the `compute/apply` seam (the rasterizer, DSP).
   Device *sensing* a GPU is feasible as pure advertisement (later); it needs no GPU compute host to exist.
   **Park the GPU compute host; it gates nothing.**

## §1 The map — where each track sits (no conflation)

The execution-substrate exploration established the universal split (Nomad/K8s/wasmCloud all do it). Placing
today's tracks on it:

| Layer | Concern | Our track | Status |
|---|---|---|---|
| **1 — sense** | read-only advertisement of host facts | **`system/device` + store op-state** | in cohort review; native provider feasible |
| **2 — schedule** | match work → resources (requests/bin-pack/affinity) | *future*; ties to COMPUTE placement | not built; vocabulary being reserved now |
| **3 — actuate** | run it (isolation-per-workload driver family) | **generic host = the COMPUTE end**; WASM-component hosting = the next driver | compute end **built** (the spike); WASM future |
| 4 — lifecycle | workload status as entities | *future* | — |
| 5 — distribution | place/run across peers | reuses NETWORK/RELAY/capability | partial (cross-peer dispatch exists) |

**The load-bearing observation:** the generic host is **not a separate island** — it is the *first actuation
driver*, sitting at the lightest, determinism-preserving end of the same spectrum whose heavier end is
WASM/container/microVM (exploration §7). "Transferable compute" and "layer-3 actuation" are one thing seen from
two docs. That is the correct place for it, and it consumes device sensing exactly as a layer-3 actuator should.

## §2 The convergence — `offered` is one hostability contract, read from both ends

This is the finding that rhymes with continuation+network. Two tracks already point at the same field, from
opposite sides, without yet naming it as one:

- **Device (sensor side)** — `PROPOSAL-SYSTEM-DEVICE §7`: *"Reserve `system/device/host/offered` — the set of
  host interfaces / capability providers this peer can supply to a workload … a scheduler consumes hostability,
  not a datasheet."*
- **Generic host (actuator side)** — `PROPOSAL-APP-CONVENTION-COMPUTE-PROGRAM §5`: admission *"= match the
  program's `imports` against the peer's offered capabilities … precisely the `system/device`
  offered-capabilities field."*

So the compute-program convention **already depends on `offered`** to admit a program, and device **already
defines** it for a future scheduler. They are the same contract. The unification, stated once (the anti-hole,
same discipline as the standing-continuation §5):

> **`system/device/host/offered` is the peer's hostability contract: the set of currently-dispatchable host
> capabilities a workload may bind. It is written by the device sensor, and read by every actuator's admission
> and by the scheduler's placement. Its accuracy is load-bearing for every reader — it MUST advertise only
> currently-dispatchable handlers (`discover_handlers`), never aspirational hostability.**

The accuracy MUST is **not** a future-scheduler concern — the device review filed it as flag §8.1 for the
scheduler, but the generic host's admission reads `offered` **today**, so a wrong advertisement mis-admits a
compute program **now**. That pulls the review's §8.1 from "forward risk" to "load-bearing for the landed
compute-program admission." State the contract in one canonical home (the device proposal, since it owns the
field) and have both admission (§5) and the future scheduler point at it — don't restate the shape in the
convention.

**Why this is the right kind of convergence** (and not over-coupling): the two tracks share a *contract*, not a
*mechanism*. Device does not learn what a compute program is; the host does not learn how device is sensed. They
meet only at `offered` — a read-only capability set — which is exactly the thin seam sense→actuate is supposed to
have. Naming it once removes the risk that a WASM actuator (the next layer-3 driver) invents a *second* offered
shape and the two drift at the cross-peer seam.

## §3 The second tie — compute placement is the scheduler's first customer

Beyond `offered`, the two tracks share the **capacity** vocabulary, and this is where the compute spike's own
findings become the scheduler's inputs:

- The compute-program descriptor carries `op_cost` (per-element) and the runtime knows `N` (state length). The
  budget cliff is exactly `N · op_cost ≤ max_ops` (app-convention §4). **That inequality, evaluated against a
  *peer's* `available_bytes`/cpu headroom, is a placement decision** — "can this peer run this program at this
  size, or must it shard / go elsewhere." It is K8s requests-vs-allocatable, in our vocabulary.
- The device review's flag §8.4 — **reserve `reserved`/`committed`** alongside total/available — is precisely
  "what compute programs has this peer already been handed." A placed compute program is the thing that consumes
  the budget the third field tracks. So the compute track is *why* the third field matters; reserving it now
  (populate two of three) is the cheap forward-proofing the review recommends, and the compute spike is its
  first concrete consumer.
- **Sensing ≠ accounting still holds (review §8.3):** `system/device/metrics/*` is a snapshot sensor; the
  reservation ledger of "which programs are placed and hold how much budget" is the scheduler's (layer 2), not
  device's. The compute host must not treat a device read as the source of truth for "is there room" — it reads
  `offered` for *admissibility* and (later) asks the scheduler for *placement*. This keeps the seam clean.

**Net:** the compute budget/sharding story and the device scheduling story are the same axis at two layers, which
is the healthy shape — device stays a dumb sensor, the scheduler owns the decision, the host actuates. They
converge on the `offered` + capacity vocabulary and nowhere else.

## §4 The GPU question — determinism is the CPU/GPU line; the GPU compute host is unnecessary and uncoupled

The operator's instinct ("I don't think a generic GPU host is really feasible") is right, and the reason sharpens
into a boundary that is *already drawn*:

- **Entity-compute is a deterministic, cross-impl-vectored tree-walk** (~153 ops/cell, budget-bounded, one
  decrement per graph step — Axis-1 measured it as an interpreter, not a kernel). A GPU wants SIMT kernels over
  flat buffers. Running entity-compute *itself* on a GPU would require a **compute → SPIR-V/kernel cross-compiler**
  — the same "real cross-compilation" lift that Doom-as-entity-compute needs, and far off. So a **generic GPU
  compute host is not near-term feasible.** Correct.
- **But it is also unnecessary, because the determinism boundary (exploration §7) is the CPU/GPU boundary.**
  The compute-program convention *already* routes GPU-shaped work to the **native** side of the `compute/apply`
  seam: rendering is a native rasterizer driver (the `display` role, §7), audio a native DSP drop-down — not
  compute. A Doom frame is rasterized natively (and *that* driver may be GPU-backed, per host); the sim stays CPU
  deterministic entity-compute. The whole-state analysis reinforced this: sparse sim (O(actors)) in compute + a
  native rasterizer for pixels. **So the GPU already has its place — as a native display/DSP driver on the far
  side of the seam — without entity-compute ever executing on it.** "Generic GPU host" in the only sensible sense
  is the native rasterizer, which exists.
- **Does `system/device` depend on any of this? No — the dependency runs the other way.** Device is layer-1
  sensing; it *stands alone* (execution-substrate §10). Actuators (the compute host, a WASM runtime, a native GPU
  rasterizer) **consume** device sensing; device consumes nothing from them. Device *sensing* a GPU — advertising
  "this host has GPU X, N GB" — is the K8s device-plugin pattern (§3: advertising a GPU is read-only; using it is
  the scheduler+runtime's job) and is feasible **as pure sensing**, independent of whether anything can run
  compute on that GPU. The device review already parks it: gpu/network are post-v1 domains, added when the
  per-platform cost justifies it, and even then they are *fields*, not a compute host.

**Answer:** park the generic GPU compute host. It is not feasible near-term, it is not necessary (GPU work is
native-side of the determinism seam and already routed there), and **system-device does not depend on it** —
device sensing a GPU is a later, self-contained field, and nothing in the compute spike or the device proposal
waits on a GPU compute host. Revisit only if/when a workload genuinely needs entity-compute *on* the GPU, which
is the cross-compilation horizon, not the substrate one.

## §5 What to watch (the misalignment risks, small)

Nothing is diverging, but three seams to keep honest as this matures:

1. **`offered` accuracy is load-bearing NOW, not later.** Because admission reads it today, adopt the device
   review's §8.1 constraint (`discover_handlers`-backed, non-aspirational) as part of the *compute-program*
   admission story, not just the future scheduler's. Folded as a pointer in app-convention §5 (this session).
2. **Don't let a second actuator invent a second `offered`.** When WASM-component hosting is designed (layer-3,
   next driver), it MUST read the same `offered` contract, not a parallel one — the convergence §2 exists to
   prevent that drift. Note it in the run-environments exploration when that work opens.
3. **Keep sense/schedule/actuate un-fused.** The temptation, once the compute host is real, is to let it read
   device metrics as a reservation ledger and self-place. That re-fuses layers 1–3 (review §8.3). The host reads
   `offered` for admissibility; placement waits for the layer-2 scheduler. Hold the seam.

## §6 Conclusions + routing

1. **Alignment confirmed.** device = sense (L1), generic host = actuate (L3, compute end), scheduler = schedule
   (L2, future). No conflation; the seams are the ones every orchestrator uses.
2. **One convergence to fold: `offered` as the shared hostability contract** (§2). Canonical home = the device
   proposal §7; admission (app-convention §5) and the future scheduler point at it; the accuracy MUST is shared
   and load-bearing for the *landed* compute admission. **Edit made this session:** app-convention §5 pointer.
   **Recommended (device track):** promote §7 `offered` from "reserve, minimal" to "the hostability contract"
   with the currently-dispatchable MUST — route to the device cohort review already in flight.
3. **Reserve the capacity triple** (total/available/**reserved**) per review §8.4 — the compute host is its first
   consumer (§3). Route to the device cohort review.
4. **GPU compute host: parked, uncoupled** (§4). Not feasible, not necessary, device independent of it. No action
   beyond this analysis; GPU re-enters only as a later device *sensing* field and as the native rasterizer, both
   already scoped.
5. **No new build for workbench** — it's correctly in a holding pattern; the compute spike is complete and the
   next compute rung (subtree-state host convention) is an arch co-design, not an unblocked build. This analysis
   is arch-internal reconciliation + two pointers into proposals already in review.
