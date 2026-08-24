# EXPLORATION — Run-environments & workload actuation (WASM-first), and what it tells device

**Phase:** Exploration / analysis (pre-proposal). 2026-07-12. **Not a proposal.**
**Purpose:** scope the **actuation** layer (layer 3 of `EXPLORATION-EXECUTION-SUBSTRATE-
ORCHESTRATION`) — running workloads — WASM-first, and **loop the findings back to sharpen the
near-term `system/device` sensing surface.** The operator's frame: understanding the space helps us
get device right before we draft its proposal.
**Method:** survey WASI 0.2 / the Component Model, WASM runtime resource metering (Wasmtime), and the
WASI/wasmCloud capability model; map onto our substrate; extract what it implies device must offer.
Cited §Sources.

---

## §0 TL;DR

Two payoffs, one of them near-term:

1. **The WASM actuation model maps almost 1:1 onto our existing substrate.** A component **carries a
   WIT "world"** — a typed, build-time-auditable manifest of every capability it **imports** and
   **exports**. WASI is **object-capability, deny-by-default** ("a capability = resource + rights,"
   the host must explicitly grant each). wasmCloud runs components where **capabilities are an
   interface + swappable providers, linked at runtime, and only linked calls are allowed.** Translate
   the vocabulary: **WIT world = capability manifest · capability provider = an entity handler · link
   = a grant · "only linked calls" = capability-gated dispatch.** Running capability-gated WASM
   components is *our capability model applied to execution* — we already have the hard part.
2. **The loop-back that sharpens device (the point of doing this now):** what a scheduler/runtime
   actually consumes is **not raw hardware specs** — it's **"what can this peer offer a workload":
   (a) which host interfaces / capability providers it can supply, and (b) resource headroom
   (available memory, CPU budget).** So `system/device` should lean toward **capability & capacity
   advertisement** ("what I can host"), with raw hardware (cpu model, gpu string) as *secondary,
   informational*. Plus the **total-vs-available** split (§5) — a scheduler needs *available*, not
   *total*.

## §1 The actuation model, entity-native (the target shape)

Putting the pieces together, "run a workload on an entity peer" becomes clean and reuses what we have:

1. **Workload = a WASM component** carrying a **WIT world** = a static, typed declaration of its
   imports (capabilities it needs) and exports (what it offers).
2. **Admission / placement** = match the component's **imports against the peer's available capability
   providers + grant policy**, and its **resource needs against headroom.** The capability half is a
   *static WIT-vs-policy match* — auditable before the component ever runs (§6).
3. **Run** = instantiate the component in a WASM runtime (Wasmtime native, or the browser's own WASM
   engine), **linking each import to an entity handler** (the host exposes `entity:tree`,
   `entity:query`, … as WIT interfaces), under **fuel/epoch/memory limits** (§4).
4. **Status/lifecycle** = the running workload is operational state — an entity at `system/…` you read
   and subscribe to (layer 4 of the orchestration family).

## §2 WASI 0.2 / the Component Model — the capability surface

- A **world** is "a complete description of both imports and exports of a component." A Preview-2
  component "**carries its own WIT world — a precise declaration of every interface it imports and
  every interface it exports**"; imports are **versioned, namespaced, typed** (unlike P1's untyped
  imports).
- The capability interfaces are granular and importable à la carte: `wasi:cli` (clocks, filesystem,
  sockets, random, stdio — POSIX-ish), `wasi:http` (request/response), `wasi:filesystem`,
  `wasi:sockets`, `wasi:clocks`, `wasi:random`. A component imports *only* what it needs.
- **Build-time auditability (load-bearing for us):** "the WIT world is **auditable at build time** —
  the output is a WIT document that **enumerates every capability the component requires**. An upload
  pipeline can **parse this document and compare it against a policy allowlist** before the component
  is stored or deployed." → capability needs are **statically checkable against grants pre-run.**

## §3 The capability mapping — why this is *our* model already

| WASM/WASI/wasmCloud | Entity-core equivalent |
|---|---|
| Capability = **resource + rights**, deny-by-default | a **grant** (`GrantEntry`: scope + ops) over a resource path |
| Component's **WIT imports** (required interfaces) | the set of **handler ops the component must be granted** |
| Capability **provider** (interface + swappable impl, WIT) | an **entity handler** implementing an op contract |
| wasmCloud **link** (operator binds component→provider at runtime) | issuing a **grant** binding the component to a handler |
| "**Only linked calls are allowed**" | **capability-gated dispatch** (verify_capability_chain at every call) |
| WIT world **auditable vs policy allowlist** pre-deploy | static **grant-coverage check** before instantiation |

The alignment is near-exact. An entity peer hosting a component is: instantiate the WASM, and provide
its imports as **entity handlers exposed through WIT interfaces**, gated by **grants**. The "host
functions" a component may call are exactly the entity ops it holds capabilities for. This is the
answer to "how do you run untrusted code safely in an entity system" — **you already have the
object-capability substrate the WASM world converged on independently.**

## §4 Resource metering & the determinism line (extends the COMPUTE boundary)

WASM runtimes bound execution three ways (Wasmtime concretely):
- **Fuel** — a fixed instruction budget; traps when exhausted; **deterministic** (exact instruction
  count).
- **Epoch interruption** — a wall-clock deadline checked at function prologues / loop back-edges;
  ~2–3× faster than fuel but **non-deterministic** (wall-time based). Enforces true timeouts even
  through host calls.
- **`ResourceLimiter` / `StoreLimits`** — memory / table / instance allocation caps, store-level.
- **Footgun:** "**neither fuel nor epochs are configured by default** — the default runs modules
  indefinitely with no limit." → for us this is a **substrate-resilience-floor MUST**: a hosted
  component runs only under explicit CPU + memory bounds (mirrors our max-payload / max-chain-depth
  bounds).

The **fuel(deterministic) vs epoch(non-deterministic)** split *is our determinism discipline inside
WASM*: fuel-metered execution could be a **conformance-determinism surface** (like COMPUTE); epoch-
metered is not. This extends the §7 COMPUTE↔actuation boundary from the previous doc — the line isn't
"WASM = non-deterministic," it's "fuel = deterministic, epoch/wall-clock/host-effects = not."

## §5 The loop-back to device (why we scoped this before drafting device)

What the runtime/scheduler consumes dictates what device must **offer**. And it is **not** a hardware
datasheet — it's **hostability**:

- **Available capability providers / host interfaces.** "Can I run this component?" ⇒ "do I provide
  the interfaces it imports (`wasi:http`? `entity:tree`? a GPU interface?)." This is **capability
  advertisement** — the K8s *device-plugin advertised-resource* analog — and it belongs in the device
  op-state surface as *a set of offered capabilities/interfaces*, not a CPU string.
- **Resource headroom, available (not total).** memory available, CPU budget (fuel/epoch feasibility),
  disk free. The **K8s `capacity` vs `allocatable`** distinction is load-bearing: a scheduler needs
  *what's schedulable*, i.e. available. → **device MUST distinguish total vs available** for at least
  memory and storage (cheap to add; the whole point for any future scheduler).
- **Raw hardware (cpu model, gpu adapter, arch) is secondary/informational** — useful for display and
  coarse capability inference, but not the load-bearing scheduling input. This *reprioritizes* the
  device field taxonomy (exploration §3/§8): lead with **offered-capabilities + available-headroom +
  total**, treat model strings as lower-priority informational fields (also the highest fingerprinting
  risk — a happy alignment with the privacy stance).

**Net sharpening for the device proposal:** frame `system/device` as **"what this peer is and what it
can offer,"** with (1) platform/idiom, (2) **capacity: total *and* available** for memory/storage/cpu,
(3) an **offered-capabilities/interfaces** set (forward-compatible with hosting), and (4) informational
hardware detail behind a higher-fidelity/opt-in grant. That's a *more purposeful, smaller-core*
surface than "mirror OSHI," and it's future-proof for scheduling without building any of it now.

## §6 Admission & permissions — the "how do we manage running code" answer

The WIT-world-auditable-vs-allowlist property (§2) is the permissions story, and it's **static +
entity-native**: a component's required capabilities are declared in its WIT world; the host **matches
that manifest against the grants the component holds** before instantiating; missing capability →
refuse to run (or run with that import unlinked → deny-by-default at call time). This is the running-
code face of the **capability-management thread** (browser-rust pull-in map, Thread 2): the same
"grant coverage, best-practice defaults, deny-by-default" model, now gating *execution* not just data
access. No new permission mechanism — the WIT manifest is just a **statically checkable grant
requirement**.

## §7 Scope & placement (updating the family map)

- **Actuation is a load-bearing driver family**, WASM-first. The **runtime driver** (Wasmtime native /
  browser WASM engine / later: container/microVM/VM via the isolation spectrum) is impl-local behind a
  common contract — same "one contract, per-runtime provider" discipline as device, but load-bearing.
- **Host interfaces = entity handlers exposed as WIT worlds** — an SDK/impl concern (how a peer offers
  `entity:*` interfaces to guests).
- **Admission/placement** (layer 2) consumes device sensing + the WIT manifest — future, and where a
  scheduler would live.
- **Determinism**: fuel-metered component execution is a candidate determinism surface (§4).
- **Horizon:** still future. This doc is scoping, not a build signal. The near-term deliverable remains
  the **device sensing proposal**, now better-informed.

## §8 What changes for the near-term device proposal (concrete)

Fold these into the device proposal when drafted (they don't expand it — they *focus* it):
1. **Add the total-vs-available split** for memory/storage (K8s capacity/allocatable). Scheduling's
   core input.
2. **Reframe the field priority:** lead with **offered-capabilities + capacity/headroom**; demote raw
   hardware (cpu/gpu model) to secondary informational (also the high-fingerprint fields → coarse/opt-
   in by default — consistent with the privacy stance).
3. **Reserve an "offered capabilities/interfaces" field** in the device vocabulary (forward-compatible
   with hosting; a peer advertises what it can provide) — even if v1 populates it minimally.
4. **Keep device strictly read-only sensing** (prior doc's ruling) — hosting/actuation never leak into
   the sensor.

## §9 Open questions & next

- **Is capability-gated WASM hosting a real target, and when?** Strategically it's the highest-leverage
  actuation entry (reuses our capability substrate, WASM-native fit) — but it's a large build. Operator
  call on timing; naming it now protects device's design.
- **Do we expose entity handlers as WIT interfaces (`entity:tree`, …)?** That's the concrete bridge
  between the WASM world and our substrate; worth its own exploration when hosting becomes a target.
- **Fuel-metered determinism** — could hosted deterministic components join the conformance model?
  (Interesting; far off.)
- **Next near-term:** the device/store proposal is now well-scoped from three angles (data model,
  operational-state shape, orchestration consumption). Recommend drafting
  `docs/proposals/PROPOSAL-…` next, folding §8 here + the shape analysis + the availability descriptor.

---

## Sources

- WASI 0.2 / Component Model: [Bytecode Alliance — WASI 0.2](https://bytecodealliance.org/articles/WASI-0.2), [wasmCloud interfaces (WIT worlds)](https://wasmcloud.com/docs/overview/interfaces/), [WASI 0.2 matters (wasmCloud)](https://wasmcloud.com/blog/wasi-preview-2-officially-launches/)
- Resource metering: [Wasmtime epoch interruption](https://www.systemshardening.com/articles/wasm/wasmtime-epoch-interruption-security/), [Wasmtime ResourceLimiter](https://docs.wasmtime.dev/api/wasmtime/trait.ResourceLimiter.html), [interrupting execution](https://docs.wasmtime.dev/examples-interrupting-wasm.html)
- Capability model: [WASI capability-based security](https://marcokuoni.ch/blog/15_capabilities_based_security/), [wasmCloud capabilities](https://wasmcloud.com/docs/v1/concepts/capabilities/), [wasmCloud security/linking](https://www.systemshardening.com/articles/wasm/wasmcloud-security/)
