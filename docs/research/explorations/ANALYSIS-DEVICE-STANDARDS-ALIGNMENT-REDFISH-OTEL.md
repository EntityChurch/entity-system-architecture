# ANALYSIS — aligning device to existing standards (OpenTelemetry + Redfish)

**Phase:** Analysis (pre-proposal). 2026-07-12. Final research pass before drafting the proposal.
**Companion to** the device/storage exploration, shape analysis, orchestration + actuation
explorations. **Purpose:** don't invent field names or an availability vocabulary — align to the two
adopted standards that already cover this ground. Cited §Sources.

---

## §0 TL;DR

Two standards validate our independently-derived design and hand us vocabulary:

1. **OpenTelemetry semantic conventions** already split **`host.*`/`os.*` (static resource identity)**
   from **`system.*` (dynamic metrics)** — *exactly* our "populate-once entities vs refresh-on-read
   gauges" split (device exploration §6). **Adopt OTel field names** (`host.arch`, `os.type`,
   `system.cpu.logical.count`, `system.memory.usage`, …) so our surface is legible to anyone who knows
   OTel and bridgeable to OTel tooling. Free interop; no bikeshedding.
2. **DMTF Redfish** — the industry hardware-management standard — models component status as
   **`Status{State, Health}`** with `State ∈ {Enabled, Disabled, Absent, UnavailableOffline, …}`.
   Direct prior art for our **availability descriptor**, and its ComputerSystem→Processor/Memory/
   Storage resource model reconfirms the §3 taxonomy. We take the **State/presence axis** (map to our
   `{value|unsupported|denied|unknown}`), **drop Health** (that's management/alerting, out of scope for
   read-only introspection).

## §1 OpenTelemetry — the naming + the static/dynamic split

- **Deliberate namespace split** (this is the important structural confirmation): *"The `host.*`
  namespace SHOULD be exclusively used to capture **resource attributes**. To report host **metrics**,
  the `system.*` namespace SHOULD be used."* Resource attributes = static identity of the host
  (`host.id`, `host.name`, `host.arch`, `host.type`; `os.type`, `os.version`, `os.build_id`). Metrics =
  dynamic gauges (`system.cpu.utilization`, `system.cpu.logical.count`, `system.cpu.physical.count`,
  `system.cpu.frequency`, `system.memory.usage`, `system.memory.limit`, `system.filesystem.usage`,
  `system.disk.*`, `system.network.*`).
- **This is our populate-vs-refresh decision, already standardized:** the `host.*`/`os.*` static set →
  populate once as operational-state entities at startup (cheap, stable); the `system.*` metric set →
  the volatile gauges we said should be refresh-on-read / hybrid (device exploration §6, shape analysis
  §7). OTel even implies the **namespace** cut we should mirror: identity vs metrics.
- **Action:** name `system/device` fields after OTel where a field exists there — `os.type`,
  `os.version`, `host.arch`, `host.name`, `system.cpu.logical.count`, `system.cpu.physical.count`,
  `system.memory.usage`/`system.memory.limit`, `system.filesystem.usage`. Our additions (offered
  capabilities, total-vs-available framing) layer on; we don't rename what OTel already named.

## §2 Redfish — the availability/state vocabulary

- Redfish is out-of-band device/hardware management (RESTful+JSON); `ComputerSystem` hyperlinks to
  `Processor`, `Memory`, `Storage`, etc. — the same domain cut (§3 of the exploration), from the
  industry-standard side.
- Every resource carries **`Status: { State, Health, HealthRollup }`**. **`State`** is the presence/
  reachability axis: `Enabled`, `Disabled`, `Absent`, `UnavailableOffline`, `StandbyOffline`, `InTest`,
  `Starting`, … **`Health`** is `OK|Warning|Critical` with a **`HealthRollup`** aggregating children.
- **Mapping to our availability descriptor:**
  | Redfish `State` | Our descriptor `state` |
  |---|---|
  | (has a value, `Enabled`) | `value` |
  | `Absent` / not-applicable-on-this-platform | `unsupported` |
  | `UnavailableOffline` / not-yet-probed | `unknown` |
  | (permission — Redfish uses HTTP 403, not a State) | `denied` |
  - Redfish confirms the **presence axis** is real and standardized; it separates presence (`State`)
    from **`Health`** — which we **omit** (health/alerting is management, and device is read-only
    introspection, not monitoring). Our `denied` comes from the auth layer (like Redfish's 403), not a
    State value — consistent with modeling it as a capability outcome.
- **HealthRollup** is a useful idea we *don't* need now but should be aware of: an aggregate status over
  children. If a `:query`-all response ever wants a single "is everything readable" summary, that's the
  pattern — defer.

## §3 What this firms up for the proposal

1. **Field names:** adopt OTel `host.*`/`os.*`/`system.*` names; don't invent. (Interop + no bikeshed.)
2. **Static/dynamic split is standardized:** `host.*`/`os.*` = static identity entities (populate once);
   `system.*` = metric gauges (refresh-on-read/hybrid). Resolves the "populate vs query" open tension
   (shape analysis §7) with an adopted precedent — possibly even mirror the **namespace** split in our
   entity layout (`system/device/host/*` identity vs `system/device/metrics/*` gauges — decide in the
   proposal).
3. **Availability descriptor:** cite Redfish `Status.State` as prior art; keep our four states
   (`value|unsupported|denied|unknown`); **omit Health** (out of scope); `denied` is an auth outcome
   (Redfish 403), not a State value. Descriptor stays small and defensible.
4. **Taxonomy reconfirmed** from the industry-management side (Redfish ComputerSystem→Processor/Memory/
   Storage), independent of the developer-library side (OSHI/sysinfo).

## §4 Next

This closes the research. The device surface is now triangulated from **five** angles: data-model
(OSHI/sysinfo/systeminformation/Godot/browser), operational-state shape, orchestration placement,
actuation consumption (hostability), and **standards alignment (OTel naming + Redfish state)**.
Recommend drafting the proposal(s) now — see the structure decision in the session/status thread:
**split** into a small reusable availability-descriptor primitive + the `system/device` proposal;
storage handled as a minor operational-state note (QUERY + GC + a small accounting entity), not an
extension.

---

## Sources

- OpenTelemetry: [Resource semantic conventions](https://opentelemetry.io/docs/specs/semconv/resource/), [Host attributes](https://opentelemetry.io/docs/specs/semconv/registry/attributes/host/), [System metrics](https://opentelemetry.io/docs/specs/semconv/system/system-metrics/)
- DMTF Redfish: [Resource and Schema Guide (DSP2046)](https://www.dmtf.org/sites/default/files/standards/documents/DSP2046_2024.2.html), [Data Model Specification (DSP0268)](https://www.dmtf.org/sites/default/files/standards/documents/DSP0268_2025.4.html), [Property Guide (DSP2053)](https://redfish.dmtf.org/schemas/v1/DSP2053_2024.1.html)
