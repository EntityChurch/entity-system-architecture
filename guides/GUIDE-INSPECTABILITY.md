# GUIDE-INSPECTABILITY — Observing what flows through an entity-core peer

**Status**: Active

**Audience:** implementers of the entity-core protocol; extension authors auditing their own observable surface per §9; deployment operators designing observability policy.

**⚠ SECURITY:** Inspect surfaces tap into capability tokens, signature material, and full entity payloads. **Read the operator security disclaimer before turning anything on.** Baseline audit covers shipped surfaces (L0/L1 hooks, workbench-go `inspect/`, core-py `entity_core.diagnostic`); designed-but-not-shipped surfaces (`system/inspect/*` runtime handlers, wire recorder, aggregator peer) require their own follow-on audits when shipped. **Multiple audits is the discipline, not the exception.**

**Scope.** What is structurally observable in an entity-core peer, where impls hook into the substrate to surface it, what the cost / activation / filtering trade-offs look like, and the fundamental design choice between **out-of-band** inspection (parallel to the entity system) and **entity-native** inspection (observations stored as entities in the system). **Not normative.** The protocol does not require any peer to expose inspection APIs — a peer with zero introspection is wire-conformant. What the protocol *does* guarantee is that everything a peer does leaves an observable trail. This guide names that trail and the patterns for projecting it onto an operator-facing surface.

---

## §1 Why a guide and not an extension

A peer can be wire-conformant with zero inspection ergonomics — accept dispatches, store entities, emit responses. So inspection isn't protocol-mandatory the way subscription delivery or revision versioning are; it's an operational concern of operators, developers, and applications that want to *see* what's happening.

What *is* structural — and what this guide is built around — is that the entity model **guarantees observability as a side effect of how it stores state**:

| Substrate primitive | Inspection consequence |
|---|---|
| Entities are content-addressed | Any state, any payload, any error has a stable hash retrievable from the content store |
| Entity types are explicit | Filter "all `system/protocol/error` entities" is structurally a type-prefix query |
| Dispatches are entities (`system/protocol/v1/execute`) | Every request, response, and continuation step is stored and hashable |
| Continuations are entities (`system/continuation/v1/data`) | The state of any multi-step chain is bound at a known path |
| Chain-error markers are entities | Failures leave a stored trail |
| Bindings live in the tree namespace | "What's at this path right now?" is one read |
| Subscriptions are entities | Notifications are observable in the content store as they're emitted |
| Capabilities are entities | Authority chains can be walked; revocations are entities too |

Inspectability emerges from the model. This guide names the **hook points** every conformant impl can surface, the **trade-offs** of doing so, and the **patterns** for projecting the observation surface onto users.

The split:

| Concern | Where it lives |
|---|---|
| **What's observable** (the structural property) | This guide |
| **How my impl surfaces it ergonomically** (Go peer builder hooks, Rust extension trait, Python middleware) | Each impl's own dev docs + `GUIDE-DEBUGGABILITY.md` cookbook |
| **What my extension contributes** (entity types, paths, chain-error vocabulary, chain participation invariants) | Each extension spec + the §9 checklist in this guide |

---

## §2 What's structurally observable

A complete enumeration of the observation surface a conformant peer offers, abstracted from any language idiom:

### §2.1 The eight observable event classes

1. **Content event.** A new entity has been Put into the content store. Fact: `(hash, type, size, timestamp, is_new: bool)`. **Fires only on genuinely new entity** — duplicate Puts of the same content_hash do not re-emit. `is_new` makes the contract explicit so observers don't have to dedup.
2. **Binding event.** A path has been bound, rebound, or unbound in the location index. Fact: `(path, kind, hash, prior_hash, timestamp, cascade_depth?)`. **`kind ∈ {created, updated, deleted}`** — without this an observer can see that a path was bound but can't distinguish a new path from an overwrite from a deletion (where `hash=None`). The optional `cascade_depth` distinguishes user-emit (depth=0) from cascade-triggered binding.
3. **Dispatch event.** A handler has been invoked on a request. Fact: `(target_uri, operation, params_hash, request_id, timestamp, response_status, response_hash, chain_id?, parent_chain_id?)`. Emitted at request entry and exit. **Note on multi-site emission:** "dispatch" in a real impl typically has multiple natural sites (internal-dispatch within a process, cross-peer remote-dispatch, local-execute fallback). Cross-impl-comparable telemetry requires impls to agree on which site(s) emit the event. **Recommended baseline:** emit dispatch events from the site at which `handler(path, op, params, ctx)` is invoked — the boundary between the dispatcher and the handler body — regardless of how the dispatcher was entered. **Chain attribution (v1.2.2 SHOULD):** when a dispatch is part of a chain step (i.e., the dispatcher entered via a continuation advance or a chain-initiating EXECUTE), the event SHOULD carry `chain_id` and, when applicable, `parent_chain_id`. Values are available wherever the dispatcher has bounds context (chain_id / parent_chain_id are already in `Bounds` per V7 §3); fan-out is event-content, not new substrate. Without this, the §2.4 chain-trace projection covers chain-error markers and suspended continuations only — application-path bindings produced during a chain step (e.g., an EXECUTE writing to `myapp/result/...`) cannot be attributed to the originating chain without operator-side stitching across event classes, which contradicts §2.4's "structural observability without instrumentation." Outside a chain context, both fields are absent (this is observably distinct from `chain_id = ""`); observers should treat absence as "not chain-attributed" and proceed.
4. **Continuation event.** A continuation has advanced one step. Fact: `(chain_id, step_index, source_status, next_target, timestamp)`. Emitted on every step transition (including error short-circuits when configured).
5. **Wire event.** A protocol frame has crossed the network boundary. Fact: `(direction, request_id, frame_bytes, peer_address, timestamp)`. Emitted on every inbound / outbound frame.
6. **Notification-emission event.** A subscription has matched a state change. Fact: `(subscription_id, source_change_uri, notification_hash, match_timestamp)`. Emitted on the emitter side when a notification is produced.
7. **Notification-delivery event.** A notification has been dispatched to a subscriber. Fact: `(subscription_id, notification_hash, deliver_uri, delivery_status, delivery_timestamp, error_code?)`. Emitted on every delivery attempt. *Note:* events 6 and 7 are deliberately split — they have different rate characteristics, different failure modes, and different operator-visibility needs. F-CIMP-2 (Cohort B) was hidden in v1.0's conflated single-class formulation because the symptom was "Python's log shows 20 deliveries accepted at status=200" while the actual failures were happening at a different layer.
8. **Error event.** A `system/protocol/error` entity has been produced. Fact: `(code, message, originating_request_id, timestamp)`. Emitted whenever a handler or framework path returns 4xx/5xx.

This is the complete event surface. Every other observable property of a peer's behavior — chains, capability grants, version history — is composed from these.

**Note on observation semantics — sync vs async.** An impl may offer event observation as **sync** (inline with the emitting code) or **async** (queued, observed one scheduling tick later). Both are valid; each has trade-offs. Sync observation gives ordered, lag-free events at the cost of holding the emitting thread during observation; async observation decouples the emitter from observers at the cost of within-peer ordering lag. Impls offering both surfaces should document the latency characteristics; impls offering only one should document the choice. Composed capabilities (§2.3) that span multiple events from the same peer must account for **within-peer async lag** as a skew source independent of **cross-peer wall-clock drift** — these are two distinct skew sources that cross-peer correlation work must handle separately.

**Note on within-handler event ordering (v1.2).** For a single-handler write that produces both a content event and a binding event for the same hash, impls SHOULD emit the content event before the binding event — the substrate-level write ordering is `content_store.put` then `location_index.set` for the same logical write. Observers composing both classes for single-handler writes can rely on this order. **Cross-handler ordering is NOT guaranteed** — concurrent goroutines / async tasks / parallel handlers can interleave their content+binding pairs arbitrarily. Observers needing pair-correlation across handlers MUST use the explicit `request_id` or `chain_id` rather than event adjacency.

### §2.2 The five derived query capabilities

From the event classes above, five derived capabilities cover the operator-facing inspection landscape. An impl that surfaces any subset of these is *more inspectable* by that increment; impls that surface none are still substrate-conformant.

| Capability | Composes from |
|---|---|
| **Path tap** — observe dispatches arriving at a specific path | Dispatch event filtered by target_uri |
| **Content stream** — observe entities as they enter the store | Content event, optionally filtered by type prefix |
| **Entity reader** — decode and present any entity at a hash or path | Content store read + binding lookup (no event hooks required) |
| **Path enumerator** — list bindings under a prefix | Location index prefix scan (no event hooks required) |
| **Wire recorder** — capture full bytes of network frames | Wire event with full payload retention |

These are the abstract primitives. Impls expose them through whatever language idiom is natural (hooks, callbacks, middleware, channels) — the guide does not bind the API shape, only the capability set.

**Entity reader has two distinct sub-modes with different failure taxonomies:**

| Mode | Failure modes |
|---|---|
| **Path-mode** — "what entity is bound at this URI?" | (a) URI unbound, (b) URI bound to hash gone from store, (c) URI bound but tree reverted before fetch |
| **Hash-mode** — "what is this hash?" | Hash not in store |

Operators inspecting "what's at this path" vs "what's at this hash" need to distinguish *why* a read returned nothing. Folding both into one "entity reader" capability hides operationally meaningful information; impls should surface the failure shape distinctly.

### §2.3 Composed inspection capabilities

The primitives above compose into higher-order capabilities that operators and applications often want directly:

- **Chain trace** — given a `chain_id`, walk all related continuation entities + chain-error markers + dispatch events. Composition: dispatch event filter + content store read on chain entities. *Honesty depends on §9 #8 extension invariants* — without declared completion contracts, chain trace cannot distinguish silent failure from "chain never started."
- **Binding stream** — observe state mutations live. Composition: binding event filtered by path prefix.
- **Subscription tracer** — observe a subscription's emit-side firings + delivery-side outcomes. Composition: notification-emission + notification-delivery events filtered by subscription_id.
- **Time-window replay** — reconstruct what happened in a past window. Composition: persisted content stream + binding event log + replay against a query.
- **Cross-peer correlation** — same `request_id` observed on two peers' wire recorders. Composition: wire recorder × 2 + structural diff. *Skew sources are dual: within-peer async lag + cross-peer wall-clock drift; tooling must handle both.*
- **Audit trail** — every state change with attribution. Composition: content event + signature/capability lookup at each entity.
- **Cap walk** — for any capability, traverse its grant chain backward to root. Composition: content store reads following the `refs.parent` chain.

None of these require new substrate primitives. They are projections.

### §2.4 From event classes to capabilities — projection table

Two outcomes worth surfacing explicitly: not every event class is needed for every capability, and two of the five derived capabilities (entity reader, path enumerator) need no event hooks at all — they read static substrate state.

| Capability (§2.2 or §2.3) | Event classes required |
|---|---|
| Path tap | Dispatch |
| Content stream | Content |
| Entity reader | (none — direct substrate read) |
| Path enumerator | (none — direct substrate read) |
| Wire recorder | Wire |
| Chain trace | Continuation + Error + Dispatch (optional: Content) |
| Binding stream | Binding |
| Subscription tracer | Notification-emission + Notification-delivery (+ optionally Content) |
| Cross-peer correlation | Wire (× N peers) |
| Audit trail | Content + (entity-level signature/capability) |
| Cap walk | (none — direct substrate read) |

An impl prioritizing minimum-cost inspectability can ship entity reader + path enumerator + cap walk *without any hook plumbing at all* — pure read-side substrate exposure. Add content + dispatch event hooks for content stream + path tap (the most-used live-observation primitives). Add the remaining event classes as workload demands.

---

## §3 The cost spectrum

Inspectability is not free. The eight event classes in §2.1 fire at varying rates depending on workload; impls and operators must choose **when** and **what** to observe. Three regimes worth naming:

### §3.1 Always-on / unfiltered

Every event of every class is captured and durably persisted. Suitable for: tests, single-peer development, forensic post-mortem. **Cost: high.** Content event alone can fire thousands of times per second on a busy peer; persisting every fact is N× substrate write cost. **Hot-path latency cost is distinct from storage volume cost** — a hook that synchronously serializes to CBOR can dominate the put cost in tight loops; the latency burden is on the emitting code path, not the storage subsystem.

### §3.2 Sampling / filtered

Selectively capture: every Nth event, events matching a type/path predicate, events within a time window, events bound to a specific `request_id` or `chain_id`. Suitable for: production observability, debug-mode peers with bounded ring buffers. **Cost: tunable.** Practical default — type-filter to `system/protocol/error/*` and `system/chain-error/*` always-on; full content stream behind an explicit opt-in.

### §3.3 On-demand / scoped

No events captured by default; an operator (or another capability-bearer) installs a tap or starts a recorder for a bounded interval, then tears it down. Suitable for: live debugging, "show me what just happened" investigations. **Cost: low except during the inspection window.** Matches the pattern most operators reach for first.

Recommended impl posture: support all three regimes, default to §3.3. Always-on §3.1 is for tests; §3.2 is for production observability with policy; §3.3 is the cheapest answer to "I need to see what's happening right now."

### §3.4 Recursion: when inspection produces more entities

The cost analysis above assumes inspection events do *not* themselves create new entities. If they do (the entity-native inspection pattern; see §4), every recorded event spawns another content event, which spawns another, etc. Without care this is unbounded.

The standard mitigation: the impl-side hook receives event facts but does **not** write them back into the local store. Recording is a parallel ring buffer / log / channel that the operator reads from outside the entity-system loop. This is the **out-of-band** pattern (§4.1). Entity-native inspection requires a carve-out (§3.4.1) or a separate aggregation peer (§5).

#### §3.4.1 Carve-out schemes — name the two stable patterns

When entity-native inspection is chosen, impls need to break the recursion loop. Two stable patterns:

1. **Type-prefix excludelist.** Hook checks `entity.type.starts_with("system/inspect/")` and skips. Cheap; requires all inspect-emitting consumers to use a well-known type-prefix.
2. **Channel-side filter.** Hook always fires; sink discards entities whose source is itself. Requires sink ↔ source identity tracking.

Workbench-go's `InstallTap` exhibits a subtler form: tap stores a debug copy at `system/runtime/tap/<path>/<seq>` which fires the content hook — but the content hook is type-filtered to chain-relevant types in practice, so recursion is bounded by filter. If an operator installs both a tap and an unfiltered content stream, the cascade hits.

**Cross-sink amplification:** even with per-sink mitigation, operators stacking inspect sinks (e.g., path tap + unfiltered content stream) can re-introduce cascade because each sink's writes trigger the other sink's hooks. Impls should document which mitigation scheme they apply; operators stacking inspect sinks should be aware of cross-sink amplification.

**Cross-peer propagation of inspect entities — convention via §9 #7 declarations (v1.2).** The type-prefix family `system/runtime/**` (covering `system/runtime/tap/*`, `system/runtime/chain-errors/*`, and adjacent diagnostic paths) is declared **local-namespace** per §9 #7 by every extension that emits markers into it (canonical declaration: EXTENSION-CONTINUATION §6.5; inherited by EXTENSION-SUBSCRIPTION §7.4, EXTENSION-INBOX §11, EXTENSION-REVISION §12). The convention is: **inspect entities live where they were bound; cross-peer observation requires explicit operator-grant on `system/inspect/*` (per §8), not implicit subscription propagation.** Subscription handlers SHOULD refuse subscriptions on `system/runtime/**` and `system/continuation/**` unless the caller's scope explicitly enumerates a narrower path with operator-class authority. Substrate enforcement (subscription engine carving out these prefixes structurally) is reserved for v1.3+ if convention proves insufficient.

---

## §4 The fundamental design choice — out-of-band vs entity-native

Every impl that surfaces inspection must answer one question: *where does the observation data live?* Two stable answers exist; pick deliberately.

### §4.1 Out-of-band (recommended default)

The impl's hooks fire and deliver event facts to a sink that is **outside the entity store**:
- In-memory ring buffer
- Log file
- Streaming channel to an external observer (test process, shell session, dashboard)
- Standard logging infrastructure (syslog, OTLP, structured logs)

Properties:
- **Cheap.** No new entities created; no impact on store growth or content hashing.
- **Recursion-safe.** Observation does not produce inputs to itself; no carve-out scheme needed; cascade is structurally impossible.
- **Local.** Observation data is not propagated cross-peer through the protocol; another peer cannot ask for "what your inspect log says happened yesterday."
- **Lost on restart.** Ring buffers vanish; logs may rotate; recordings are operator-managed artifacts.
- **Not capability-controlled at the substrate level.** Whoever has direct access to the impl process can read the buffer.

Out-of-band is what `WithNamedContentHook` + log streams + in-process taps deliver. It maps onto the way mature systems (Git GC logs, IPFS event logs, Nix evaluation traces) handle introspection: parallel to the substrate, not inside it.

### §4.2 Entity-native (heavier pattern, occasionally needed)

The impl's hooks fire and the observation event is itself stored as an entity in the content store (`system/inspect/event/v1` or similar), bound at a known path (`system/runtime/observations/<class>/<seq>`).

Properties:
- **Inspectable through the same query surface as everything else.** No special API needed — `find_under("system/runtime/observations/")` enumerates.
- **Capability-controlled.** Cap-scope on the inspect path family limits who can read.
- **Durable.** Survives restart along with the rest of the store.
- **Cross-peer propagatable.** Another peer can subscribe to `system/runtime/observations/` if granted; audit-trail use cases benefit.
- **Recursive cost.** Every observation creates an entity, which triggers a content event, which (without §3.4.1 carve-out) creates an observation, etc.
- **Storage growth.** Inspection generates many small entities; needs GC policy (per `GUIDE-GC.md`).

Entity-native is the heavier pattern. It's the right choice when:
- Inspection data must be cross-peer auditable.
- Capability-controlled access matters.
- Durability across restart is required.
- The inspect data is itself part of the application's data model (e.g., audit log is a business artifact).

For most debugging and operator use cases, out-of-band §4.1 is the correct default. Entity-native §4.2 is a deliberate choice with a specific use case.

### §4.3 Hybrid

Both patterns can coexist. A peer might:
- Stream content events out-of-band to a ring buffer for live debugging (§4.1).
- Emit `system/audit/event` entities for capability-controlled, durable audit (§4.2).

The two are orthogonal and serve different needs.

---

## §5 The aggregation pattern — when entity-native inspection needs a second peer

The recursion problem in §3.4 has a structural solution: **offload observation aggregation to a separate peer.**

```
                    primary peer (where work happens)
                          │
                          │ wire events (out-of-band, low cost)
                          ▼
                    aggregator peer (records observations as entities)
                          │
                          │ query, subscribe, audit (standard protocol)
                          ▼
                    operator / dashboard / audit consumer
```

The aggregator peer ingests events from the primary as out-of-band wire data (a recording stream, an extension-defined `system/inspect/feed` subscription, etc.), then materializes them as entities in *its own* content store. The primary peer's hooks never fire on its own observation entities because those entities don't live on the primary.

Properties of the aggregation pattern:
- Primary peer pays only the out-of-band hook cost; storage growth and recursion cost moves to the aggregator.
- Operators query the aggregator using standard entity-protocol queries (`find_under`, `subscribe`, etc.) — no special API.
- Cap-controlled access at the aggregator boundary.
- Durable, cross-restart, cross-peer auditable — full entity-native ergonomics on the inspection side without the primary peer paying recursion cost.

This is the **"separate concerns by peer"** instance of a recurring pattern: when an aspect of system behavior wants entity-native semantics but the work happens elsewhere, push it to a sibling peer rather than entangling it with the work peer.

Trade-offs:
- More complex deployment (two peers minimum).
- Aggregator peer becomes a single point of inspection failure; replicate it like any other peer.
- Capability flow needs design: who can read the aggregator? Who can subscribe?

**Note: the aggregator pattern is currently impl-bilateral.** Primary and aggregator must agree on the feed shape (§11). Cross-impl aggregation (e.g., Go primary, Python aggregator) requires that feed shape to be pinned; until then, deploy primary and aggregator from the same impl.

The aggregation pattern is overkill for "I'm debugging on my laptop." It is appropriate for "production deployment needs durable audit." Pick based on use case.

---

## §6 Activation, filtering, and the policy surface

Independent of out-of-band vs entity-native (§4), an impl must answer six operator-facing policy questions. **These are knobs, not spec values** — the impl exposes each as configurable with a conservative default; the guide does not pick specific numeric values.

| Knob | Recommended default | Override surface |
|---|---|---|
| **Hook activation** | Runtime install via cap-gated API; not active by default in production builds | Operator command / handler call |
| **Filter predicate** | Empty = match all | Predicate language: type-prefix, path-prefix, time-window, request_id, chain_id |
| **Sink** | In-memory ring buffer, bounded capacity | Operator-redirectable to log file, OTLP, aggregator peer |
| **Cap requirement** | Cap-scoped install in production | Local-debug builds may grant unconditionally |
| **Sample rate** | All events when active | Every Nth / rate-limited / first N then drop |
| **Overflow behavior** | Drop newest + observable counter (operator sees "N events lost") | Back-pressure available as opt-in; spill-to-disk as opt-in |
| **Event class emission** | Required event classes always emittable; optional classes opt-in per substrate-build flag | Substrate-build-time flag (e.g., observability-enabled-build) |

The last row addresses a multi-tier substrate availability question: in some impls, content events come from one storage implementation tier (e.g., Python's `NotifyingContentStore`) while another tier (bare `ContentStore`) doesn't surface them. Impls should make this distinction explicit so cross-impl tooling can probe for capability presence rather than assume universal availability.

---

## §7 The two audiences

Inspection serves two audiences with different ergonomic needs. Both compose from the same primitives (§2.2); they differ in surface and framing.

### §7.1 The implementer-debugger

A developer or operator debugging a problem in an entity-system component. Wants:
- Live observation of dispatches at a specific path.
- Decoded view of entities at suspect locations.
- Cross-impl wire-shape comparison.
- Replay of a captured failure.

Surfaces: shell commands, test-time programmatic API, GUI panels for live observation.

This audience operates *within* the entity-system frame but switches to *impl-side debugging* when the bug is in opaque handler code (a Rust extension's internal logic, a Python handler body, etc.). For those, classical debuggers (gdb, lldb, pdb, language IDEs) are the right tool.

**The boundary discipline (corrected from v1.0):** inspect primitives *surface the symptom* at the entity-system boundary — they make what's happening visible faster than reading source does. They do not *fix* anything. Fixing typically still requires handler-body work, where classical debuggers are the appropriate tool. Both are necessary; **inspect doesn't replace classical debuggers, it shortens the path to the right handler.** v1.0's wording ("classical debuggers take over inside opaque handler bodies") understated the role of source-reading and handler-body work; this revision frames inspect as *symptom localization*, not *root-cause analysis*.

### §7.2 The entity-native user

An application author or end user reasoning natively in the entity-system frame. Wants:
- Visualizations of continuation chains as graphs.
- Trees of binding namespaces.
- Live animation of entities flowing through subscriptions.
- "What capabilities do I currently hold?" answered as a grant-chain walk.

Surfaces: GUI panels, IDE / editor integrations, application UX, Godot-style visualizations.

**Two distinct rendering paradigms with different policy-spectrum fits (v1.2.1):**
- **DOM-based-WASM apps** (e.g., `egui-entity-core-rust` — repo name is historical, the app is DOM-WASM per its CLAUDE.md banner since the eframe removal): subscription-driven dirty-flag pipeline; DOM rebuild loop saturates at high event rates. Natural fit: §3.3 on-demand default, §3.2 sampling for high-rate streams, §3.1 always-on NEVER as UX-facing option (debug-build opt-in only). Window-spawn = tap install, window-close = teardown maps cleanly.
- **Immediate-mode apps** (e.g., Godot when they engage): per-frame query is the rendering pattern anyway, so §3.1 always-on at low cost is viable. Live animation of entity flows fits naturally.

Both valid; different impls pick the regime that matches their rendering model.

This audience is debugging *less* than they are *exploring* or *operating*. The same primitives serve both audiences; the projection layer differs.

Implementers building application UX should expect both modes to coexist: an app that visualizes its own entity flows for end users *and* surfaces lower-level inspect primitives for developers tracing what went wrong.

**egui-entity-core-rust + godot-entity-core-rust invited to review this section + §10 forward direction; v1.2 will absorb their L3-application perspective.**

---

## §8 Cross-impl coordination — v1.0 reference protocol shape

If inspection is per-impl-ergonomic but the abstract event classes (§2.1) and derived capabilities (§2.2) are shared, cross-impl tooling becomes possible. A shell-side inspection command can talk to any conformant impl through a common protocol-level inspect interface.

**The minimum cross-impl-portable surface — v1.0 reference shape:**

| Capability | Protocol shape |
|---|---|
| Path tap install | `EXECUTE` on `system/inspect/tap` with `{path, ops, sink_uri}`; returns tap_id |
| Path tap read | `QUERY` on `system/inspect/tap/<id>/captures` returns capture entities |
| Content stream subscribe | `SUBSCRIBE` on `system/inspect/content` with type-prefix filter |
| Entity read | `QUERY` on `system/inspect/entity/<path-or-hash>` returns decoded entity |
| Path enumerate | `QUERY` on `system/inspect/under/<prefix>` returns binding list |
| Wire recorder install | `EXECUTE` on `system/inspect/recorder` with `{filter, sink}`; returns recorder_id |

**Status:** non-normative as guide-level text, but stable enough that workbench-go ships the first impl against it. **A second impl that diverges from this shape owes a divergence rationale** — not a spec violation, but an explicit "we did it differently because X" so cross-impl shell tooling knows what adapters to write. This is the convening-point move: not a spec mandate, but a stable surface a second impl can read off the reference impl.

Cap-scope `system/inspect/*` as appropriate for the deployment.

Cross-impl coordination implications:
- Path schemas (e.g., `system/inspect/tap/`) should be common across impls so shell tooling doesn't need per-impl adapters.
- Sink URI semantics should agree (where does captured data flow?).
- **Filter predicate grammar should agree** (how do operators say "only type=system/protocol/error"? glob? URI prefix? regex?). See §11 open question.

---

## §9 Extension-author checklist

Every extension contributes to the observable surface. To make cross-impl inspect tooling portable, extension authors should **declare** their contribution explicitly. The recommended checklist for an extension spec (mirror of `GUIDE-GC.md §10`):

**Vocabulary (v1.2) — §9 #7 cross-peer observability convention.** Four terms used in extension declarations:

| Term | Meaning |
|---|---|
| **local-namespace** | Path family is per-peer working state. MUST NOT propagate via subscriptions or revision sync. Cross-peer access requires an explicit operator-grant on `system/inspect/*` or equivalent. |
| **convergent** | Cross-peer copies are expected to converge to the same content via the extension's own convergence model (e.g., subscription mirror recipe, revision DAG). Observers see a consistent eventually-converged view. |
| **chain-bundled** | Propagates cross-peer only as part of an envelope's `included` map (V7 §3.1) when chain authority requires it. Not subscription-eligible. |
| **transport-only** | Carried on the wire (e.g., for delivery construction) but not stored cross-peer. |

**Declaration discipline (v1.2 — promoted from "recommended" to required for L3 application UX safety claims).** Extensions claiming L3-application-UX safety MUST declare §9 #4 privacy classifications for every entity type they store and §9 #7 cross-peer observability for every path family they bind. Conservative-default for inspect tooling: undeclared entity types are treated as `sensitive` (over-redact rather than over-expose). Extensions that don't declare can still be wire-conformant; they just can't claim L3-app-safe.

1. **Entity types this extension stores.** Enumerate every `system/<extension>/...` type the extension produces, with a one-line description of when it's emitted.
2. **Path patterns this extension binds.** Enumerate the path prefixes (`system/inbox/...`, `system/continuation/...`, etc.) the extension writes. Note which paths are local-namespace vs cross-peer-published.
3. **Chain-error vocabulary.** If the extension participates in continuations or chains, enumerate the error codes / shapes that can appear in chain-error markers attributable to this extension.
4. **Privacy classification.** For each entity type: public / capability-controlled / sensitive. This guides production observability sampling.
5. **Rate characterization.** For each entity type: expected emission frequency under typical / peak workloads. Guides production sampling rate defaults.
6. **Replay safety.** Does replaying a captured dispatch to this extension produce idempotent effects, side effects, or destructive effects? Guides replay tooling.
7. **Cross-peer observability.** Does this extension's state propagate cross-peer (e.g., via subscriptions, attestations)? If yes, observers on remote peers may see partial views; document the convergence model.
8. **Chain participation invariants.** If the extension participates in continuations or chains, declare: (a) the path family where chain state persists, (b) the path family where chain-error markers persist, (c) the **completion contract** — "if a chain succeeds, an entity appears at path X; if it fails, a marker appears at path Y." So **the absence of both is detectable as anomalous.** Without these declarations, inspection tooling cannot distinguish silent failure from "the chain never ran" — `chain_trace` is best-effort guesswork. With them, the trace can detect silent failure as "chain neither succeeded nor recorded a marker." Promoted from "nice to have" to required for inspection-tooling honesty after Python's I4 finding implementing `chain_trace` against an unbounded substrate.

The eight items are extension-spec-author owed. They don't need normative MUST language to be useful — an extension spec that includes a §X "Observable Surface" section listing these makes cross-impl inspect tooling significantly easier to write.

---

## §10 What this guide does NOT cover

To keep scope clear:

- **Impl-side tooling patterns.** Workbench-go's `inspect/`, core-py's `entity_core.diagnostic`, Godot's visualization layer — these belong in `GUIDE-DEBUGGABILITY.md` (forthcoming, impl-team-owned cookbook). This guide names what's observable; that one names what works to surface it.
- **Specific language APIs.** The five derived capabilities (§2.2) are abstract; each impl projects them through its own idiom. This guide does not say "use Go peer builder hooks" or "use Rust trait methods."
- **OpenTelemetry / structured logging integration.** Out-of-band sinks (§4.1) can integrate with standard observability stacks; the integration is impl-side wiring, not substrate concern.
- **Production observability dashboards.** Live dashboards built on inspection events are an ops design question; this guide enumerates what they can show, not how to build them.
- **Classical-debugger handoff.** When debugging opaque handler bodies (§7.1), classical debuggers (gdb, lldb, pdb, language IDEs) are the appropriate tool; the inspect facility shortens the path to the right handler, doesn't replace the debugger.
- **Replay, revert, edit-and-replay (L2 / L3 / L4 forward direction).** Beyond observation (L1, this guide), the entity model supports re-executing captured flows (L2 replay), rolling back to past binding states (L3 revert, partially supported via revision), and modifying captured flows then re-executing from a chosen step (L4 edit-and-replay). These are composed capabilities building on L1 primitives + the wire recorder (§2.2 #5) + a continuation-resume hook. **The substrate guarantees these are possible; the surface plumbing is future work.** Classical systems cannot have L2-L4 because state isn't content-addressed, mutations aren't reified, operations aren't inspectable entities — these layers are where the entity-system value proposition diverges most sharply from classical systems. Designed in a future exploration (`EXPLORATION-REPLAY-REVERT-EDIT-AND-REPLAY.md`, to follow); named here as forward direction.
- **Stepping debugger primitive.** Pause-and-step at handler boundaries needs a substrate-level "wait for resume" hook in the dispatcher; that's a continuation-engine concern, not inspect. Belongs in future `EXPLORATION-STEPPING-DEBUGGER.md`.

---

## §11 Open questions

Multi-impl-quorum items — answer when a second (or third) impl needs the answer:

- **Should there be a `system/inspect/*` URI domain in the spec?** §8 ships the v1.0 reference shape; whether to bind it normatively or leave it as a guide-recommended pattern remains open. Likely answer crystallizes when a third impl ships handlers against it.
- **Should `system/audit/*` (the entity-native pattern) be standardized?** Separate question from inspect — audit is a durable persistent surface; might warrant its own extension if multiple impls converge on it.
- **Filter pattern grammar convergence + peer-relative semantics.** §8's cross-impl portable taps need a common filter syntax (glob? URI prefix? regex? mixed?). Per-impl grammar today; pinning requires multi-impl agreement. **Signal:** for binding-event hook registration, two impls have aligned on a small set of grammar primitives — `"*"` (or empty) matches everything, `"prefix/*"` matches anything under `prefix/` (and the bare `prefix`), otherwise exact match. The pattern is checked at the cascade-walk site so non-matching hooks are never invoked. **The primitives converge; the coordinate-space semantics for the peer-relative form do NOT, by deliberate design.** Two valid interpretations have surfaced for a pattern like `"foo/*"`:
   - **Local-PID-scoped:** canonicalize at registration to `/{localPID}/foo/*` — the hook fires only on events under the registering peer's own namespace. Fits surfaces where the hook lives in the write path and naturally watches its own peer's writes.
   - **Namespace-agnostic:** match the suffix under any peer namespace — the hook fires on events under `/localPID/foo/...`, `/remoteA/foo/...`, `/remoteB/foo/...`. Fits surfaces whose job is cross-peer observation, where the local tree holds other peers' namespaces from sync-mirror / revision-cache / cross-peer entity flow, and where local-PID-scoping silently loses every event from those remote namespaces.

  Both interpretations are valid against the substrate; they answer different questions. Surfaces with absolute patterns (`"/PID/foo/*"` or explicit any-peer `"/*/foo/*"`) compose cleanly under both interpretations. The substantive cross-impl question is therefore **not "what glob grammar"** but **"what does a peer-relative pattern MEAN against a multi-peer tree"** — a §11 sub-question worth pinning when a ruling pass happens. The spec does not pick today; both interpretations are documented so a third impl picks with eyes open. Other hook classes (content / dispatch / wire) discriminate on different fields (type prefix / target URI / direction+root) where this grammar doesn't directly apply — separate decision before grammar matters there.
- **Clock discipline contract for cross-peer correlation.** Event facts include `timestamp` without specifying monotonic vs wall-clock vs source-attributed. §2.3 cross-peer correlation depends on the choice. The guide doesn't have to pick one; it needs to name the choice exists and is load-bearing.
- **Cross-impl dispatch-event site agreement.** §2.1 #3 names a recommended baseline; if impls drift on site, cross-impl telemetry is non-comparable. Pinning the baseline requires multi-impl convergence. **Convergence signal:** two impls have aligned on the dispatcher-side wrapper around the handler body — both fire entry + exit per dispatch with `response_hash` populated on exit. A third impl has no dispatch-hook surface today; when it adds one, the surfaced baseline above is the working reference. Not promoted to MUST until a third independent surface confirms.
- **Within-peer sync vs async observation semantics.** §2.1 names the choice; whether the cross-impl portable surface (§8) returns sync or async observation events is an open coordination question.
- **Entity reader path/hash failure taxonomy convergence.** §2.2 names the two sub-modes; impls today may surface them differently. Worth pinning when cross-impl tooling consumes the reader.
- **Production observability spec position.** If multiple impls converge on sampling + OTLP export, does the substrate gain a normative observability section? Unclear; revisit when ops use cases land.

**Deferred (no action until trigger):**

- **Aggregation peer protocol shape.** §5 sketches the pattern; specific feed/subscription shape impl-defined today. Defer until a second impl needs to be the aggregator for a different-impl primary.
- **Cross-impl wire-recorder portability.** Wire frames are CBOR; recording is local. Whether the recording format becomes cross-impl-portable so a Go peer's recording can be replayed against a Rust peer is open. Defer until two impls have wire recorders.

---

## §12 Cross-references

- Exploration backing this guide: the "inspectability as first-class" exploration.
- v1.0 → v1.1 review synthesis: the synthesis of the cross-impl inspectability reviews.
- Workbench-go review (L2): the workbench-go feedback on this guide.
- Core-py review (L1): the core-py Python-side review of this guide.
- Pattern precedent: `GUIDE-GC.md` (substrate-property guide + extension-author checklist)
- Companion (forthcoming, impl-team-owned): `GUIDE-DEBUGGABILITY.md`
- Reference impls: `entity-workbench-go/inspect/`; `entity-core-py/packages/entity-core/src/entity_core/diagnostic/`
- Workbench-go direction memo: `entity-workbench-go/docs/architecture/DIAGNOSTIC-DIRECTION.md`
- Underlying substrate hooks (Go reference): `entity-core-go/core/peer/builder.go::WithNamedContentHook`, `WithDebugLog`; `entity-core-go/core/store/notifying_content.go::AddNamedContentHook`
- Underlying substrate hooks (Python reference): `packages/entity-core/src/entity_core/storage/content.py::NotifyingContentStore.add_content_hook`; `packages/entity-core/src/entity_core/emit/pathway.py::EmitPathway.subscribe`
- Related guides: `GUIDE-EXTENSION-DEVELOPMENT.md` (where the §9 checklist gets pointed at), `GUIDE-CROSS-PEER-MESSAGING.md` (wire-event semantics), `GUIDE-OPERATIONAL-STATE.md` (production policy)
- Spec touchpoints: `EXTENSION-CONTINUATION.md` (chain events), `EXTENSION-INBOX.md` (delivery events), `EXTENSION-SUBSCRIPTION.md` (notification emit + deliver events), `V7 §3` (envelope, content-hash)
