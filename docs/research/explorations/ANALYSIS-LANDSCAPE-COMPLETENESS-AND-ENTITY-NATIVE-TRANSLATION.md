# ANALYSIS — landscape completeness + the entity-native translation principle

**Phase:** Analysis (pre-proposal refinement). 2026-07-12. Rounds out the device landscape and fixes
the rule for **how we relate to external standards** — so the proposal is entity-system-native, not a
hodgepodge of borrowed syntaxes. **Companion to** the device explorations + the OTel/Redfish alignment
analysis.

---

## §0 TL;DR

- **The landscape is complete.** We've now surveyed the full lineage — firmware inventory → object
  model → network management → modern management → observability → dev libraries → sandbox → game
  engine → cross-platform app frameworks → orchestration. Nothing structurally new is out there (§1).
- **Redfish is current, not legacy** (honest check): DMTF's IPMI replacement, spec DSP0266 **v1.23.1
  (2025-12)**, implemented by every server vendor + OpenBMC, OCP-required. We borrow its data-shape
  wisdom, not the protocol (§2).
- **They all converge** on the same handful of concepts (§3) — which is what lets us adopt confidently.
- **The rule (the operator's steer):** adopt the convergent **semantics**, express them **entity-
  native**, and **justify each within our system** — one coherent idiom (kebab paths, snake_case keys,
  the one availability descriptor, entities-in-the-tree, capability-path grants). We do **not** import
  dotted namespaces, PascalCase, CIM class models, or any external wire/protocol form (§4–§5).

## §1 The full landscape (completeness)

| Era / role | Standard / system | What it is | What we take |
|---|---|---|---|
| Firmware inventory | **SMBIOS / DMI** (DMTF) | the vendor table format under everything; still mandated (Windows HCK) | nothing directly — it's the providers' data source |
| Object model | **CIM / WBEM** (DMTF), WMI | the class model Windows exposes hardware through | nothing — too heavy an ontology |
| Network mgmt | **SNMP HOST-RESOURCES-MIB** (RFC 2790) | "objects common across computer architectures" — storage/device/process | confirms the domain taxonomy is decades-stable |
| Modern mgmt | **DMTF Redfish** (current) | REST/JSON BMC management; IPMI replacement | the `Status.State` presence model; taxonomy (§2) |
| Observability | **OpenTelemetry** semconv | `host.*`/`os.*` identity vs `system.*` metrics | the **static/dynamic split**; field *concepts* |
| Dev libraries | **OSHI / sysinfo / systeminformation** | cross-platform host/hardware query libs | the concrete field taxonomy; the provider pattern |
| Sandbox | **browser** `navigator.*`/WebGPU | deliberately-narrowed, coarsened host facts | the **coarsening mechanisms**; the honesty need |
| Game engine | **Godot** `OS`/`has_feature` | abstract API + feature detection | the **support/feature axis** of the descriptor |
| App frameworks | **.NET MAUI / Flutter / Qt** | unified device abstraction + Idiom | the `idiom` concept; abstraction-with-providers |
| Orchestration | **K8s / Nomad / wasmCloud** | sense→schedule→actuate; capacity/allocatable | total-vs-available; device=sensing; hostability |

The convergence across firmware→cloud and 1990s→2026 is the completeness signal: no era or role
introduces a *structurally* new concept beyond what §3 lists.

## §2 Redfish, honestly (is it real / still used?)

Yes — it's the current industry standard for out-of-band server management, DMTF's explicit **secure,
multi-node replacement for IPMI-over-LAN** (HTTPS/JSON/REST, TLS-required). Latest **DSP0266 v1.23.1,
2025-12-04**; implemented by Dell iDRAC, HPE iLO, Lenovo XCC, Supermicro, Cisco, Intel, and the OpenBMC
reference stack; **OCP-compliant servers are required to support Redfish profiles**; IPMI is now the
legacy compatibility layer. **But we do not implement Redfish** — it's a BMC management protocol, not an
in-process introspection API. We take exactly two things: its **`Status.State` presence vocabulary**
(prior art for the availability descriptor) and its **ComputerSystem→Processor/Memory/Storage taxonomy**
(an independent, industry-side confirmation of the same domains the dev libraries show). Nothing else —
no HTTP semantics, no CIM lineage, no health/rollup machinery.

## §3 What they all converge on (the adoptable core)

Independent of syntax, every system above agrees on:
1. **The ~8 domains** — platform · cpu · memory · storage · network · gpu · sensors/power · display.
2. **Total vs available** — installed capacity ≠ schedulable headroom (K8s capacity/allocatable).
3. **A presence/availability axis** — value / absent / unavailable / not-permitted (Redfish State +
   Permissions API + Godot has_feature).
4. **Static identity vs dynamic metric** — stable facts vs volatile gauges (OTel host.*/os.* vs
   system.*).
5. **One abstract contract, per-platform providers, honest partiality** — every cross-platform lib.

These five are the design, and they're what we adopt. Everything else is that system's local syntax.

## §4 The entity-native translation principle

**Adopt the semantics; express them entity-native; justify each within our system.** For every borrowed
concept, the test is *"does this make sense in the entity model on its own terms?"* — not *"does a
standards body do it?"*

| Convergent concept | External form (NOT adopted) | Entity-native form (adopted) | Justified within our system because… |
|---|---|---|---|
| host/os identity | OTel dotted `host.arch`, `os.type` | snake_case keys `arch`, `os_type` in a `system/device/host/platform` entity | our naming convention (kebab path / snake key); one `platform` entity, not OTel's host/os split — simpler for us |
| static vs dynamic | OTel `host.*` vs `system.*` namespaces | `system/device/host/*` (populate-once) vs `system/device/metrics/*` (refresh-on-read) | it's our operational-state **population lifecycle** (startup vs on-read), not a telemetry-pipeline distinction |
| presence/availability | Redfish `Status:{State:"Absent"}` PascalCase | the `availability<T>` descriptor `{state: unsupported}` | it's our **deliver-or-signal floor** at the field level, one descriptor everywhere |
| total vs available | K8s `capacity`/`allocatable` | `total_bytes` / `available_bytes` fields | what a consumer needs to reason about headroom; plain snake keys |
| feature/support detection | Godot `OS.has_feature("...")` | the descriptor's `unsupported` state | folds into the one descriptor — no separate feature-query surface |
| capability advertisement | K8s device-plugin resources | a read-only `offered` field of entity handler/interface names | it's just handlers we already `discover_handlers`; no new concept |
| read over a connection | Redfish HTTP GET | ordinary `tree:get` (cross-peer, capability-gated) | our substrate already makes any entity remotely readable |

**Coherence rules (one idiom, checked against `STYLE-NAMING-CONVENTIONS`):** path segments kebab; map
keys snake_case; enum values kebab; **all** optional/partial fields use the **single** availability
descriptor (never a per-domain "missing" convention); facts live as **entities in the tree** (operational
state), read/aggregated via `tree`/`query`, observed via `subscription`; access via **capability-path
grants**. If a borrowed idea can't be said cleanly in that idiom, we restate it until it can, or we drop
it.

## §5 What we explicitly do NOT adopt

- **Syntax:** dotted namespaces (OTel), PascalCase properties (Redfish), CIM/WBEM class ontology, SNMP
  OIDs, SMBIOS table layouts.
- **Scope creep:** Redfish `Health`/monitoring/alerting; writable device control; the management-protocol
  surface (HTTP verbs, actions, event subscriptions in their form — we already have `subscription`).
- **Their transports/wire:** we implement none of these protocols; providers read the OS by whatever
  native means (sysinfo/navigator/…) and project into our entity form.

We are magpies for **data-shape wisdom**, not adopters of anyone's stack.

## §6 Consequence for the proposals (refinements)

1. **Add a "relationship to external standards" note** to `PROPOSAL-SYSTEM-DEVICE` stating §4's principle
   once: names are entity-native; external references are **concept provenance**, not adopted syntax.
2. **Reframe inline citations** in the device proposal from "= OTel os.type" (reads like syntax adoption)
   to "concept per OTel; name entity-native." Consolidate host/os into one `platform` entity (a
   deliberate divergence from OTel, justified).
3. **Confirm the coherence rules** hold across both DRAFTs (they do — snake keys, kebab paths, one
   descriptor); note it explicitly so a reviewer sees it was checked.
4. Landscape research is **closed** — five angles (data-model, shape, orchestration, actuation,
   standards) + this completeness/translation pass. Ready to refine + route the DRAFTs.

---

## Sources

- Redfish currency: [Redfish (Wikipedia)](https://en.wikipedia.org/wiki/Redfish_(specification)), [DMTF Redfish](https://www.dmtf.org/standards/redfish), [Dell iDRAC Redfish](https://www.dell.com/support/kbdoc/en-us/000178045/redfish-api-with-dell-integrated-remote-access-controller)
- Inventory-standard lineage: [SMBIOS (DMTF)](https://www.dmtf.org/standards/smbios), [DMI (Wikipedia)](https://en.wikipedia.org/wiki/Desktop_Management_Interface), [Host Resources MIB (RFC 2790)](https://www.rfc-editor.org/rfc/rfc2790), [net-snmp host MIB](https://www.net-snmp.org/docs/mibs/host.html)
