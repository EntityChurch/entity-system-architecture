# SDK Extension Operations — Draft

**Version**: 0.9
**Status**: Active
**Depends**: SDK-OPERATIONS.md (v1.7+), ENTITY-CORE-PROTOCOL.md (v7.40+), SYSTEM-COMPOSITION.md (v1.5+), EXTENSION-COMPUTE.md (v3.18+); SDK-IDENTITY-INFRASTRUCTURE.md (v0.1+) for the identity stack

---

> **Relationship to SDK-OPERATIONS.md.** That spec covers the base layer — tree ops, execute, query, watch, connect, peer lifecycle. This document covers the extension operations that become available when extensions are registered. What extensions your peer has determines what operations are available. The peer builder composes this.

> **Status.** Working draft. Covers all system extensions defined in the spec suite.

---

## 1. How Extension Operations Work

Every extension operation is an `execute()` call to the extension's handler. The SDK wraps these into ergonomic APIs. When you register the revision extension, you get `commit()`, `merge()`, `log()`, etc. When you don't register it, those operations don't exist.

```
; Without SDK wrapper:
peer.execute("system/revision", "commit", {prefix: "knowledge/"})

; With SDK wrapper (same dispatch, better ergonomics):
peer.revision.commit("knowledge/")
```

The SDK's job is to provide typed, discoverable wrappers. The underlying mechanism is always `execute(handler_pattern, operation, params)`.

**Capability scoping applies.** Extension operations go through the same dispatch chain as tree operations (SDK-OPERATIONS.md §2.3). The application's grant must cover the handler pattern and operation.

---

## 2. Continuation Extension

**Handler:** `system/continuation`
**What it does:** Durable async execution chains. Request goes out, result comes back to inbox, continuation advances to next step.

> **Coherent Capability (v0.3).** Continuation entities carry a `dispatch_capability` field that authorizes future dispatches. Use `system/continuation:install` to create them — the install operation validates the embedded capability against the writer's authority chain. Direct `tree:put` of continuation entities is reserved for system-extension use. See EXTENSION-CONTINUATION §1.X Coherent Capability.

### Operations

**install** — Create a continuation entity with validated dispatch capability.
```
install(params: InstallParams) → InstallResult

  Creates a continuation entity at the specified path. Validates
  dispatch_capability: the writer's identity must be in the embedded
  cap's authority chain (R1 chain-root check). Persists the cap and
  its chain to the local content store.

  InstallParams := {
    path:                 string        ; Path under system/continuation/suspended/*
    target_uri:           string        ; Handler to dispatch to on advance
    operation:            string        ; Operation name
    resource_target:      object?       ; Resource scope for the dispatch
    params_template:      any?          ; Params template for the dispatch
    dispatch_capability:  hash          ; Cap token authorizing the future dispatch
    on_error:             object?       ; Error delivery spec
    remaining_executions: uint?         ; Max advance count
  }

  InstallResult := {
    status: uint
    path:   string                      ; Path where continuation was installed
  }
```

The same shape applies to `system/continuation/join` (for fan-in joins) via a parallel `install-join` operation or a discriminator field.

**advance** — Move a continuation forward.
```
advance(continuation_id: string, result: Entity) → AdvanceResult

  Dispatches to the continuation's target handler, injecting the result.
  Handles fan-in joins (wait for multiple results before advancing).

  AdvanceResult := {
    status:    uint
    next:      string?     ; Next continuation ID if chain continues
    completed: bool        ; Whether the chain terminated
  }
```

**resume** — Retry a suspended continuation.
```
resume(continuation_id: string) → Status

  Reconstructs a suspended continuation and re-dispatches with fresh bounds.
  For manual retry of failed operations.
```

**abandon** — Clean up a pending continuation.
```
abandon(continuation_id: string) → Status

  Deletes the suspended continuation entity and cleans up delivery state.
```

### Forward continuation assembly modes

A forward continuation assembles dispatch params from three components: the static `params` template, the post-transform result value, and the assembly mode.

| Mode | Triggered by | Behavior |
|---|---|---|
| **Pass-through** | `result_field` absent, `result_merge` absent | post-transform value replaces `params` directly |
| **Inject** | `result_field: "name"` | post-transform value nested under `name` in `params` |
| **Merge** (v1.16) | `result_merge: true` | post-transform value (**MUST** be a map) shallow-merged into `params` at top level; result keys win on collision |

Merge mode is the assembly path for `op(static_scaffold, dyn1, dyn2, …)` — Inject can only place one dynamic value under one key; Merge places several flat. `result_field` and `result_merge` are mutually exclusive — `install` rejects with `400 invalid_continuation` if both are set. A non-map value under `result_merge` degrades to static-only params and binds a `merge_value_not_map` lost-error marker per EXTENSION-CONTINUATION §3.4.

### Structural-transform continuations

In addition to forward-dispatch continuations (`install` shape above), the continuation extension defines structural-transform continuations (`system/continuation/transform`) that apply a closed, total, pure, bounded set of `transform_ops` to a step's result before passing it to the next step. Transforms are the declarative leg of chain expressibility — they let plain continuation chains express cross-peer-notification → local-path rewrites, key projections, and envelope navigation without inserting opaque handler steps.

The closed admissibility contract (total / pure / bounded / statically analyzable) is in EXTENSION-CONTINUATION §2.2. Unrecognized `op` is rejected at install with `400 unknown_transform_op`. SDKs **SHOULD** expose the transform vocabulary alongside the install surface.

Notable ops (the closed set grows on demonstrated need under the same admissibility contract):

**collect_keys** (v1.15) — Projects a map's keys into an array.
```
{op: "collect_keys", field: "<path>", into: "<dest>"}
        OR
{op: "collect_keys", fields: ["<path1>", "<path2>"], into: "<dest>"}
```
Singular form projects one map's keys; plural form concatenates keys from multiple maps in list order. Used to thread a diff result's path-keyed map into a `tree:extract.paths` array for incremental sync chains. Mutually exclusive `field` / `fields` — install rejects with `400 invalid_transform_args`.

**deref_included** (v1.17) — Reads `field` as a `system/hash` reference and replaces it with the entity bound to that hash in the **envelope's `included` map**.
```
{op: "deref_included", field: "<path>"}
```
Pure in-flight navigation — the entity was bundled by the sender per V7 §3.1 — not a tree/store read. Used to consume an `include_payload`-bundled entity (EXTENSION-SUBSCRIPTION §2.2) from a plain continuation chain without an opaque handler step. Requires the request-side envelope-`included` preservation invariant (V7 §3.3 v7.51; SDK-OPERATIONS §4.1.3) so `included` reaches the continuation.

### The Reactive Chain Pattern

Continuations are the backbone of async workflows. The common pattern:

```
1. Application sends EXECUTE to remote peer
2. Remote peer processes asynchronously
3. Result delivered to local peer's inbox (system/inbox receive)
4. Inbox handler triggers continuation advance
5. Continuation dispatches to next step
6. → loop until chain completes
```

This is how subscribe → notify → process → respond works across peers. The SDK should make building these chains ergonomic:

```
; Build a continuation chain (L3 — pipeline builder, not yet specified)
pipeline = peer.continuation.chain()
    .then("system/revision", "fetch", {peer: remote_id, prefix: "knowledge/"})
    .then("system/revision", "merge", {strategy: "three-way"})
    .on_error(log_and_retry)
    .build()

peer.continuation.start(pipeline)
```

Cross-peer mirror chain using the v1.15/v1.16/v1.17 transform vocabulary:

```
; subscribe + include_payload → deref_included → tree:put with CAS
pipeline = peer.continuation.chain()
    .subscribe(remote_id, "knowledge/", include_payload=True)
    .transform(deref_included(field="hash"), result_field="entity")
    .then("system/tree", "put", {path: "$notification.path",
                                 expected_hash: "$notification.previous_hash"},
                                result_merge=True)
    .build()
```

The chain stays declarative end-to-end — no opaque handler step is needed because `deref_included` resolves the bundled entity from the envelope's `included`, and `result_merge` flattens the dynamic notification fields into the static `tree:put` params.

**Status:** The pipeline builder API above is aspirational (L3/L4). The raw operations (advance, resume, abandon) are L1 — they work through execute().

---

## 3. Subscription Extension

**Handler:** `system/subscription`
**What it does:** Push notifications on tree changes matching a pattern.

> **Coherent Capability (v0.3).** The subscribe handler validates the `deliver_token`'s authority chain against the subscriber's identity (R1 chain-root check). The subscriber can only use deliver tokens whose chain includes their own identity — tokens they issued or hold via legitimate delegation. This prevents an actor from referencing another peer's deliver token to force notifications to that peer's inbox. See EXTENSION-SUBSCRIPTION §1.X Coherent Capability.

> **Three capabilities in a subscribe flow.** A subscribe request involves three distinct capabilities: (1) the **caller capability** (outer EXECUTE) authorizing the subscribe operation itself, rooted at the subscribee's peer; (2) the **subscription deliver_token** (`params.deliver_token`) authorizing future async delivery to the subscriber's inbox, rooted at the subscriber's peer; (3) the **inbox EXECUTE-level deliver_token** (per-delivery) authorizing a specific async delivery. In cross-peer scenarios, the caller capability and deliver_token root at different peers. See EXTENSION-SUBSCRIPTION §2.X for details.

### Operations

**subscribe** — Register for change notifications.
```
subscribe(params: SubscribeParams) → SubscriptionInfo

  SubscribeParams := {
    pattern:         string       ; Path pattern (exact match or prefix/*)
    events:          [string]?    ; Event types to filter ("put", "remove"); null = all
    deliver_to:      string       ; URI where notifications go (usually inbox)
    deliver_token:   hash         ; Capability token authorizing inbox delivery
    include_payload: bool?        ; Bundle changed entity in notification's included (default false; v3.14)
    limits: {
      max_events:      uint?     ; Stop after N events
      max_duration_ms:  uint?    ; Stop after N ms
      rate_limit:      uint?     ; Max events per second
    }?
  }

  SubscriptionInfo := {
    subscription_id: string
    pattern:         string
    events:          [string]
    limits:          object       ; Merged (server may tighten client limits)
  }
```

**`include_payload` requires `tree:get` authorization (normative, EXTENSION-SUBSCRIPTION §2.3 v3.13).** When set, the subscribe handler verifies the caller's capability covers `tree:get` on the subscribed resource. Rejected with `403 payload_unauthorized` if the caller holds `subscribe` but not `get`. The check moves enforcement from a subscriber-side `tree:get` pull to a server-side check before the content is pushed — net authorization is identical. A caller with `subscribe` but not `get` still receives lean (hashes-only) notifications by omitting `include_payload`.

**Semantics (EXTENSION-SUBSCRIPTION §2.2 v3.14).** Server bundles the **direct entity** at `notification.hash` (no closure) into the notification envelope's `included`. Removed events bundle nothing. Source-side resolution failure delivers hash-only + debug-log; the receiver MAY fall back to GET. Additive / opt-in — default `false` preserves the lean hashes-only delivery shape; existing subscribers unaffected.

**unsubscribe** — Cancel a subscription.
```
unsubscribe(subscription_id: string) → Status
```

### Cross-Peer Subscription Flow

```
1. Peer A connects to Peer B (grant exchange)
2. Peer A subscribes on Peer B: pattern "knowledge/*", deliver_to "entity://peer_a/system/inbox/sub1",
                                 include_payload=true (requires tree:get cap on Peer B's knowledge/*)
3. Peer B stores subscription + deliver token + include_payload flag
4. Something writes to Peer B's knowledge/ tree
5. Subscription engine matches, builds notification with the changed entity in `included`
6. Notification dispatched to Peer A via connection (remote EXECUTE)
7. Peer A's inbox receives notification + the bundled entity
8. Continuation chain applies: deref_included → tree:put with CAS (expected_hash = notification.previous_hash,
                               or zero-hash for created events — see SDK-OPERATIONS §3.2 put_cas)
```

This is the convergent-mirror recipe (`PROPOSAL-CONVERGENT-MIRRORING` in `proposals/implemented/`). With `include_payload`, the subscriber needs no follow-up cross-peer GET — the entity is delivered alongside the notification, and the local `put_cas` with the threaded `previous_hash` (or zero-hash for bootstrap) prevents stale-lap amplification. Without `include_payload`, the flow is unchanged: hashes-only notification, subscriber pulls via `tree:get` on demand.

### SDK Watch Wrapper

SDK-OPERATIONS.md §6 defines `watch(pattern) → ChangeStream` as the ergonomic wrapper. Internally:

1. SDK creates a deliver token scoped to the subscription
2. SDK calls `subscribe` with deliver_to pointing at local inbox
3. SDK listens for inbox notifications matching the subscription
4. SDK delivers events through platform-native mechanism (channel, callback, signal)
5. On `unwatch`, SDK calls `unsubscribe` and cleans up

---

## 4. Revision Extension

**Handler:** `system/revision`
**What it does:** Structural version control for entity trees. Content-addressed snapshots, three-way merge, cross-peer sync.

> **Important:** Version entries are purely structural — `{root: hash, parents: [hash]}`. No message, author, or timestamp in the version entry itself. The version DAG is a content-addressed data structure, not a commit log. Application-level metadata (messages, authors) can be stored as separate entities at convention paths if needed, but the revision handler does not manage them.

> **Deletions are explicit canonical-entity marks (v3.1).** Bound paths in a version's trie are explicit entries; a path that is "deleted" between versions has its trie entry bound to the canonical `system/deletion-marker` hash (`ecf-sha256:689ae4679f69f006e4bf7cb7c7a9155d0de5fb9fe31e81692dca5769eda9e0a6`; ENTITY-NATIVE-TYPE-SYSTEM §4.9). Three-way merge treats deletion markers as ordinary entity hashes; the merge-config `deletion_resolution` strategy (see `merge-config` op below) governs the deletion-vs-edit case. Deletion markers MUST NOT appear in the live location index — version-transcription apply (merge / fast-forward / checkout) translates marker bindings to live-tree unbinds.
>
> **Backwards-compatibility warning:** pre-v3.1 versions that expressed deletes via absence will reanimate historically-deleted paths under post-v3.1 merge semantics. Production deployments with material delete history require one-time version-rewrite migration before cross-peer interop. Prototype deployments unaffected.

### Handler Operations

These are actual handler operations — each dispatches via `execute("system/revision", op, params)`.

**commit** — Snapshot current tree state as a version.
```
commit(prefix: string) → CommitResult

  Builds a Merkle trie from current tree state under prefix.
  Creates a structural version entry {root, parents} pointing to the trie root.
  Updates HEAD pointer at system/revision/head/{prefix}.

  CommitResult := {
    version: hash            ; Hash of the new version entry
    root:    hash            ; Root of the snapshot trie
    parent:  hash?           ; SDK-local convenience (previous HEAD); NOT a wire-entity field
  }
  ; Wire entity is system/revision/commit-result {version, root} per
  ; EXTENSION-REVISION §4.3.1 (authoritative for the wire shape). Field names
  ; here MUST match those wire keys — prior version_hash/trie_root labels were a
  ; spec-vs-spec contradiction (cross-impl F4).
```

**log** — Walk the version DAG.
```
log(prefix: string, count?: uint, cursor?: hash) → LogResult

  LogResult := {
    versions: [{
      hash:    hash          ; Version entry hash
      root:    hash          ; Trie root hash
      parents: [hash]        ; Parent version hashes (sorted)
    }]
    cursor:   hash?          ; For pagination
  }
```

**status** — Current revision state.
```
status(prefix: string) → RevisionStatus

  RevisionStatus := {
    head:    hash?           ; Current HEAD version hash
  }
```

**merge** — Three-way merge.
```
merge(prefix: string, theirs: hash, strategy?: string) → MergeResult

  Finds common ancestor, computes diffs, applies changes.
  Type-aware: delegates per-type merge to type handlers when configured.

  MergeResult := {
    status:    uint          ; 200 clean, 409 conflicts
    version:   hash?         ; New merge version (if clean) — matches EXTENSION-REVISION §4.3.4 {status, version}
    conflicts: [{path: string, base: hash?, ours: hash?, theirs: hash?}]
  }
```

**resolve** — Resolve a merge conflict.
```
resolve(prefix: string, path: string, resolution: Entity) → Status
```

**fetch** — Get version DAG from remote peer.
```
fetch(peer: PeerID, prefix: string) → FetchResult

  Retrieves version entries and trie nodes.
  Doesn't fetch entity content — just the structure.

  FetchResult := {
    versions:   [VersionEntry]
    trie_nodes: [TrieNode]
  }
```

**fetch-entities** — Get entities by hash from remote peer.
```
fetch_entities(peer: PeerID, hashes: [hash]) → FetchEntitiesResult

  Retrieves actual entity content for hashes discovered during fetch.
  Scoped to the fetched snapshot — can't request arbitrary hashes.
```

**config** — Set revision configuration for a prefix.
```
config(prefix: string, settings: RevisionConfig) → Status

  RevisionConfig := {
    exclude_patterns: [string]?    ; Paths to exclude from versioning
    merge_strategy:   string?      ; Default merge strategy
    auto_version:     bool?        ; Enable/disable auto-versioning
  }
```

**fetch-diff** (v3.4) — Incremental content transport for cross-peer revision-follow.
```
fetch_diff(prefix: string, base: hash) → DiffEnvelope

  Returns the changed closure between caller's `base` version and the
  handler peer's current head, bundled as a `system/envelope`. The target
  version is implicit (handler peer's head) — the op is single-dynamic-
  field, so a standing continuation chain can drive it via:

    subscribe head → fetch-diff(prefix, base=$notification.previous_hash) → tree:merge

  Pass base = zero-hash for a full-closure bootstrap.

  DiffEnvelope := system/envelope        ; The changed-closure bundle.

  Errors:
    400  invalid_params           Bad prefix or base shape.
    403  capability_denied        Chain not granted fetch-diff for this prefix.
    404  no_local_state           Handler peer has no state for this prefix.
    404  base_not_found           Base hash not known locally.
    400  base_not_a_version       Base hash exists but is not a version entry.
```

This is the V-1 closure feature — cross-peer follow at any tree size, declaratively expressed. Validated end-to-end by workbench-go POC. Pairs with the convergent-mirror recipe under Subscription §3 above.

**merge-config** (v3.3) — Canonical write path for the merge-config namespace.
```
merge_config(path: string?, type: string?, config: MergeConfig) → Status

  Writes a per-path or per-type merge-config entry under the handler-owned
  namespace system/revision/config/merge/{path,type}/*. Enforces the
  strategy-rejection contract at config-write time.

  MergeConfig := {
    strategy:             string?     ; e.g. "three-way", "ours", "theirs", "deterministic"
    deletion_resolution:  string?     ; "preserve-on-conflict" (default),
                                      ;  "deletion-wins", "three-way-fallthrough",
                                      ;  "deterministic"
                                      ;  ("lww" and "keep-both" rejected at write time)
    ; other strategy-specific fields
  }

  Errors:
    400  invalid_strategy         deletion_resolution is "lww" or "keep-both" (per §2.3).
    403  capability_denied
```

**`system/revision/config/merge/{path,type}/*` is a handler-owned namespace.** The canonical write path is `merge-config` — `system/tree:put` on this namespace bypasses the handler's strategy-rejection contract and is a deployment misconfiguration, not a protocol-supported write path. SDK capability presets **MUST NOT** include `system/tree:put` on this namespace by default. See `GUIDE-CAPABILITIES.md §8.1` and `EXTENSION-REVISION §2.3`.

### SDK Orchestration (not handler operations)

These compose multiple handler dispatches and/or cross-peer operations. They are SDK-level workflows, not single `execute()` calls. Implementations **SHOULD** provide them.

**pull** — Fetch and merge remote versions.
```
pull(peer: PeerID, prefix: string) → MergeResult
  Orchestrates: connect → fetch(peer, prefix) → fetch_entities(peer, hashes) → merge(prefix, theirs).
  Multi-step, multi-peer.
```

**push** — Send local versions to a remote peer.
```
push(peer: PeerID, prefix: string) → PushResult
  Orchestrates: local status → extract envelope → remote peer merge.
  Multi-step, multi-peer.
```

**diff** — Compare two versions.
```
diff(prefix: string, from: hash, to: hash) → [Change]
  Delegates to tree trie diff (SDK-OPERATIONS.md §3.7).
```

**cherry-pick** — Apply a single version's changes.
```
cherry_pick(prefix: string, version: hash) → CommitResult
  Orchestrates: diff(version.parent, version) → apply delta → commit.
```

**revert** — Undo a version's changes.
```
revert(prefix: string, version: hash) → CommitResult
  Orchestrates: inverse diff → apply → commit.
```

**find-ancestor** — Find common ancestor of two versions.
```
find_ancestor(a: hash, b: hash) → hash?
  Bilateral BFS on version DAG. Returns null if no common ancestor.
```

### Tree Data (not handler operations)

These are tree reads/writes at convention paths. The SDK **MAY** provide typed helpers.

```
; Branches — tree entities at system/revision/branches/{name}
put("system/revision/branches/{name}", "system/revision/branch", {head: version_hash})

; Tags — tree entities at system/revision/tags/{name}
put("system/revision/tags/{name}", "system/revision/tag", {version: version_hash})

; Checkout — read trie, batch-write bindings (destructive — overwrites current tree state)
; SDK SHOULD warn if uncommitted changes exist under prefix
```

### Sync Pattern (Continuation + Subscription + Revision)

The full sync workflow composes three extensions:

```
1. Peer A subscribes to Peer B's revision changes (subscription)
2. Peer B commits new content (revision)
3. Peer B's subscription engine notifies Peer A (notification → inbox)
4. Peer A's continuation chain triggers: fetch → fetch-entities → merge (revision)
5. Peer A's tree is now up to date
```

This is the auto-sync pattern documented in GUIDE-REVISION-AUTO-VERSION.md.

---

## 5. History Extension

**Handler:** `system/history`
**What it does:** Per-path transition history. Every tree write at a path records what changed.

### Operations

> **Name collision note.** The history handler's operation is `query`, same name as the query extension's `find`. At the dispatch level there's no collision (different handlers), but SDK wrappers should disambiguate: `peer.history.query(path)` vs `peer.query.find(expr)`.

**query** — Get history for a path.
```
history_query(path: string, count?: uint, cursor?: hash) → HistoryResult

  HistoryResult := {
    transitions: [{
      hash:          hash
      previous_hash: hash?
      operation:     string      ; "put" | "remove"
      author:        PeerID?
      timestamp:     uint
    }]
    cursor: hash?                ; For pagination
  }
```

**rollback** — Restore a path to a previous state.
```
rollback(path: string, target_hash: hash) → RollbackResult

  Validates target_hash exists in the path's history chain.
  Binds the path to the target entity.
  Records the rollback as a new transition.
```

---

## 6. Query Extension

**Handler:** `system/query`
**What it does:** Find entities by type, field values, path patterns, references.

### Operations

**find** — Search for entities.
```
find(expression: QueryExpression) → QueryResult

  QueryExpression := {
    type_filter:  string?        ; Type glob pattern
    field_filters: [{
      field:    string
      operator: string           ; eq, neq, in, exists, gt, lt, gte, lte, prefix, substring, contains
      value:    any
    }]?
    path_filter:  string?        ; Path prefix
    ref_filter:   hash?          ; Entities referencing this hash
    order_by:     string?        ; Field to sort by
    limit:        uint?
    cursor:       string?        ; Pagination
  }

  QueryResult := {
    entries: [{path: string, content_hash: hash, entity?: Entity}]
    cursor:  string?
    total:   uint?
  }
```

**count** — Count matching entities.
```
count(expression: QueryExpression) → uint
```

### SDK Query Builder (L3)

```
results = peer.query
    .type("app/state/setting")
    .field("category", eq="display")
    .path_prefix("app/browser/settings/")
    .limit(50)
    .find()
```

---

## 7. Inbox Extension

**Handler:** `system/inbox`
**What it does:** Receives async deliveries. The mailbox for notifications, continuation results, and cross-peer messages.

### Operations

**receive** — Accept a delivery.
```
receive(delivery: Entity) → Status

  Stores delivery in tree (write-ahead at system/inbox/{path}).
  If continuation registered for this delivery: triggers advance.
  On successful advance: cleans up inbox entry.
```

The inbox is primarily consumed by other extensions (subscription notifications land here, continuation results land here). Direct application use is less common, but the SDK should expose inbox contents for inspection:

```
; List pending inbox items
peer.list("system/inbox/")

; Read a specific delivery
peer.get("system/inbox/some/delivery")
```

---

## 7A. Transaction Extension

**Handler:** `system/transaction`
**What it does:** Multi-path atomic operations — groups multiple tree bindings into a single logical unit with pre-commit validation, observation boundaries, and compensating rollback.

### Operations

**execute** — Single-shot transaction. Apply a group of bindings atomically.
```
execute(intent: TransactionIntent) → TransactionResult

  TransactionIntent := {
    bindings:    [{path, hash, expected_hash?}]  ; Ordered, sorted by path
    isolation:   string?     ; "read-committed" (default) | "snapshot" | "serializable"
    rollback_on_failure: bool?  ; Compensate on partial failure (default: false)
  }

  TransactionResult := {
    transaction_id: string
    status:         string   ; "committed" | "partial" | "rolled-back" | "rejected"
    applied:        [path]
    failed:         [{path, code, message}]?
    snapshot_root:  hash?    ; Trie root at begin (snapshot/serializable)
    commit_root:    hash?    ; Trie root after all bindings applied
  }
```

**begin** — Start an interactive transaction.
```
begin(isolation?: string) → TransactionState

  TransactionState := {
    transaction_id: string
    isolation:      string
    snapshot_root:  hash?    ; Captured trie root (snapshot/serializable)
  }
```

**write** — Stage a binding in an interactive transaction.
```
write(transaction_id: string, binding: TransactionBinding) → Status
```

**read** — Read within a transaction's isolation level.
```
read(transaction_id: string, path: string) → Entity?

  Under snapshot isolation: reads against the trie root captured at begin.
  Under serializable: same, plus tracks the read for commit-time verification.
  Under read-committed: reads the live tree.
```

**commit** — Commit an interactive transaction.
```
commit(transaction_id: string) → TransactionResult

  Under serializable: verifies nothing in the read set changed since begin.
  If verification fails: status "rejected", no bindings applied.
```

**rollback** — Abort an interactive transaction.
```
rollback(transaction_id: string) → Status
```

### When to use transactions

- **Multi-path consistency.** Two or more bindings that must agree (config + tracking state, entity + index entry).
- **Batch operations.** Applying N changes where observers should see all-or-nothing.
- **Read-then-write.** Read state A, decide what to write to B, guarantee A didn't change.

You don't need transactions for: single-path writes (tree.put is already atomic per path with CAS), append-only content-store writes, or handler operations that already coordinate internally (revision/commit, revision/merge).

### Composition with revision

Transaction + revision compose naturally: the transaction groups writes, revision snapshots the result. A continuation pipeline can automate: begin transaction → apply bindings → commit → revision commit. See GUIDE-TRANSACTION.md §7 for the pattern.

---

## 8. Compute Extension

**Handler:** `system/compute/*`
**What it does:** Entity-native computation. Install expressions that reactively evaluate when dependencies change.

> **Coherent Capability (v0.3).** Compute subgraphs and the expressions they reference are load-bearing: their `installation_grant`, `authorized_data_hashes`, and embedded `compute/apply.capability` fields authorize future evaluations and dispatches. Use `system/compute:install` to create subgraphs. The install operation audits the expression graph, validates literal capability references against the installer's authority chain (chain-root check), and persists subgraph metadata under the handler's own grant. See EXTENSION-COMPUTE §1.X Coherent Capability.

> **Standard-IR floor (EXTENSION-COMPUTE v3.18).** The expression surface now includes three additional core inline types — `compute/index`, `compute/length`, `compute/numeric-cast` — and the collection stdlib `map`/`filter`/`fold` is **MUST-given-COMPUTE** with spec-pinned argument types (`system/compute/{map,filter,fold}-args`). Integer arithmetic follows the pinned WASM/LLVM/JVM model (sign-agnostic `add`/`sub`/`mul`, signed-default `div`/`mod`/`compare`, point-of-use casts). An expression builder (when specified here) covers these. See GUIDE-COMPUTE §4.4 / §10.2 and GUIDE-COMPUTE-PROGRAMMING §2.3 for the authoring patterns.

### Operations

**eval** — Evaluate an expression.
```
eval(expression: Entity) → any

  Expression types: literal, lookup, apply, if, let, lambda;
    inline: arithmetic, compare, logic, field, construct, index, length, numeric-cast;
    collection builtins: map, filter, fold (MUST-given-COMPUTE).
  Content-addressed memoization — same expression + same inputs = cached result.
```

**install** — Register a reactive computation.
```
install(params: InstallParams) → InstallResult

  Installs a compute subgraph. The install operation:
  1. Audits the expression graph (cycle detection, closure env walk,
     impure operation collection)
  2. Checks capabilities (caller's grant covers all read_paths,
     handler_targets, write_paths, data_hashes)
  3. Chain-root validates static-literal compute/apply.capability
     references against the installer's authority chain
  4. Creates subgraph metadata with installation grant hash
  5. Registers reactive dependencies on compute/lookup/tree paths

  InstallParams := {
    root_expression_path: string  ; Path to the root expression entity
    result_path:          string? ; Where results are written (default: {root}/result)
  }

  InstallResult := {
    status:          uint
    installation_id: string
  }
```

**uninstall** — Remove a reactive computation.
```
uninstall(installation_id: string) → Status
```

### Security model (v3.10)

**Installation grant.** The caller's capability at install time becomes the installation grant — the authorization ceiling for all future re-evaluations. If revoked, the subgraph freezes.

**Dual-check for `compute/apply.capability`.** When an expression dispatches with a caller-provided capability override, both the handler grant AND the provided capability must cover the target (handler + operation + resource). The `compute/apply.resource` field MUST be present when `capability` is present — without it, the resource-level ceiling is unenforced.

**Chain-root on static literals.** At install audit, static-literal `compute/apply.capability` references are validated: the installer's identity must appear in the embedded cap's authority chain. Dynamic capabilities (computed at eval time) are checked at runtime via the dual-check.

See GUIDE-COMPUTE.md §4.3 for the full security narrative; EXTENSION-COMPUTE.md §3.3 and §4.1 for normative pseudocode.

Class T transferability (running entity-native programs on any peer) depends on this extension.

### Expression builder (E7 — reference design)

Building compute IR by hand (the bottom-up `put_expr` pattern, GUIDE-COMPUTE-PROGRAMMING §2) is verbose. The expression **builder** is the SDK's typed surface for emitting IR: one constructor per expression type, composed in memory, with a terminal `Build` that puts the graph bottom-up. The builder's *semantics* are extension-level (defined here); its *shape* is language-specific (a per-language appendix). The reference shape below is workbench-go's shipped, tested S1 design — the first compute-authoring builder in the ecosystem, and therefore the cross-impl reference. Rust/Python builders SHOULD mirror it, in particular the four normative author-notes that follow.

```
// One node in the in-memory expression DAG. Constructors return a node; nesting composes.
// Terminal Build() topologically sorts and puts entities, returning the root content hash.

// Leaves — the three impurity tiers are explicit in the surface (a frontend/analyzer
// asks "which impurity tier?" at every junction; the builder must make it answerable).
Literal(value)                         // value node
LookupScope(name)                      // pure   — scope.{name}
LookupTree(path, relative)             // IMPURE — live tree read; a reactive edge
LookupHash(hash, pathHint, relative)   // pure   — content-addressed, fixed

// Inline (unary/binary)
Field(target, name)        Index(array, index)        Length(array)
NumericCast(value, toType) Arithmetic(op, l, r)       Compare(op, l, r)   Logic(op, l, r)

// Complex
Construct(entityType, fields{name→node})
Apply(path, operation, args{name→node}, ...ApplyOption)   // HANDLER mode — WithCapability / WithResource
ApplyClosure(fnNode, args{name→node})  // CLOSURE mode — fnNode produces a compute/closure (LookupHash/LookupTree of a stored lambda); emits compute/apply with `fn`, not `path`. The recursion/self-reference primitive (EXTENSION-COMPUTE §2.2 closure mode + §4.x TCO).
BuiltinsCall(builtin, args)            // sugar over Apply("system/compute/builtins/{builtin}", ...)
If(cond, then, else)   Let(bindings{name→node}, body)   Lambda(params[], body)

// Terminal
Build(rootPath) → (rootHash, error)

// Registration / install — distinct surfaces (see below)
RegisterComputeHandler(spec, exprNode) → HandlerHandle     // dispatches system/handler:register
InstallComputeSubgraph(exprNode, resultPath, ...opt) → SubgraphHandle  // dispatches system/compute:install

// Bookends
PrimitiveAny(value) → Entity           // build a primitive/any params/return entity
UnwrapComputeResult(response) → (value, exprHash, error)   // strip the SA-4 compute/result envelope
```

**Four normative author-notes (the cross-impl alignment points).** A conforming builder MUST encode these — they turn runtime/semantic surprises into build-time errors:

1. **Rule 11 — refuse `NumericCast` as a `Let` binding.** `compute/numeric-cast` is eager and consumed by the immediately-following operation; binding it via `Let` (or routing it through `If`, `Construct`, scope) silently drops the unsigned intent (EXTENSION-COMPUTE §2.2 rule 11). The builder MUST reject a `Let` binding whose value is a `NumericCast`, citing Rule 11, and direct the author to inline the cast at the operand site. (GUIDE-CORE-COMPUTATIONAL-ARCHITECTURE §9.)
2. **F5 — `WithCapability` requires `WithResource`.** An `Apply` carrying a capability override but no resource is a structural violation (PROPOSAL-COMPUTE-APPLY-RESOURCE-CEILING §F5; rejected at install and eval). The builder MUST reject it at `Build` time, before any dispatch.
3. **Lambda produces an *expression*, never a pre-evaluated closure.** There is no closure constructor. A `compute/closure` is a runtime value; storing one as a builtin argument is non-portable. `BuiltinsCall("map", {fn: …})` takes a `Lambda` *expression* node.
4. **Kernel-vs-handler — never `tree:put` a handler/subgraph.** `RegisterComputeHandler` dispatches `system/handler:register` (R0 atomic: manifest + grant together); `InstallComputeSubgraph` dispatches `system/compute:install` (audit + grant + dependency capture). `Build` writes only the *expression tree* (pure data). Bypassing these via `tree:put` leaves the grant entity missing → 403 fail-closed, or bypasses R1 chain-root validation on embedded static-literal capabilities. The builder MUST route handler/subgraph creation through the proper ops, never a raw put. (SDK-OPERATIONS §11.2A.)

**`Register` vs `Install` are distinct, not interchangeable.** `RegisterComputeHandler` creates an entity-native *handler* dispatched on demand per request (request/response shape — chain steps, drop-down targets). `InstallComputeSubgraph` creates a *reactive subgraph* that re-evaluates when tree dependencies change, writing to `resultPath` (derived state / computed views). Plain one-shot `compute:eval` is reachable through the base execute surface and needs no builder method.

**Build-side properties to preserve.** `Apply` args are canonically sorted by the CBOR encoder (length-then-lex), so identical logical expressions hash identically regardless of construction order — the precondition for Stage-2 memoization correctness; a builder relying on the standard encoder gets this for free. `Build` always emits source compute entities to the tree (never compiled-only output) — the preserve-IR invariant (GUIDE-CORE-COMPUTATIONAL-ARCHITECTURE §6).

Per-language shapes (Go: `ap.Compute()` method returning a builder; Rust/Python idioms TBD by those tracks) live in the per-language SDK guides; the surface and the four author-notes above are the shared contract. Reference implementation: `entity-workbench-go/entitysdk/compute_builder.go` (+ `compute_helpers.go`, `register_compute_handler.go`).

### Lowering toolkit (E7.1 — the layer above the builder)

The builder (E7) is typed *IR assembly*; a frontend would not hand-author programs at that level. The **lowering toolkit** is the layer above it (GUIDE-CORE-COMPUTATIONAL-ARCHITECTURE §8 layer 2, §9): the small set of reusable functional decompositions every frontend compiles down to. Workbench-go shipped and tested the first one (`entitysdk/compute_lower.go`); it is the cross-impl reference, and Rust/Python SHOULD mirror its shape. The decompositions:

```
LowerArithmetic(cb, intent, op, l, r)   // intent ∈ {Signed, Unsigned}; Rule-11 cast inlined for div/mod
LowerCompare(cb, intent, op, l, r)      // signed-default; Unsigned inlines numeric-cast at the operand site
LowerFold(cb, collection, initial, step(acc, elem) → body)   // loop-over-collection (the common iteration)
LowerFilter(cb, collection, pred(elem) → body)               // single-param; absorbs the F11 arg-name
LowerMap(cb, collection, xform(elem) → body)                 // single-param transform
LowerRecord(cb, entityType, fields{name → node|value})       // Construct sugar; auto-wraps bare values
LowerRecurse(cb, params[], body(self, params...) → node)     // recursion/TCO via ApplyClosure self-reference (the pure-hash fixpoint)
LowerMatch(cb, value, tagField, arms{tag → bind(inner) → node}, default)  // sum-type discrimination → if/eq chain on a .data tag; binds the matched value
```

**The load-bearing principle: the toolkit is *sugar, not transformation*.** Each decomposition emits IR **byte-identical** to the hand-written builder form (workbench's `LowerRecord` equivalence test asserts this). A frontend lowering through the toolkit produces exactly what hand-assembly would — which is what keeps the IR canonical (dedup-correct, hash-stable) regardless of which layer authored it. This is the §8/§9 claim made concrete: lowering functional source to this IR is *mechanical*, the known functional-compiler middle-end, not novel design.

**The two former gaps — now resolved (`PROPOSAL-COMPUTE-RECURSION-AND-SUM-TYPES.md`), both smaller than they looked:**
- **`LowerRecurse` / TCO** — *no spec change.* `compute/apply` already has closure mode and the evaluator already does TCO; the only gap was the builder exposing it. The `ApplyClosure(fnNode, args)` constructor (above) closes it; `LowerRecurse` is the decomposition (the pure-hash fixpoint, §8.2/§9). Ships now.
- **Sum-type discrimination (4-B)** — *no spec change for the common case.* Two corrections reshaped it: the type system has **no `union_of`** (so there is nothing to "bind to"), and `compute/field` reads only `.data` (not the entity `type`). The entity-native answer: lower a variant to an entity carrying a `.data` tag, and `match` to an `if`/`eq` chain on it (`LowerMatch`, above) — unblocks Rust `enum`/`Result`/`Option` now. A first-class **`compute/match` primitive (+ `compute/type-of`)** and a **type-system `union_of`** are pinned as **v3.20-candidate / deferred**, gated on a real driver (static exhaustiveness, or discriminating on the entity `type` rather than a `.data` tag) — not built ahead of it. See the proposal §3 for the analysis and the workbench/Rust validation hand-off.

### Typed conveniences (E7.2 — SHOULD-provide ergonomic helpers)

Battle-testing the builder + toolkit through real workbench surfaces (the `compute aggregate` verb; the multi-step continuation chain; the reactive-install POC, per the closed-out compute-foundation feedback) surfaced a small set of recurring decode/dispatch boilerplate. These are **ergonomic, not semantic** — the spec primitives are sufficient; the helpers remove friction. An SDK **SHOULD** provide them; names stay implementation-flavored, **semantics align**:

| Helper | Removes | Note |
|---|---|---|
| `UnwrapComputeResultAsMap` / `…AsList` | the two-shape (`map[string]any` vs `map[any]any`) decode + key-conversion dance after a compute eval | extends the generic `UnwrapComputeResult` (E7) with the common record/array shapes |
| `IsHandlerRegistered(pattern)` (or `Status(pattern)`) | coupling a caller to the internal expression-path layout to detect prior registration | internal layout stays impl-private behind the query |
| `UnwrapChainStepDelivery(reqParams) → (status, innerValue, err)` | the trampoline envelope-peel (chain delivery wraps the prior result as `InboxDeliveryData`, then the **full entity envelope** `{type,data,content_hash}` inside `.result`) | **also a doc gap:** the double-wrap wire shape is not obvious from `EXTENSION-CONTINUATION` — worth a one-paragraph clarification in `GUIDE-CROSS-PEER-MESSAGING` |
| `LookupTreeLocal(path)` | hand-qualifying `/{peerID}/{path}` for every local-peer `compute/lookup/tree` dep | the existing `LookupTree(path, relative)` stays for the cross-peer case |

> **`LookupTreeLocal` — the highest-leverage one; analyzed → a clean spec-ambiguity fix (`PROPOSAL-COMPUTE-LOOKUP-TREE-LOCAL-QUALIFICATION.md`).** The footgun: `compute/lookup/tree` stores `Path` **verbatim** (so reactive dep-tracking keys on the literal string) while the tree layer canonicalizes bare paths — so a bare-path dep silently never recomputes. The spec is *ambiguous*, not conflicted: §1.4 (line 11) says unqualified paths resolve to the local peer, but §4.1's `relative`-false wording says "used as-is." Resolution: pin §1.4's rule — a bare `lookup/tree` path with `relative` false/absent is **local-peer-qualified** for resolution *and* dep-tracking. **No collision with the transferable relative-path story** — that is the explicit `relative: true` → subgraph-root flag, and cross-peer refs are explicit *absolute* paths; both orthogonal to the bare case (the earlier "collision, defer" read was wrong). Non-breaking. This helper stays as ergonomic sugar on top of the fix.

### L3 SDK wrapper signatures (S4)

SDKs **SHOULD** expose typed L3 wrappers around the L1 `execute(...)` plumbing for the three top-level compute operations. The wrappers are thin — they marshal typed arguments into the `params` map and unwrap the result — but pinning the **signature shape across impls** prevents each impl team from inventing a different surface for the same operation. Names follow language idiom; semantics align.

| Wrapper | L1 equivalent | Typed surface (semantic) |
|---|---|---|
| `eval(expression_hash, scope?, budget?, depth?) → ComputeResult` | `execute("system/compute", "eval", {expression: hash, scope?, budget?, depth?})` | One-shot evaluation; returns the result value (or a propagated `compute/error` at status 200 per §7.1). |
| `install(root_hash, result_path, triggers?, grant?) → SubgraphHandle` | `execute("system/compute", "install", {root: hash, result_path, triggers?, grant?})` | Install a reactive subgraph; the engine wires triggers and writes results to `result_path`. The returned handle exposes the subgraph path for inspection / `uninstall`. |
| `uninstall(subgraph_path) → ()` | `execute("system/compute", "uninstall", {subgraph_path})` | Tear down a reactive subgraph; reverses install (removes triggers, leaves the most recent result in place per the extension spec). |

These wrappers are L3 ergonomic surface; they do not change the protocol contract. Impls **MAY** add typed overloads (e.g., `install_with_grant`, `eval_in_scope`) — the three above are the convergence baseline. See `GUIDE-COMPUTE-PROGRAMMING.md` for usage patterns.

---

## 9. Network Extension

**Handler:** `system/network`
**What it does:** Peer lifecycle management — keepalive, reconnection, subscription restoration.

### Operations

**maintain-peer** — Start managing a peer relationship.
```
maintain_peer(peer_id: PeerID, address: string) → MaintainResult

  Creates continuation graph for: connect → keepalive → reconnect on failure.
  Sets up subscription restoration after reconnection.
```

**release-peer** — Stop managing a peer relationship.
```
release_peer(peer_id: PeerID) → Status
```

**status** — Get network state.
```
network_status() → NetworkStatus

  NetworkStatus := {
    peers: [{
      peer_id:         PeerID
      state:           string     ; "connected" | "reconnecting" | "failed"
      pending_deliveries: uint
      subscriptions:   uint
    }]
  }
```

**close** — Graceful connection close.
```
close(peer_id: PeerID, reason?: string) → Status
```

This extension automates the reliability layer — keepalive, reconnection, subscription restoration — that would otherwise be manual.

---

## 10. Clock Extension

**Handler:** `system/clock`
**What it does:** Logical/physical time tracking. Contributes clock state to the execution context.

### Operations

**now** — Get current clock state.
```
clock_now() → ClockState

  ClockState := {
    mode:      string     ; "physical" | "logical" | "hybrid" | "vector"
    timestamp: uint?      ; Physical time (ms since epoch)
    logical:   uint?      ; Logical counter
    vector:    map?       ; Vector clock entries
  }
```

**compare** — Compare two clock values.
```
clock_compare(a: ClockState, b: ClockState) → "before" | "after" | "concurrent" | "equal"
```

---

## 11. Content Extension

**Handler:** `system/content/*`
**What it does:** Large content chunking and retrieval; cap-checked closure completion.

### Operations

**get** — Retrieve content chunks by hash.
```
content_get(hashes: [hash]) → ContentResponse

  Returns found/missing chunk map (per CONTENT v3.6 §6.2 spec-literal
  arrays + envelope-Included delivery). Access-gated by capability.
```

Chunking and ingest are handler-specific (files handler chunks files, media handler chunks media). The content extension provides the shared retrieval and storage layer.

### SDK Closure-Completion Surface (paired with the content-materialization-first-class proposal, Amendment A)

The content extension exposes one SDK-level affordance over its `system/content:get` op: **closure completion** — a cap-checked sequencer that drains `missing` until the requested blob's full closure (blob entity + every chunk it references) is locally present in the content store.

#### `EnsureClosure` — the load-bearing primitive

```
content.EnsureClosure(ctx, dispatcher, hash, namespace) → error

  Sequences cap-checked system/content:get dispatches until the closure
  (blob + all referenced chunks) is locally present.

  Parameters:
    ctx        — language-idiomatic cancellation / deadline context
    dispatcher — Dispatcher (per §11.SDK below); satisfied by both
                 outer-caller (AppPeer) and handler-internal (HandlerContext)
                 shapes; for handler-cross-peer use content.AtPeer
    hash       — the blob hash to drive closure for (system/hash)
    namespace  — cap-scope target; the namespace prefix the cap covers
                 (per CONTENT v3.6 §6.4 cap-matrix). Default for the
                 system content handler when no sub-namespace is in play:
                 "system/content"

  Returns:
    nil      on closure-complete locally
    403      cap denial on any sub-dispatch (per V7 §5.2)
    404      missing entity not expected to arrive (sync-state-visibility
             predicate per CONTENT v3.6 §3.4)
    503      blob_pending_sync; closure-incomplete; caller retries on next
             sync event per the partial-sync taxonomy
```

**Behavior — pure cap-checked sequencer:**

1. **Blob check.** If the blob entity (at `hash`) is not in the local content store, dispatch `system/content:get` against `entity://{peer}/{namespace}` with `params: {hashes: [hash]}`. On success, store the blob locally from `envelope.included`.
2. **Enumerate.** Read the blob's `data.chunks` array.
3. **Drain.** For each chunk hash not in the local content store, batch into windows of `GET_BATCH_SIZE` (initial value 16 per CONTENT v3.6 §7.1) and dispatch `system/content:get` per batch. Store fetched chunks locally from `envelope.included`.
4. **Loop on `missing`.** If `response.missing` is non-empty (e.g., due to frame-budget overflow per CONTENT v3.6 §6.2 receiver-side discipline), retry the missing batch.
5. **503 retry policy (Amendment per Go's post-landing audit Finding 1; Rust + Go converged after independent diagnosis).** On `503 blob_pending_sync` response to any sub-dispatch (blob fetch OR chunk fetch), implementations **SHOULD** retry up to `MAX_PENDING_SYNC_RETRIES` (recommended initial value 3) before propagating the 503 to the caller. The retry applies symmetrically to blob and chunk paths — the early-blob-fetch failure case must not short-circuit the retry policy with a premature 404. Each retry SHOULD inspect the `pending` sidecar (per CONTENT v3.6 Amendment 2 §6.2) to confirm the hash is still classified as sync-pending; if the receiver flips it to terminal between retries, propagate as 404 to the caller.
6. **Return.** On full closure-completion, return success. On unrecoverable failure (404 / repeated 503 past `MAX_PENDING_SYNC_RETRIES` / 403), return error matching the partial-sync taxonomy.

**Key property — entity-shape end-to-end.** `EnsureClosure` does NOT return bytes. The SDK contract stays in entity-and-hash terms. Byte extraction is a separate local concern (§Reassembly below).

#### `AtPeer` — peer-aimed Dispatcher (cross-peer-from-handler)

```
content.AtPeer(handler_ctx, source_peer_id) → Dispatcher

  Returns a peer-aimed Dispatcher derived from a HandlerContext + target
  peer ID. Use when a handler running on peer B needs to dispatch
  system/content:get cross-peer against peer A's namespace (e.g.,
  subscription-driven cross-peer materialization per workbench's
  Stage 3 case 1.5 chain shape).

  The namespace argument to EnsureClosure stays purely a cap-scope concept
  (which namespace prefix the cap is granted on); peer authority is
  the Dispatcher's concern.
```

#### Dispatcher interface

Both `AppPeer` (outer-caller / cross-peer) and `HandlerContext` (handler-internal) satisfy a shared `Dispatcher` interface:

```
Dispatcher.Execute(ctx, ExecuteRequest) → (ExecuteResponse, error)

  ExecuteRequest carries: URI, Operation, Resource (path-as-resource per
  V7 §3.2), Params (ECF-encoded), Capability (per V7 §6.8
  propagated-cap-not-a-gate).

  ExecuteResponse carries: Result (ECF-encoded), Included map (per V7
  §3.3 v7.51 envelope-included preservation).
```

This unifies the §7.2 closure-fetch algorithm into one implementation per impl. Per-language form differs; the contract is the same.

**Placement is per-impl (descriptive, not normative — per Go's post-landing audit Finding 3).** This spec describes the SDK affordance (`EnsureClosure` + `AtPeer` + Dispatcher contract); placement in each impl's module tree is per-impl per language idiom and module-layering constraints. Reference placements:

- **Go**: `entitysdk` package in workbench-go (the SDK boundary that imports both `core/peer` and `core/handler` surfaces). Per the user's direction ("SDK extension operations hitting core protocol layer doesn't seem right; workbench-go owns the SDK"). The Dispatcher interface lives with the SDK affordance — no core-layer addition; workbench-go's app-layer SDK is the home. The two adapter constructors (Connection.Execute → Dispatcher; HandlerContext.Execute → Dispatcher) compose existing core surfaces.
- **Python**: package's SDK module (`entity_core_py.sdk.content` or analogous). Python's module layering allows placement closer to the user-facing SDK boundary; choose what matches the existing extension surface convention.
- **Rust**: `entitysdk` crate (or analogous SDK boundary). Same reasoning — wherever the SDK convention is established for cross-extension affordances.

The earlier draft language ("each impl's content extension") was ambiguous between normative ("MUST live in ext/content") and descriptive ("lives in whichever SDK boundary the impl uses"). **Descriptive is canonical.** Implementations choose the placement that respects their language's module-layering discipline and SDK convention. The wire contract + Dispatcher interface semantics + EnsureClosure behavior are the load-bearing surfaces; physical placement is impl-detail.

#### Byte extraction (NOT in the SDK protocol surface)

After `EnsureClosure(hash, namespace)` returns successfully, the blob + all chunks are locally present in the content store. Byte extraction is a **pure local helper** the consumer calls in their own language idiom. **Library locations pinned:**

- **Go:** `content.Reassemble(store store.ContentStore, blob hash.Hash) → ([]byte, error)` lives publicly in `ext/content/builder.go` (alongside existing `BuildBlob` / `IngestBlob`).
- **Python:** `reassemble_content(store, hash) → bytes` lives publicly in `entity_core_py/ext/content/__init__.py` (or analogous package-public entry).
- **Rust:** `content::reassemble(store: &ContentStore, hash: Hash) → Result<Vec<u8>>` lives publicly in `extensions/content/src/lib.rs`.

`Reassemble` is **substrate-internal trusted code** per CONTENT §3.4 L10 framing — it reads from the local content store after closure completion. The cap-checked wrapper (`EnsureClosure`) guards the closure-completion side; reassembly itself is a pure local function with no protocol surface.

Streaming variants (Go `io.Reader`, Python `Iterator[bytes]`, Rust `Stream`) are likewise per-language library affordances over the local content store — NOT protocol surface. Each impl provides them in the same package.

**Worked usage — direct SDK call (outer caller, cross-peer):**

```
peer  = AppPeer.Connect(remote_peer_uri, cap)
err   = content.EnsureClosure(ctx, peer, blob_hash, "system/content/shared/team-alpha")
bytes = content.Reassemble(local_store, blob_hash)
```

**Worked usage — handler-internal, cross-peer (the workbench Stage 3 shape):**

```
// Inside a handler running on peer B
remote = content.AtPeer(handler_ctx, source_peer_id_A)
err    = content.EnsureClosure(ctx, remote, file_entity.content, "system/content")
// closure now local on B; downstream handler step or this handler's continuation
// can call content.Reassemble or trigger local/files:write content-mode, etc.
```

**Worked usage — chain composition (L11 idiom per `GUIDE-EXTENSION-DEVELOPMENT.md` §4.10):** the chain dispatches a handler whose body calls `EnsureClosure`; the next chain step operates on the now-locally-complete closure via existing transform ops (`deref_included`, `extract`) or via another handler that does the application work.

**Cap-flow.** Each `system/content:get` inside `EnsureClosure` is independently cap-checked at the dispatcher. The caller's cap must cover `system/content:get` on `namespace`. For handler-internal callers, the handler's own `internal_scope` grant is the surface; for cross-peer use, standard V7 §5.10 cap delegation applies. No privilege amplification beyond what direct `system/content:get` dispatch already permits.

---

## 12. Type Extension

**Handler:** `system/type/constraint/*`
**What it does:** Type validation, constraint evaluation, type analysis.

### Operations

**validate** — Check a value against a constraint.
```
validate(constraint_type: string, value: any, params: any) → ValidateResult

  Standard constraints: min, max, min_length, max_length, min_count,
  max_count, pattern, one_of, not_one_of, format, type_pattern.
```

Type analysis operations (compare, converge, compatible, adopt, reconcile) are defined in the spec but not yet exposed as handler operations in any implementation.

---

## 13. Role Extension

**Handler:** `system/role`
**What it does:** Role-based access control — role definitions, assignments, exclusions, re-derivation, and member-to-member delegation. Assignments derive capability tokens for the assignee per the role definition. Caps are root caps (parent: null, granter: local peer's identity hash) per role v2.0 PR-1, structurally identical to startup-time L0 derivation.

> **Coherent Capability.** Role definition and assignment entities are load-bearing: creating either derives or scopes capability tokens. Use `system/role:define` and `system/role:assign` rather than direct `tree:put` to role-namespace paths. The handlers validate caller authority (RL2: caller cap covers proposed grants); direct `tree:put` bypasses these checks. See `EXTENSION-ROLE.md` §1.5.2 / §6.6.

> **Identity composition.** When the identity extension is registered, the typical caller cap for role ops is the local peer→controller cap issued by `system/identity:configure`. See `sdk-domain/specs/SDK-IDENTITY-INFRASTRUCTURE.md` §7 for role↔identity composition (multi-role per (peer, context); multi-agent concurrent re-derive; bootstrap composition; member-to-member delegation chain depth).

### Operations (per role v2.0 + Amendments 1+2)

**define** — Write or update a role definition.
```
define(params: DefineParams) → DefineResult

  Validates RL2 (caller cap covers proposed role grants).
  Triggers re-derive cascade on definition mutation.

  DefineParams := {
    context: string         ; Role context
    role:    string         ; Role name
    grants:  [GrantEntry]   ; Templated grants (may contain {peer_id}, {context})
    ttl:     duration?      ; Default TTL on derived caps
  }
```

**assign** — Create a role assignment and derive capability tokens.
```
assign(params: AssignParams) → AssignResult

  Validates RL2 + zero-grantee rejection (per role v2.0 PR-3 / SEC-18).
  Issues role-derived cap (root cap: parent: null, granter: local
  peer's identity hash) and writes the assignment + linkage entities.

  AssignParams := {
    context:  string
    role:     string
    assignee: hash          ; Identity hash of the assignee peer (NOT zero)
    metadata: any?
  }

  AssignResult := {
    status:          uint
    assignment_path: string
    cap_path:        string  ; Where the role-derived cap is bound
  }
```

**unassign** — Remove a role assignment and revoke the role-derived cap.
```
unassign(context: string, role: string, assignee: hash) → Status
```

**exclude** — Layer-1 sweep: revoke all role-derived caps for the assignee in this context.
```
exclude(params: ExcludeParams) → Status
  ; Three-layer exclusion model per EXTENSION-ROLE §6:
  ;  Layer 1 (this op): token revocation — sweeps system/capability/grants/role-derived/{context}/{peer_hex}/*
  ;  Layer 2: blocks new derivation (consulted by :assign / :re-derive / :delegate)
  ;  Layer 3: opt-in handler-level context-aware checks
```

**unexclude** — Remove the exclusion entity. Does NOT auto-restore revoked caps; subsequent :assign or :re-derive will issue fresh caps.
```
unexclude(context: string, peer: hash) → Status
```

**re-derive** — Re-issue caps for all assignees of a role after definition mutation.
```
re-derive(context: string, role: string) → ReDeriveResult
  ; Lifecycle: T_new before T_old revocation (per IA9).
  ; SI-15 skipped_grantees on RL2 mid-cascade failures.

  ReDeriveResult := {
    re_derived_count: uint
    skipped_grantees: [hash]
  }
```

**delegate** — Member-to-member delegation (opt-in per role).
```
delegate(params: DelegateParams) → DelegateResult
  ; Requires the role definition to include "delegate" in its grants list
  ; (per PR-8.2 — delegate-ability is opt-in per role).
  ; Scope MUST be literal (no template variables; per SI-20 / PR-9.3).
  ; Delegation cap is rooted at delegator's runtime peer (parent: role-derived
  ; root cap; chain depth 2). Layer-2 exclusion checked.

  DelegateParams := {
    context:    string
    role:       string
    delegate:   hash        ; Recipient's identity hash
    scope:      [GrantEntry]  ; MUST be literal subset of role's grants
    expires_at: timestamp?
  }
```

### Role definitions and tree data

Role definitions are now created via `system/role:define` (per ARC-FIXES IA11). Direct `tree:put` to role-namespace paths is reserved for system-extension and administrative use; SDK application grants SHOULD cover handler ops, not raw `tree:put`.

### Multi-role per (peer, context)

Per role v2.0 §5.5 — `:assign` may be called multiple times for the same (peer, context) with different role names; tokens compose; verification per V7 picks the presented token.

### Initial grant policy (connect-time)

The `system/role/initial-grant-policy` entity governs anonymous-allow / anonymous-deny / recognize-on-attestation behavior at the connection handler's grant-resolution step. Per role v2.0 PR-5 + §7, the resolver is an SDK seam, not a role-spec normative concern. SDK frameworks SHOULD provide a built-in role-aware resolver. See `sdk-domain/specs/SDK-IDENTITY-INFRASTRUCTURE.md` §7 for the role↔identity composition.

---

## 14. The Full SDK Surface

When all extensions are registered, the SDK exposes approximately:

| Category | Handler Operations | SDK Orchestration | Source |
|----------|-------------------|-------------------|--------|
| Tree | get, put, list, remove, has, snapshot, diff, merge, extract | — | system/tree |
| Dispatch | execute | — | core |
| Revision | commit, log, status, merge, resolve, fetch, fetch-entities, config | pull, push, diff, cherry-pick, revert, find-ancestor | system/revision |
| Transaction | execute (single-shot), begin, write, read, commit, rollback | — | system/transaction |
| Subscription | subscribe, unsubscribe | watch, unwatch | system/subscription |
| Continuation | install, advance, resume, abandon | pipeline builder (L3) | system/continuation |
| Query | find, count | query builder (L3) | system/query |
| History | query, rollback | — | system/history |
| Clock | now, compare | — | system/clock |
| Type | validate | — | system/type/constraint |
| Compute | eval, install, uninstall | — | system/compute |
| Network | maintain-peer, release-peer, status, close | — | system/network |
| Content | get | — | system/content |
| Inbox | receive | — | system/inbox |
| Role | assign, unassign, exclude | grant derivation from role definitions | system/role |
| Capability | request, delegate, revoke | — | system/capability |
| Connection | hello, authenticate | — | system/protocol/connect |

**~55 handler operations + ~10 SDK orchestration workflows.** Not all needed at once — the peer builder determines what's available.

### What the Peer Builder Controls

```
; Minimal — KV store only
peer = create_peer(extensions=[])
; Available: get, put, list, remove, has, execute

; With subscription — KV store + reactive
peer = create_peer(extensions=[subscription, inbox, continuation])
; Handler ops: subscribe, unsubscribe, advance, resume, abandon, receive
; SDK wrappers: watch, unwatch

; With revision — versioned KV store
peer = create_peer(extensions=[subscription, inbox, continuation, revision, history])
; Handler ops: + commit, log, status, merge, resolve, fetch, fetch-entities, config, history query, rollback
; SDK orchestration: + pull, push, diff, cherry-pick, revert

; Full standard peer
peer = create_peer(extensions=[all_standard])
; Everything above + query, clock (implemented)
; + compute, network, content, type constraint (when implemented)
```

---

## 15. Extension Availability

Extensions are independently registrable. Not every peer needs every extension. The peer builder composes what's needed.

| Extension | Spec Status | Dependencies | SDK doc |
|-----------|-------------|--------------|---------|
| Inbox | Stable | None | This doc §7 |
| Continuation | Stable | Inbox | This doc §2 |
| Subscription | Stable | Inbox | This doc §3 |
| History | Stable | None | This doc §5 |
| Query | Stable | None | This doc §6 |
| Clock | Stable | None | This doc §10 |
| Revision | Stable | Tree snapshot/diff/merge | This doc §4 |
| Transaction | Draft | Tree (core) | This doc §7A |
| Type | Stable | None | This doc §12 |
| Compute | Designed | None | This doc §8 |
| Network | Designed | Continuation, Subscription, Inbox | This doc §9 |
| Content | Designed | None | This doc §11 |
| Role | Stable (v2.0) | Capability system; identity (when registered) | This doc §13 |
| **Attestation** | Stable (v1.2) | None | `sdk-domain/specs/SDK-IDENTITY-INFRASTRUCTURE.md` §5.1 |
| **Quorum** | Stable (v1.2) | Attestation | `sdk-domain/specs/SDK-IDENTITY-INFRASTRUCTURE.md` §5.2 |
| **Identity** | Stable (v3.5) | Attestation, Quorum | `sdk-domain/specs/SDK-IDENTITY-INFRASTRUCTURE.md` §6 |
| Group | v1.5 sweep queued | Attestation, Quorum, Identity | `sdk-domain/specs/SDK-IDENTITY-INFRASTRUCTURE.md` §10 (placeholder) |

**Identity stack (attestation + quorum + identity, plus role-coherence aspects) is documented in `sdk-domain/specs/SDK-IDENTITY-INFRASTRUCTURE.md`** rather than per-extension subsections in this doc, because the surface composes across all three and includes a non-handler-op helper library (bootstrap, pairing, custody, rotation, recovery). The role section (§13) covers role's plain handler-ops surface; identity-coherence aspects (caller-cap composition, multi-agent re-derive, bootstrap composition, member-to-member delegation) live in the identity-infra spec.

The standard peer configuration includes: subscription, continuation, inbox, history, query, clock, revision, transaction, and type. Identity-aware deployments add attestation + quorum + identity (and role; group when v1.5 lands). Compute, network, and content are designed — their SDK surfaces will solidify as implementations mature.

---

## 15.1 Coherent Capability Authority — Cross-Cutting Rule

**Generalized deliver_token rule (GR1).** Any handler operation that accepts a `deliver_token` (or equivalent embedded capability authorizing future async delivery) in its EXECUTE input MUST validate the token's authority chain against the EXECUTE author's identity at the time of receipt.

This rule covers subscription (`deliver_token` in subscribe), inbox EXECUTE-level `deliver_token`, and any future async-delivery handler. The validation uses the `check_creator_authority(cap, identity, ctx)` primitive (V7 §5.5), which collects the full authority chain via `collect_authority_chain` and verifies both reachability and identity match. Handlers that store, persist, or forward an embedded capability reference without this check accept tokens whose creator authority is unverified.

**Kernel-vs-handler principle.** Load-bearing entity types — types whose data carries semantic content that affects later runtime behavior (capability references, dispatch targets, state-machine fields, scoped grants) — should be created via the owning handler's operation, not via direct `tree:put`. Application grants SHOULD cover handler operations, not raw `tree:put` to system handler namespaces. See V7 §6.3 for the normative statement; SDK-OPERATIONS.md §2.7.2 for SDK guidance.

This applies to: continuation `install`, subscription `subscribe`, role `assign`, compute `install`, revision `config`, handler `register`. Each extension's "Coherent Capability" subsection documents which entity types are load-bearing and which operation to use.

---

## 16. Open Questions

1. **Pipeline builder API shape.** Continuation chains are powerful but building them is verbose. The L3 pipeline builder needs design work. What does a fluent API for multi-step async workflows look like?

2. **Extension discovery.** How does application code discover which extensions are available on a peer it connects to? Currently: `list("system/handler/")` and check. Should the SDK provide typed extension discovery?

3. **Cross-extension composition patterns.** The subscribe → notify → continuation → revision chain is the core reactive pattern. Should the SDK provide pre-built compositions (e.g., `peer.sync(remote_peer, "knowledge/")` that wires up the full chain)?

4. **Revision metadata convention.** Version entries are structural (root + parents). Applications that want commit messages, authors, or timestamps need a convention for storing that metadata alongside versions. Should the SDK provide a metadata helper that associates annotation entities with version hashes? Or should the revision spec be extended to support optional metadata fields?

5. **Checkout safety.** Checkout overwrites current tree state under a prefix. Should the SDK require explicit confirmation when uncommitted changes exist? Should there be a `--force` equivalent and a default safe mode?

6. **Compute expression builder.** When compute is implemented, building expressions programmatically needs an API. The expression types (literal, lookup, apply, if, let, lambda) map naturally to a builder pattern.

7. **Dynamic handler loading.** Language-specific mechanisms for registering handlers after peer build (Go plugins, Python modules, Rust dylibs, Godot GDScript). The registration contract (§11.3 in SDK-OPERATIONS.md) needs a runtime variant.

---

## 17. Document History

- **v0.3:** Coherent capability authority alignment. **Continuation**: added `install` operation (creates continuations with R1 chain-root validation on `dispatch_capability`). **Subscription**: added R1 chain-root check note on `deliver_token`, three-cap model clarification. **Compute**: expanded install operation detail (audit, chain-root on static literals, installation grant), added security model subsection (dual-check, resource field, chain-root). **Role**: rewritten from tree-data-only to handler with `assign`, `unassign`, `exclude` operations (R1-equivalent check on derived grants). Added §15.1 cross-cutting coherent capability authority rules (GR1 generalized deliver_token, kernel-vs-handler principle). Updated summary table (continuation +install, role +assign/unassign/exclude). Depends bumped: V7 v7.20+ → v7.30+, added EXTENSION-COMPUTE v3.10+. Source: PROPOSAL-COHERENT-CAPABILITY-AUTHORITY (adopted), PROPOSAL-COMPUTE-APPLY-RESOURCE-CEILING (adopted), EXTENSION-COMPUTE v3.10.
- **v0.2:** Review fixes. Revision: aligned version entries with structural reality (no message/author/timestamp), separated handler operations from SDK orchestration, branches/tags as tree data. History: noted query name collision. Availability table: added implementation status column, corrected unimplemented extensions from "Stable" to "Designed". Summary table: split handler ops vs SDK orchestration.
- **v0.1:** Initial working draft. Maps extension handler operations to SDK surface. Source: extension specs + SDK-SYNTHESIS.md.
