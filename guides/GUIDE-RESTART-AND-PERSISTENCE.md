# Guide: Restart and Persistence

**Status**: Active
**Audience:** Implementers building peers with durable storage; extension authors whose handlers maintain in-memory state.
**Related spec material:**
- ENTITY-CORE-PROTOCOL.md §1.1 (emit primitive — no-op suppression)
- SYSTEM-COMPOSITION.md §6.7 (persistence and implementation freedom)
- SYSTEM-COMPOSITION.md §7.1 (the tree IS the monitoring surface)
- EXTENSION-ATTESTATION.md §5.7 (persistence and rebuild)
- EXTENSION-COMPUTE.md §7.1 (dependency index rebuild)
- EXTENSION-SUBSCRIPTION.md §1 (subscription entity is source of truth, internal index is cache)

---

## 1. Why this guide exists

The protocol takes a simple position: **the tree is the state**. Anything that should survive a stop/restart belongs in the tree. In-memory state in a handler is acceptable as a performance optimization (an index, a cache, a routing structure) provided it can be rebuilt from tree state.

This guide gathers the patterns that follow from that position. It is not normative — implementations are free to satisfy the existing spec requirements (SYSTEM-COMPOSITION §6.7.4 mandates that derived state rebuild after hydration; EXTENSION-ATTESTATION and EXTENSION-COMPUTE already require restart-consistency for their indexes) however they like. The patterns here are observations from impl work, surfaced so the next implementer doesn't rediscover them by hitting the same gaps.

---

## 2. Three kinds of canonical peer-internal entities

A peer creates a small handful of entities about itself: handler entities, type definitions, transport bindings, the peer's own identity entity, self-issued capability grants, signatures over those grants, the peer's runtime status. These fall into three rough patterns based on what determines their content.

The patterns are descriptive vocabulary, not normative architecture. They help reason about what handling each entity needs.

### 2.1 Class D — declarative-of-current-code

Content is fully determined by the current code and the peer's identity material. Both inputs are available at every start; the peer can compute the full canonical content.

Examples: type definitions, handler entities, handler interface entities, local transport bindings, the peer's identity entity (`system/peer/self`, deterministic from `{peer_id, public_key, key_type}`).

Handling pattern: idempotent re-creation at every start works fine. When the code is unchanged, the resulting content hash matches the existing tree binding and §1.1 no-op suppression eliminates the emit. When the code changes (a release with new handler manifest shape, new type field, new transport configuration), the new content hash differs and the upgrade cascade fires naturally — old binding overwritten, consumers notified.

Implementations may also choose "read existence and skip re-creation" if performance demands. Both produce equivalent observable behavior.

Class D content MUST be fully determined by current code state and identity material. A canonical peer-internal entity with a non-deterministic field (timestamp, nonce, runtime-derived data) is not Class D — see Class I or Class L.

### 2.2 Class I — install-once

Content includes data captured at first creation that isn't derivable from code or identity material. The install-event itself is the source of truth; the entity records that event.

Current example: self-issued capability grants (`system/capability/grants/*`) and their signatures. The cap-token type schema requires `created_at`, which is install-event data not derivable later.

Handling pattern: at restart, check existence at the canonical path before constructing the entity. If present, skip — the install-time value is canonical. Don't re-mint with a fresh timestamp; that produces drift in the content hash and churns path bindings.

If the existence check finds an entity at the canonical path with an unexpected content hash (operator intervention, sync arrival, impl drift), the diff is signal worth understanding. Surface as a diagnostic finding so the operator, test suite, or sync mechanism can investigate. Do NOT overwrite (would reintroduce the re-mint problem) and do NOT error (conflates legitimate intervention with startup failure).

Class I exists today as a class only because of a type-overload: `system/capability/token.created_at` is shared between runtime delegated tokens (where the timestamp matters for TTL) and peer-internal grants (where it doesn't). A future amendment splitting the type into `system/capability/static-grant` (no `created_at`) for peer-internal grants and `system/capability/runtime-token` for delegated tokens would migrate peer-internal grants from Class I to Class D, retiring this class.

### 2.3 Class L — live state

Content reflects current runtime state and is updated continuously through the emit pathway as state changes.

Current example: the peer's own runtime status (`system/peer/self/status` — see the spec amendment).

Handling pattern: write through the normal emit pathway when state changes. The "re-create at start" framing doesn't apply — these entities are written when their state changes, including during the start sequence as the peer transitions through phases (`starting` → `ready`).

---

## 3. Handler in-memory shadows must be rebuildable

Many extensions maintain in-memory state derived from tree entities for performance: subscription routing indexes, attestation graphs, history recorder caches, revision auto-version state, tree root-tracker state, compute dependency indexes.

The pattern: each such handler's tree state is the source of truth. The in-memory structure is a cache over the tree state. At start, the handler reads its tree entries and rebuilds its in-memory structure.

Each handler's spec describes what state it keeps and where in the tree the source lives:

| Handler | Tree source | In-memory structure | Impl status |
|---|---|---|---|
| Subscription | `system/subscription/*` | Routing index (subscriptions, pathIndex) | Go: shipped · Rust: shipped · Python: shipped |
| Attestation | `system/attestation/*` | Attestation graph (attesting/attested/kind/supersedes) | Go: shipped · Rust: pending · Python: pending |
| History (recorder) | `system/history/config/*` | Recorder config cache | Go: shipped · Rust: shipped · Python: shipped |
| Revision (auto-version) | `system/revision/head/*`, `system/revision/config` | Versioning state | Go: shipped · Rust: shipped · Python: shipped |
| Tree (root-tracker) | `system/tree/root-config` | Tracked roots | Go: shipped · Rust: shipped · Python: shipped |
| Compute (engine) | `system/compute/processes/*` | Dependency index | Go: shipped · Rust: pending · Python: pending |
| Query | content store + tree (full scan) | Type / reverse-hash / path-link indexes | Go: shipped · Rust: shipped · Python: shipped |
| Local-files (watcher) | `system/config/local-files/*` | Active watcher set keyed by root name | Go: **pending** (P2 on `CORE-GO-IMPL-WORK §1.5`; workbench glue-codes the reload at the SDK layer today) · Rust: pending · Python: pending |

*The impl-status column comes from the production-readiness amendments (A.6). Initial cells filled from current observations; impl teams update as they ship. The `local/files` row was added in the same pass (PERSISTENCE-FEEDBACK Finding 4 — localfiles handler has no `Load()` method).*

Existing spec material requires this rebuild:
- SYSTEM-COMPOSITION §6.7.4: "Query indexes and other derived state rebuild after hydration completes."
- EXTENSION-ATTESTATION §5.7: "Implementations MUST guarantee that index lookups are consistent with current tree state across process restarts."
- EXTENSION-COMPUTE §7.1: "On peer restart, the compute handler MUST rebuild the dependency index by scanning for installed subgraphs."

Two handlers with state that intentionally does NOT rebuild from the tree:

- **Identity signing-key registry.** Per EXTENSION-IDENTITY §7 default custody, signing keys live in ceremony tools, not the daemon. Lazy refill from external custody is the contract.
- **Quorum signer-set cache.** Pure cache, lazy refill on first access. No eager rebuild needed.

When designing an extension that holds in-memory state, the question to ask: is the state derivable from tree entries the extension manages? If yes — declare what tree paths feed the rebuild and implement the rebuild. If no — the state isn't really durable and the extension is correctly stateless across restart, OR the state should be moved into the tree.

---

## 4. Don't redo work that's already in the tree

The cap-grant rebinding bug that triggered this whole thread of work is a specific instance of a general pattern. At start, before re-creating an entity at a canonical path, check whether the tree already has it. If the entity is present and the install-time value is canonical (Class I), leave it alone. Re-creating it with a fresh timestamp churns content hashes and path bindings for no reason.

For Class D entities, idempotent re-creation is fine — §1.1 suppresses no-ops, and code changes flow through the upgrade cascade naturally.

The point: implementations encounter this distinction concretely. The right behavior follows from "tree is the state."

---

## 5. Memory-only peers operate under a different model

A memory-only peer (no durable storage) loses its tree state on stop. When a new process starts, it is a fresh peer regardless of identity. The §6.7.4 invariant doesn't apply — there is no durable state to be equivalent to.

This is correct and intentional. The protocol does not require persistence; deployments that want fresh-start-every-start (CI, ephemeral peers, workbench testing) operate this way by choice. None of the patterns in this guide apply to memory-only peers; they are documented for peers with durable storage.

---

## 6. Known open issues — tracked

These are real issues that surfaced during impl-team analysis but are NOT solved by the patterns in this guide or by the spec amendments in PROPOSAL-RESTART-EQUIVALENCE.md. Each one is a concrete piece of follow-up work in a specific extension or area. Tracking them here so they are not lost; each will become its own focused proposal/amendment when picked up.

| # | Issue | Severity | Where the fix lives | Status |
|---|---|---|---|---|
| 1 | Compute pipelines stuck mid-flight after crash-restart | Correctness gap for crash recovery | EXTENSION-COMPUTE rebuild logic | Open — needs proposal |
| 2 | Zombie subscriptions from dead subscribers | Operational accumulation | EXTENSION-SUBSCRIPTION lifecycle | Open — needs proposal |
| 3 | Content store grows without bound | Operational storage | New GC mechanism | Open — needs design pass |
| 4 | Subscriber reconnection delays during peer down-window | Operating as designed; interacts with #2 | EXTENSION-NETWORK / inbox replay | Documented; not a bug |
| 5 | History accumulates without `max_depth` | Operator config concern | Operator-side configuration | Documented; not a bug |

Detailed write-ups below. Each section names the issue, what triggers it, current workarounds (if any), and what a real fix would look like.

### 6.1 Compute pipelines stuck mid-flight after crash-restart

**Severity:** Correctness gap. Affects any compute pipeline that crashes during evaluation.

**Trigger:** Input `X = 5` is set; computation starts; peer dies before writing result `Y`. On restart: `X` is still `5`, no `Y`. The compute engine's dependency-index rebuild picks up the active subgraph and wires reactive triggers correctly — but the engine fires on new tree changes, not on existing state. Nothing triggers re-fire of the stuck computation. The pipeline sits with `X` set but no `Y` until something else writes to `X` again.

**Current workarounds:** Subgraph designer detects missing outputs in their expression and re-fires; or an operator manually touches the input. Neither is adequate for production.

**What a real fix looks like:** The compute engine's rebuild step (`RebuildDependencyIndex` or equivalent in each impl) does a "for each active subgraph, check whether declared outputs exist for current inputs, and re-fire if not" pass during the rebuild phase. This catches all stuck pipelines at startup and brings them to settled state.

**Where:** Belongs in a focused amendment to EXTENSION-COMPUTE (probably §7.1 alongside the existing dependency-index rebuild text).

### 6.2 Zombie subscriptions from dead subscribers

**Severity:** Operational accumulation. Long-running peers accumulate stale subscriptions over time.

**Trigger:** A subscriber subscribes, then dies (process crash, network partition, deployment shutdown) without calling `unsubscribe`. The `system/subscription` entity stays bound. The engine loads it on every restart, attempts delivery on every matching tree change, fails (subscriber unreachable), keeps the subscription bound. Forever.

**Current state:** `system/subscription` entities have no expiration field. `SubscriptionLimitsData` exists for rate/count limits, not expiry. No "remove after N consecutive delivery failures" logic.

**What a real fix looks like:** TTL field on the subscription entity (subscriber renews periodically, expiry cleans up), or failure-count-based cleanup (after N delivery failures, the engine unbinds the subscription), or both. Each has trade-offs (TTL adds renewal traffic; failure-count needs careful definition of "failure" for transient vs permanent).

**Where:** Focused amendment to EXTENSION-SUBSCRIPTION. Worth pursuing soon — operational impact accumulates over time.

### 6.3 Content store grows without bound

**Severity:** Operational storage. Disk usage grows indefinitely.

**Trigger:** Every entity ever written stays in the durable content store. Path bindings are correctly removed when subscriptions cancel, subgraphs uninstall, attestations are revoked, etc. — but the entity bytes themselves stay in the content store. Old type definitions from prior code versions, revoked attestations, cancelled subscriptions, every historical state — all persist.

**Current state:** No GC mechanism exists.

**Severity nuance:** Not a correctness issue. Handlers look at bound paths, not raw content-store entries; orphaned content-store entries are invisible to normal operation. It's a disk-space issue. After days/weeks/months, the content store is mostly historical garbage.

**What a real fix looks like:** A GC pass that computes reachability from active sources (current path bindings + revision tracked roots + history retention windows + envelope reference closures + signatures over reachable entities) and removes unreachable content-store entries. The reachability computation is non-trivial — needs care around content-only entities (envelope ingestion, merge material, sub verification chains) that are referenced by hash but not bound at a path.

**Where:** Out of scope of restart equivalence entirely. Needs its own design pass — possibly an exploration first, then a proposal that touches the persistence consumer model in SYSTEM-COMPOSITION §6.7 plus a new GC extension or built-in.

### 6.4 Subscriber reconnection delays during peer down-window

**Severity:** Operating as designed. Documented for clarity, not a bug.

TCP (or QUIC) connections die when a peer shuts down. After restart, outbound subscription delivery requires reaching the subscriber. If the subscriber's transport address is registered at `system/peer/transport/{peer_id}` (per V7 §3.13), the peer attempts a fresh connection and delivery resumes when the subscriber is reachable. If the transport wasn't registered, deliveries fail until the subscriber reconnects to the local peer.

In workbench-style local-peer-to-local-peer usage, subscriber reconnection happens fast. In distributed deployments, "delivery resumes when subscriber comes back" is the contract; that's correct, but it interacts with #6.2 — from the local peer's perspective, a subscriber that's just slow to reconnect looks identical to a subscriber that's gone forever. Whatever cleanup mechanism #6.2 lands needs to distinguish the two (typically via a longer timeout or explicit subscriber heartbeat).

### 6.5 History accumulates without `max_depth`

**Severity:** Operator config concern. Not a bug.

If a history config (`system/history/config/{name}`) is set without `max_depth`, history transitions accumulate indefinitely. After days of activity that's a lot of entries.

**Mitigation:** Operators using EXTENSION-HISTORY at scale should set `max_depth` in their config. Documented in EXTENSION-HISTORY's own conformance material.

This interacts with #6.3 (content-store GC): even with `max_depth` pruning bound paths, the underlying entity bytes for old transitions remain in the content store until GC removes them. A complete solution requires both pruning and GC.

---

## 7. What to tell yourself when designing a handler

Three questions to ask:

**1. What state do I keep that needs to survive a restart?**

If durable: it belongs in the tree. Define the entity type, write it through the normal emit pathway when state changes. The handler reads the tree to rebuild any in-memory shadow at start.

If genuinely transient: don't persist anything. Be honest in the handler's spec that this state doesn't survive.

**2. What state do I create about myself at start?**

If Class D (deterministic from code+identity): re-create idempotently every start; §1.1 handles no-op and upgrade.

If Class I (has install-event content): check existence first; skip-if-present.

If something else: figure out which class fits. If none fit, the state probably belongs in the tree as runtime state (Class L) instead of as something the peer creates about itself.

**3. What can go wrong at restart that I should test?**

Cancelled-then-restarted: did the cancel actually remove the path binding? Or does my state come back zombie?

Mid-operation crash: is the partial state visible at restart? Does the next operation recover, or does it sit stuck?

Long-running accumulation: do my entities have an expiration / cleanup story? Or do they grow forever?

Dead client: if a client of my handler dies without cleaning up, do my entities stay bound forever? What's the cleanup path?

These questions don't have spec answers. They're handler-design questions each extension author has to answer for their extension.

---

## 8. Document History

- Initial. Lifted from PROPOSAL-RESTART-EQUIVALENCE.md §5 (which got bigger than belongs in a spec proposal). Adds §6 known open issues from impl-team analysis (compute mid-flight, subscription TTL, content-store GC, subscriber reconnection, history accumulation) tracked as a table with severity/location/status, and §7 design questions for extension authors. Not normative; complements the spec amendments in PROPOSAL-RESTART-EQUIVALENCE.md (RE-1 invariant, RE-2 self-status type).
- §6 reframed from "known gaps" prose to a tracked-issues table with per-issue severity, fix location, and status. Five issues now have explicit tracking entries so they don't get lost as separate work items: compute mid-flight recovery (open, EXTENSION-COMPUTE), subscription TTL/cleanup (open, EXTENSION-SUBSCRIPTION), content-store GC (open, needs design pass), subscriber reconnection (operating-as-designed, documented), history accumulation (operator config, documented). Each has its own write-up.
