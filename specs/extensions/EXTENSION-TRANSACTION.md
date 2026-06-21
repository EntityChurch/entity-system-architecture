# Transaction Extension — Normative Specification

**Version**: 0.1
**Status**: Draft
**Depends**: ENTITY-CORE-PROTOCOL.md (v7.26+), EXTENSION-TREE.md (v3.3+)
**Optional**: EXTENSION-REVISION.md (v2.5+) — continuation-pipeline versioning (see §7)
**Optional**: EXTENSION-CONTINUATION.md (v1.5+) — post-commit workflows (see §7)
**Source**: EXPLORATION-NORMALIZATION-TRANSACTIONS-AND-COORDINATION.md §4, §11

---

> **Path notation.** Paths in this document use peer-relative notation (without leading `/{peer_id}/`). All peer-relative paths resolve to the local peer's namespace. See ENTITY-CORE-PROTOCOL.md §1.4 for the path model.

## 1. Overview

The transaction extension provides **multi-path atomic operations** — grouping multiple tree bindings into a single logical unit. The tree layer provides per-binding atomicity only (ENTITY-CORE-PROTOCOL.md §6.10). This extension composes per-binding tree.put into grouped operations with pre-commit validation, completion signaling, and compensating rollback.

### 1.1 What this provides

- **Grouped writes.** Multiple tree bindings applied as a unit with a completion signal.
- **Pre-commit validation.** CAS guards, capability checks, and constraint verification before any binding is applied.
- **Observation boundary.** Consumers of the transaction layer see committed states, not intermediate bindings. Raw-tree consumers see individual bindings as usual.
- **Compensating rollback.** On partial failure, previously-applied bindings can be reverted to their prior state.
- **Isolation levels.** Snapshot isolation and serializable isolation via trie-root snapshots and read-set tracking.

### 1.2 What this does NOT provide

- **True atomicity at the location-index level.** Individual tree.puts are sequential. Intermediate states exist at the raw tree level between the first and last binding application. L8 (atomic multi-path tree operations) would eliminate these; this extension works without it.
- **Distributed transactions.** This extension is local — one peer. Cluster transactions (multi-peer coordination) compose on top using this extension's intent/result types. See §8.
- **Before-triggers on arbitrary tree.put.** The transaction handler validates writes routed through it. Direct tree.put to the same paths (if capability permits) bypasses transaction validation.

### 1.3 Architecture

The entity system's core protocol provides per-binding atomicity. Extensions compose to provide stronger guarantees:

```
Layer 0: tree.put               — per-binding atomic
Layer 1: Handler operations     — multi-write, handler-scoped
Layer 2: EXTENSION-TRANSACTION  — multi-write, grouped, observation boundary  ← this extension
Layer 3: Cluster transaction    — multi-peer, coordinated (future extension)
```

Each layer composes from the layer below. Lower layers remain accessible. A transaction's individual tree.puts fire normal emit cascades; the transaction adds a completion signal on top.

---

## 2. Type Definitions

### 2.1 Transaction Intent

The unit of work. Lists bindings to apply as a group.

```
system/transaction/intent := {
  fields: {
    prefix:         {type_ref: "system/tree/path"}
                     ; Tree-path prefix this transaction operates on.
                     ; All bindings MUST be under this prefix (400 if not).
                     ; Required for snapshot/serializable isolation (trie root capture).
                     ; For read-committed, used for binding path validation only.
    bindings:       {array_of: {type_ref: "system/transaction/binding"}}
                     ; Ordered list of (path, hash) pairs to apply.
                     ; MUST be sorted by path for deterministic cross-peer
                     ; behavior (see §5.8). The handler rejects unsorted intents.
    isolation:      {type_ref: "primitive/string", optional: true}
                     ; "read-committed" (default) | "snapshot" | "serializable"
                     ; Controls validation behavior at commit time. See §6.
    rollback_on_failure: {type_ref: "primitive/bool", optional: true}
                     ; Default: false. If true, compensating writes are issued
                     ; for applied bindings when a subsequent binding fails.
                     ; Compensating writes are best-effort (see §5.4).
  }
}
```

### 2.2 Transaction Binding

A single path-to-hash binding within a transaction.

```
system/transaction/binding := {
  fields: {
    path:           {type_ref: "system/tree/path"}
                     ; Absolute path for the binding.
    hash:           {type_ref: "system/hash", optional: true}
                     ; Content hash to bind at the path.
                     ; Null means delete the binding.
    expected_hash:  {type_ref: "system/hash", optional: true}
                     ; CAS guard. If present, the binding is applied only if
                     ; the current hash at the path matches. Null/absent means
                     ; unconditional.
  }
}
```

### 2.3 Transaction Result

Returned by `execute` and `commit` operations.

```
system/transaction/result := {
  fields: {
    transaction_id:    {type_ref: "primitive/string"}
    status:            {type_ref: "primitive/string"}
                        ; "committed" | "partial" | "rolled-back" | "rejected"
    applied:           {array_of: {type_ref: "system/tree/path"}}
                        ; Paths whose bindings were successfully applied.
    failed:            {array_of: {type_ref: "system/transaction/binding-failure"}, optional: true}
                        ; Bindings that failed. Present when status is "partial" or "rejected".
    compensated:       {array_of: {type_ref: "system/tree/path"}, optional: true}
                        ; Paths reverted by compensating writes. Present when status is "rolled-back".
    cascade_warnings:  {array_of: {type_ref: "system/transaction/cascade-warning"}, optional: true}
                        ; Bindings whose tree.put returned 207 (cascade halted).
    snapshot_root:     {type_ref: "system/hash", optional: true}
                        ; Trie root at transaction begin (snapshot/serializable only).
    commit_root:       {type_ref: "system/hash", optional: true}
                        ; Trie root after all bindings applied (on success).
  }
}
```

### 2.4 Binding Failure

Detail for a binding that could not be applied.

```
system/transaction/binding-failure := {
  fields: {
    path:           {type_ref: "system/tree/path"}
    reason:         {type_ref: "primitive/string"}
                     ; "cas_mismatch" | "capability_denied" | "write_rejected" | "constraint_violated"
    expected_hash:  {type_ref: "system/hash", optional: true}
    actual_hash:    {type_ref: "system/hash", optional: true}
    error:          {type_ref: "system/protocol/error", optional: true}
  }
}
```

### 2.5 Cascade Warning

Binding whose cascade halted (207) during application.

```
system/transaction/cascade-warning := {
  fields: {
    path:               {type_ref: "system/tree/path"}
    consumer_halted:    {type_ref: "primitive/string"}
    error_code:         {type_ref: "primitive/string"}
  }
}
```

### 2.6 Transaction State (interactive transactions)

Stored at `system/transaction/pending/{id}` during an interactive transaction.

```
system/transaction/state := {
  fields: {
    transaction_id:    {type_ref: "primitive/string"}
    status:            {type_ref: "primitive/string"}
                        ; "pending" | "committing" | "rolling-back"
    prefix:            {type_ref: "system/tree/path"}
                        ; Tree-path prefix this transaction operates on.
    intent:            {type_ref: "system/transaction/intent"}
                        ; Accumulated bindings.
    snapshot_root:     {type_ref: "system/hash", optional: true}
                        ; Trie root for `prefix`, captured at begin (snapshot/serializable).
    read_set:          {array_of: {type_ref: "system/tree/path"}, optional: true}
                        ; Paths read during the transaction (serializable only).
    created_at:        {type_ref: "system/clock/timestamp"}
  }
}
```

### 2.7 Committed Record

Written to `system/transaction/committed/{id}` on successful commit. This is the **observation boundary** — consumers of the transaction layer watch this path.

```
system/transaction/committed := {
  fields: {
    transaction_id:    {type_ref: "primitive/string"}
    intent_hash:       {type_ref: "system/hash"}
                        ; Content hash of the intent entity (audit trail).
    applied:           {array_of: {type_ref: "system/tree/path"}}
    snapshot_root:     {type_ref: "system/hash", optional: true}
    commit_root:       {type_ref: "system/hash", optional: true}
    committed_at:      {type_ref: "system/clock/timestamp"}
  }
}
```

---

## 3. Tree Layout

```
system/transaction/                          ; Extension root
system/transaction/pending/{id}              ; Interactive transaction state (§4.3)
system/transaction/committed/{id}            ; Committed record (observation boundary)
system/transaction/rolled-back/{id}          ; Rollback record (when rollback_on_failure applied)
```

Pending transaction entities are cleaned up after commit or rollback. Committed and rolled-back records persist for audit and are subject to normal GC policies.

---

## 4. Handler

### 4.1 Handler Manifest

```
system/handler := {
  data: {
    pattern:    "system/transaction"
    name:       "transaction"
    operations: {
      execute:    {input_type: "system/transaction/intent",          output_type: "system/transaction/result"}
      begin:      {input_type: "system/transaction/begin-params",    output_type: "system/transaction/begin-result"}
      write:      {input_type: "system/transaction/write-params",    output_type: "system/transaction/write-result"}
      read:       {input_type: "system/transaction/read-params",     output_type: "system/transaction/read-result"}
      commit:     {input_type: "system/transaction/commit-params",   output_type: "system/transaction/result"}
      rollback:   {input_type: "system/transaction/rollback-params", output_type: "system/transaction/result"}
      status:     {input_type: "system/transaction/status-params",   output_type: "system/transaction/state"}
    }
  }
}
```

Manifest at pattern path `system/transaction`. Index entry at `system/handler/system/transaction`. Handler grant at `system/capability/grants/system/transaction`.

### 4.2 Core and Convenience Operations

**Core operations** (MUST implement):

| Operation | Purpose |
|-----------|---------|
| `execute` | Single-shot: validate and apply an intent atomically |
| `begin` | Start interactive transaction, capture snapshot |
| `commit` | Apply accumulated bindings from interactive transaction |
| `rollback` | Discard interactive transaction, optionally compensate |

**Convenience operations** (SHOULD implement):

| Operation | Purpose |
|-----------|---------|
| `write` | Stage a binding in an interactive transaction |
| `read` | Read a path through the transaction's snapshot |
| `status` | Query current state of an interactive transaction |

`write` and `read` are convenience because callers could manage the intent entity themselves and issue a single `commit`. But interactive use (read-then-write patterns) requires handler-mediated reads for isolation guarantees.

### 4.3 Capability Model

Transaction operations require capability for the `system/transaction` handler. The transaction handler checks the caller's capability against each binding's path before applying — the caller must have `put` permission on every path in the intent.

The transaction handler's own grant covers `system/transaction/*` (pending, committed, rolled-back paths). The handler writes to these paths under its own authority.

---

## 5. Operation Semantics

### 5.1 execute (single-shot)

The primary operation. Takes a complete intent and applies it.

```
handle_execute(ctx, intent):
  transaction_id = generate_id()
  prefix = intent.prefix

  ; --- Pre-commit validation ---

  ; V0: All bindings must be under the declared prefix
  for binding in intent.bindings:
    if not binding.path starts with prefix:
      return error(400, "transaction/binding-outside-prefix", {
        path: binding.path, prefix: prefix
      })

  ; V1: Capability check on all paths
  for binding in intent.bindings:
    if not check_path_permission("put", binding.path, ctx.capability):
      return error(403, "transaction/capability-denied", {
        path: binding.path
      })

  ; V2: Isolation validation (snapshot/serializable)
  if intent.isolation == "snapshot" or intent.isolation == "serializable":
    root_path = "system/tree/root/" + normalize_prefix_key(prefix)
    snapshot_root = ctx.entity_tree.get_hash(root_path)
    if snapshot_root == null:
      return error(400, "transaction/no-tracked-root", {
        prefix: prefix
      })
  else:
    snapshot_root = null

  ; V3: CAS pre-check on all bindings
  for binding in intent.bindings:
    if binding.expected_hash != null:
      current = ctx.entity_tree.get_hash(binding.path)
      if current != binding.expected_hash:
        return {
          status: 200,
          result: {
            type: "system/transaction/result",
            data: {
              transaction_id:  transaction_id,
              status:          "rejected",
              applied:         [],
              failed:          [{
                path:          binding.path,
                reason:        "cas_mismatch",
                expected_hash: binding.expected_hash,
                actual_hash:   current
              }]
            }
          }
        }

  ; --- Capture previous state (for rollback) ---
  previous = {}
  if intent.rollback_on_failure:
    for binding in intent.bindings:
      previous[binding.path] = ctx.entity_tree.get_hash(binding.path)

  ; --- Apply bindings ---
  applied = []
  failed = []
  cascade_warnings = []

  for binding in intent.bindings:
    if binding.hash != null:
      result = ctx.entity_tree.put(binding.path, binding.hash)
    else:
      result = ctx.entity_tree.put(binding.path, null)  ; delete

    if result.status in [200, 207]:
      applied.append(binding.path)
      if result.status == 207:
        ; Cascade halted — binding landed, but some consumers skipped.
        ; Continue applying remaining bindings.
        cascade_warnings.append(extract_cascade_warning(result))
    else:
      ; Pre-write rejection (4xx/5xx). Binding did NOT land.
      failed.append({
        path:   binding.path,
        reason: "write_rejected",
        error:  result.error
      })
      ; Stop applying further bindings.
      break

  ; --- Determine outcome ---
  if len(failed) == 0:
    ; All bindings applied successfully.
    committed = {
      type: "system/transaction/committed",
      data: {
        transaction_id:  transaction_id,
        intent_hash:     content_hash(intent),
        applied:         applied,
        snapshot_root:   snapshot_root,
        commit_root:     ctx.entity_tree.get_hash("system/tree/root/" + normalize_prefix_key(prefix)),
        committed_at:    system_clock_ms()
      }
    }
    ctx.content_store.put(committed)
    ctx.entity_tree.put("system/transaction/committed/" + transaction_id, content_hash(committed))

    return {
      status: 200,
      result: {
        type: "system/transaction/result",
        data: {
          transaction_id:    transaction_id,
          status:            "committed",
          applied:           applied,
          cascade_warnings:  cascade_warnings or null,
          snapshot_root:     snapshot_root,
          commit_root:       committed.data.commit_root
        }
      }
    }

  else:
    ; Partial failure.
    if intent.rollback_on_failure:
      compensated = compensate(ctx, applied, previous)
      return {
        status: 200,
        result: {
          type: "system/transaction/result",
          data: {
            transaction_id:    transaction_id,
            status:            "rolled-back",
            applied:           [],         ; net effect after compensation
            failed:            failed,
            compensated:       compensated,
            cascade_warnings:  cascade_warnings or null
          }
        }
      }
    else:
      return {
        status: 200,
        result: {
          type: "system/transaction/result",
          data: {
            transaction_id:    transaction_id,
            status:            "partial",
            applied:           applied,
            failed:            failed,
            cascade_warnings:  cascade_warnings or null
          }
        }
      }
```

### 5.2 begin (interactive)

```
system/transaction/begin-params := {
  fields: {
    prefix:         {type_ref: "system/tree/path"}
                     ; Tree-path prefix this transaction operates on.
                     ; Required for snapshot/serializable isolation — the handler
                     ; captures the tracked trie root for this prefix at begin time.
                     ; All reads and writes MUST be under this prefix (400 if not).
                     ; Capability checking uses full tree paths (prefix + relative).
    isolation:      {type_ref: "primitive/string", optional: true}
                     ; "read-committed" | "snapshot" | "serializable"
    rollback_on_failure: {type_ref: "primitive/bool", optional: true}
  }
}

system/transaction/begin-result := {
  fields: {
    transaction_id:    {type_ref: "primitive/string"}
    snapshot_root:     {type_ref: "system/hash", optional: true}
  }
}
```

```
handle_begin(ctx, params):
  transaction_id = generate_id()
  isolation = params.isolation or "read-committed"
  prefix = params.prefix

  snapshot_root = null
  if isolation in ["snapshot", "serializable"]:
    ; Capture the tracked trie root for this prefix.
    ; Requires a tracking-config for the prefix (EXTENSION-TREE §3.4.1a).
    root_path = "system/tree/root/" + normalize_prefix_key(prefix)
    snapshot_root = ctx.entity_tree.get_hash(root_path)
    if snapshot_root == null:
      return error(400, "transaction/no-tracked-root", {
        prefix: prefix,
        message: "snapshot/serializable isolation requires a tracked trie root for the prefix"
      })

  state = {
    type: "system/transaction/state",
    data: {
      transaction_id:       transaction_id,
      status:               "pending",
      prefix:               prefix,
      intent:               { bindings: [], isolation: isolation,
                              rollback_on_failure: params.rollback_on_failure },
      snapshot_root:        snapshot_root,
      read_set:             [] if isolation == "serializable" else null,
      created_at:           system_clock_ms()
    }
  }

  ctx.content_store.put(state)
  ctx.entity_tree.put("system/transaction/pending/" + transaction_id, content_hash(state))

  return {
    status: 200,
    result: {
      type: "system/transaction/begin-result",
      data: {
        transaction_id: transaction_id,
        snapshot_root:  snapshot_root
      }
    }
  }
```

### 5.3 write (interactive — stage a binding)

```
system/transaction/write-params := {
  fields: {
    transaction_id: {type_ref: "primitive/string"}
    binding:        {type_ref: "system/transaction/binding"}
  }
}

system/transaction/write-result := {
  fields: {
    transaction_id: {type_ref: "primitive/string"}
    binding_count:  {type_ref: "primitive/uint"}
  }
}
```

The `write` operation appends a binding to the pending transaction's intent. The binding is NOT applied to the tree — it's staged. Application happens at `commit`.

### 5.4 read (interactive — snapshot read)

```
system/transaction/read-params := {
  fields: {
    transaction_id: {type_ref: "primitive/string"}
    path:           {type_ref: "system/tree/path"}
  }
}

system/transaction/read-result := {
  fields: {
    path:           {type_ref: "system/tree/path"}
    hash:           {type_ref: "system/hash", optional: true}
                     ; null if no binding at path
    source:         {type_ref: "primitive/string"}
                     ; "snapshot" — read from captured trie root
                     ; "staged"  — read from transaction's staged bindings
                     ; "live"    — read from current tree (read-committed only)
  }
}
```

Read resolution:

1. Check staged bindings in the transaction's intent. If the path has a staged binding, return it (`source: "staged"`).
2. For snapshot/serializable isolation: the transaction handler resolves the path against its captured `snapshot_root` by walking the trie internally via `content_store.get`. The handler holds the prefix (from the transaction state) and performs capability checking against the full tree path (`prefix + relative_path`) using the caller's capability. Return the result (`source: "snapshot"`).
3. For read-committed: delegate to `tree.get` — reads from the live location index (`source: "live"`).
4. For serializable: also add the path to the `read_set` for commit-time validation.

### 5.5 commit (interactive)

```
system/transaction/commit-params := {
  fields: {
    transaction_id: {type_ref: "primitive/string"}
  }
}
```

Commit reads the pending transaction state, validates, and applies bindings using the same logic as `execute` (§5.1). Additional validation for serializable isolation:

```
; Serializable validation: check that read-set paths haven't changed since snapshot
if state.intent.isolation == "serializable":
  for path in state.read_set:
    current = ctx.entity_tree.get_hash(path)
    relative = strip_prefix(path, state.prefix)
    snapshot_value = resolve_against_trie(state.snapshot_root, relative)
    if current != snapshot_value:
      return {
        status: 200,
        result: {
          type: "system/transaction/result",
          data: {
            transaction_id: state.transaction_id,
            status:         "rejected",
            applied:        [],
            failed:         [{
              path:    path,
              reason:  "serializable_conflict",
              expected_hash: snapshot_value,
              actual_hash:   current
            }]
          }
        }
      }
```

After successful application, the pending state is deleted and a committed record is written (same as `execute`).

### 5.6 rollback (interactive)

```
system/transaction/rollback-params := {
  fields: {
    transaction_id: {type_ref: "primitive/string"}
  }
}
```

Rollback discards the pending transaction. If bindings have been applied (shouldn't be the case in normal interactive flow — bindings are staged, not applied, until commit), compensating writes are issued.

In the normal case (no bindings applied yet), rollback deletes the pending state:

```
handle_rollback(ctx, params):
  state_hash = ctx.entity_tree.get_hash("system/transaction/pending/" + params.transaction_id)
  if state_hash == null:
    return error(404, "transaction/not-found", { transaction_id: params.transaction_id })

  ctx.entity_tree.put("system/transaction/pending/" + params.transaction_id, null)

  return {
    status: 200,
    result: {
      type: "system/transaction/result",
      data: {
        transaction_id: params.transaction_id,
        status:         "rolled-back",
        applied:        [],
      }
    }
  }
```

### 5.7 Compensating writes

Compensating writes are the rollback mechanism. They write the previous binding back to each applied path. This is NOT true undo — it's a forward write that restores the prior state.

```
compensate(ctx, applied_paths, previous_hashes):
  compensated = []
  for path in reverse(applied_paths):
    prev = previous_hashes[path]
    result = ctx.entity_tree.put(path, prev)  ; prev may be null (delete)
    if result.status in [200, 207]:
      compensated.append(path)
    ; If compensating write fails, log and continue.
    ; Best-effort — can't guarantee all compensations succeed.
  return compensated
```

**Compensating writes fire their own emit cascades.** Raw-tree consumers see: original state → committed state → reverted state (three transitions). Consumers above the transaction layer see only the rolled-back record.

**Compensating writes are best-effort.** If a compensating write fails (capability denied, cascade-depth exceeded, etc.), the path remains at its committed value. The transaction result lists which paths were compensated and which weren't. This is honest about the guarantee: compensating rollback achieves eventual restoration, not atomic undo.

### 5.8 Binding application order

Bindings MUST be sorted by path in the intent's `bindings` array. The handler MUST reject intents with unsorted bindings (status 400, code `transaction/unsorted-bindings`). Bindings are applied in array order during commit.

Deterministic ordering ensures:
- Two peers applying the same intent produce the same sequence of emit events.
- Auto-version intermediates (if enabled) produce the same version chain.
- Audit trails (history transitions) are comparable across peers.
- Content hash of the intent entity is identical across peers for the same logical transaction.

---

## 6. Isolation Levels

The transaction extension supports three isolation levels, composable from existing primitives:

### 6.1 Read-committed (default)

- Each read sees the latest committed state.
- No snapshot captured at begin.
- Writes are staged until commit; commit applies them sequentially.
- CAS guards on individual bindings prevent lost writes.

This is the default because it requires no additional state and provides the guarantees most applications need: no dirty reads, CAS-protected writes.

### 6.2 Snapshot isolation

- A trie root is captured at `begin` (or at `execute` entry).
- Reads resolve against the captured trie root, not the live tree.
- The transaction sees a consistent point-in-time view of the tree.
- Writes are staged; commit applies against the current tree.
- No read-set validation at commit — first-committer-wins.

Snapshot isolation requires the trie-as-index mechanism (EXTENSION-TREE.md §3.4). The captured trie root IS the snapshot. Reads resolve by walking the trie structure from the captured root. This is O(depth) per read.

**Write skew.** Snapshot isolation allows write skew: two concurrent transactions read overlapping paths and write to non-overlapping paths based on stale reads. This is the standard snapshot-isolation trade-off. Use serializable isolation to prevent write skew.

### 6.3 Serializable isolation

- Snapshot isolation plus read-set tracking.
- At commit, the transaction validates that no read-set path has been modified since the snapshot.
- If any read-set path changed, the transaction is rejected with `serializable_conflict`.
- The caller retries from `begin`.

Serializable isolation prevents write skew at the cost of potential commit-time rejection. The read-set check is O(|read_set|) — one hash comparison per read path.

---

## 7. Emit Behavior and Versioning Interaction

### 7.1 Per-binding emits fire normally

Each tree.put during commit fires its own cascade. Emit consumers (persistence at 0, query at 1, clock at 2, history at 4, compute at 5, structural summaries at 6, auto-version at 7, subscription at 8) all fire as usual. The transaction layer does NOT suppress per-binding emits.

This is deliberate: persistence needs to mirror each binding, query needs to index each entity, history needs to record each transition. These are per-binding concerns that must run regardless of whether the binding is part of a transaction.

### 7.2 Transaction-committed event

After all bindings are applied, the transaction handler writes the committed record to `system/transaction/committed/{id}`. This write fires its own emit cascade. **This is the observation boundary.**

Consumers that want transactional observation — seeing committed groups rather than individual bindings — subscribe to or watch `system/transaction/committed/*`.

### 7.3 Versioning interaction

Auto-version (EXTENSION-REVISION.md §6.1) fires at position 7 on every tree write. A transaction with N bindings produces N intermediate version entries — identical to merge's intermediate-version behavior (EXTENSION-REVISION.md §4.4.4).

For per-transaction versioning (one version entry per commit instead of per binding), a continuation pipeline bridges the committed event to `revision/commit`. This is a composition pattern described in GUIDE-TRANSACTION.md, not a transaction-extension concern — the transaction extension writes the committed entity; downstream workflows are independent.

---

## 8. Cluster Extensibility

The local transaction extension is designed so cluster transactions compose on top.

### 8.1 Intent as the unit of exchange

`system/transaction/intent` is self-contained — it lists all bindings and their CAS guards. A cluster coordinator creates an intent and sends it to participants. Each participant validates locally and applies via the local `transaction/execute` operation.

### 8.2 Result as the response

`system/transaction/result` is self-contained — it reports what was applied, what failed, and why. Participants send results back to the coordinator.

### 8.3 What cluster transactions add

A future EXTENSION-CLUSTER-TRANSACTION would add:

- **Vote types**: `system/cluster/transaction/vote` (accept/reject per participant).
- **Decision type**: `system/cluster/transaction/decision` (commit/abort after quorum).
- **Coordination protocol**: leader-based (coordinator applies + pushes) or consensus-based (vote collection + quorum).
- **Recovery**: handling participant failure, coordinator failure, network partition.

The local transaction extension doesn't know about clusters. It provides the execution mechanism (apply intent, report result). The cluster extension provides the coordination protocol.

### 8.4 Leader-based (available with current primitives)

No new extension needed. A leader peer:

1. Executes `transaction/execute` locally.
2. On success, produces a version entry via `revision/commit`.
3. Pushes the version to followers via `revision/push`.
4. Followers merge the version into their tree.

This uses: EXTENSION-TRANSACTION (local) + EXTENSION-REVISION (version + push + merge). No cluster-specific types or protocols. The limitation: single point of coordination, no pre-commit consensus.

---

## 9. Conformance

### 9.1 MUST

- `execute` operation applies all bindings or reports partial failure.
- CAS guards (`expected_hash`) are checked before any binding is applied.
- Capability is verified for every path in the intent before any binding is applied.
- Committed record is written to `system/transaction/committed/{id}` on successful commit.
- Compensating writes are attempted for all applied paths when `rollback_on_failure` is true and a binding fails.
- Per-binding tree.put calls follow standard cascade semantics (SYSTEM-COMPOSITION v1.5).
- Status 207 from a sub-write (cascade halt) is treated as success — the binding landed. The transaction continues.
- Status 4xx/5xx from a sub-write (pre-write rejection) stops the transaction.
- Bindings in the intent MUST be sorted by path. Unsorted intents are rejected (400).

### 9.2 SHOULD
- Snapshot and serializable isolation supported (requires trie-as-index).
- Interactive operations (`begin`, `write`, `read`, `commit`, `rollback`, `status`) supported.
- Rolled-back record written to `system/transaction/rolled-back/{id}` on compensating rollback.

### 9.3 MAY

- Timeout for pending interactive transactions (prevent abandoned transactions from accumulating).
- Metrics for transaction throughput, failure rates, cascade-warning frequency.
- Integration with EXTENSION-HISTORY for transaction-level audit (one history entry per transaction, not per binding).

---

## 10. Constants and Limits

| Constant | Default | Description |
|----------|---------|-------------|
| `max_bindings_per_transaction` | 1000 | Maximum bindings in a single intent. Prevents unbounded cascade depth. |
| `max_pending_transactions` | 100 | Maximum concurrent interactive transactions per peer. |
| `pending_transaction_timeout` | 300s | Time after which a pending interactive transaction is automatically rolled back. |

Implementations MAY adjust these defaults. The values are implementation-defined; the existence of limits is normative.

---

## 11. Document History

- v0.1: Initial draft. Core types, operations, isolation levels, emit behavior, versioning interaction, cluster extensibility.
