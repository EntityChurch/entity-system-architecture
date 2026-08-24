# EXPLORATION — Execution substrate & orchestration: where device sits, and the full scope

**Phase:** Exploration / analysis (pre-proposal). 2026-07-12. **Not a proposal.**
**Question (operator):** is read-only `system/device` just informational, or does it "play more of
a role" — and how does it connect to *using* a system: **scheduling, running environments,
containers, VMs**? What's the **full scope and range** of the substrate concern before we commit?
**Companion to** the device/storage exploration + shape analysis.
**Method:** survey how orchestration systems (Nomad, Kubernetes, wasmCloud, the container/VM/WASM
isolation spectrum) actually structure this, then place device and scope the family. Cited §Sources.

---

## §0 TL;DR

**Device is the *sensing* layer of a three-part architecture that every orchestration system keeps
separate: sense → schedule → actuate.** It answers your question directly: `system/device` stays
**read-only and passive**; it is **consumed by** separate scheduling and execution concerns that
"handle their own concerns and parcel it up" — you do **not** fold scheduling or execution into the
sensor. Confirmed by:

- **Nomad:** the client **fingerprints** the node (sense) → the server **bin-packs** against
  constraints (schedule) → **task drivers** (docker/exec/qemu/java/raw_exec) run it (actuate).
- **Kubernetes:** Node `capacity`/`allocatable` + device-plugin advertisement (sense) → scheduler
  matches **requests** + taints/affinity (schedule) → **CRI + RuntimeClass** run it, over
  runc/gVisor/Kata/Firecracker (actuate).

The **full "execution substrate" concern is a 5-layer family**, not one extension (§8). Most of it
is **future**; near-term we're only building layer 1 (sensing). Two findings that matter for us:
(1) the actuation layer is the **same "one contract, per-runtime driver" shape we already chose for
device** — CRI/RuntimeClass and Nomad drivers are exactly that; (2) because we are **WASM-native
with a capability substrate**, the natural *first* actuation target is **WASM components**
(wasmCloud/WASI-shaped) — and wasmCloud's "default-deny, capabilities linked at runtime" model is
**our capability grant model already**, which is a strong hint about where this goes.

## §1 The question, sharpened

"Is the read-only device thing just informational, or does it play more of a role?" The worry is
right to raise *now*: if device is secretly the front of a scheduling/execution system, we'd design
it differently than if it's a passive sensor. The way to answer is to look at systems that *do*
orchestration and see whether they fuse or separate these concerns.

## §2 The universal architecture — sense → schedule → actuate

Every mature orchestrator splits into the same three layers with clean seams:

| Layer | Nomad | Kubernetes | wasmCloud | = our world |
|---|---|---|---|---|
| **Sense** (what the substrate has) | client **fingerprint** of node resources/attributes | Node `capacity`/`allocatable`; **device plugins** advertise GPU/FPGA/NIC | host capabilities | **`system/device` + `system/store`** (read-only op-state) |
| **Schedule** (match work → resources) | server **bin-packing** + `constraints` | scheduler uses **requests** (not limits); **taints/tolerations**, **affinity** | lattice placement | a *future* placement concern; ties to COMPUTE (§7) |
| **Actuate** (run it) | **task drivers** (docker/exec/qemu/java/raw_exec, allowlist) | **CRI** plugin + **RuntimeClass** → runc/gVisor/Kata/Firecracker | component runtime over **wRPC** | a *future* "run-environments" driver family (§5) |
| **Status/lifecycle** | alloc status | Pod status | component status | operational state again (workloads as entities) |

The seams are load-bearing: sensing is read-only and dumb; scheduling is pure decision; actuation is
the only privileged, side-effectful part. Nothing conflates them.

## §3 Where device sits — the sensing layer (answer: consumed, not controlling)

Device/store is **layer 1: sensing.** It is *read-only advertisement* of what the host has, exactly
like:
- **Kubernetes Node `.status.allocatable`** — a passive report (and notably `allocatable < capacity`
  because system daemons reserve some; a real modeling nuance — "total" ≠ "available to schedule").
- **Nomad fingerprint** — the client just *reports* attributes/resources upward; it doesn't decide
  or run anything.
- **K8s device plugins** — even special hardware (GPU) is *advertised* to the kubelet as a
  schedulable resource; **sensing ≠ allocation.** Advertising a GPU is read-only; *allocating* it is
  the scheduler + runtime's job.

So: **`system/device` stays read-only and non-load-bearing even though orchestration consumes it.**
It is the sensor other concerns read; it never schedules or runs. This confirms the operator's
"handle their own concerns and parcel it up" instinct — device parcels out facts; separate handlers
consume them. (One consequence worth pinning: like K8s capacity-vs-allocatable, device should
distinguish **total** from **available/reservable** where it matters — a scheduler needs the latter.)

## §4 The scheduling layer (what "using it for scheduling" actually means)

Scheduling is the **decision** concern that reads sensing and places work. The reusable ideas:
- **Requests vs limits** (K8s): the scheduler places on **requests** (the minimum a workload needs);
  limits are a runtime ceiling, invisible to placement. → if we ever schedule, "what a workload
  needs" and "what caps it at runtime" are different fields.
- **Constraints + bin-packing** (Nomad): match required driver + resources + constraints; pack for
  density.
- **Taints/tolerations + affinity** (K8s): nodes *repel* (taint) unless a workload *tolerates*;
  affinity *attracts/repels* by topology. A rich vocabulary for "where may this run."

For us this is a **future** concern and it's where **`EXTENSION-COMPUTE` placement** lives (§7): "can
this peer accept this compute workload / how much" is a scheduling question answered from device
sensing. It is *not* part of device.

## §5 The actuation layer — running environments (the big future piece)

If we go beyond sensing to *running things*, the field is unambiguous about the shape: a **pluggable
driver/runtime abstraction** — one contract, many backends. This is **the same shape we already
chose for device providers**:
- **Kubernetes CRI** — a gRPC plugin interface so the kubelet runs *any* runtime without recompiling;
  **RuntimeClass** selects per-workload. Under it: runc / gVisor(runsc) / Kata / Firecracker.
- **Nomad task drivers** — docker/exec/java/qemu/raw_exec, allowlist-gated.
- **The isolation spectrum** (a real design axis, not a binary): process (namespaces+cgroups, shared
  kernel) → **gVisor** (userspace syscall interception) → **Firecracker microVM** (~125 ms boot,
  ~5 MB, hardware-virt isolation) → **full VM / Kata** (QEMU/Cloud-Hypervisor/Firecracker) → and at
  the *lightest* end, **WASM** (§6). Kata itself is a driver-of-drivers (picks the VMM per threat
  model) — evidence that "pick your isolation per workload" is the mature pattern.

**Implication for us:** a "run-environments" concern would be a **load-bearing handler family** —
one op contract (`start/stop/status a workload`), per-runtime **drivers** (process / container /
microVM / WASM), the isolation level chosen per workload. Same "one contract, per-runtime provider"
discipline as `system/device`, but **load-bearing** (it acts), so a different tier than the
read-only sensor. This is future scope; naming it here so device isn't accidentally designed to
carry it.

## §6 The WASM-native angle (why our first actuation target is probably WASM, not Docker)

We are WASM-native (browser-rust is a WASM peer) and we already have a **capability substrate**. That
changes the natural entry point:
- **wasmCloud** runs WASM **components** (portable, WASI 0.2 / Component-Model binaries) across a
  **lattice** of hosts over **wRPC**, and — critically — **each component is default-deny, isolated
  behind a capability contract; capabilities (HTTP, KV, blob, messaging) are linked at runtime by an
  operator.** That is **our capability-grant model**, applied to execution. An entity peer that runs
  a WASM component and gates its host access through **entity capability grants** is wasmCloud-shaped
  *for free*.
- **WASI/Component Model** is the portable ABI — the "what syscalls may this thing make," which in
  our world is "what handler ops may this component dispatch," i.e. **capabilities**.

So the cheapest, most-aligned actuation target is **entity-capability-gated WASM components**, not
containers/VMs. Containers/microVMs/VMs are the heavier end of the same driver spectrum, added later
if a workload needs full-syscall or non-WASM code. This is a strong steer for scope: **WASM
component hosting first; the container/VM drivers are optional heavier backends.**

## §7 Relationship to `EXTENSION-COMPUTE` (draw the boundary now)

COMPUTE is **in-process, deterministic expression evaluation** (the lowering toolkit) — pure,
conformance-testable, no side effects. Running environments is **out-of-process, side-effectful,
non-deterministic execution.** They are different tiers and must not be conflated. But they sit on a
**spectrum of "executing something":**

```
COMPUTE (pure expr, deterministic)  →  WASM component (sandboxed, capability-gated, side-effects via caps)
                                     →  container (full syscalls, shared kernel)  →  microVM / VM (hw isolation)
```

The determinism boundary is the key line: COMPUTE is a determinism surface (cross-impl vectors);
anything past it is not — actuation is impure and lives outside the conformance-determinism model.
Scheduling/placement is the concern that decides *which peer* runs a COMPUTE job or a component — the
first real consumer of device sensing (§4).

## §8 The full scope & range (the 5-layer family — the operator's ask)

The "execution substrate" is a **layered family**, most of it future. Named so we know the shape and
don't over-build layer 1:

| # | Layer | What | Load-bearing? | Level | Horizon |
|---|---|---|---|---|---|
| 1 | **Substrate sensing** | `system/device` + `system/store` — read-only op-state | no | op-state vocab + provider | **near-term (current work)** |
| 2 | **Workload declaration + scheduling** | requirements (requests/limits/constraints) + placement (bin-pack/affinity) | no (decision) | SDK/L5 logic + COMPUTE tie-in | future |
| 3 | **Runtime actuation** | run-environments driver family (process/container/microVM/**WASM**) | **yes** | new handler family, driver-pluggable | future (WASM first) |
| 4 | **Workload lifecycle/status** | start/stop/health/usage as entities | partly | operational state again | future |
| 5 | **Distribution / lattice** | place & run across many peers | yes | reuses NETWORK/RELAY + capability + cross-peer | future |

Two things fall out: layer 5 (distribution) **already partly exists** in our stack (cross-peer
dispatch, NETWORK/RELAY, the capability model) — wasmCloud's lattice is the analog, and we have the
pieces. And layer 1 (what we're building) must stay clean: **a read-only sensor, designed not to grow
scheduling or execution into itself.**

## §9 Architectural rulings this suggests (for the proposal + future work)

1. **`system/device`/`system/store` are read-only sensing, permanently.** They advertise; they never
   schedule or run. Design them so orchestration *consumes* them (add the total-vs-available
   distinction, §3). This protects the near-term work from scope creep.
2. **Scheduling and actuation are separate concerns, added later** — not folded into device, not
   folded into each other (sense/schedule/actuate seam is load-bearing).
3. **If/when we actuate, it's a driver family** (one contract, per-runtime drivers, isolation-per-
   workload) — the same discipline as device providers, but load-bearing tier.
4. **WASM-component hosting is the natural first actuation target**, and it **reuses our capability
   substrate directly** (wasmCloud's runtime-linked capabilities ≈ our grants). Containers/VMs are
   heavier optional drivers.
5. **Keep the COMPUTE/actuation boundary explicit** (determinism line, §7).

## §10 Open questions & next

- **Is running-environments a near-term target at all, or purely forward context?** (Operator call.)
  The device sensing work stands alone and is useful without it; naming the family prevents
  mis-scoping, but layers 2–5 need not be built now.
- **Does device need the total-vs-available split in v1** (K8s capacity/allocatable)? Cheap to add,
  and it's what any future scheduler needs — lean yes, at least for storage/memory.
- **When we're ready:** a dedicated `EXPLORATION-RUN-ENVIRONMENTS-AND-WORKLOAD-ACTUATION` (the driver
  family + the WASM-first path + the capability-gated execution model). Not now — this doc scopes it;
  the near-term deliverable remains the device/store sensing proposal.
- Fold the "device is sensing, permanently read-only; total-vs-available" ruling into the device
  proposal when drafted.

---

## Sources

- Nomad: [client/fingerprint config](https://developer.hashicorp.com/nomad/docs/configuration/client), [task drivers](https://developer.hashicorp.com/nomad/docs/deploy/task-driver), [QEMU driver](https://developer.hashicorp.com/nomad/docs/deploy/task-driver/qemu)
- Kubernetes: [resource requests/limits](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/), [taints & tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/), [CRI](https://kubernetes.io/docs/concepts/containers/cri/), [device plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)
- wasmCloud: [platform overview](https://wasmcloud.com/docs/v1/concepts/), [components/component model](https://wasmcloud.com/docs/overview/workloads/components/), [1.0 + WASI 0.2](https://wasmcloud.com/blog/wasmcloud-1-brings-components-to-enterprise/)
- Isolation spectrum: [Kata vs Firecracker vs gVisor (Northflank)](https://northflank.com/blog/kata-containers-vs-firecracker-vs-gvisor), [isolation compared (Edera)](https://edera.dev/stories/kata-vs-firecracker-vs-gvisor-isolation-compared)
