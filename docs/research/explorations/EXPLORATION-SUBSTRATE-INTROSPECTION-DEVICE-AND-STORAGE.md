# EXPLORATION — Substrate introspection: device + storage capabilities

**Phase:** Exploration / analysis (pre-proposal). 2026-07-11. **Not a proposal, not ratified.**
**Intake:** `entity-browser-rust/.../PROPOSAL-SYSTEM-DEVICE-AND-STORAGE-CAPABILITIES.md` (`8ad781de`).
**Method:** survey of how the cross-platform world actually exposes host/hardware/OS
introspection (7 systems, real API surfaces, cited §Sources), then synthesis into a taxonomy,
an availability model, a privacy/coarsening model, and concrete op/entity sketches for us.

---

## §0 TL;DR (what the research changed)

The browser-rust proposal was right that this is a real gap and a `§5.5` read-only capability —
but the interesting design does not live in "add two handlers." It lives in **three findings the
survey surfaces**:

1. **The taxonomy is already convergent.** OSHI, Rust `sysinfo`, Node `systeminformation`, and
   Godot cut the host into the *same ~8 domains* (platform · cpu · memory · storage · gpu ·
   network · sensors/power · display). Our per-domain op split isn't a guess — it matches how
   every mature library already carves it. §3.
2. **"Absent" has two independent axes that no single system unifies** — **support** (does this
   platform expose it: Godot `has_feature()`) vs **permission** (are you allowed: Web Permissions
   API `granted/denied/prompt`). Our availability descriptor's real contribution is *unifying both*
   into one field-state `{value | unsupported | denied | unknown}`. §4.
3. **Coarsening is a concrete, load-bearing mechanism in the field, not a nicety** — the browser
   quantizes `deviceMemory` to a power of 2 then clamps to bounds; rounds network `rtt` to 25 ms;
   the W3C is adding *bucketing + limit-clamping* to WebGPU because adapter info alone is 25–35
   bits of entropy. And fingerprinting is **cumulative** (stacked across fields), which is the real
   argument for grant-scoped fidelity + opt-in-per-domain, not just per-field masking. §5.

Two things the intake missed that the survey forces: **gauge-vs-rate metrics** (some facts are
point-in-time; CPU-load and net-throughput are *rate* metrics meaningless from one read — `sysinfo`
computes them on the diff of two snapshots), §6; and the **use cases that justify it inside our own
stack** (COMPUTE placement, SUBSTITUTE/CDN capacity, GC pressure, adaptive L5 serving), §7.

---

## §1 The gap (brief — established in the intake)

A peer exposes files (`local/files`) but can expose **nothing about the store it keeps or the
machine it runs on over a connection.** Run the native peer headless and manage it from a phone:
Tauri IPC and `navigator.storage.estimate()` are both gone; "how big is its store / how much disk /
is it under memory pressure" is a capability on the peer or it is unanswerable. Greenfield — an
exhaustive search of the archived workspace (60 explorations / 189 reviews / 225 proposals) found
**zero** prior device/host/substrate work.

## §2 Prior-art survey (how the world does it)

| System | Surface | What it exposes (real fields) | Heterogeneity / absence handling |
|---|---|---|---|
| **OSHI** (Java, JNA) | `HardwareAbstractionLayer` + `OperatingSystem` | The most complete taxonomy: ComputerSystem (mfr/model/serial/firmware/baseboard), CentralProcessor (physical+logical count, groups, NUMA, load), GlobalMemory (total/avail, swap/virtual), **PhysicalMemory per-DIMM** (bank/capacity/clock), Sensors (CPU temp, fan RPM, voltage), PowerSource (battery % / remaining / state), Displays (EDID), DiskStores (model/serial/size/RW), GraphicsCards (VRAM), NetworkIFs (MAC/IP/bandwidth/TCP-UDP), SoundCards, USB; OS: family/version/build/**bitness**, processes/threads, uptime, FileStores (mount/type/usable/total/IO), net params (hostname/DNS/gateway) | one abstract API, native backends per-OS; fields simply empty/zero where a platform can't provide |
| **Rust `sysinfo`** (the native provider we'd actually use) | `System` + `Disks`/`Networks`/`Components`/`Users` | System: total/used RAM+swap, name/kernel/OS-version/hostname, CPUs, processes; Disks; Networks (per-IF bytes); Components (temperatures); Users | **explicit refresh; computes on the diff of two snapshots** — rate metrics need two reads (§6) |
| **Node `systeminformation`** | 50+ async fns | `cpu` (mfr/brand/speed/cores/physicalCores), `mem`, `graphics` (multi-display, size, connection), `diskLayout`, `fsSize` (bytes RW + **per-second** rates), `networkInterfaces`, `osInfo` | function-per-domain; rate fields explicitly time-derived |
| **Godot** `OS` + `RenderingServer` | engine singleton | `get_memory_info()` → `{physical, free, available, stack}`; `get_processor_name()`/`get_processor_count()`; `get_name()`/`get_distribution_name()`/`get_version()`; `get_video_adapter_driver_info()`; `get_static_memory_usage()` | **`has_feature(tag)` = runtime capability detection** — the abstract "is this supported here" query, answered *before* you call. The support axis (§4). |
| **Browser** `navigator.*` / `StorageManager` / WebGPU | sandboxed | `deviceMemory` (**GB, rounded to power-of-2 then clamped**), `hardwareConcurrency` (Safari clamps), `storage.estimate()` → `{quota, usage}` (**obscured**), Network Info (`effectiveType` 2g/3g/4g, `rtt` **rounded to 25 ms**, `downlink`, `saveData`), `GPUAdapterInfo` (vendor/architecture/device) | deliberately *narrowed*; **coarsened by design** to fight fingerprinting (§5); no NIC enumeration |
| **.NET MAUI** `IDeviceInfo` (+ Battery/Connectivity/Sensors) | unified abstraction | `DeviceInfo.Current`: manufacturer, model, name, version, **Platform**, **Idiom** (Phone/Tablet/Desktop/TV/Watch), DeviceType (physical/virtual); sibling abstractions for Battery, Connectivity, sensors | one cross-platform interface, per-platform impls; "the most common scenarios" abstracted, rest platform-specific |
| **Flutter `device_info_plus` / Qt `QSysInfo`+`QStorageInfo`+`QNetworkInterface`** | plugin / core lib | OS version, model, manufacturer, hardware; Qt `QStorageInfo` (volume space/mount/label/fs), `QSysInfo` (kernel/product/arch) | per-platform getters; web build returns the browser subset |
| **Web Permissions API** (the permission axis) | `navigator.permissions.query()` | `PermissionStatus.state` ∈ **`granted` / `denied` / `prompt`** | the standardized way to ask "am I allowed" *without* triggering the feature (§4) |

## §3 The convergent taxonomy (evidence our domain cut is right)

Cross-tabulating the surveyed systems, the **same domains recur everywhere** — strong evidence for a
per-domain op split rather than one opaque blob:

| Domain | OSHI | sysinfo | systeminformation | Godot | Browser | Our proposed op |
|---|---|---|---|---|---|---|
| **platform** (os/arch/version/idiom/hostname) | ✓ | ✓ | ✓ | ✓ | UA-CH (partial) | `system/device:platform` |
| **cpu** (cores, model, load) | ✓ | ✓ | ✓ | ✓ | `hardwareConcurrency` | `:cpu` |
| **memory** (total/available) | ✓ | ✓ | ✓ | ✓ | `deviceMemory` (coarse) | `:memory` |
| **storage** (disk total/free; fs) | ✓ | ✓ | ✓ | — | `storage.estimate` (quota/usage) | `:storage` |
| **gpu** (adapter/vendor/vram) | ✓ | — | ✓ | ✓ | `GPUAdapterInfo` (coarse) | `:gpu` |
| **network** (interfaces / reachability) | ✓ | ✓ | ✓ | — | Network Info (no NIC list) | `:network` |
| **sensors / power** (battery/temp/fan) | ✓ | ✓ (temps) | ✓ | — | Battery API (deprecating) | `:power` (later) |
| **display** (count/size/EDID) | ✓ | — | ✓ | ✓ | `screen.*` | `:display` (later) |

**Reading:** the peer's own store (`content_blobs`, `live_paths`) is a *different* domain — it's
entity-native, always known, and belongs to `system/storage` (the store), not `system/device`
(the host). The host's physical disk lives under `system/device:storage`. That's the two-handler
seam, and it falls out of the taxonomy cleanly. Ship the **top four** (platform/cpu/memory/storage)
in v1 (every system provides them, they cover the motivating use cases); gpu/network/power/display
layer on later.

## §4 The absence problem — support vs permission vs unknown (the real primitive)

The survey's sharpest finding: **two independent axes explain "why is this field absent," and no
surveyed system unifies them.**

- **Support axis** — *can this platform/build even produce it?* Godot answers this with
  `has_feature(tag)`: a **pre-flight capability query**. A browser peer has no NIC enumeration; a
  headless server has no battery. This is intrinsic, not a policy choice.
- **Permission axis** — *are you (this grantee) allowed to see it?* The Web Permissions API models
  this as `granted / denied / prompt` — orthogonal to support (a supported field can be denied).
- **Liveness axis** — *has it been probed yet?* `sysinfo`'s refresh model makes "not yet sampled"
  a real third state distinct from "zero."

The browser-rust proposal's `{value | unsupported | denied | unknown}` descriptor is exactly the
**union of these three prior-art axes into one field-state** — and framing it that way is the
argument for blessing it: it is not invented, it is *Godot's feature-tag + the Permissions API's
tri-state + sysinfo's probe-state*, collapsed to one honest field. Concretely:

```
availability<T> := { state: "value" | "unsupported" | "denied" | "unknown",
                     value?: T,               # present iff state = "value"
                     fidelity?: "exact" | "coarse" }   # §5
```

This is **reusable far beyond device** — any cross-platform, cross-grant, or partial capability
(network reachability, discovery results, a `MAY` a peer can't honor). That breadth is why it should
land as a **standalone small primitive** (its own micro-spec or a `GUIDE-CAPABILITIES` section),
*before* the device handler that first needs it. It is also our own "deliver-or-signal, never fake"
substrate floor expressed at the field level.

## §5 Privacy & coarsening — concrete mechanisms, not a disclaimer

Device introspection is *the* fingerprinting surface, and the browser world has already worked out
the mitigations — we should lift the mechanisms, not re-reason them:

- **Quantize then clamp.** `deviceMemory` is rounded to the nearest **power of two**, divided to GB,
  then **clamped to [lower, upper] bounds** so very-small / very-large devices don't stand out.
  → adopt: coarse memory/storage report bucketed (power-of-two or fixed bucket table) + bounds.
- **Fixed rounding granularity for rates.** Network `rtt` is rounded to the nearest **25 ms**.
  → adopt: rate/latency fields report at a fixed granularity, never raw.
- **Bucketing + limit-clamping for high-entropy surfaces.** WebGPU adapter info is **25–35 bits**
  of entropy (vs 15–20 for WebGL); the W3C mitigation is **bucketing similar GPUs** and **clamping
  reported limits** to standard tiers. → gpu is the highest-risk domain; default to a coarse
  adapter *class*, expose exact model only under an explicit high-fidelity grant.
- **Fingerprinting is cumulative.** Entropy comes from *stacking* deviceMemory + cores + platform +
  GPU + screen. → per-field masking is insufficient; the **grant** must gate the whole device
  surface, and **opt-in-per-domain** matters because a consumer combines domains. This is the
  concrete argument for: **coarse/bucketed by default; exact only under an explicit fidelity grant;
  device facts off unless the grant names the domain** (`GUIDE-CAPABILITIES §10`).

The `fidelity: exact|coarse` tag on the descriptor (§4) is how a consumer *knows* which it got —
so it never mistakes a bucket for a measurement.

## §6 Liveness — snapshot vs rate (the gauge/rate split)

`sysinfo` and `systeminformation` both make this explicit and the intake glossed it: **some facts
are gauges (meaningful from one read), some are rates (meaningless without two).**

- **Gauges** — disk total/free, RAM total/available, core count, platform, gpu model. A `:stats`
  snapshot returns these directly.
- **Rates** — CPU load %, network throughput, disk IOPS. `sysinfo` computes them on the **diff of
  two refreshes**; `systeminformation` exposes per-second fields. A single `execute` can't produce
  a truthful rate.

Design consequence for the op: **v1 `:stats` / `:query` returns gauges only.** Rate metrics are
either (a) explicitly a "sampled over N ms" op that takes two internal reads, or (b) deferred to a
subscription-backed stream (the intake's stated non-goal — "a `stats` snapshot; subscription can
layer on later via `system/subscription`"). Pin this so an impl doesn't return a bogus instantaneous
"CPU 0%."

## §7 Use cases — what a consumer *does* with it (why it's worth exposing)

The intake justified it only by the Storage-window UI. The survey's canonical uses are richer, and
several land *inside our own stack* — which is the real case for standardizing it:

- **Adaptive serving (L5).** The documented headline use of Network Info + deviceMemory: serve
  lighter assets on `saveData`/`slow-2g`, a lighter experience on low-RAM. Directly relevant to the
  `APP-CONVENTION-SEMANTIC-CONTENT-SITE` render path.
- **Compute placement / scheduling (`EXTENSION-COMPUTE`).** core count + memory pressure decide
  "can this peer accept this compute workload / how much." A peer that can report its cpu/memory is
  schedulable; one that can't isn't.
- **Storage-substrate capacity (`EXTENSION-SUBSTITUTE` / CDN corridor).** disk free on the host
  volume decides whether a peer can serve as a storage substrate / accept a mirror. "How much can
  you hold" is `system/device:storage` + `system/storage:stats`.
- **GC / retention pressure (`GUIDE-GC`).** `system/storage:stats` reclaimable/save-state churn is
  the signal for when to prune — the append-only-store growth the app already computes locally.
- **Graphics auto-config.** Godot's own pattern: `get_processor_name` + video adapter → benchmark
  annotation + automatic quality tier. An L5 client can do the same from `:cpu`/`:gpu`.
- **Honest remote management (the intake).** the headless-peer Storage/System window, now uniform
  local-or-remote through one handler path.

The through-line: device/storage introspection is **infrastructure the rest of the extension family
consumes** (COMPUTE, SUBSTITUTE, GC, L5), not just a UI nicety. That is why it belongs in the spec,
not an app.

## §8 Concrete shape sketch (to feed the proposal — not final)

> **Shape refined by `ANALYSIS-DEVICE-STORAGE-SHAPE-OPERATIONAL-STATE.md`:** these are better
> modeled as **operational-state entities** a peer populates in its own tree (read via `tree:get`,
> subscribed, QUERY-aggregated) than as bespoke `:stats`/`:query` handler ops — `system/storage`
> is not a new extension, and `system/device` is an operational-state *vocabulary + host provider*,
> not a query extension. The field shapes below still hold; read them as **entity types**, not op
> results. Remote-readability is satisfied by `tree:get` being cross-peer dispatchable.

Grounded in §3's taxonomy and §4's descriptor:

```
# system/storage:stats  → system/storage/stats   (entity-native; mostly exact, always known)
{ content_blobs: int, content_bytes: int, live_paths: int,
  save_state_paths: int, reclaimable_bytes: availability<int> }   # estimate → descriptor

# system/device:platform → system/device/platform
{ os: availability<str>, arch: availability<str>, version: availability<str>,
  idiom: availability<str>,                       # MAUI's Idiom: desktop|tablet|phone|tv|watch|server
  hostname: availability<str> }                   # PII → denied/coarse by default

# system/device:memory   { total_bytes: availability<int>, available_bytes: availability<int> }   # browser: coarse bucket
# system/device:storage  { volume_total_bytes: …, volume_free_bytes: …,
#                          origin_quota_bytes: …, origin_usage_bytes: … }   # browser fills the origin pair, unsupported the volume pair
# system/device:cpu      { logical_cores: …, physical_cores: …, model: availability<str> }   # load = rate, §6, deferred
# system/device:gpu      { present: …, adapter_class: …, vendor: …, vram_bytes: … }   # highest fingerprint risk, coarse default
# system/device:network  { interfaces: availability<list>(native)/unsupported(browser),
#                          effective_type: …, rtt_ms: …, save_data: … }
```

`system/device:query` returns all installed domains at once; per-domain ops return one. Units are
fixed integers (bytes/counts/ms) so a consumer renders identically regardless of provider.

## §9 Namespace + level (grounded, brief)

- **Namespace = `system/`.** Principle the survey supports: `system/` = the peer's own *substrate*
  (store, host device, identity, clock — intrinsic; every peer runs *somewhere*); `local/` =
  *optional mounted* resources (files). Device is always-present substrate, so it belongs in
  `system/` more firmly than files do. Write this rule down; it recurs (processes, local net).
- **Level (refined by the shape analysis).** Not a new query-handler extension. The
  cross-impl-observable parts — the **availability descriptor** and the **`system/device` entity
  vocabulary** — are spec'd as an **operational-state addition** (V7 §3.13 sibling), read via the
  ordinary `tree`/`query` handlers; store self-accounting is a small operational-state entity +
  QUERY + the GC signal. The **per-OS host provider** (Rust `sysinfo` / Go/Py equivalent / browser
  `navigator.*`) is **impl-local, not spec.** The **SDK** carries the typed reader + descriptor
  decoding. Not kernel, not L5, not a standalone stats extension. See the analysis.

> **Sharpened by `EXPLORATION-RUN-ENVIRONMENTS-AND-WORKLOAD-ACTUATION.md §5/§8`:** what a
> scheduler/runtime consumes is *hostability*, not a datasheet. Fold in: **(a) total-vs-available**
> for memory/storage (K8s capacity/allocatable); **(b) lead with offered-capabilities + capacity/
> headroom, demote raw cpu/gpu model to secondary informational** (also the high-fingerprint fields
> → coarse/opt-in); **(c) reserve an "offered capabilities/interfaces" field** (forward-compatible
> with WASM hosting). Focuses the surface; doesn't expand it.

## §10 Open questions for the proposal (surfaced by the research)

1. **Availability descriptor: standalone primitive?** Strong yes (§4) — its own micro-spec /
   `GUIDE-CAPABILITIES` section, landed *first*. Confirm the field name + the four states + the
   `fidelity` tag.
2. **Coarsening table.** What exact buckets/rounding per field (memory power-of-two + bounds; rtt 25
   ms; gpu adapter-class list)? Is coarsening a **capability-model** concern (grant carries fidelity)
   or a **provider** concern? Lean: grant carries the *ask*, provider *honors* it (§5).
3. **Gauge vs rate.** v1 = gauges only; rates via an explicit sampled op or subscription later (§6).
   Pin so no impl fakes an instantaneous rate.
4. **v1 domain set.** platform/cpu/memory/storage (the convergent four) vs including gpu/network now.
   Lean: four in v1; gpu/network/power/display as a follow-on (gpu needs the coarsening design first).
5. **Provider sharing.** A shared native provider crate across impls, or each impl its own over the
   shared contract? (browser is necessarily its own.) Impl convenience, not a spec matter.
6. **Idiom vocabulary.** Adopt MAUI's `Idiom` enum (desktop/tablet/phone/tv/watch) + add `server`/
   `headless`? Useful, low-cost, cross-impl-observable → pin the enum.

## §11 Where this lands / next

- **Draft `docs/proposals/PROPOSAL-SYSTEM-DEVICE-AND-STORAGE-CAPABILITIES.md`**, folding the intake,
  sequenced: **availability descriptor first** (§4, the reusable unlock) → `system/storage:stats`
  (entity-native, simplest) → `system/device:{platform,cpu,memory,storage}` with the coarsening
  table → SDK client. Route the DRAFT to the cohort (Go/Rust/Py + browser) pre-impl — device
  providers differ most across exactly those runtimes, and the coarsening rules need cross-checking.
- **Non-goals** (from intake + §6): no writable device control; no continuous telemetry (snapshot,
  not stream); no rate metrics in v1; no bespoke wire format.
- **Separate exploration:** *sync under churn* (liveness / reconnect-without-hammering / resume) —
  distinct design space, queued; captured in `docs/status/STATUS-2026-07-11-pullin-round2-device-and-churn.md §C`.

---

## Sources

- Rust `sysinfo`: [docs.rs](https://docs.rs/sysinfo/latest/sysinfo/), [System struct](https://docs.rs/sysinfo/latest/sysinfo/struct.System.html), [GitHub](https://github.com/GuillaumeGomez/sysinfo)
- Node `systeminformation`: [npm](https://www.npmjs.com/package/systeminformation), [systeminformation.io](https://systeminformation.io/)
- OSHI: [oshi.ooo](https://www.oshi.ooo/), [BoxLang OSHI taxonomy](https://boxlang.ortusbooks.com/boxlang-framework/modularity/hardware-and-system-info)
- Godot: [OS class](https://docs.godotengine.org/en/stable/classes/class_os.html), [feature tags](https://docs.godotengine.org/en/stable/tutorials/export/feature_tags.html), [get_processor_name PR](https://github.com/godotengine/godot/pull/44716)
- Browser device APIs: [Navigator.deviceMemory (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/deviceMemory), [hardwareConcurrency](https://www.testmuai.com/learning-hub/navigator-hardwareconcurrency/), [Storage API (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/Storage_API), [NetworkInformation (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/NetworkInformation), [GPUAdapterInfo (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/GPUAdapterInfo), [adaptive serving (AddyOsmani)](https://addyosmani.com/blog/adaptive-serving/)
- Fingerprinting/coarsening: [deviceMemory fingerprinting (Castle)](https://blog.castle.io/deep-dive-how-navigator-devicememory-can-be-used-for-fingerprinting-and-bot-detection/), [WebGPU fingerprinting entropy](https://botbrowser.io/en/blog/webgpu-fingerprinting/)
- Cross-platform abstractions: [.NET MAUI DeviceInfo](https://learn.microsoft.com/en-us/dotnet/maui/platform-integration/device/information), [Flutter device_info_plus](https://pub.dev/packages/device_info_plus), [Qt QStorageInfo](https://doc.qt.io/qt-6/qstorageinfo.html), [Qt QSysInfo](https://doc.qt.io/qt-6/qsysinfo.html)
- Permission model: [Web Permissions API (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/Permissions_API), [PermissionStatus.state](https://developer.mozilla.org/en-US/docs/Web/API/PermissionStatus/state)
