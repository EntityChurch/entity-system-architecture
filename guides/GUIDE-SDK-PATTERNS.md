# Guide: SDK Patterns

**Status**: Active
**Source:** SDK-OPERATIONS.md L2 advisory patterns, EXPLORATION-SDK-ACCESS-MODEL.md

---

## What This Guide Covers

Patterns for building applications on the entity system SDK. These are advisory — the SDK operations spec (SDK-OPERATIONS.md) defines the normative interface. This guide describes how to use those operations effectively.

---

## 1. Scoped Handles

A scoped handle binds a peer, a tree prefix, and a capability context. All operations through the handle are relative to the prefix.

```
scope = peer.scope("app/browser/")

scope.get("settings/theme")
; → resolves to /{peer_id}/app/browser/settings/theme

scope.put("workspace/layout", layout_entity)
; → resolves to /{peer_id}/app/browser/workspace/layout

scope.watch("workspace/*", on_change)
; → watches /{peer_id}/app/browser/workspace/*

scope.execute("system/query", "find", {path_filter: "workspace/"})
; → query scoped to /{peer_id}/app/browser/workspace/
```

**Benefits:**

- **Prefix isolation.** Operations can't accidentally touch paths outside the scope.
- **Lifetime management.** Subscriptions created through the handle cancel when the handle is dropped.
- **Implicit peer binding.** The handle knows which peer it targets — simplifies multi-peer apps.
- **Capability context.** The handle carries the grant chain that authorizes operations.

**Path canonicalization.** Scoped handles accept peer-relative paths. The SDK canonicalizes internally: `relative_path` → `/{peer_id}/{prefix}/{relative_path}` per the absolute path model (V7 §1.4). The application works with short paths; the protocol sees absolute paths. See SDK-OPERATIONS.md §2.5.

The SDK **SHOULD** provide scoped handles. The SDK **MUST** also provide unscoped operations with absolute or peer-relative paths.

---

## 2. Entity-Backed Application State

Application state stored as typed entities in the tree. This is the universal pattern across entity-native applications — UI state, settings, selection, layout are all entities at known paths.

### Convention Paths

```
app/{app-id}/workspace/     UI state (windows, layout, selection)
app/{app-id}/settings/      Application configuration
```

The `{app-id}` scoping lets multiple applications coexist on one peer without colliding.

### Convention Types

| Concept | Type name |
|---------|-----------|
| Generic setting | `app/state/setting` |
| Selection state | `app/state/selection` |
| Window configuration | `app/state/window` |
| Layout | `app/state/layout` |

The `app/state/` prefix is language-neutral. Not `rust_workspace/settings`, not `go_workspace/state`.

### Usage

```
; Write a setting
peer.put("app/browser/settings/theme", "app/state/setting", {key: "theme", value: "dark"})

; Read it back
setting = peer.get("app/browser/settings/theme")

; Within a scoped handle:
scope = peer.scope("app/browser/")
scope.put("settings/theme", "app/state/setting", {key: "theme", value: "dark"})
```

### Why Entity-Backed State

- **Inspectable** — any connected peer can read it via standard tree operations.
- **Syncable** — entity sync (revision extension) works on it automatically.
- **Auditable** — history extension records every change.
- **Queryable** — query extension can find entities by type and field values.

This is preferred over in-memory or file-based state because the entity system's infrastructure (sync, history, query, subscription) works on it for free.

---

## 3. Type Rendering

> **Status: advisory app-side pattern.** Type rendering is intrinsically per-presentation-layer (CLI, GUI, web — all have different idioms). The three-level chain and `ValueKind` enum below are recommended *patterns* for app-side renderer libraries to converge on; they are not an SDK contract. SDKs MAY ship a renderer-registry helper, but apps SHOULD NOT depend on one being present. This section reads as conventions for cross-app interop on rendered output, not as required SDK surface area.

Applications displaying entities to users **SHOULD** implement the three-level degradation chain:

1. **Type-specific renderer.** A registered render function for the entity's type. Best experience.
2. **Structural renderer.** Reads the type definition at `system/type/{type_path}` and renders field-by-field using field metadata. Meaningful but generic.
3. **CBOR diagnostic.** Renders raw CBOR data. Always available as fallback.

### ValueKind

Renderer-neutral semantic tagging for formatted values:

```
ValueKind := "null" | "bool" | "string" | "number" | "bytes"
           | "hash" | "path" | "key" | "index" | "error" | "unknown"
```

The SDK **SHOULD** provide:
- A renderer registry (type path → render function).
- A structural renderer that reads type definitions.
- ValueKind tagging so renderers can apply platform-specific styling without understanding entity content.

---

## 4. The Reactive Cycle

Individual SDK operations compose into a cycle that drives entity-native applications:

```
put(path, entity)
  → emit pathway fires (SYSTEM-COMPOSITION.md §1)
    → subscription engine matches patterns
      → notification delivered to subscriber's inbox (local or remote)
        → inbox triggers continuation advance
          → continuation dispatches next step (e.g., fetch + merge)
            → resulting puts trigger further emit...
              → watch/event stream delivers to application UI
                → application re-renders
```

This is the core reactive pattern: **write → emit → notify → process → write.** The SDK operations (put, watch, execute) are the application's entry and exit points. The emit pathway, subscription engine, inbox, and continuation machinery run between them.

For local-only use (no cross-peer), the cycle simplifies to: put → emit → watch fires → application re-renders.

### Change Detection Mechanisms

The SDK provides three mechanisms suited to different application architectures:

| Mechanism | Delivery | Best for |
|-----------|----------|----------|
| `subscribe_events()` | All events, unfiltered | Event-driven architectures (callbacks, event loops) |
| `watch(prefix, callback)` | Filtered by prefix | Fine-grained reactive UI (specific component watches specific paths) |
| `generation()` | Poll in render loop | Immediate-mode UI (egui, game loops — check if anything changed since last frame) |

All three ultimately observe the emit pathway's Phase 2 async notifications (SYSTEM-COMPOSITION.md §1.3).

---

## 5. Connection Lifecycle Patterns

### Manual

Store transport address, connect explicitly, execute:

```
peer.tree_put("system/peer/transport/{remote_id}",
    entity("system/network/transport-address", {peer_id: remote_id, address: "192.168.1.10:9100"}))
peer.connect("192.168.1.10:9100")
peer.execute("entity://{remote_id}/system/tree", "get", ...)
```

Works for development, static networks, CLI tools.

### Automated (Network Extension)

The network extension's `maintain-peer` builds continuation pipelines:

```
peer.execute("system/network", "maintain-peer", {peer_id: remote_id, address: "192.168.1.10:9100"})
; → connects, builds continuation graph:
;   - subscription on system/peer/status/{remote_id} for disconnect detection
;   - on disconnect: reconnect with exponential backoff
;   - on reconnect: restore subscriptions that were active before disconnect
```

Entity-native: connection state is in the tree (`system/peer/status/*`), continuations are entities (`system/inbox/network/*`), subscriptions trigger the lifecycle. Inspectable, debuggable, composable.

See SDK-EXTENSION-OPERATIONS.md §9 for the full network extension surface.

---

## 6. Multi-Peer Applications

The SDK supports multiple independent peers in one process. Common patterns:

**Application peer + device peers.** The application creates one internal peer (in-process, Level 0+1 access). Device peers run as separate processes or services — the application connects to them via WebSocket/TCP (Level 2 access).

**Multiple internal peers.** The application creates several in-process peers for isolation — each has its own identity, storage, and grants. Useful for testing, sandboxing, or separating concerns.

**The EntitySDK container** (or equivalent) manages multiple PeerContexts. Each PeerContext wraps one local peer. Backend/network peers are registered as metadata — reached through a local peer's Level 2 routing. The container provides `peer(id)` to access a specific context and `default_peer()` for the primary one.

---

## 7. Entity-Native Handler Pattern

When handler logic needs to be **transferable**, **inspectable**, or **survive restart without re-registration**, write it as a compute expression entity and register a handler manifest pointing to it. This is the entity-native model from SDK-OPERATIONS §11.3.

**Shape:**

```
1. Build the expression bottom-up (literals → operators → control flow → root).
   For each sub-expression: encode → store in tree → record content hash.
   See GUIDE-COMPUTE.md §8.1 and GUIDE-COMPUTE-PROGRAMMING.md.

2. Write the root expression entity to a tree path you control (e.g., app/handlers/{name}/body).

3. Register the handler manifest via the system/handler:register protocol operation:
   {
     pattern: "app/{name}",
     expression_path: "app/handlers/{name}/body",
     interface: { operations: [...] },
   }

4. The manifest is a regular tree entity — no SDK primitive is needed because
   the body is already in the tree. Dispatch evaluates the expression at the
   declared path (V7 §6.6).
```

**When to use:**

- Logic that needs to move between peers (transfer manifest + expression + libraries, register at the target).
- Logic that needs static analysis (the compute install audit walks the expression graph and validates capability references; opaque code cannot be audited).
- Logic that needs to survive restart without code-side re-registration (the expression is in the tree; the peer rehydrates dispatch from the manifest).
- Hot-swappable logic (replace the expression at the tree path; subsequent dispatches use the new body).

**When not to use:**

- Tight inner loops or numerically intensive paths — interpreted expressions carry ~5x overhead vs. compiled code. Use SDK language-native for those, or bridge with `compute/apply` handler mode (entity-native expression calling a language-native handler for a specific operation).
- Logic that fundamentally needs language-native I/O (filesystem, network sockets) — use language-native handlers; entity-native is for pure logic + dispatched effects.

**Comparison** (SDK-OPERATIONS §11.3 has the full table): entity-native is transferable, inspectable, hot-swappable, restart-surviving, statically auditable; SDK language-native is faster but ephemeral and opaque.

---

## 8. Reactive Computation Pattern

When derived state needs to **stay current as inputs change**, install a reactive compute subgraph. The engine watches the expression's dependencies, re-evaluates on change, and writes the result to a result path. This is the spreadsheet model (GUIDE-COMPUTE §5).

**Shape:**

```
1. Build the root expression (typically a compute/lookup/tree on input + transform).
   Store the expression entity in the tree.

2. Install the subgraph:
   sdk.compute.install(
     root: expression_hash,
     result_path: "app/views/{name}",
     triggers: { auto },                // engine walks audit_subgraph for triggers
     grant: ctx.installation_grant,     // attenuated cap covering deps + result_path
   ) → SubgraphHandle

3. Reads from app/views/{name} return the current value. As inputs change,
   the engine re-evaluates and updates the result entity at result_path.

4. To tear down: sdk.compute.uninstall(subgraph_handle.path)
```

**When to use:**

- Derived state: a tree projection that needs to stay coherent with its sources without polling or manual recomputation.
- Computed views: aggregations, joins, transformations of underlying data.
- Cross-prefix coordination: subgraph reads from `domain/a/*` + `domain/b/*` and writes to `app/views/{name}`; consumers watch the view path, not the underlying paths.

**Key properties:**

- **Authorization is by installation grant.** Re-evaluations run under the installer's cap, attenuated and stored at install time. If the grant is revoked, re-evaluation fails and the subgraph **freezes** with `installation_grant_invalid`.
- **Cascade limits apply.** A reactive write that triggers another reactive write counts as a cascade step (SYSTEM-COMPOSITION §1.6). Exceeding the depth limit freezes the subgraph with `cascade_limit`. Frozen subgraphs don't re-evaluate until manually unfrozen.
- **The result path is the contract.** Consumers watch the result path; the dependency graph behind it is internal to the engine.

**Composes with §2 entity-backed state** (the result path *is* an entity-backed state slot) and **§4 reactive cycle** (the result-path update emits, which can be watched, which can trigger further reactive computations or UI re-renders).

See GUIDE-COMPUTE §5 for the install/trigger/result model, GUIDE-COMPUTE §7.2 for the freeze error codes, and GUIDE-COMPUTE-PROGRAMMING.md for debugging frozen subgraphs.

---

## 9. Capability Grant Pattern

Grant configuration is where the **kernel-vs-handler principle** (SDK-OPERATIONS §2.7.2) meets practice. The rule: grant handler create operations, not raw `tree:put` to handler-managed namespaces. Handler operations validate embedded capabilities at creation time; `tree:put` bypasses this validation.

**Per-extension shape:**

```
; Antipattern — broad tree:put on handler-managed namespace
{handlers: {include: ["system/tree"]},
 operations: {include: ["put"]},
 resources: {include: ["system/continuation/suspended/*"]}}

; Coherent — route through the handler's create op
{handlers: {include: ["system/continuation"]},
 operations: {include: ["install"]},
 resources: {include: ["*"]}}
```

**The per-extension checklist** (full list in GUIDE-CAPABILITIES §5):

| Extension | Grant this | Not this |
|---|---|---|
| Continuation | `system/continuation:install` | `system/tree:put` on `system/continuation/suspended/*` |
| Subscription | `system/subscription:subscribe` | `system/tree:put` on `system/subscription/*` |
| Role | `system/role:assign` | `system/tree:put` on `system/role/{ctx}/assignment/*` |
| Compute | `system/compute:install` | `system/tree:put` on `system/compute/processes/*` |

**Grant progression** (SDK-OPERATIONS §11.2A; full progression in GUIDE-CAPABILITIES §7):

- **L0** — peer-owner code uses direct store access for its own peer; never exposed to delegated grants.
- **L1** — application grants handler ops for system extensions it uses (continuation install, subscription subscribe, role assign, compute install).
- **L2** — delegated actors get attenuated caps. Attenuation can only narrow; chain validation ensures no escalation.
- **L3** — runtime delegation via the SDK's grant-issue primitive. The chain root remains the issuer; the delegate cannot embed caps it doesn't itself hold (R1; GUIDE-CAPABILITIES §3).

**Common pitfalls** (GUIDE-CAPABILITIES §8 has the full list): broad `tree:put` on a system namespace; embedding a delegated cap as your own dispatch capability; mixing role definition (template writes) with role assignment (token derivation); forgetting chain reachability when serializing requests.

---

*These patterns are advisory. The SDK operations spec (SDK-OPERATIONS.md) defines what the operations are. This guide describes how to use them well.*
