# SDK Operations — Normative Specification

**Version**: 1.10

**Status**: Active
**Depends**: ENTITY-CORE-PROTOCOL.md (v7.40+), SYSTEM-COMPOSITION.md (v1.5+), EXTENSION-COMPUTE.md (v3.18+)

---

> **Scope.** This document specifies the operations an SDK implementation **MUST**, **SHOULD**, and **MAY** expose for application code to interact with the entity system. It defines operation semantics — what goes in, what comes out, what errors can occur — not API shapes. Language-idiomatic differences (method chaining, options pattern, fluent builder) are expected and encouraged.

> **Audience.** SDK implementors in any language. Application developers reference per-language SDK documentation, not this spec.

> **What this spec does NOT cover.** Peer roles, namespace conventions, and path organization are advisory patterns documented in `GUIDE-PEER-CONCERNS-AND-NAMESPACES.md`. This spec is about the interface — it works regardless of how you organize your tree.

---

## 1. SDK Levels

| Level | Name | What it provides | Audience |
|-------|------|-----------------|----------|
| L0 | Wire protocol | CBOR framing, envelope construction, handshake | Core library internals |
| L1 | Operations | Tree ops, dispatch, query, subscription, connection, lifecycle | All SDK consumers |
| L2 | Patterns | Scoped handles, state management, type rendering (see `GUIDE-SDK-PATTERNS.md`) | Application developers |
| L3 | Extension operations | Extension handler wrappers (revision, subscription, history, query, etc.), handler registration | Entity-native applications |
| L4 | Composition patterns | Reactive pipelines, cross-peer workflows, workspace transfer | Advanced applications |

**This specification covers L1 and advisory L2 patterns.** L0 is internal to the core library. L3 and L4 are documented in `guides/` as patterns mature.

---

## 2. How Operations Actually Work

The SDK manages one or more local peers. When you call an operation, you're calling it on one of those managed peers. How the peer was created — its keypair, storage, extensions, listener, grants — determines what happens. An ephemeral client peer with no extensions handles a `get()` differently than a full listening peer with revision and subscription (§2.6 covers the spectrum, §2.7 covers the access levels).

This section describes the full flow from peer creation through operation execution.

### 2.1 Peer Startup and Capability Bootstrap

When a peer is created:

1. **Keypair loaded or generated.** The peer's identity (Ed25519 keypair) is its cryptographic root.
2. **Storage initialized.** Content store (hash → entity) and location index (path → hash).
3. **Bootstrap handlers pre-loaded.** `system/tree`, `system/handler`, `system/type`, `system/protocol/connect` are installed before the capability system exists (V7 §6.9). They have to be — they're needed to install everything else.
4. **Self-grants generated.** For each registered handler, the peer creates a capability grant entity at `system/capability/grants/{pattern}`. The grant is derived from the peer's root capability, attenuated to the handler's declared scope (`internal_scope`). This is how handlers get permission to do their work.
5. **Extensions registered.** Each extension contributes a handler, emit consumers, and/or an engine. Extensions are registered in the order specified by SYSTEM-COMPOSITION.md §2.2.
6. **Peer is live.** Engines started, listener bound (if configured), ready for operations.

**The capability chain is fundamental.** Every operation — get, put, execute, query — goes through capability verification. The chain is: peer root capability → handler grant → caller's capability token. The four scope dimensions checked on every dispatch are: handler pattern, operation name, peer identity, and resource path (V7 §5.2 `check_permission`).

**The SDK manages this transparently.** When application code calls `get("knowledge/articles/intro")`, the SDK constructs the authorized request internally — the application doesn't build EXECUTE entities or manage tokens by hand. The SDK holds a scoped capability grant for the code running against it. The application knows what permissions it has because the SDK was configured with those grants at instantiation.

For local operations, the SDK handles authorization internally. For remote operations, capability tokens come from connection grant exchange. Either way, the application code looks the same — the SDK is the interface that manages grants, connections, and scoped access on behalf of the application.

### 2.2 Connection and Grant Exchange

When two peers connect:

1. **Two-round-trip handshake** (V7 §4). HELLO (request/response) → AUTHENTICATE (request/response). Each step is a dispatched EXECUTE to the `system/protocol/connect` handler, enforced in order.
2. **Grant exchange.** The authenticate response carries the initial capability grant. The grant specifies four scope dimensions:
   - `handlers`: which handlers the remote peer can call (e.g., `system/tree`, `local/files`)
   - `operations`: which operations (e.g., `get`, `put`, `list`)
   - `resources`: which paths (e.g., `knowledge/*`, `public/*`)
   - `peers`: which peer identities (usually the local peer)
3. **Connection established.** Remote operations are now authorized by the exchanged grants.

For development, the SDK **MAY** provide a debug/full-access grant mode. For production, grants **SHOULD** be scoped per-identity and per-connection. See §11 for the capability management surface.

### 2.3 Operation Dispatch Flow

In the examples below, `peer` is a managed local peer — created via `create_peer()` (§8.1) and held by the application. The peer's configuration (§2.6) and access level (§2.7) determine the actual dispatch pathway. This section shows the full Level 1 handler dispatch path; Level 0 and Level 2 are described in §2.7.

Every Level 1 SDK operation follows this path:

```
Application calls peer.get("knowledge/articles/intro")
  │
  ├─ SDK constructs EXECUTE entity:
  │    type: "system/protocol/execute"
  │    data: {uri: "entity://{peer}/system/tree",
  │           operation: "get",
  │           resource: {targets: ["knowledge/articles/intro"]},
  │           author: {peer's identity hash},
  │           capability: {grant hash}}
  │
  ├─ SDK signs the EXECUTE, builds envelope with included entities
  │    (author identity, capability token, signature)
  │
  ├─ Dispatch chain (V7 §6.5):
  │    1. Verify content hashes
  │    2. Verify signature
  │    3. Verify capability chain (author → grantee match, chain to peer root)
  │    4. Canonicalize URI path
  │    5. Resolve handler (longest-prefix match in tree → system/tree)
  │    6. Check permission (handler × operation × peer × resource)
  │    7. Build handler execution context
  │    8. Handler processes request
  │
  └─ Handler returns response → SDK returns entity to application
```

For local calls, the SDK **MAY** optimize this path (skip envelope construction, skip signature verification for self-signed local calls). The observable behavior **MUST** be equivalent — if a capability check would fail, the operation **MUST** fail with 403 even for local calls.

**The application doesn't see any of this.** The application calls `peer.get(path)` on a managed local peer. The peer knows its identity, its grants, and its connections. The SDK constructs the request, handles dispatch, and returns the result.

### 2.4 The Reactive Cycle

The SDK operations compose into a reactive cycle: write → emit → notify → process → write. Every tree mutation fires the emit pathway (SYSTEM-COMPOSITION.md §1), which triggers extension consumers (history, subscription, query indexing, clock, revision). Subscriptions deliver notifications to inboxes. Continuations advance to next steps. Resulting writes trigger further emit.

The SDK operations (put, watch, execute) are the application's entry and exit points into this cycle. See `GUIDE-SDK-PATTERNS.md` §4 for the full cycle description and change detection mechanisms.

### 2.5 Path Model

**Every path in the entity system is absolute — rooted at a peer identity.** The path `/{peer_id}/knowledge/articles/intro` is how data actually lives in the tree. There is no path outside a peer namespace. The location index stores absolute paths. Wire messages carry absolute paths (after URI normalization). Storage is absolute.

When the SDK accepts a path without a leading `/` (like `knowledge/articles/intro`), this is peer-relative notation — a caller convenience. The SDK resolves it to `/{local_peer_id}/knowledge/articles/intro` before any operation. This is not a different addressing mode. It's shorthand.

```
; These are equivalent on the local peer:
peer.get("knowledge/articles/intro")
peer.get("/{local_peer_id}/knowledge/articles/intro")

; This targets a remote peer's tree:
peer.get("/{remote_peer_id}/knowledge/articles/intro")
```

**The universal namespace is the reality.** Every peer sees one tree with paths organized by peer identity. Under `/{local_peer_id}/` is the peer's authoritative data. Under `/{other_peer_id}/` is cached remote data — what this peer knows about that peer, which may be stale or incomplete. Only the key holder for a peer_id is the authority for that namespace.

**Scoped handles and tree scopes are convenience.** A scoped handle bound to prefix `knowledge/` on the local peer provides `get("articles/intro")` which resolves to `/{local_peer_id}/knowledge/articles/intro`. The scoping is syntactic sugar over the universal namespace.

See V7 §1.4 for the full path model. See `GUIDE-PEER-CONCERNS-AND-NAMESPACES.md` for path organization conventions (reserved prefixes: `system/`, `host/`, `app/{app-id}/`, `bridge/`).

### 2.6 The Peer Spectrum

A peer is always rooted in a keypair. Everything else is progressive enhancement:

| Configuration | Storage | Listener | Extensions | Example |
|---------------|---------|----------|------------|---------|
| **Ephemeral client** | In-memory | None | None | CLI tool: load keypair, connect, execute, exit |
| **Long-running client** | In-memory or SQLite | None | Standard set | Application's internal peer: maintains connections, stores state |
| **Listening peer** | SQLite | TCP/WS | Standard set | Server: accepts inbound connections, serves content |
| **Persistent service** | SQLite | TCP/WS | Standard + domain | System service: starts on boot, runs as daemon |

The keypair is the constant — you can't operate on the network without identity. Storage, extensions, listener, and lifecycle are additive. The SDK serves the entire spectrum.

### 2.7 Access Levels

The SDK exposes three access levels to a local peer. All three are always available.

**L1 is the default; L0 is the carve-out (normative).** Application-level writes SHOULD go through L1 (capability-checked dispatch) by default. L0 (direct store access) is a documented carve-out for purely-internal bookkeeping that no peer will ever observe (e.g., window geometry, host-private layout caches, ephemeral per-process state) — and for the bootstrap / handler-internal cases enumerated in §2.7.1. The moment L0 data becomes subject to multi-peer / service-peer profile access, L0-by-default code paths require restructuring; establishing L1 as default keeps the multi-peer option open structurally. Per the kernel-vs-handler principle (§2.7.2), application grants SHOULD cover handler create operations (continuation install, subscription subscribe, role assign, compute install) rather than raw `tree:put` to handler-managed namespaces. The Rust EOS-discipline reframe documents the rationale and a worked enforcement pattern for this discipline.

**Level 0 — Direct store access.** Synchronous read/write to the peer's content store and location index. No handler dispatch. The application is operating as the peer.

- Every tree mutation fires the emit pathway. This is the system's fundamental guarantee — history records the transition, subscription engine matches, query indexes update, clock advances. The emit pathway is not a handler feature. It's the primitive.
- The execution context carries the peer's standing grant (root capability from build). Capability context is never absent.
- Level 0 under a full-admin root grant (the default) is well-defined — you have permission to do everything, you're just skipping dispatch overhead. If the peer was constructed with a constrained grant and Level 0 writes exceed those constraints, this is a **policy bypass, not a system inconsistency.** The tree state is correct. The emit consumers fire and record the mutation. History, subscription, query indexing all see the write. The grant constraints are simply not enforced because dispatch was not invoked. Applications using Level 0 on constrained peers are responsible for ensuring writes are consistent with their intended grant policy.

**Level 1 — Local handler dispatch.** `execute()` on the local peer. Resolves the handler (longest-prefix match), builds execution context (handler grant, caller capability, scoped tree access), dispatches. The handler processes the operation and returns a result. Capability attenuation applies for handler-to-handler sub-dispatch.

**Level 2 — Remote dispatch.** Same `execute()` call, but the URI targets a different peer_id. The local peer resolves the transport address from `system/peer/transport/{remote_peer_id}`, gets or creates a connection (dial + handshake if first time), constructs a signed EXECUTE envelope with the capability token from the handshake, sends it on the wire, receives the response.

**The application doesn't choose the level for execute().** The same `execute()` function handles both Level 1 and Level 2. If the peer_id in the URI matches the local peer, it's Level 1. If it's different, it's Level 2. Peer-relative paths always resolve to local.

**Level 0 is a separate, explicit API surface.** Direct store operations are distinct from dispatched operations. The developer explicitly opts in. This is the same pattern as ORMs that separate query builders (safe by default, parameterized) from raw SQL access (powerful, bypasses protections, developer takes responsibility):

```
; Dispatched (Level 1) — goes through tree handler, capability-checked
peer.get("knowledge/articles/intro")
peer.execute("system/tree", "get", {resource: {targets: ["knowledge/articles/intro"]}})

; Direct store (Level 0) — bypasses dispatch, peer owner's authority
peer.store.get("knowledge/articles/intro")      ; or: peer.direct.get(), peer.raw_get()
peer.store.put("knowledge/articles/intro", entity)
```

The naming is language-idiomatic — the spec doesn't prescribe method names. What the SDK **MUST** do is make the boundary visible: direct store operations **SHOULD** be accessed through a distinct interface (separate namespace, separate accessor, or clearly differentiated names) so that a developer reading the code can tell which level they're operating at. Mixing dispatched and direct operations under the same names (both called `get`) is the anti-pattern — it hides the security boundary.

Application developers building higher-level abstractions (custom handlers, application frameworks, multi-user systems) should route through dispatched operations (`execute()`, `get()`) to get capability enforcement. Developers who need Level 0 for performance know they're operating as the peer and accepting the responsibility.

The SDK **SHOULD NOT** expose Level 0 to code that runs under delegated or attenuated grants (handlers processing remote requests, compute expressions evaluating under installation grants). Those contexts use the `execute()` function from their handler context, which enforces capability attenuation. Level 0 is for the peer owner's code — the code that built and configured the peer.

**Conformance under open-grants mode.** Until §11 grant enforcement lands kernel-side, "the execution context carries the peer's standing grant" is forward-looking. SDKs operating in open-grants mode satisfy §2.7 by preserving the emit pathway (every L0 mutation fires the broadcast); the standing-grant-context aspect is satisfied vacuously when grants are unenforced. Once §11.2 / §11.2A enforcement lands, the standing-grant context becomes load-bearing and SDKs MUST plumb it through L0.

### 2.7.1 L0 Use Cases

L0 (direct store access) is a well-defined access level — not a hack or debug mode. Every L0 tree mutation fires the emit pathway (history, subscription, query, clock all observe it). The distinction from L1 is: no handler dispatch, no capability verification at the dispatch layer.

| Use case | L0 acceptable? | Why |
|----------|---------------|-----|
| Bootstrap (pre-handler tree writes) | Yes, required | No handlers exist yet to dispatch through |
| Identity bootstrap (`:configure` first call, before local peer→controller cap exists) | Yes, **required** | Per `EXTENSION-IDENTITY.md` §6.5 startup boundary — no controller authority exists yet. SDK MUST route through the bootstrap helper library defined in `sdk-domain/specs/SDK-IDENTITY-INFRASTRUCTURE.md` §7. |
| Identity ops post-bootstrap (`:create_attestation`, `:supersede_attestation`, `:publish_attestation`, `:rotate_*`, etc.) | No | After local cap exists, dispatched EXECUTE under controller authority is the only conformant path. |
| Peer-owner application code | Yes, explicit | Owner authority; naming discipline makes bypass visible |
| Render-time reads (sync UI) | Yes, acceptable | Read-only; no security or semantic implications |
| Pre-runtime state seeding (application startup) | Yes, with rationale | Same as bootstrap — runs before dispatch is meaningful |
| Handler-internal writes to own namespace | Yes, by design | Handlers need direct tree access; authorization recorded via mutation context |
| Handler-internal writes beyond own namespace | Caution | Should use `execute_fn` for cap-checked cross-handler calls |
| Delegated/attenuated code | No | Bypasses restrictions the grant was designed to enforce |
| Extension state writes that bypass handler semantics | No | Produces state the extension handler doesn't expect |

**Principle:** L0 is the peer owner's prerogative. It's well-defined, not a bug, and not going away. The SDK's job is to make it visible, not to prevent it.

SDKs MAY provide cloneable L0 read handles for use in callbacks and async contexts where the originating scope has ended. These handles MUST be accessed through the L0 API surface (`store()`) and SHOULD be read-only.

### 2.7.2 Kernel-vs-handler principle

`tree:put` is kernel-level access. The tree handler provides direct location-index and content-store mutation, bypassing every domain handler's validation logic. This is intentional — but for **load-bearing entity types** (types whose data carries semantic content that affects later behavior: capability references, dispatch targets, state-machine fields, scoped grants), the proper creation path is the owning handler's operation.

**What this means for SDK applications:**

- Grant `system/continuation:install`, not `system/tree:put` on `system/continuation/suspended/*`
- Grant `system/subscription:subscribe`, not `system/tree:put` on `system/subscription/*`
- Grant `system/role:assign`, not `system/tree:put` on `system/role/{context}/assignment/*`
- Grant `system/compute:install`, not `system/tree:put` on `system/compute/processes/*`

Handler operations validate embedded capabilities, check authority chains, and enforce domain invariants. `tree:put` bypasses all of this. An actor with `tree:put` on a system handler's namespace can embed arbitrary capability references in load-bearing entities — capabilities they don't legitimately hold — and trigger later dispatches that wield those capabilities.

**SDK guidance:** application-level capability grants SHOULD cover handler create operations, not raw `tree:put` to handler-managed namespaces. L0 direct store access on these namespaces falls under the same caution: if the L0 writer embeds capability references, those references are unvalidated. L0 writes by the peer owner (author = local peer identity) are safe by construction (their embedded caps chain to the local peer), but L0 should never be exposed to delegated or attenuated code.

See V7 §6.3 for the normative statement. See SDK-EXTENSION-OPERATIONS.md §15.1 for per-extension Coherent Capability guidance.

### 2.8 You Operate ON a Local Peer

There is no "remote peer handle." You operate on your local peer. Your local peer routes to remote peers when the URI says so. The connection is an internal implementation detail — cached in the peer's connection pool, keyed by peer_id, reused for subsequent requests.

```
; Create your local peer
peer = create_peer(config)

; Local operations (Level 0 or Level 1)
peer.tree_get("knowledge/articles/intro")                        ; Level 0: direct store
peer.execute("system/query", "find", {type_filter: "article"})   ; Level 1: handler dispatch

; Remote operations (Level 2 — routed by URI peer_id)
peer.execute("entity://{remote_id}/system/tree", "get",
    {resource: {targets: ["knowledge/articles/intro"]}})
  ; → resolves address from system/peer/transport/{remote_id}
  ; → dials + handshakes if no cached connection
  ; → sends EXECUTE on wire, receives response

; connect() pre-establishes a connection (optional — execute() connects on demand)
peer.connect("192.168.1.5:9100")
  ; → dial, hello exchange, authenticate exchange, grant exchange
  ; → connection cached, keyed by remote peer_id
  ; → subsequent execute() calls to that peer_id use the cached connection
```

**Transport addresses are entities.** Remote peer addresses live at `system/peer/transport/{peer_id}` in the local tree. The execute function resolves them on demand. The `connect()` operation can also store the transport address after handshake. See SDK-EXTENSION-OPERATIONS.md §9 (network extension) for automated lifecycle management via continuation pipelines.

**Connection failure.** If a cached connection fails during a remote dispatch, the operation returns a transport error (500) to the caller. The connection pool removes the failed connection. The SDK does not automatically retry — reconnection is either explicit (`connect()` again) or automated by the network extension (`maintain-peer` builds reconnection continuation pipelines). No silent retry at the pool level.

---

## 3. Core Tree Operations (L1)

All tree operations accept absolute paths (`/{peer_id}/path`) or peer-relative paths (`path`). Peer-relative paths resolve to the local peer's namespace (§2.5). The access pathway — local dispatch or remote execute — is determined by peer_id resolution (§2.8).

### 3.1 get

```
get(path: string) → Entity | null

  Returns: The entity at path, or null if no binding exists.

  Errors:
    403  Capability grant does not cover this path.
    500  Internal error.
```

### 3.2 put

```
put(path: string, type: string, data: any) → hash

  Returns: The content hash of the stored entity.

  Errors:
    400  Invalid path, type, or data.
    403  Capability grant does not cover this path or operation.
    409  Conflict (compare-and-swap failure when applicable).
    500  Internal error.
```

Implementations **MAY** accept an already-constructed entity instead of separate type and data parameters.

A put triggers the emit pathway (SYSTEM-COMPOSITION.md §1). The operation returns after all synchronous emit consumers have completed (Phase 1). Phase 2 async notifications fire after return.

**Compare-and-swap.** The SDK **SHOULD** support conditional puts:

```
put_cas(path: string, type: string, data: any, expected_hash: hash?) → hash

  Writes only if the current binding matches expected_hash.
  expected_hash = null means "path must not exist" (create-only).

  Errors:
    409  Current binding does not match expected_hash.
```

**V7 §3.9 v7.50 cross-reference (normative).** The SDK's `expected_hash` parameter maps to V7 §3.9's three-value `system/tree/put-request.expected_hash` semantics:

| SDK call | V7 wire `expected_hash` | Semantics |
|---|---|---|
| `put(...)` (no CAS) | field absent | unconditional |
| `put_cas(... expected_hash=null)` (or equivalent language sentinel) | zero-hash | expect-absent — succeeds only if path is currently unbound (CAS-create) |
| `put_cas(... expected_hash=H)` for non-zero `H` | `H` | succeeds only if current binding equals `H` |

SDK implementations **MUST** translate the language-native "null" sentinel into V7's zero-hash on the wire. A server receiving a `put-request` with `expected_hash = zero-hash` treats it as CAS-create per V7 §3.9 v7.50; the previously-undefined zero-value case is now distinguished from the absent-field case.

### 3.3 list

```
list(prefix: string) → [Entry]

  Entry := {
    name:         string       ; Segment name (not full path)
    path:         string       ; Full path
    content_hash: hash
    has_children: bool
  }

  Returns: Direct children of prefix. Empty array if none. Ordering
           is implementation-defined.

  Errors:
    403  Capability grant does not cover this prefix.
    500  Internal error.
```

Single-level. Direct children only, not recursive.

**L0 vs L1 list shapes — principled divergence.** The Entry shape defined above is normative for L1 dispatched `list`. L0 implementations MAY return a different shape (e.g., a recursive index walk) reflecting raw store access semantics. The divergence is principled: L1 is the dispatched protocol surface where single-level immediate-children matches the dispatch contract; L0 is raw store access where the underlying index naturally supports recursive walks at lower cost than chained dispatch. SDKs that expose both levels SHOULD use distinct return types to make the level distinction explicit, and document the L0 shape's divergence in their SDK reference.

**Conformance:** L1 `list` MUST return entries conformant to the Entry shape (single-level, immediate children, all four fields populated).

### 3.4 remove

```
remove(path: string) → ()

  Errors:
    403  Capability grant does not cover this path or operation.
    404  No binding at path (implementations MAY silently succeed).
    500  Internal error.
```

Unbinds the path. Entity remains in content store. Triggers emit pathway.

### 3.5 has

```
has(path: string) → bool

  Errors:
    403  Capability grant does not cover this path.
    500  Internal error.
```

Convenience. Implementations **MAY** implement as `get(path) != null`.

### 3.6 snapshot

**Note on SDK exposure (§3.6-3.9).** §3.6-3.9 specify operations on the tree handler. SDKs reach them via the standard handler-dispatch primitive (`execute("system/tree", "snapshot", ...)`); typed SDK wrappers are MAY-tier convenience. They are not in §16 conformance because the SDK contract is the dispatch primitive, not the typed wrappers. Implementations that ship typed wrappers SHOULD use the parameter and return shapes defined in §3.6-3.9.

```
snapshot(prefix: string) → hash

  Captures current tree state under prefix as a Merkle trie.
  Returns the root hash of the snapshot.

  Errors:
    403  Capability grant does not cover this prefix.
    500  Internal error.
```

### 3.7 diff

```
diff(from: hash, to: hash) → [Change]

  Compares two snapshots. Trie-aware — skips matching subtrees.

  Change := {
    path:      string
    change:    string       ; "added" | "removed" | "modified"
    old_hash:  hash?
    new_hash:  hash?
  }

  Errors:
    400  Invalid snapshot hash (not found in content store).
    500  Internal error.
```

### 3.8 merge

```
merge(base: hash, ours: hash, theirs: hash) → MergeResult

  Three-way tree merge at the trie level.

  MergeResult := {
    status:    uint         ; 200 clean, 409 conflicts
    root:      hash?        ; Merged trie root (if clean)
    conflicts: [{path: string, base: hash?, ours: hash?, theirs: hash?}]
  }

  Errors:
    400  Invalid snapshot hash.
    409  Conflicts detected (returned in MergeResult.conflicts).
    500  Internal error.
```

### 3.9 extract

```
extract(snapshot: hash, prefix: string) → Envelope

  Extracts a subtree from a snapshot as a transferable envelope.
  Includes trie nodes + entity content for the prefix scope.

  Errors:
    400  Invalid snapshot hash.
    403  Capability grant does not cover prefix.
    500  Internal error.
```

These operations are in the tree handler alongside get/put/list/remove. They're the foundation for the revision extension's version management.

---

## 4. Handler Dispatch (L1)

### 4.1 execute

```
execute(target: string, operation: string, params?: any) → Response

  target:    Path or URI identifying the handler target.
  operation: Operation name.
  params:    Operation-specific parameters (optional).

  Response := {
    status:   uint           ; HTTP-style status code
    type:     string         ; Response entity type
    data:     any            ; Response payload
    hash:     hash           ; Content hash of response entity
    included: [Entity]?      ; Supporting entities (signatures, capabilities, multi-entity results)
  }

  Errors:
    400  Invalid target, operation, or params.
    403  Capability grant does not cover this handler, operation, or resource.
    404  No handler matched the target path (longest-prefix match failed).
    429  Rate limited.
    500  Handler error.
    501  Handler does not support the requested operation.
```

Execute is the universal dispatch mechanism. Tree operations (§3) are ergonomic wrappers around execute calls to `system/tree`. The SDK exposes both: tree operations for the common case, execute for everything else (extension handlers, domain handlers, custom handlers).

Handler resolution uses longest-prefix matching against the target path (V7 §6.6). The handler at `system/tree` catches `system/tree/any/sub/path`. A handler at `local/files` catches `local/files/readme.md`.

#### 4.1.1 Result carrier and dispatch-surface equivalence (V7 §3.3 v7.49, normative)

A handler returning multiple entities **MUST** wrap them as a `system/envelope` carrying the domain subtree in its `included` map. The `execute()` result shape is identical across external (cross-peer), internal sub-dispatch, and remote dispatch surfaces — internal and remote **MUST NOT** drop the envelope's `included`. SDK wrappers materializing `execute()` returns into language-native shapes **MUST** preserve `included` across all three surfaces; an internal optimization that strips `included` for in-process dispatch is non-conformant. The in-process representation is implementation-private; the surface contract is invariant.

#### 4.1.2 Internal sub-dispatch authorization (V7 §6.8 v7.49, normative)

When a handler dispatches internally (a sub-call inside the handler's own execution), the authorization decision gates on the **executing handler's grant**, not on the caller's propagated `caller_capability`. The propagated `caller_capability` is for caller-specified-path checks and history attribution only and **MUST NOT** be a dispatch gate (the confused-deputy dual of "No silent escalation"). SDK wrappers providing internal-dispatch helpers **MUST** honor this rule.

#### 4.1.3 Request-side `included` preservation (V7 §3.3 v7.51, normative)

A dispatcher routing an EXECUTE envelope (whether to a local handler, an internal sub-dispatch, or a remote peer) **MUST** preserve the envelope's `included` map across the surface. A dispatcher **MUST NOT** drop `included` before the wire — bundled hash-refs in `EXECUTE.data` resolve against `included`; if dropped, the handler and any downstream continuations see inconsistent referents. SDK wrappers materializing outbound EXECUTEs **MUST** forward `included` across local/internal/remote dispatch boundaries identically. Load-bearing for `deref_included` (EXTENSION-CONTINUATION §2.2; SDK §2 Continuation transform_ops) consuming an `include_payload`-bundled entity.

---

## 5. Query (L1)

### 5.1 query

```
query(expression: any) → [Result]

  expression: Query expression per EXTENSION-QUERY.md.

  Result := {
    path:         string
    content_hash: hash
    entity:       Entity?    ; If requested in expression
  }

  Errors:
    400  Invalid query expression.
    403  Capability grant restricts query scope.
    500  Internal error.
```

Dispatches to `system/query` handler, operation `find`. The query handler's capability scope limits which tree prefixes the query can see.

The SDK **SHOULD** provide query builder helpers at L3 but **MUST** accept raw expression data at L1.

---

## 6. Change Notification (L1)

**Conformance note.** The `watch(pattern) → ChangeStream` shape is the spec's *logical* surface — the semantic pieces (pull-style change notification, pattern-filtered, exact-or-prefix-glob) are conformance-bearing. The exact factory name and return-type construction are platform-idiomatic: Rust uses `mpsc::Receiver<ChangeEvent>` *or* a callback shape; Go uses `chan ChangeEvent`; Python uses async iterators; Godot uses signals. SDKs satisfy §6 conformance by exposing pull-style change notification with the specified semantics; cross-impl conformance is on the capability, not the literal factory signature.

`watch()` is an L1 operation with implementation-defined backend. It **MAY** be powered by the subscription extension (cross-peer capable, full pattern matching), the raw emit pathway event stream (local only, all events), or polling (generation counter). The backend choice is transparent to the application — the `watch()` contract is the same regardless.

For full control over cross-peer subscriptions, deliver tokens, and limits, use the subscription extension directly via `execute("system/subscription", "subscribe", ...)` — see SDK-EXTENSION-OPERATIONS.md §3.

### 6.1 watch

```
watch(pattern: string) → ChangeStream

  pattern: Path pattern. Two forms only:
    "knowledge/articles/intro"    exact path match
    "knowledge/articles/*"        prefix match (all paths under prefix)

  Only exact and prefix/* patterns are specified. Deeper patterns
  (e.g., "knowledge/*/intro") are reserved for future specification.

  ChangeStream: Platform-specific delivery.
    Go:     chan ChangeEvent
    Rust:   mpsc::Receiver<ChangeEvent> or callback
    Python: async iterator / asyncio.Queue
    Godot:  signal

  ChangeEvent := {
    event_type: string     ; "put" | "remove"
    path:       string     ; Path that changed
    new_hash:   hash?      ; New content hash (null on remove)
  }

  Errors:
    400  Invalid pattern syntax.
    403  Capability grant does not cover the pattern scope.
    500  Internal error.
```

**Delivery semantics:**

- Events delivered asynchronously (Phase 2 per SYSTEM-COMPOSITION.md §1.3).
- Per-path ordering preserved. Cross-path ordering implementation-defined.
- Implementations **MAY** coalesce rapid changes to the same path but **MUST NOT** silently drop changes unless the underlying subscription has `rate_limit` configured (EXTENSION-SUBSCRIPTION.md — rate-limited events are dropped, not queued).

### 6.2 unwatch

```
unwatch(handle: SubscriptionHandle) → ()
```

Implementations **SHOULD** auto-cancel on handle drop/GC. Explicit unwatch **MUST** also be available.

### 6.3 Raw Event Stream

The SDK **MAY** expose the raw tree change event stream below the `watch()` abstraction:

```
subscribe_events() → EventStream    ; All tree mutations, unfiltered

  TreeChangeEvent := {
    event_type:    string     ; "put" | "remove"
    path:          string
    new_hash:      hash?
    previous_hash: hash?
  }
```

These are the local emit pathway Phase 2 notifications (SYSTEM-COMPOSITION.md §1.3). Useful for application frameworks that need all events regardless of pattern.

### 6.4 Generation Counter (Optional)

The SDK **MAY** expose a generation counter for polling-based change detection:

```
generation() → uint         ; Monotonically non-decreasing
```

**Semantics (when exposed).** The counter is monotonically non-decreasing and **MUST** advance at every tree mutation observable to readers. Concretely: if a sequence of `tree:put` calls completes and a subsequent `get` would return new data, `generation()` must have advanced at least once between those points. Implementations **MAY** coalesce a batch of mutations into a single bump at a commit boundary (e.g., a transaction or revision merge), but the counter **MUST NOT** lag behind reader-visible state — a reader that observes a value and re-reads `generation()` must see the bump.

**Bulk-write performance is the SDK's problem, not the reader's.** Applications observing this counter (game-loop polls, immediate-mode UI) rely on the counter to reflect reality with bounded latency. If bulk operations are slow at the bump granularity, the SDK is responsible for batching mutations behind a single bump (e.g., transactional commit), not for skipping bumps and forcing readers to scan.

**This is one of several change-detection styles, not the default.** Three styles in common use:

| Style | Mechanism | Best for |
|---|---|---|
| **Counter polling** | `generation()` checked each frame/tick; if changed, scan state of interest. | Immediate-mode UI with simple state, game loops where a per-frame check is already happening. |
| **Path-targeted observation** | `store().watch(prefix)` (§6.5 L0) — pattern-filtered local event stream, no global counter scan. | Retained-mode UI with many independent panels watching disjoint subtrees. Avoids the per-consumer rescan cost of a global counter. |
| **Dispatched subscription** | `subscribe(pattern, callback)` (§6.5 L1) — capability-checked, cross-peer capable. | Cross-peer reactivity, capability-gated subscribers, anything that needs the inbox handler delivery model. |

`generation()` is the simplest and most coarse; `watch(pattern)` is finer and cheaper at scale; `subscribe` is the only one that crosses peer boundaries. Application teams choose per panel/component based on retained vs immediate rendering, fan-out, and whether the observation needs to leave the local peer. Implementations that don't need polling-based change detection can omit `generation()` entirely.

### 6.5 Notification Access Levels

The §2.7 access-level boundary applies to change notification. Three observation primitives, at two access levels:

**Level 0 — Raw observation (always available, local-only):**

- `store().subscribe_events()` — all tree mutations, unfiltered (§6.3).
- `store().watch(pattern)` — filtered by pattern, local-only. Uses the raw emit pathway broadcast with client-side pattern matching. No capability check. Always available regardless of extensions installed.

**Level 1 — Dispatched observation (requires subscription extension, cross-peer capable):**

- `subscribe(pattern, callback)` — dispatched through `system/subscription`. Capability-checked. Delivers via inbox handler. Supports rate limiting, redirect, and bounded fanout per EXTENSION-SUBSCRIPTION.

The SDK MUST expose L0 observation through the Level 0 API surface (the `store()` accessor or equivalent) and L1 observation through the Level 1 API surface. The naming MUST carry the boundary.

**When the subscription extension is not installed:** L0 observation primitives (`store().subscribe_events()`, `store().watch()`) are always available. L1 `subscribe()` returns an error indicating the subscription extension is required. A minimal peer (no extensions) still has pattern-filtered local observation via `store().watch()`.

---

## 7. Connection (L1)

Connection operations run on a local peer — the local peer provides the identity (keypair) and the grants to issue to the remote peer.

**Inbound frame processing concurrency (V7 §4.8 v7.48, normative).** SDK implementations of the connection surface **MUST** support inbound frame processing concurrent with outbound dispatch initiated from handlers. While a handler is processing a frame received on a connection, the SDK **MUST** be able to read and dispatch additional frames received on that same connection, and to send outbound EXECUTEs (including responses to those additional frames) without waiting for the original handler to complete. This is a correctness requirement — without it, bidirectional symmetric P2P deadlocks. SDKs **MAY** bound concurrency via worker pools, semaphores, or back-pressure; they **MUST NOT** serialize inbound processing on outbound dispatch on the same connection. Architecture is impl-defined (per-frame goroutines, request-response multiplexing, async-task-per-frame all valid); the forbidden shape is the synchronous-inline-dispatch default.

### 7.1 connect

```
connect(address: string) → Connection

  address: "host:port" for TCP, "ws://host:port" for WebSocket.

  Steps:
    1. Dial transport to address.
    2. HELLO exchange: send local peer_id + nonce + supported protocols,
       receive remote peer_id + nonce + protocols. Negotiate common protocol.
    3. AUTHENTICATE exchange: send public key + signed nonce,
       receive remote public key + signed nonce + capability grant.
       Send our capability grant to the remote peer.
    4. Connection established. Both sides hold:
       - The other peer's identity (peer_id, public key)
       - The capability grant received from the other peer
       - The capability grant issued to the other peer

  Connection := {
    remote_peer_id: PeerID     ; Remote peer's identity
    address:        string
    protocols:      [string]   ; Negotiated versions
    grants:         [GrantEntry] ; Grant entries received from remote peer
  }

  Errors:
    400  Invalid address.
    403  Handshake failed (incompatible protocols, signature verification
         failed, or remote peer rejected connection).
    500  Transport error (dial failed, connection reset).
```

After connect, the local peer can dispatch operations to the remote peer using the received grants (see §2.8). The connection is pooled — subsequent operations to the same remote peer reuse it.

The grants we issue to the remote peer come from the local peer's connection grant configuration (§8.1 `grants` in PeerConfig). The grants we receive are what the remote peer chose to give us — they determine what operations we can dispatch to that peer.

### 7.2 listen

```
listen(address: string) → Listener

  Binds the address and accepts incoming connections. For each incoming
  connection, the local peer acts as the responder in the handshake:
    1. Receive HELLO, respond with local peer_id + nonce.
    2. Receive AUTHENTICATE, verify signature, respond with capability grant.
    3. Connection established — remote peer can now dispatch operations
       authorized by the grant we issued.

  The grants issued to connecting peers come from the local peer's
  connection grant configuration (§8.1).

  Errors:
    400  Invalid address or port in use.
    500  Transport error.
```

### 7.3 connected_peers

```
connected_peers() → [PeerInfo]

  PeerInfo := {
    peer_id:   PeerID
    address:   string
    direction: string       ; "inbound" | "outbound"
  }
```

---

## 8. Peer Lifecycle (L1)

### 8.1 create_peer

```
create_peer(config: PeerConfig) → Peer

  PeerConfig := {
    keypair:     Keypair?       ; Identity. Generated if omitted.
    storage:     StorageConfig?  ; Backend. In-memory if omitted.
    handlers:    [Handler]?      ; Custom application handlers.
    listen_addr: string?
    grants:      [Grant]?       ; Connection-time grants for remote peers.
  }
```

See §2.1 for the full startup flow. The peer builder:

1. Loads/generates keypair.
2. Initializes storage.
3. Pre-loads bootstrap handlers (tree, handler, type, connect).
4. Registers system extensions. Each contributes handler + emit consumer + engine, in SYSTEM-COMPOSITION.md §2.2 order.
5. Registers custom application handlers (from config).
6. Generates self-grants for each handler at `system/capability/grants/{pattern}`.
7. Starts engines.

**System extensions vs custom handlers.** System extensions (subscription, continuation, inbox, revision, history, query, clock) are the standard set defined by the extension specs. How they're enabled is language-specific:

- **Compile-time selection.** The SDK is built with a set of extensions baked in (Rust feature flags, build-time constants). Extensions not compiled in are unavailable. This is the typical pattern for system extensions — they're part of the peer implementation, not runtime-configurable by application code.
- **Runtime registration.** The builder wires extensions during peer construction — registering handlers, emit consumers, and engines. The set is fixed after build. This is how Go's options pattern and Python's per-extension methods work.
- **All-or-nothing convenience.** The SDK **SHOULD** provide a default configuration that includes the standard seven extensions. Most peers want all of them. Selective enabling is for constrained environments (WASM with limited capabilities, minimal embedded peers).

Custom application handlers register through the builder alongside system extensions. They participate in the same dispatch, emit, and capability system. The difference: system extensions are defined by the extension specs; application handlers are defined by the application developer. See §11.3 for the handler registration contract.

**Connection grants** (`grants` in config) define what remote peers can do when they connect. These are the grants exchanged during the handshake (§2.2). The SDK is the interface for managing these grants — creating scoped grants for different connections, attenuating grants for delegation, revoking grants when needed. See §11 for the full capability management surface.

**Application grant.** The application code itself runs within a capability scope. The SDK holds the grant that authorizes the application's operations. The application can further attenuate and delegate within its scope but cannot escalate. This is how an application "knows the permissions the code is running at" — the SDK was configured with that grant at instantiation.

**Multi-peer.** The SDK **MUST** support creating multiple independent peers in the same process. Each peer has its own identity, storage, handlers, and grant context. Multi-peer is a runtime concern, not a protocol concern.

**Local-only mode.** The SDK **MUST** support peers that run without networking — no listener, no connections. This is the embedded/library/WASM deployment mode. The peer handles local operations only, via `execute()`. Extensions still work (emit pathway, history recording, query indexing). Networking operations (connect, listen) are unavailable.

### 8.2 close_peer

```
close_peer() → ()

  Graceful shutdown:
    1. Stop accepting new connections.
    2. Flush pending async deliveries (best-effort, with timeout).
    3. Close all active connections.
    4. Stop engines (subscription, continuation, etc.).
    5. Release storage resources.

  Errors:
    500  Shutdown error (partial cleanup).
```

Implementations **SHOULD** support a timeout parameter. Implementations **SHOULD** cancel in-flight operations with context cancellation.

### 8.3 peer_id

```
peer_id() → PeerID

  PeerID: Base58(key_type || hash_type || hash_bytes)
```

### 8.4 Diagnostics

```
entity_count() → uint      ; Total entities in content store
path_count() → uint         ; Total bindings in location index
```

Direct store queries, not handler-dispatched. Useful for UI status displays, health checks, and debugging. These bypass the dispatch chain — they're store-level introspection.

---

## 9. Discovery (L1)

### 9.1 discover_handlers

```
discover_handlers() → [HandlerInfo]

  HandlerInfo := {
    pattern:    string
    name:       string
    operations: [OperationInfo]
  }

  OperationInfo := {
    name:        string
    input_type:  string?
    output_type: string?
  }
```

**`HandlerInfo.pattern` semantics (normative).** The `pattern` field carries the manifest's advertisement pattern — which **MAY** include glob notation (e.g., `"system/type/constraint/*"`) per V7 §3.7. The dispatcher does NOT interpret this field; handler resolution is by V7 §6.6 longest-prefix walk against handler entities at literal prefix paths. Consumers building EXECUTE targets from `HandlerInfo.pattern` MUST handle the convention per V7 §6.6 / `GUIDE-EXTENSION-DEVELOPMENT.md` §4.9: strip trailing `/*` to obtain the dispatch prefix, or rely on the dispatcher's longest-prefix walk-back at execute time. The `pattern` is advertisement-only, NOT a dispatch URI.

Implemented as: `list("system/handler/")` + reading each manifest/interface entity. The SDK **SHOULD** provide this as a typed helper.

### 9.2 discover_types

```
discover_types() → [TypeInfo]

  TypeInfo := {
    type_path: string
    fields:    [FieldInfo]
  }

  FieldInfo := {
    name:     string
    type_ref: string
    optional: bool
  }
```

**Storage shape vs typed output (normative).** The schemas above describe the SDK's typed *caller-facing output* (`Array<FieldInfo>`). The *stored tree-entity shape* is distinct and pinned at `ENTITY-NATIVE-TYPE-SYSTEM.md §4.1` (`system/type`) and §4.2 (`system/type/field-spec`):

```
system/type.data := {
  name:        system/type/name,
  fields:      map_of system/type/field-spec    ; Map<field_name, field-spec>
  (extends, layout, type_params, type_args: optional)
}

system/type/field-spec := {
  ; exactly one of:
  type_ref:    system/type/name
  array_of:    system/type/field-spec
  map_of:      system/type/field-spec
  union_of:    array of system/type/field-spec
  type_param:  primitive/string
  ; plus modifiers: optional, default, key_type, type_args, byte_size
}
```

The stored entity's envelope `type` is the canonical meta-type `system/type` (ENTITY-NATIVE-TYPE-SYSTEM §4.1, §2.6, §4.4). The SDK reader (e.g., `entity-core-rust/bindings/sdk/src/sdk.rs:TypeInfo::from_entity`) reads the stored `Map<field_name, field-spec>` and synthesizes the typed `Array<FieldInfo>` for callers. Implementations writing type entities MUST use the storage shape per §4.1 / §4.2; writing in the typed-output shape (`Array<FieldInfo>`) produces structurally invalid type entities even if the bytes are valid CBOR.

The `array_of` / `map_of` / `union_of` keys in field-spec are part of the public alphabet (§4.2). The exactly-one-of invariant applies. Open-types semantics (`ENTITY-NATIVE-TYPE-SYSTEM.md §2.4`) permit additional keys to be present without rejection, BUT `ENTITY-NATIVE-TYPE-SYSTEM.md §2.5` normatively forbids documentation fields (`description`, `doc`, etc.) in type entity `data` to prevent content-hash divergence across implementations. Implementations needing per-field documentation MUST keep it in companion entities, NOT in the canonical type entity.

Implemented as: `list("system/type/")` + reading each type definition entity.

### 9.3 Cross-impl conformance (normative)

The `discover_handlers` and `discover_types` operations are SDK helpers over tree state — their typed outputs MUST conform to §9.1 / §9.2 across implementations. The following clauses pin behavior that has been observed-equivalent across the Rust / Go / Python SDKs and was previously implicit:

**Ordering.** Result ordering is implementation-defined. SDKs MAY sort (e.g., the Rust SDK sorts by `pattern` for stable UI rendering). Consumers MUST NOT depend on result ordering across SDK implementations. Consumers that require stable presentation MUST sort caller-side.

**Membership.** The returned set MUST include every `system/handler/interface` (resp. `system/type`) entity reachable under the caller's capability scope, modulo the V7 §6.6 advertisement-vs-registration-path convention. Handlers and types blocked by capability scope MUST NOT appear; their absence is not an error and is not signaled.

**Encoding equivalence.** Host-language-specific shaping at the SDK boundary (e.g., a Godot binding's `Array[Dictionary]` vs the Rust SDK's `Vec<HandlerInfo>` struct) is permitted. The contract is over the semantic content of the typed schema (§9.1 / §9.2 field names, types, optionality), not over the in-memory representation. Frontends building shared visual patterns MUST consume the typed schema, not the host-language representation. See `EXPLORATION-WIRE-ENCODING-AND-INTEGRATION.md` §6.2 on host-language frontend boundary integration as a recurring divergence class.

---

## 10. Protocol-First Default

The SDK's standard operations (§3-§6) route through handler dispatch. This is the default and recommended path because it makes local and remote peers interchangeable — `get("knowledge/articles/intro")` works the same on a local peer as on a remote peer.

The SDK **SHOULD** default to handler-dispatched operations. The SDK **MAY** also expose lower-level access (direct store reads, raw execute construction, content store operations) for advanced use cases, debugging, and extension development. The SDK doesn't prevent you from working at whatever level you need.

Default configuration **SHOULD** favor higher security and capability-scoped access. But the SDK is a tool for using the system — not a gatekeeper.

---

## 11. Capability Management (L1-L3)

The SDK's core job is managing grants and scoped access. This section covers what exists, what's needed, and what's not yet specified.

> **Identity-aware peers.** When the identity, attestation, quorum, role, or group extensions are registered, an additional SDK surface is available — bootstrap helpers, identity-stack operations, rotation lifecycle hooks, and `rotation_reissue_outstanding_grants`. See `sdk-domain/specs/SDK-IDENTITY-INFRASTRUCTURE.md` for the dedicated surface and `sdk-domain/guides/GUIDE-IDENTITY-SDK.md` for the application-developer walkthrough. The configuration-directory formalization (§15) covers the on-disk layout the helpers operate on.

### 11.1 Grant Lifecycle

**Self-grants during build.** The peer builder generates a capability grant for each handler at `system/capability/grants/{pattern}`. These authorize the peer's own operations.

**Connection grants.** On connect, peers exchange grants (§2.2). Four scope dimensions: handlers, operations, resources, peers. The SDK **SHOULD** support both static grant configuration (config files) and dynamic per-identity grant generation.

**The capability handler.** V7 §6.2 defines `system/capability` with operations: `request`, `delegate`, `revoke`. The SDK **SHOULD** expose these as typed operations.

### 11.2 What the SDK Needs to Expose

The SDK is the interface for applications to manage their capability context.

**GrantScope** — the four dimensions that define what a grant allows:

```
GrantScope := {
  handlers:   {include: [string], exclude: [string]?}   ; Handler patterns
  operations: {include: [string], exclude: [string]?}   ; Operation names
  resources:  {include: [string], exclude: [string]?}   ; Path patterns
  peers:      {include: [string], exclude: [string]?}   ; Peer ID patterns
}

; Each dimension has include (what's allowed) and optional exclude (what's denied).
; A dispatch must match all four dimensions from a single grant entry.
; This is the same structure as system/capability/scope in V7 §3.6.
```

**SDK operations for capability management:**

```
create_grant(scope: GrantScope, grantee?: PeerID) → Grant
delegate_grant(parent: Grant, attenuated_scope: GrantScope, grantee: PeerID) → Grant
revoke_grant(grant: Grant) → ()
inspect_grants(connection?: PeerID) → [GrantInfo]
```

These dispatch to the `system/capability` handler. An application instantiated with a scoped grant can further attenuate and delegate within its scope but cannot escalate beyond it.

### 11.2A Grant Progression

Capability management has a natural progression. Each level is independently useful.

**Level 0 — Open access.** Single grant entry, wildcard on all four dimensions. Development/single-developer use.

```
grants: [{
  handlers:   {include: ["*"]}
  operations: {include: ["*"]}
  resources:  {include: ["*"]}
  peers:      {include: ["*"]}
}]
```

**Level 1 — Per-handler entries.** Multiple grant entries, each scoped to specific handlers, operations, and resource paths. First production step.

```
grants: [
  ; Read-only tree access to public content
  {handlers: {include: ["system/tree"]},
   operations: {include: ["get", "list"]},
   resources: {include: ["public/*", "knowledge/*"]}},

  ; Query access
  {handlers: {include: ["system/query"]},
   operations: {include: ["find", "count"]},
   resources: {include: ["public/*", "knowledge/*"]}},

  ; Subscribe to changes
  {handlers: {include: ["system/subscription"]},
   operations: {include: ["subscribe", "unsubscribe"]},
   resources: {include: ["knowledge/*"]}}
]
```

**Grant handler operations, not `tree:put` to system namespaces.** For load-bearing entity types (continuation, subscription, role assignment, compute subgraph), grant the handler's create operation — e.g., `system/continuation:install` instead of `system/tree:put` on `system/continuation/suspended/*`. Handler operations validate embedded capabilities at creation time; `tree:put` bypasses this validation. See §2.7.2 for the full rationale.

Level 0→1 is configuration-only — change the grant entries in `grants.toml` or the builder config. No new SDK operations needed.

**Level 2 — Per-identity grant resolution.** Different connecting identities get different grants. The SDK needs a grant policy mechanism:

```
set_grant_policy(policy: GrantPolicy)

GrantPolicy := {
  default: [GrantEntry]         ; Fallback for unknown identities
  rules: [{
    match:  IdentityPattern     ; Peer ID pattern or group
    grants: [GrantEntry]
  }]
}
```

Level 1→2 needs one new SDK operation. Same grant entry structure — the policy just selects which entries a given connection receives.

**Level 3 — Runtime delegation.** Application receives a grant, attenuates it for components or other peers. Uses `delegate_grant`, `revoke_grant`, `inspect_grants` from §11.2. Each child grant entry must be covered by a parent grant entry (V7 §5.6 — no escalation). Child inherits all parent excludes and may add more.

### 11.2B Rotation Re-issuance Helper

Long-lived caps held by third parties die on issuer rotation. This is a direct consequence of V7 chain validity composed with identity rotation: when a runtime peer is retired or rotated, the local peer→Op cap that authorized its outstanding issuances is revoked, and downstream caps lose their authorizing chain on the next `is_revoked` walk. Signatures are immutable; caps don't "follow" identity changes, and the protocol does not silently re-issue them.

The mitigation is the **rotating peer's** responsibility, not the consuming extension's. The rotating peer holds an outstanding-grants tree (`system/capability/grants/...`) listing the caps it has issued; on rotation, it iterates that tree, identifies grants that should survive, re-issues each from the new authority, and pushes the re-issued caps to consuming peers via the consuming extension's normal flow (re-subscribe, inbox push, etc.). Consuming extensions stay rotation-agnostic — they see a normal "new cap arrived" flow.

**SDK helper:**

```
rotation_reissue_outstanding_grants(rotated_peer: PeerID, new_authority: Grant) → [Grant]
```

Iterates the rotating peer's outstanding-grants tree, identifies grants that should survive rotation (filtered by deployment policy — long-TTL grants, subscription deliver_tokens, inbox dispatch caps, continuation roots), re-issues each from `new_authority`, and emits the re-issued caps for delivery to consuming peers via the appropriate flow.

**Conformance.** SHOULD be provided by SDK implementations targeting deployments with long-lived cross-peer flows. SDKs targeting short-TTL deployments (where natural renewal cycles cover rotation events) MAY omit it. Filter policy is implementation-defined; see GUIDE-IDENTITY for deployment-pattern guidance.

### 11.3 Handler Execution Models

Every handler in the entity system is a `system/handler` entity in the tree, dispatched by longest-prefix match (V7 §6.6), capability-checked on every request. What differs is where the handler's logic lives and how it executes. Three models exist, forming a progression from compiled infrastructure to transferable logic.

**1. Precompiled system extensions.**

Bootstrap handlers (`system/tree`, `system/handler`, `system/type`, `system/protocol/connect`) are compiled into the peer binary and pre-loaded during startup before the capability system exists (V7 §6.9). System extensions (revision, subscription, history, query, clock, etc.) are also precompiled: handler code is in the binary, registered in emit-pipeline order (SYSTEM-COMPOSITION §2.2) during peer initialization. All system infrastructure uses this model. The logic is compiled, not inspectable from within the system, not transferable between peers.

**2. SDK-registered language-native handlers.**

After peer creation, application code registers handlers at runtime via the SDK's `register_handler` primitive (§11.6). The handler manifest goes into the tree (interface entity + handler entity + grant), and the handler body is a language-native callable (Go closure, Rust `Fn`, Python `async def`) bound in the in-memory dispatch index. This is how applications extend the system without recompiling the peer. The trade-off: the callable is ephemeral. On restart, the tree entries survive but the dispatch index is empty — the application must re-register (§11.6.6). See §11.6 for the full registration contract.

**3. Entity-native compute-backed handlers.**

The handler manifest includes `expression_path` (V7 §3.7). Dispatch evaluates the compute expression at that tree path instead of calling compiled code (V7 §6.6). The handler body is a content-addressed compute expression entity — transferable, inspectable, auditable, and survives peer restart without re-registration. Registration uses the standard `system/handler:register` protocol operation; no SDK primitive is needed because the body is already in the tree. Requires the compute extension (EXTENSION-COMPUTE). See GUIDE-COMPUTE.md for expression language details, reactive subgraphs, TCO, and library-as-entities patterns.

**Comparison:**

| Property | Precompiled | SDK language-native | Entity-native |
|---|---|---|---|
| Registration | Peer startup (binary) | `register_handler` (SDK §11.6) | `system/handler:register` (protocol) |
| Body location | Compiled binary | In-memory dispatch index | Tree (expression entity) |
| Transferable | No | No | Yes (content-addressed) |
| Inspectable | No | No | Yes (walk expression graph) |
| Hot-swappable | No (recompile) | Close handle + re-register | Replace expression at tree path |
| Restart survival | Yes (in binary) | No (re-register on startup) | Yes (expression in tree) |
| Auditable | Implementation trust | Implementation trust | Static analysis (compute install audit) |
| Performance | Native | Native | Interpreted (~5x overhead) |

**Choosing a model.** Precompiled for system infrastructure and extensions that need native performance. SDK language-native for application handlers that need compiled performance without recompiling the peer. Entity-native for handler logic that needs to be transferred between peers, audited by capability inspection, or updated without re-registration. `compute/apply` handler mode bridges entity-native expressions to language-native handlers when an entity-native handler needs native performance for a specific operation.

**Handler-context chain-root primitive (SEC-3).** For SDK language-native handlers that accept caller-provided capability references in their input and embed them in entities the handler creates (continuations, subscriptions, role assignments, compute installs, or any handler-created entity carrying an embedded cap), the handler context (`ctx`) **MUST** expose a chain-root check — `ctx.identity_in_authority_chain(cap_hash) → bool` or an equivalent named primitive. The handler uses it to validate that the EXECUTE author appears as a granter in the embedded cap's authority chain before persisting the entity. Without this primitive at the handler tier, the handler is forced to either (a) reconstruct the chain walk independently (fragile, error-prone) or (b) trust the caller (recreating Finding 3 at the application level — see GUIDE-CAPABILITIES §8.3). Entity-native handlers don't need the primitive directly: the compute install audit validates static-literal cap references at install time, and runtime-passed caps go through R1 at the underlying handler operation.

### 11.4 Handler Registration (Build Time)

Handlers register during peer build. The registration contract:

1. **Pattern** — the path prefix this handler serves (e.g., `local/files`).
2. **Scope declaration** — what tree paths and operations the handler needs (`internal_scope`). The builder generates a self-grant from this.
3. **Manifest** — a `system/handler` entity stored at the pattern path. Includes handler name and operation specs.
4. **Interface** — a `system/handler/interface` entity at `system/handler/{pattern}`. Lists operations with input/output types for discovery.
5. **Handle function** — receives execution context (peer identity, caller capability, handler grant, scoped tree access, execute function) and request (operation, params, resource).

The builder registers the handler, stores the manifest, generates the grant, and creates the interface entity. After build, the handler participates in dispatch via longest-prefix matching (V7 §6.6).

This is L3 — needed for application developers building custom handlers, not for those using standard extensions.

### 11.5 Watch Implementation

§6 defines `watch()` as an L1 operation with implementation-defined backend. When the subscription extension is registered, the recommended implementation: create a deliver token → call subscribe with `deliver_to` pointing at local inbox → listen for inbox notifications → deliver events through platform-native mechanism. When subscription is not registered, the emit pathway event stream or polling are acceptable backends.

For direct control over cross-peer subscriptions, deliver tokens, and limits, applications use the subscription extension via `execute("system/subscription", "subscribe", ...)` — see SDK-EXTENSION-OPERATIONS.md §3.

### 11.6 Dynamic Handler Registration

The entity system supports two kinds of handler bodies:

- **Entity-native** (compute-backed): the handler manifest includes `expression_path`; dispatch evaluates the compute expression. Registration via `system/handler:register` is complete — no SDK primitive needed.
- **Language-native** (implementation-specific): the handler body is a language-native callable (closure, function, coroutine) that cannot be entity-encoded. The SDK MUST couple the protocol-side declaration (tree entities) with the implementation-side binding (in-memory dispatch index).

This section specifies the SDK primitive for language-native handler registration.

SDKs that support runtime handler registration — registering handlers after `create_peer()` returns — MUST expose a `register_handler` primitive for the language-native case. Direct access to the underlying handler dispatch index MUST NOT be part of the SDK's public API surface. The dispatch index is an internal implementation detail; the SDK primitive is the only public mutation path.

**Signature (illustrative, language-idiomatic):**

```
register_handler(spec: HandlerSpec, body: HandlerBody) → Handle

  HandlerSpec := {
    pattern:        string           ; Bare pattern (e.g., "app/myapp/greeter").
                                     ; The SDK qualifies to /{peer_id}/{pattern} internally.
                                     ; A leading slash in the bare pattern is an error.
    name:           string           ; Display name for manifest + interface
    description:    string?          ; Human-readable, for discovery consumers
    operations:     [OperationSpec]  ; Operations this handler accepts
    internal_scope: [GrantEntry]?    ; Self-grant scope. Null = no outbound calls.
    types:          map<string, TypeDef>?
                                     ; Type definitions to install at system/type/*.
                                     ; For handlers that define custom operation types.
                                     ; Usually null.
  }

  HandlerBody: Language-native callable that receives HandlerContext,
               returns HandlerResult. See §11.6.3 for cross-language shapes.

  Handle: Opaque type whose close/drop/dispose unregisters both sides.
          See §11.6.2 for lifecycle.

  Errors:
    409  Pattern collision — a handler is already registered at this pattern.
    400  Invalid spec (empty pattern, empty operations list).
    500  Internal error (partial write failure after compensation).
```

**Relationship to `system/handler:register`.** The `HandlerSpec` fields map directly to the existing `system/handler/register-request` type (V7 §3.12): `pattern` and `operations` form the manifest; `internal_scope` maps to `requested_scope`; `types` maps to the `types` field. The tree entities this primitive writes are identical to the entities `system/handler:register` would write. The primitive adds one thing the protocol cannot specify: binding the language-native callable body.

**Registration level progression.** Currently, this primitive writes tree entities at L0 (direct store access, same as bootstrap). The architectural target is for the primitive to dispatch through `system/handler:register` (L1) for the declarative side, then bind the callable locally. The existing `register-request` / `register-result` types already support this. The progression:

- **V1.0:** L0 writes. Caller is peer-owner code. No capability check on registration.
- **V2.0 target:** Dispatches `system/handler:register` (L1, capability-checked) for tree entities, then binds the callable. Required for plugin systems and multi-tenant scenarios where registration itself needs authorization. Bootstrap handlers continue to use the L0 direct path (§11.3); V2.0 applies to dynamic registration after `create_peer()` returns.

#### 11.6.1 What `register_handler` writes

The SDK constructs a `system/handler/manifest` from the `HandlerSpec`, then decomposes it into two stored entities per V7 §6.2. Registration produces four mutations, in this order:

1. **Tree: interface entity** at `/{pid}/system/handler/{bare_pattern}`, type `"system/handler/interface"` (V7 §3.7). Contains pattern, name, and operations — the handler's public contract and single source of truth for its external description. Does NOT include `max_scope` or `internal_scope`. This is what `discover_handlers()` reads, `system/query` returns, and remote peers see. Written first because the handler entity references it by path.
2. **Tree: handler entity** at `/{pid}/{bare_pattern}`, type `"system/handler"` (V7 §3.7). Contains `interface` path reference (pointing to step 1's entity), `max_scope`, and `internal_scope`. This is the dispatch target — what tree walk (V7 §6.6) finds. Security configuration lives here, not on the interface, so it is not exposed to remote peers through discovery.
3. **Tree: grant entity** at `/{pid}/system/capability/grants/{bare_pattern}`, if `internal_scope` is non-null. Self-grant derived from the peer's root capability, attenuated to the declared scope. The tree binding — not the content-store put — is the declaration.
4. **Dispatch index: callable entry.** For language-native handlers, the SDK registers a wrapper in the implementation's dispatch index that delegates to the caller's body. For entity-native handlers (compute-backed, with `expression_path`), no dispatch index entry is needed — dispatch evaluates the expression from the tree.

If `types` is provided, type definitions are additionally written at `/{pid}/system/type/{type_name}` for each entry, before step 1.

**Ordering matters.** Tree first, dispatch index second. If the dispatch index write fails after tree writes succeed, the SDK MUST compensate by removing the tree entries (§11.6.4). The reverse ordering creates a window where dispatch reaches a handler the tree doesn't declare.

**Collision check.** Before any writes, `register_handler` MUST check whether a handler is already registered at the pattern (either in the dispatch index or the tree). If so, return 409. Silent overwrite is not permitted. Replacement requires explicit close (via handle) followed by `register_handler`.

**The authoritative pattern** is the one passed in `HandlerSpec.pattern`, not any pattern declared inside the body's manifest or interface.

#### 11.6.2 Handle lifecycle

`register_handler` returns a handle. The handle's close (or language-idiomatic equivalent) unregisters both sides:

1. **Dispatch index first** — stop accepting dispatch immediately.
2. **Tree entries second** — remove handler entity, interface entity, and grant binding. Type definitions installed via `types` are NOT removed — they have independent lifecycle and may be referenced by other handlers, queries, or remote peers.

Close MUST be idempotent. Repeated close is a no-op. In languages where value types are copyable, the SDK MUST return a handle whose semantics are reference-based or MUST make close idempotent with an internal closed-flag.

The SDK MUST provide both:
- An explicit close method (`handle.close()`, `handle.Close()`, `await handle.close()`).
- The most-idiomatic scoped construct the language has (`impl Drop` in Rust, `defer handle.Close()` in Go, `await using` / `async with` in TS/Python).

The SDK MUST NOT rely on garbage collection or finalizers for correctness. GC-based cleanup MAY be provided as a safety net but MUST NOT be the only cleanup path.

#### 11.6.3 Handler body contract

Dynamic handler bodies run under the same handler contract as bootstrap handlers. They receive a `HandlerContext` and return a `HandlerResult`. The SDK's job is to wrap language-native callables into that contract.

**Body concurrency.** The SDK does not guarantee serial invocation of handler bodies. Concurrent dispatches to the same handler pattern MAY invoke the body in parallel. Body implementers are responsible for their own synchronization. Single-threaded runtimes naturally serialize; this is acceptable but not a guarantee the spec makes.

**Cancellation.** Handler bodies MUST respect cancellation signals from their language-native runtime. SDKs MUST propagate cancellation from the dispatch-level context to the handler body's runtime primitive.

**Internal scope.** A dynamic handler registered without `internal_scope` (null) cannot call other handlers from its body. The SDK MUST NOT silently default to a wildcard grant — that is the capability equivalent of running as root. The caller declares intent; the SDK enforces it.

**Cross-language body shapes:**

| Language | Body shape | Capture semantics | Cleanup idiom |
|----------|-----------|-------------------|---------------|
| Rust | `Fn(&HandlerContext) -> Future<Result>` + `Send + Sync + 'static` | `Arc<T>` captures; `'static` required | `impl Drop` |
| Go | `func(context.Context, *HandlerContext) (HandlerResult, error)` | Closure captures by reference | `Close() error` + `defer` |
| TS/JS | `async (ctx: HandlerContext) => HandlerResult` | Closure retains by default | `await handle.close()` or `Symbol.asyncDispose` |
| Python | `async def handler(ctx: HandlerContext) -> HandlerResult` | Closure retains via scoping | `async with` or `await handle.close()` |

#### 11.6.4 Partial-failure compensation

If any write step in §11.6.1 fails, the SDK MUST compensate by removing all writes that succeeded before the failure. The SDK tracks writes-so-far and removes them in reverse order on failure.

```
tree write 1 (interface)  → OK     (tracked)
tree write 2 (handler)    → OK     (tracked)
tree write 3 (grant)      → FAIL
  → compensate: remove handler, remove interface
  → return error to caller
  → no handle returned — caller knows registration failed entirely
```

Compensation is best-effort — if a compensation removal itself fails, the SDK logs the orphaned entries and returns the original error. Orphaned tree entries without a dispatch index body are harmless (dispatch returns 404). Type definitions installed via `types` are NOT compensated — they have independent lifecycle (§11.6.2).

#### 11.6.5 Dispatchable vs non-dispatchable: the litmus test

The requirement "if it's dispatchable, it must be tree-declared" has a precise boundary:

> Can some piece of code — internal or external — cause this callback to fire by performing a dispatch to a path?

If **yes**: the callback is a handler in the protocol sense. It MUST be registered via `register_handler` and tree-declared.

If **no**: it is not a handler — SDK-internal callbacks, event-bridge listeners, channel receivers, in-process plumbing. These MUST NOT be registered in the dispatch index. They should be expressed as language-level constructs that are explicitly not reachable through dispatch.

**No middle ground.** No "registered in the dispatch index but hidden from the tree." That middle ground is the invariant violation this section exists to prevent.

#### 11.6.6 Peer restart behavior

Dynamic handlers are ephemeral — their callable bodies exist only in memory. On peer restart with a persistent tree:

- Tree entries from prior dynamic registrations may survive.
- The dispatch index is empty — no callable bodies exist.
- Dispatch to a surviving tree entry returns 404 (no dispatch index match).

The recommended approach: applications re-register dynamic handlers on startup. `register_handler` replays the tree writes (idempotent if the entries already exist). The tree stays truthful as long as registration is deterministic from application state.

SDKs MUST NOT automatically re-register handlers from surviving tree entries — the callable body is not recoverable from the tree. (For entity-native handlers with `expression_path`, the body IS in the tree and dispatch works without re-registration.)

#### 11.6.7 Runtime-instantiated handler placement

Handlers registered via `register_handler` can use any pattern the caller chooses — `app/`, domain-specific prefixes, or any path appropriate to the handler's purpose. No namespace restriction applies to application-owned dynamic handlers.

Handlers minted at runtime as system machinery (subscription delivery, continuation callbacks, etc.) live under one of two organizational patterns:

- **Within the owning extension's namespace.** When an extension owns the relevant concept, runtime-minted handlers live under `system/{ext}/...` — the layout below the extension prefix is the extension's prerogative. Common shapes:
  - Single-level: `system/subscription/{nonce}`, `system/continuation/{nonce}`.
  - Two-level by purpose: `system/subscription/delivery/{nonce}`, `system/inbox/sub-{id}`, `system/continuation/callback/{nonce}`.
  - Nested with caller-specified sub-namespace (when the extension exposes it): `system/subscription/delivery/{caller-tag}/{nonce}` — useful for organizational grouping. Whether to support caller-specified sub-namespacing is the extension's API decision.
- **Under `system/runtime/`.** A semantic category for *system-privileged, runtime-instantiated, unfitted-elsewhere* state, parallel to `app/{app-id}/` for application content. Three properties: system-privileged (lives under `system/...`); runtime-instantiated (per-call, ephemeral); unfitted-elsewhere (no single extension owns it, or it crosses extension boundaries).

Both patterns coexist. Whichever is used, the layout MUST be explicitly enumerated in the owning normative spec — extension specs document the layout (and any caller-specified sub-namespace parameters) they use; specs describing particular `system/runtime/` purposes enumerate sub-purposes.

Application code SHOULD NOT register handlers under `system/runtime/` or directly under another extension's namespace.

**Deprecation.** The previous reservation `system/sdk/{purpose}/{identifier}` is deprecated. Existing impls using `system/sdk/...` continue to function during a deprecation window; new machinery uses the per-extension or `system/runtime/` patterns above. (entity-core-go and entity-workbench-go never adopted `system/sdk/...`; entity-core-rust's egui-app uses it in `entity-sdk/src/subscription.rs` and migrates as part of `proposals/implemented/PROPOSAL-OPERATIONAL-STATE-AND-SDK-CONVERGENCE.md`.)

#### 11.6.8 Open Questions (Deferred)

1. **Serial dispatch opt-in.** §11.6.3 specifies that the SDK does not guarantee serial invocation. For subscription delivery handlers specifically, out-of-order events can produce incorrect state. A future `serial: bool` field on `HandlerSpec` (or equivalent) could let handlers opt into serialized dispatch. Deferred: the current non-guarantee is correct as the default.

2. **Owner attribution.** Should tree entries record who registered the handler? Useful for plugin revocation. Not needed for the initial primitive.

3. **Atomic replacement.** Is `replace_handler(spec, body)` needed? `unregister` then `register` is adequate for now.

4. **Handler status type.** A `system/handler/status` entity (state: active/declared/disabled, boot_id, bound_at) would help with restart-recovery and posture B precision.

5. **Delivery-token tree binding.** The subscription bridge mints a scoped capability token (delivery token) authorizing the subscription engine to dispatch `receive` onto the subscriber's inbox. This token is currently stored in the content store only — no tree binding. The same "floating entity" pattern that R fixes for handlers applies to these capabilities. Scope: either an addendum to §11.6, a subscription extension amendment, or a cross-cutting "SDK-minted capability lifecycle" proposal.

### 11.7 Tree-Gated Dispatch (Migration Target)

The entity system's architectural target is **tree-gated dispatch**: the handler resolution step verifies that a `system/handler/interface` entity exists at `/{pid}/system/handler/{bare_pattern}` before dispatching to a handler in the dispatch index. If absent, dispatch returns status 503 with error code `handler_undeclared`.

This is distinct from 404 (no handler matched): 503 means the handler IS in the dispatch index but is NOT declared in the tree.

Implementations SHOULD support a peer configuration flag:

```
require_tree_declared_handlers: bool  (default: false)
```

When `false`: dispatch index only. When `true`: tree-gated dispatch.

This section is informative — it describes the architectural target, not a current requirement. The requirement is §11.6 (SDK-enforced paired writes).

---

## 12. Error Model

### 12.1 Status Codes

| Code | Meaning | When |
|------|---------|------|
| 200 | Success | Normal completion |
| 202 | Accepted | Async delivery accepted; result will arrive via inbox |
| 207 | Partial success | Primary operation succeeded; emit consumer(s) failed |
| 303 | Redirect | Subscription redirect to another peer |
| 400 | Bad request | Invalid params, path, type, expression |
| 403 | Forbidden | Capability grant does not authorize this operation |
| 404 | Not found | No binding at path, or no handler matched |
| 409 | Conflict | CAS failure, merge conflict |
| 429 | Rate limited | Peer or handler capacity exceeded |
| 500 | Internal error | Unrecoverable failure |
| 501 | Not supported | Handler does not implement requested operation |
| 503 | Handler undeclared | Tree-gated dispatch (§11.6): handler in dispatch index but not declared in tree |

### 12.2 Error Entity

```
system/protocol/error := {
  fields: {
    status:  {type_ref: "primitive/uint"}
    code:    {type_ref: "primitive/string"}       ; Machine-readable
    message: {type_ref: "primitive/string"}       ; Human-readable
    details: {type_ref: "primitive/any", optional: true}
  }
}
```

### 12.3 SDK Error Mapping

The SDK **MUST** preserve status codes. How they surface is language-idiomatic (Go error returns, Rust Result, Python exceptions). The SDK **MUST NOT** collapse all errors into a single generic type.

The SDK **SHOULD** distinguish: client errors (400, 404, 409 — caller can fix), authorization errors (403 — grant problem), and system errors (500, 501 — something broke).

### 12.4 Status 207 (Partial Success)

Primary operation succeeded. Emit consumer(s) failed. Response includes normal result. Application code **SHOULD** log consumer errors. Application code **MUST NOT** treat 207 as failure.

### 12.5 Registration Error Codes

Error codes introduced by `register_handler` (§11.6):

| Status | Code string | Meaning |
|--------|-------------|---------|
| 409 | `pattern_collision` | A handler is already registered at this pattern |
| 400 | `invalid_handler_spec` | Spec is malformed (empty pattern, empty operations) |
| 500 | `partial_registration_failure` | Compensation succeeded but original write failed |
| 503 | `handler_undeclared` | Tree-gated dispatch (§11.6): handler in dispatch index but not declared in tree |

These code strings MUST be consistent across SDK implementations.

---

## 13. SDK Patterns (Advisory)

Scoped handles, entity-backed state, type rendering, reactive cycle patterns, connection lifecycle, and multi-peer composition are documented in `GUIDE-SDK-PATTERNS.md`. These are advisory patterns — recommended ways to use the operations defined in this spec.

---

## 14. Cross-Language Contracts

### 14.1 Content Identity

Same entity **MUST** produce same content hash. Requires: ECF canonical encoding (ENTITY-CBOR-ENCODING.md) + SHA-256 with format code (V7 §1.2).

Use `compare-types` or `validate-peer` to verify.

### 14.2 Type Definitions

All implementations **MUST** register the same core type definitions at `system/type/*`.

### 14.3 Extension Composition

Same extensions → same emit pipeline ordering (SYSTEM-COMPOSITION.md §2.2).

---

## 15. Configuration Directory

The on-disk layout for keypairs, peer configurations, and (when the identity extension is registered) identity bundles.

### 15.1 V7-only mode (legacy / minimal)

For peers without the identity extension installed:

```
~/.entity/
├── identities/{name}/          Ed25519 keypairs
│   ├── public_key
│   └── private_key             (restricted permissions)
├── peers/{name}/
│   ├── keypair
│   ├── config.toml             (listen address, storage, extensions list)
│   └── grants.toml             (connection-time grants)
```

The flat `identities/{name}/{public_key, private_key}` form is **legacy and load-bearing** — V7-only peers continue to use this layout indefinitely. Absence of `peers/{name}/identity.toml` signals V7-only mode.

### 15.2 Identity-aware mode

For peers running the identity extension, the `identities/{name}/` entry becomes a directory bundle. See `sdk-domain/specs/SDK-IDENTITY-INFRASTRUCTURE.md` §8.4 for the full layout including quorum-constituent custody state, controller/agent/identifier keypairs, identity metadata, and the per-peer `identity.toml` referencing the bundle.

**Three modes coexist:**
1. **V7-only** (§15.1) — flat keypair files; no identity extension; absence of `peers/{name}/identity.toml`.
2. **Identity-aware single-identifier** — bundle directory + `peers/{name}/identity.toml`. Default for identity-extension users.
3. **Multi-identity host** — multiple `peers/{name}/` directories on the same host, each referencing a different identity bundle. Per `EXTENSION-IDENTITY.md` §3.8, peer-configs MUST NOT share state across identities; structural separation enforces this.

### 15.3 Conformance

- Layout structure: **SHOULD** for cross-impl interop. Implementations converging on the layout means a Rust impl's identity bundle is operable by a Go peer-manager; users can move between impls' CLI tools without re-provisioning.
- Path naming: **SHOULD** (cross-impl tools should be able to operate on each other's bundles).
- File formats inside the bundle: implementation-defined as long as cross-impl tools can read at least the public components (public keys, peer IDs, attestation hashes).
- Override via env var or CLI flag: **MUST** be supported (operational deployments need this).
- Standardized `identity.toml` minimal core (when identity-aware): identity-side `{name, identifier_id, controller_id, quorum_id, schema_version}`; peer-side `{identity_name, trusts_quorum, agent_grants, identifiers}`. Implementations MAY extend with their own fields.

### 15.4 Migration

A V7-only peer becomes identity-aware via the `BootstrapFromExistingKeypair` helper (per `SDK-IDENTITY-INFRASTRUCTURE.md` §8.1): bundle is created in `identities/{name}/`, `peers/{name}/identity.toml` is added pointing at the bundle, the peer's keypair is unchanged (peer_id stable). No retroactive migration of contacts.

---

## 16. Conformance

### 16.1 MUST Implement

- §3.1-3.4: get, put, list, remove
- §4.1: execute (with local/remote routing per §2.7-2.8)
- §6.1-6.2: watch, unwatch (implementation-defined backend)
- §7.1: connect with full handshake and grant exchange
- §8.1: create_peer with capability bootstrap
- §11.6: `register_handler` primitive with tree-paired writes, handle lifecycle, collision semantics, compensation
- §12.1-12.3: Status code preservation and error mapping
- §12.5: Registration error code strings (cross-SDK consistency)
- §14.1-14.2: Content identity and type definitions

### 16.2 SHOULD Implement

- §2.7 Level 0: Direct store access with standing grant context and emit pathway
- §2.7.1: L0 use-case guidance
- §3.5: has
- §5.1: query (if EXTENSION-QUERY registered)
- §6.5: Notification access level boundary (three primitives at two levels)
- §7.2-7.3: listen, connected_peers
- §9.1-9.2: discover_handlers, discover_types as typed helpers
- §11.6: Tree-gated dispatch migration flag (`require_tree_declared_handlers`)
- §15: Standard configuration directory

### 16.3 Advisory (see guides/)

- Scoped handles — `GUIDE-SDK-PATTERNS.md` §1
- Entity-backed state — `GUIDE-SDK-PATTERNS.md` §2
- Type rendering — `GUIDE-SDK-PATTERNS.md` §3

### 16.4 Implementation-Defined

- API shape and naming conventions
- Subscription delivery mechanism
- Error type hierarchy
- Internal optimization strategies (polling, caching)
- List entry ordering
- Cross-path event ordering in watch

---

## 17. Open Questions

1. **Batch operations.** No `put_many`, `get_many`, `remove_many`. Sync workflows that merge hundreds of entities pay per-dispatch overhead for each one. Should the SDK spec batch operations, or is this an implementation optimization behind the existing API?

2. **Entity construction helpers.** `put(path, type, data)` takes raw data. How does application code construct data that conforms to a type definition? A `build_entity(type_name, fields)` that validates against `system/type/{type_name}` would complement the type rendering pattern (see `GUIDE-SDK-PATTERNS.md` §3) — rendering is display, construction is input.

3. **Version negotiation.** When two peers connect, what happens if they support different extension versions (e.g., revision v2.4 vs v2.1)? The handshake exchanges protocol versions but not extension versions. Should `discover_handlers` include version information?

4. ~~**Dynamic handler registration.**~~ Resolved — see §11.6.

---

## 18. Relationship to Other Specs

| Spec | Relationship |
|------|-------------|
| ENTITY-CORE-PROTOCOL.md | SDK operations wrap protocol dispatch. §5 capability system authorizes every operation. §6 handler model is the dispatch foundation. |
| SYSTEM-COMPOSITION.md | SDK put triggers emit pathway. SDK watch observes Phase 2. |
| EXTENSION-QUERY.md | SDK query dispatches to query handler. |
| EXTENSION-SUBSCRIPTION.md | SDK watch uses subscription extension. |
| ENTITY-CBOR-ENCODING.md | Entity construction uses ECF for content hashing. |

---

## Addendum A: SDK Specification Notation

### What the pseudocode in this document means

The code examples throughout this spec are **illustrative, not prescriptive.** They show the logical structure of operations — what goes in, what comes out, what the call looks like conceptually. They are not API definitions. Each language implementation translates these into its own idioms.

```
; This is a spec example:
peer.get("knowledge/articles/intro")

; A Go implementation might look like:
entity, err := executor.TreeGet("knowledge/articles/intro")

; A Rust implementation might look like:
let entity = peer_ctx.tree_get("knowledge/articles/intro");

; A Python implementation might look like:
entity = await peer.get("knowledge/articles/intro")

; A functional language might look like:
(peer-get peer "knowledge/articles/intro")
```

All of these implement the same operation: retrieve the entity at a path. The spec defines the semantics (what the operation does, what errors it returns, what guarantees it provides). The implementation defines the syntax.

### Notation conventions used in this spec

| Notation | Meaning |
|----------|---------|
| `operation(param: type) → ReturnType` | Operation signature — inputs and output |
| `TypeName := { field: type ; comment }` | Result type definition — fields and their types |
| `type?` | Optional (may be null/absent) |
| `[Type]` | Array/list of Type |
| `; comment` | Inline comment (consistent with CBOR diagnostic notation) |
| `peer.operation(...)` | Operation on a managed local peer (§2.8) |
| `peer.store.operation(...)` | Level 0 direct store access (§2.7) |
| `peer.execute(handler, op, params)` | Level 1/2 handler dispatch (§2.7) |
| `entity(type, data)` | Construct an entity with the given type and data |
| `{include: [...], exclude: [...]}` | Capability scope dimension (V7 §3.6) |

### What's normative vs what's guidance

**Normative (must be consistent across implementations):**
- Operation semantics — what each operation does, under what conditions, with what guarantees
- Error conditions — which status codes are returned and when
- System guarantees — emit pathway fires on every mutation, capability context is always present, content identity (same entity = same hash)
- Extension composition — same extensions registered = same emit pipeline ordering
- These derive from the core protocol spec (ENTITY-CORE-PROTOCOL.md), SYSTEM-COMPOSITION.md, and the extension specs. The SDK spec formalizes how they surface to application code.

**Guidance (language-idiomatic, expected to vary):**
- Method names, parameter order, return types
- Error representation (exceptions, result types, error returns)
- Builder patterns (method chaining, options, fluent)
- Async model (callbacks, futures, channels, signals)
- Naming of access levels (store.get vs direct_get vs raw_get)
- Extension wrapper namespacing (peer.revision.commit vs peer.execute("system/revision", "commit"))

An implementation is conformant if the normative behavior matches. How it looks is up to the language.

### Where to look for normative definitions

The SDK spec describes patterns of use. The normative foundations live in the specs below. When there's ambiguity in the SDK spec, these are authoritative:

| Concern | Normative source | What it defines |
|---------|-----------------|-----------------|
| **Entity structure, types, hashing** | ENTITY-CORE-PROTOCOL.md §1-§2 | Entity := {type, data, content_hash}. Type definitions at system/type/*. Hash = SHA-256 with format code. |
| **Canonical encoding (ECF)** | ENTITY-CBOR-ENCODING.md | CBOR canonical form. Same entity bytes → same hash in every language. L0 algorithm library — must be vendored per language, must produce identical output. |
| **Capability system** | ENTITY-CORE-PROTOCOL.md §5 | Grant structure, four scope dimensions, chain verification, attenuation rules, revocation model. |
| **Handler model and dispatch** | ENTITY-CORE-PROTOCOL.md §6 | Handler registration, longest-prefix matching, dispatch chain, execution context, authority model. |
| **Connection handshake** | ENTITY-CORE-PROTOCOL.md §4 | Hello/authenticate exchange, pre-authorization rules, initial capability delivery. |
| **Wire format** | ENTITY-CORE-PROTOCOL.md §1.6 | Envelope structure, CBOR framing, length-prefixed messages. |
| **Emit pathway and extension composition** | SYSTEM-COMPOSITION.md | Consumer ordering, cascade depth, two-phase delivery (sync + async), convergence guarantees. |
| **Tree operations** | EXTENSION-TREE.md | Snapshot (trie), diff, merge, extract. View trees (capability-scoped projections). |
| **Individual extensions** | EXTENSION-*.md (in specs/extensions/) | Each extension's handler operations, types, algorithms, and conformance requirements. |
| **Cryptographic algorithms** | ENTITY-CORE-PROTOCOL.md §7, ENTITY-CBOR-ENCODING.md | Ed25519 signatures, SHA-256, Base58 peer ID encoding. L0 algorithms — shared across all layers. |

The SDK wraps these into an application-facing interface. The types referenced in SDK operation signatures (Entity, Hash, PeerID, GrantScope) are defined in the core protocol spec. The algorithms (hashing, signing, encoding) are in the L0 specs. The handler behavior behind each `execute()` call is in the extension specs.

---

## 19. Document History

- **v1.10:** Godot intake clarifications per the Godot intake-clarifications proposal. §2.7 L1-default / L0-carve-out pin paragraph (Amendment G); §9.1 `HandlerInfo.pattern` advertisement-only semantics cross-ref to V7 §6.6 + GUIDE-EXTENSION-DEVELOPMENT §4.9 (Amendment D); §9.2 storage-shape-vs-typed-output cross-ref to ENTITY-NATIVE-TYPE-SYSTEM §4.1 / §4.2 + §2.5 documentation-field prohibition (Amendment F); new §9.3 cross-impl conformance subsection — ordering, membership, encoding equivalence (Amendment E). Companion Amendments A + B touch core-protocol-domain (V7 §3.7; EXTENSION-QUERY §4.1 / §5.4) and remain Workstream 2 sign-off. No SDK API redesign. Trigger: Godot β-track-close intake (commit `eae8d4b`) — Q1 / Q3 / Q4 / D2 discipline.
- **v1.6:** Absorbed IA26 from PROPOSAL-IDENTITY-ARC-FIXES. New §11.2B "Rotation Re-issuance Helper" — `rotation_reissue_outstanding_grants(rotated_peer, new_authority)` SDK helper for the rotating peer to re-issue still-needed long-lived caps from a new authority on rotation, so consuming extensions (subscription, inbox, continuation, compute) stay rotation-agnostic. Documents the cross-cutting property (long-lived third-party-held caps die on issuer rotation by V7 chain validity + rotation) and assigns mitigation to the rotating peer. Conformance: SHOULD for SDKs targeting deployments with long-lived cross-peer flows. Source: PROPOSAL-IDENTITY-ARC-FIXES IA26.
- **v1.5:** Coherent capability authority alignment. New §2.7.2 "Kernel-vs-handler principle" — normative guidance that application grants should cover handler create operations (continuation install, subscription subscribe, role assign, compute install) rather than raw `tree:put` to handler-managed namespaces. §11.2A Level 1: added load-bearing type grant guidance. Source: PROPOSAL-COHERENT-CAPABILITY-AUTHORITY (adopted), PROPOSAL-SDK-AND-GUIDE-COMPUTE-ALIGNMENT.
- **v1.4:** PROPOSAL-COMPUTE-APPLY-RESOURCE-CEILING H1: §11.6 subsection renumbering — subsections previously at §11.5.x renumbered to §11.6.x to match the §11.5→§11.6 section rename from PROPOSAL-HANDLER-EXECUTION-MODELS (v1.3). No content changes; mechanical renumbering only. Internal cross-references to §11.5.x updated to §11.6.x. Header `Depends:` bumped: V7 v7.20+ → v7.30+, EXTENSION-COMPUTE v3.2+ → v3.10+ (reflects landed extension-spec dependencies).
- **v1.3:** PROPOSAL-HANDLER-EXECUTION-MODELS H1: new §11.1 "Handler Execution Models" overview covering all three handler models (precompiled system extensions, SDK-registered language-native handlers, entity-native compute-backed handlers). Existing §11 subsections renumbered: §11.1→§11.2, §11.2→§11.3, §11.3→§11.4, §11.4→§11.5, §11.5→§11.6. GUIDE-COMPUTE.md §9 trimmed to remove the system-level overview (which now lives here), kept compute-specific content (§9.1-§9.4). Source: PROPOSAL-HANDLER-EXECUTION-MODELS.md.
- **v1.2:** Dynamic handler registration (§11.5 — now §11.6 after v1.3 renumber), tree-gated dispatch target (§11.6 — now §11.7), registration error codes (§12.5), notification access levels (§6.5), L0 use-case taxonomy (§2.7.1), 503 status code, deferred questions (§11.5.8 — now §11.6.8). §11.5.1 (now §11.6.1) updated to post-normalization handler shape (handler entity + interface entity, per PROPOSAL-HANDLER-NORMALIZATION N1-N8). Resolved open question Q4. Source: PROPOSAL-SDK-HANDLER-REGISTRATION-AND-NOTIFICATION-BOUNDARY, PROPOSAL-HANDLER-NORMALIZATION.
- **v1.1:** Review fixes. Corrected handshake to two-round-trip HELLO→AUTHENTICATE. Added 202 status code, `included` on Response, `close_peer`, diagnostics section. Fixed watch pattern syntax to match subscription spec. Fixed delivery drop semantics. Removed duplicate §11.2 paragraph. Added open questions (batch, entity construction, version negotiation, dynamic handlers).
- **v1.0:** Initial draft. Formalizes converged operations from Go, Rust, and Python implementations. Source: `SDK-SYNTHESIS.md` cross-project analysis.
