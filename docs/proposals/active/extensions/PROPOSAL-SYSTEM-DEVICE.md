# PROPOSAL — `system/device`: read-only host-substrate introspection

**Status:** DRAFT — 2026-07-12.
**Depends on:** ~~`PROPOSAL-AVAILABILITY-DESCRIPTOR`~~ — **DISCHARGED 2026-08-15**: the descriptor is landed at `SPECIFICATION-FORMAT` §8.2a (vectors at §8.5). Every field here is `availability<T>` and now references a normative shape rather than a pending proposal. **Nothing blocks this proposal.**
**Scope:** a peer exposes **read-only facts about the host it runs on** — as **operational-state
entities** in its own tree, read via the ordinary `tree`/`query` handlers over a connection. No wire
change; no new query-handler extension. **Storage introspection is out of scope here** (§9 — it's
QUERY + GC + a small op-state entity, not part of this proposal).
**Intake:** `entity-browser-rust/.../PROPOSAL-SYSTEM-DEVICE-AND-STORAGE-CAPABILITIES.md` (`8ad781de`).
**Research (design record):** `docs/research/explorations/EXPLORATION-SUBSTRATE-INTROSPECTION-DEVICE-AND-STORAGE.md`,
`ANALYSIS-DEVICE-STORAGE-SHAPE-OPERATIONAL-STATE.md`, `EXPLORATION-EXECUTION-SUBSTRATE-ORCHESTRATION.md`,
`EXPLORATION-RUN-ENVIRONMENTS-AND-WORKLOAD-ACTUATION.md`, `ANALYSIS-DEVICE-STANDARDS-ALIGNMENT-REDFISH-OTEL.md`,
`ANALYSIS-SUBSTRATE-CONVERGENCE-COMPUTE-DEVICE-SCHEDULER.md` (the `offered` convergence, §7).
**Cohort absorption (folded 2026-07-19):** `entity-core-go/docs/reviews/REVIEW-SYSTEM-DEVICE-NATIVE-PROVIDER-GO-2026-07-13.md`
— native reference-impl review: **full native provider feasible on stdlib + build-tagged `syscall`, no new
dependency** (recommend *against* `gopsutil` for v1 — an adapter-shaped dep for what stdlib+syscall covers).
Its edges are folded below: `available_bytes` native degradation (§4), the storage volume key (§3/§4a), the
`reserved` third field (§4), the exact-fidelity scheduling grant (§5), the `offered` accuracy MUST (§7), the
sensing≠accounting boundary (§8), and the injectable host seam + `Freeram` trap (§10).

---

## §1 Concept

A peer can expose its files (`local/files`) but nothing about the machine it runs on. When a peer runs
headless and is managed from another device, the only channel is its handlers over the connection —
so host facts (disk free, memory pressure, cpu, platform) must be **a capability on the peer or they
are unanswerable**. This proposal adds that capability the way the system already lets a peer describe
itself: **operational state** — self-describing entities in the tree (`GUIDE-OPERATIONAL-STATE §1`),
read via `tree:get`, subscribed to, capability-gated by path. It is **read-only sensing**: it advertises
what the host is; it never schedules or runs anything (§8).

## §2 Shape — operational-state vocabulary, not a query extension

Device facts live as entities under `system/device/…`, populated by a per-runtime **provider**, read
through the ordinary tree handler (cross-peer dispatchable, capability-gated) and observed via
`system/subscription`. This mirrors the five V7 §3.13 operational-state types (`transport`, `config`,
`connection`, …) — device is a sibling vocabulary, not a new handler-op surface.

**Static-identity vs dynamic-metric split** (an entity-native population distinction — static facts
populate once, gauges refresh on read; the same cut OTel draws as `host.*`/`os.*` vs `system.*`, §2a):

| Class | Entities | Population |
|---|---|---|
| **Identity** (static) — platform, arch, cpu model/count, os | `system/device/host/*` | populated **once at startup**; cheap, stable |
| **Metrics** (dynamic/gauge) — memory available, disk free, cpu load | `system/device/metrics/*` | **refresh-on-read / hybrid** (`GUIDE-INSPECTABILITY §4.3`); a read MAY trigger a bounded re-probe. **v1 = gauges only; no rate metrics** (cpu-% / throughput need two samples — deferred to a sampled op or subscription) |

Reading `system/device/*` remotely is an ordinary capability-gated `tree:get` — the whole
"manage a headless peer's facts over a connection" requirement, satisfied with no bespoke op.

## §2a Relationship to external standards (entity-native, not adopted syntax)

This surface **borrows convergent semantics, not syntax** (`ANALYSIS-LANDSCAPE-COMPLETENESS-AND-ENTITY-
NATIVE-TRANSLATION §4`). The landscape (SMBIOS→CIM→SNMP→Redfish→OpenTelemetry→dev-libs→browser→Godot→
orchestration) agrees on a small set of concepts — the ~8 domains, total-vs-available, a presence/
availability axis, the static-identity-vs-dynamic-metric split — and **those** we adopt. But every
name here is **entity-native** (kebab path segments, snake_case keys per `STYLE-NAMING-CONVENTIONS`, the
one `availability<T>` descriptor, entities-in-the-tree read via `tree`/`query`, capability-path grants).
External references below (e.g. "cf OTel `os.type`") are **concept provenance, not imported syntax** — we
do not adopt dotted namespaces, PascalCase, CIM ontology, Redfish `Health`, or any external wire form.
Where our model is cleaner we diverge deliberately (e.g. one `platform` entity instead of OTel's
separate `host.*`/`os.*`).

## §3 The entity vocabulary (v1 domains; entity-native names, concept-traced)

Every field is `availability<T>` (per the descriptor proposal). **Names are entity-native**; the `cf`
comments cite concept provenance only (§2a). **v1 ships the convergent four**
(`platform`, `cpu`, `memory`, `storage`); `gpu`/`network`/`power`/`display` are named for
forward-compat and land later (gpu needs the §5 coarsening design first).

```
system/device/host/platform      { os_type: av<str>,        ; cf OTel os.type
                                    os_version: av<str>,     ; cf OTel os.version
                                    arch: av<str>,           ; cf OTel host.arch
                                    idiom: av<str>,          ; desktop|tablet|phone|tv|watch|server|headless (cf MAUI Idiom + server/headless)
                                    hostname: av<str> }      ; cf OTel host.name — PII → denied/coarse by default (§5)

system/device/host/cpu           { logical_count: av<int>,  ; cf OTel system.cpu.logical.count
                                    physical_count: av<int>, ; cf OTel system.cpu.physical.count
                                    model: av<str> }         ; secondary/informational (high fingerprint → coarse/opt-in)

system/device/metrics/memory     { total_bytes: av<int>,        ; total installed          (capacity)
                                    available_bytes: av<int>,    ; OS-schedulable headroom  (allocatable) — see §4
                                    reserved_bytes: av<int> }    ; this peer's own committed budget — see §4; v1 MAY be `unknown`

system/device/metrics/storage    { volume: av<str>,             ; which volume this entity reports (§4a); v1 = the peer's store volume
                                    volume_total_bytes: av<int>,
                                    volume_free_bytes: av<int>,
                                    origin_quota_bytes: av<int>,  ; cf browser StorageManager.estimate quota
                                    origin_usage_bytes: av<int> } ; browser usage; volume_* → unsupported in-sandbox

; forward (named, not v1): system/device/host/gpu, system/device/metrics/network, .../power, .../display
```

`av<T>` = `availability<T>`. Units are fixed integers (bytes/counts) so a consumer renders identically
regardless of provider. `idiom` adds `server`/`headless` to MAUI's enum for our deployment reality.

## §4 The capacity triple: total / available / reserved (K8s capacity / allocatable / allocated)

For memory (and forward, storage), **total** and **available** are distinct required fields — a
scheduler/consumer needs *schedulable headroom* (`available_bytes`, `volume_free_bytes`), not just installed
total (the Kubernetes `capacity` vs `allocatable` distinction).

**The third field — `reserved` (native review §8.4).** `total − available` is OS-level free; it does **not**
tell you what *this peer's own scheduler* has already committed to running workloads (the fuel/epoch/memory
budgets handed to live compute programs / components — K8s `allocated`). That number is layer-4 lifecycle
accounting, **not** an OS reading, so v1 **MAY** report `reserved_bytes` as `unknown` — but the **vocabulary
reserves the slot now** so the metrics domain isn't reshaped once a scheduler exists. Cheapest forward-proofing:
name the triple, populate two. The compute-program host is its first concrete consumer — a placed program is
exactly what `reserved` tracks (`ANALYSIS-SUBSTRATE-CONVERGENCE-COMPUTE-DEVICE-SCHEDULER §3`).

**`available_bytes` degrades natively too — it is not a browser-only story (native review §3).** `total` is
cheap everywhere; `available` — the load-bearing scheduler field — is not uniformly cheap: **macOS needs
`host_statistics64`, awkward from pure Go without cgo**, so a native provider may legitimately report
`available_bytes` as `unsupported`/`coarse` on Darwin. Conformance (§10) MUST tolerate that on some native
platforms, not only in the WASM arm. **Cross-impl trap to pin as a fixture:** `Sysinfo.Freeram ≠ MemAvailable`
— `Freeram` excludes reclaimable cache and silently *under-reports* schedulable headroom; an impl using it
would diverge on the most scheduling-relevant field.

## §4a Storage volume key (native review §4)

`system/device/metrics/storage` reports **one** `volume_total/free`, but a host has many volumes (the peer's
store, scratch, `/` may be different mounts). **v1 normative:** the storage entity reports **the volume backing
the peer's store**, and the `volume` field (§3) names it — answering the motivating "how much room do I have"
without multi-entity machinery. **Forward:** storage-aware scheduling needs per-volume, so the full form is a
keyed set `system/device/metrics/storage/{volume}` (the natural op-state-tree fit, matching how
`system/transport/{protocol}` is keyed). Pin the single-volume semantics now; the keyed form lands with
scheduling (open-Q §5.2).

## §5 Privacy & coarsening

Device facts are the canonical fingerprinting surface, and it is **cumulative** (entropy comes from
stacking fields). So the grant gates the surface and coarsening is default:

- **Opt-in per domain.** Reading `system/device/host/{domain}` requires a read grant naming that path.
  Default grants (e.g. a system-backend manager grant) SHOULD scope to the domains actually needed.
- **Coarse by default; exact only under an explicit high-fidelity grant.** Coarsened fields set
  `fidelity: "coarse"` (descriptor §4). Concrete rules (from the browser's own mitigations):
  - memory/storage totals → **bucketed** (nearest power-of-two, or a fixed bucket table) + clamped to
    `[lower, upper]` bounds;
  - `cpu.model`, `gpu` adapter → a coarse **class**, not the exact string, unless high-fidelity;
  - (future) any rate/latency field → fixed rounding granularity.
- **High-risk fields default to `denied` or coarse** without an explicit grant: `hostname`, `cpu.model`,
  `gpu` detail.

**Coarsening vs. scheduling precision — the exact-fidelity grant tier (native review §8.2).** A scheduler
bin-packs on exactly the capacity fields this section coarsens for fingerprinting resistance — and it
**cannot safely pack on a bucket** (coarsen-down → under-utilize; round-to-nearest → over-commit). This is a
real tension, and it resolves as a **deliberate grant tier, not a surprise:** **scheduling requires an
exact-fidelity read grant** on capacity fields — *a scheduler is a high-fidelity-granted reader.* A `coarse`
capacity read is for display/telemetry, never placement; a consumer MUST NOT bin-pack on a `fidelity: coarse`
value. So the privacy default (coarse) and the scheduling need (exact) coexist: the grant scope decides which
provider path runs, and placement is gated on holding the exact tier.

## §6 Capability model

Read-only, **non-load-bearing** (`GUIDE-CAPABILITIES §5.5`) — nothing downstream acts on a device fact
as *authority*; no Coherent-Capability (R1) machinery. Access is ordinary **path-scoped read grants** on
`system/device/*` (same as any tree read). `denied` in a field (descriptor) is the auth outcome when a
grant doesn't cover it.

## §7 Offered capabilities — the hostability contract (the shared seam)

`system/device/host/offered` is the set of **host interfaces / capability providers** this peer can supply to
a workload (`EXPLORATION-RUN-ENVIRONMENTS §5`: a scheduler consumes *hostability*, not a datasheet). It is
**more than a reserved forward field** — it is the **one contract where the read-only sensor meets its
consumers**, and it already has a landed reader.

**The convergence (`ANALYSIS-SUBSTRATE-CONVERGENCE-COMPUTE-DEVICE-SCHEDULER §2`).** `offered` is read from two
ends: this sensor **writes** it, and every layer-3 **actuator's admission reads** it. The compute-program
generic host's admission already matches a program's `imports` against `system/device/host/offered`
(`PROPOSAL-APP-CONVENTION-COMPUTE-PROGRAM §5`); the future scheduler and any later actuator (WASM-component
hosting) read the *same* field. This is the sense→actuate seam, thin by design — consumers meet the sensor
only at this capability set, nowhere else. This proposal is its **canonical home**; consumers point here and
MUST NOT fork its shape.

**Accuracy MUST (native review §8.1 — load-bearing *now*, not only for a future scheduler).** The instant an
admission or a scheduler binds work on `offered`, an inaccurate advertisement is a broken contract (the K8s
device-plugin "advertised but can't deliver" failure), and read-only-ness does **not** make a wrong
advertisement harmless. Because the generic host's admission reads `offered` **today**, this is already live:

> **`offered` MUST advertise only *currently-dispatchable* handlers (`discover_handlers`-backed), never
> aspirational hostability.** It names what the peer *can dispatch right now*, not what it *could host*.

**Read-only** — advertising what is dispatchable, never granting or running it. v1 populates it from
`discover_handlers` (open-Q §11.3 is now *ruled*: populate it, don't leave `unknown` — it has a landed reader).

## §8 Read-only ruling & non-goals

- **`system/device` is read-only sensing, permanently.** It advertises; it never schedules, allocates,
  or runs. Scheduling and actuation are separate concerns that *consume* it
  (`EXPLORATION-EXECUTION-SUBSTRATE-ORCHESTRATION §3/§9`). This is a standing boundary, not a v1 scoping.
- **Sensing ≠ reservation accounting (native review §8.3).** `system/device/metrics/*` is a **snapshot
  sensor**, never a reservation ledger. `refresh-on-read` gives a point-in-time reading; a consumer that reads
  `available = 8 GB` and places 6 GB races another placement that took the memory in between (TOCTOU). K8s
  solves this by making the scheduler the **single writer of an allocatable ledger** (reservations) and *not*
  re-reading the node sensor per placement. So a consumer **MUST NOT treat a device read as source-of-truth for
  "is there room"** — the accounting/reservation layer is the scheduler's (layer 2). This protects device from
  mis-consumption the same way the read-only ruling protects it from mis-extension. (The `reserved_bytes` field,
  §4, is the sensor *reporting* the scheduler's committed total — not device doing the accounting.)
- **Gauge-subscription cost at scale — forward flag (native review §8.5).** A consumer watching N peers'
  volatile gauges via `system/subscription` is the kubelet→apiserver heartbeat-storm shape. Fine for v1
  (snapshot reads), but subscription-driven scheduling MUST debounce / rate-limit device-gauge subscriptions —
  do not wire an unthrottled per-gauge subscription. Named so a later scheduler doesn't drown.
- No writable device control (fan/power/mount). No health/monitoring (`Health` is Redfish's job; we
  carry presence via the descriptor, not health). No continuous telemetry — a snapshot; streaming layers
  on later via `system/subscription`. No rate metrics in v1 (§2). No bespoke wire format.

## §9 Storage is NOT in this proposal

Per `ANALYSIS-DEVICE-STORAGE-SHAPE-OPERATIONAL-STATE`: the peer's own store facts are **not** a new
extension. Entity/path counts = `system/query:count` (+ §2.5 zone-summaries); physical store accounting
(blobs/bytes/reclaimable) = a small operational-state entity whose reclaimable-churn number **is** the
`GUIDE-GC` signal. Handle storage there as a minor operational-state addition; do **not** fold it into a
device extension. (The `system/device/metrics/storage` domain above is the *host disk*, not the entity
store — different owners, §the shape analysis.)

## §10 Provider contract & conformance (when promoted)

- The **op-nothing contract**: the cross-impl-observable surface is the **entity vocabulary + field
  names + units + the availability descriptor**. The **per-runtime provider** (Rust `sysinfo` / a Go/Py
  equivalent / browser `navigator.*`+`StorageManager`) is **impl-local, not specified** — each fills what
  it can and reports `unsupported` honestly for the rest. Browser is necessarily its own provider.
- **Native feasibility — confirmed, no new dependency (native review §0/§1/§7).** The Go native provider is
  feasible on **stdlib + build-tagged `syscall`** (the `ext/localfiles` `statcache_{linux,darwin}.go` idiom;
  populate = build entity → `cs.Put` → `li.Set`), **no new dependency** — `gopsutil` is **not** recommended for
  v1 (an adapter-shaped dep + large transitive tree for four fields; revisit only if `gpu`/`network` land and
  per-platform cost justifies it, with numbers). Some fields are honestly per-platform (`cpu.physical_count`,
  `os_version`) or `unsupported` where hard (`memory.available_bytes` on Darwin, §4).
- **The provider needs an injectable host seam (native review §8.6 — provider-contract gap).** A provider
  reads the *real* host, so the "fixed fake host" fixture only works cross-impl if the **host-reading seam is
  interface-injectable** — the conformance harness supplies a synthetic `HostSource`. The provider contract
  MUST state this seam is injectable; left implicit, every impl mocks differently and "same fixture across
  impls" fails. One line, load-bearing for the cross-impl goal.
- **Conformance:** for a fixed fake host, the populated entities match a fixture (fields, units,
  descriptor states); every impl reports `unsupported` for the same platform-absent fields; coarsened
  fields carry `fidelity: "coarse"`. Fixtures MUST also: **tolerate `available_bytes` = `unsupported`/`coarse`
  on some native platforms** (not only WASM, §4); and pin the **`Sysinfo.Freeram ≠ MemAvailable` trap** (§4) so
  no impl silently under-reports headroom. **Route the DRAFT to the cohort (Go/Rust/Py + browser) pre-impl** —
  providers differ most across exactly those runtimes, and the coarsening rules need cross-checking.

## §11 Open questions

1. **Namespace split** — mirror OTel literally with `system/device/host/*` (identity) vs
   `system/device/metrics/*` (gauges) as above, or one flat `system/device/*`? (Leaning: the split — it
   encodes populate-once vs refresh-on-read.)
2. **Coarsening table** — exact buckets/bounds per field; is fidelity requested via the grant (leaning
   yes) with the provider honoring it? (Now with the exact-fidelity *scheduling* tier pinned, §5.)
3. **~~`offered` in v1~~ — RULED (§7).** Populate from `discover_handlers` (it has a landed reader — the
   compute-host admission); the open remainder is only the *exact enumeration shape* of the capability set,
   not whether to populate it.
4. **Idiom enum** — confirm `desktop|tablet|phone|tv|watch|server|headless`. (Native-vantage note: a native
   provider mostly returns `server`/`headless`/`unknown`; a scheduler MUST NOT assume `idiom` is always
   present — the mobile idioms are browser/mobile-framework signals.)
5. **Refresh mechanism** — refresh-on-read vs an explicit `system/device:refresh` to repopulate the
   metric entities (the one place a thin op might earn its keep). Decide with the cohort.
6. **Storage volume key (§4a)** — v1 ships the single "peer's store volume" form; confirm the keyed
   `metrics/storage/{volume}` set is the right forward shape for storage-aware scheduling (native review §4).
7. **`reserved`/committed population (§4)** — v1 MAY leave it `unknown`; when a scheduler exists, confirm it
   (not device) is the writer of the committed total, with device only *reporting* it (the §8 sensing≠accounting
   boundary).
