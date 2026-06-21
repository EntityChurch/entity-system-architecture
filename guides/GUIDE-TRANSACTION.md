# Transaction Extension: Developer Guide (Exploratory)

**Status**: Draft

**Audience:** Application developers building on the entity system who need multi-path consistency. Not a spec. EXTENSION-TRANSACTION.md is normative; this guide is descriptive.

**Scope:** When to use transactions, how reads work, how to avoid intermediate-state visibility, and composition patterns with versioning and continuation pipelines.

---

## 1. The one-paragraph mental model

The entity system's tree provides per-path atomic writes. If your operation touches one path, you don't need transactions. If it touches multiple paths that must be consistent — a config and its corresponding tracking state, a document and its index entry, a batch of settings that should all change together — the transaction extension groups those writes so that observers above the transaction layer see only the committed result, never an intermediate state.

The raw tree still sees individual writes. Transactions don't change the tree's behavior — they add an observation boundary on top.

---

## 2. When to use transactions

### You need transactions when:

- **Multi-path consistency.** Two or more bindings must agree. Example: revision config + tracking-config. Example: a user profile entity + its index entry in a lookup table.
- **Batch operations.** Applying N config changes where observers should see all-or-nothing. Example: migrating a schema across 50 entities.
- **Read-then-write patterns.** You read state A, decide what to write to state B based on A, and need a guarantee that A didn't change before B lands. Example: checking a balance before issuing a transfer.

### You don't need transactions when:

- **Single-path operations.** tree.put is already atomic per path. CAS (`expected_hash`) handles concurrent writes.
- **Append-only data.** Writing immutable entities to the content store has no consistency concern — content addressing deduplicates naturally.
- **Independent writes.** Two paths that don't need to agree. Write them separately.
- **Handler operations that already coordinate.** `revision/commit`, `revision/merge`, and similar ops internally coordinate multi-path writes. You don't need to wrap them in a transaction — they already handle their own consistency.

---

## 3. How reads work — the important part

This is the most common source of confusion. The transaction extension groups writes. But **what about reads?**

### 3.1 The problem

When a transaction is in progress (bindings being applied one by one via tree.put), the raw tree is in an intermediate state. If you read the raw tree during this window, you see a mix of pre-transaction and post-transaction bindings. This is not a bug — it's how the raw tree works (per-binding atomicity, no multi-path snapshot).

### 3.2 Three read strategies

**Strategy A: Read through the transaction handler (interactive transactions).**

If you're inside an interactive transaction (`begin` → `read` → `write` → `commit`), the `transaction/read` operation provides isolation:

- Under **snapshot isolation**: reads resolve against the trie root captured at `begin`. You see a consistent point-in-time view. Changes by other writers are invisible until your transaction ends.
- Under **serializable isolation**: same as snapshot, plus the transaction tracks what you read. At commit, it verifies nothing you read has changed. If it has, commit is rejected and you retry.
- Under **read-committed**: reads go to the live tree. You see the latest committed state per path, but no cross-path consistency.

**Strategy B: Use the transaction handler for consistent reads (even without writes).**

If you're not inside a write transaction but want consistent reads, use an interactive transaction in read-only mode:

```
; Begin a snapshot-isolation transaction (no writes needed)
EXECUTE system/transaction  operation: "begin"
  params: { prefix: "your-app/", isolation: "snapshot" }
→ { transaction_id: "tx-readonly-1" }

; Read through the transaction — sees a consistent snapshot
EXECUTE system/transaction  operation: "read"
  params: { transaction_id: "tx-readonly-1", path: "your-app/config/settings" }
→ { hash: "sha256:...", source: "snapshot" }

EXECUTE system/transaction  operation: "read"
  params: { transaction_id: "tx-readonly-1", path: "your-app/users/alice" }
→ { hash: "sha256:...", source: "snapshot" }

; When done, rollback (no writes to apply)
EXECUTE system/transaction  operation: "rollback"
  params: { transaction_id: "tx-readonly-1" }
```

The transaction handler captures a trie root at `begin` and resolves reads against it internally. The capability check uses full tree paths (the handler knows the prefix). You get a consistent point-in-time view without manually handling trie roots or worrying about intermediate states.

**When do you advance your view?** Start a new read-only transaction. Each `begin` captures the current trie state. Subscribe to `system/transaction/committed/*` if you want to know when to refresh.

**Why not `tree.get` with a root parameter?** A direct snapshot-read on `tree.get` was considered and reverted (PROPOSAL-REVERT-TREE-SNAPSHOT-READ). Content-addressed trie roots carry no prefix metadata, so the tree handler can't do correct capability checking against trie-relative paths. The transaction handler solves this because it holds the prefix in its own state.

**Strategy C: Read the raw tree and accept eventual consistency.**

For applications where brief inconsistency is acceptable (dashboards, metrics, debugging), read the raw tree directly. Individual reads are always consistent per-path. Cross-path consistency is not guaranteed but converges quickly (the transaction completes in milliseconds).

### 3.3 Choosing a strategy

| Use case | Strategy | Why |
|---|---|---|
| Interactive read-write workflow | A (transaction handler) | Isolation guarantees, read-set tracking |
| Application UI displaying consistent data | B (read-only transaction) | Consistent view, no intermediate states, correct capability checking |
| Monitoring / debugging | C (raw tree) | Simplicity, real-time visibility |
| Emit consumers (persistence, query, etc.) | C (raw tree) | Per-binding processing, always see latest |

**For most applications, Strategy A or B is the default.** Read through the transaction handler for consistency. Use raw tree reads only when you don't need cross-path consistency.

---

## 4. Single-shot transactions (the common case)

Most transactions are single-shot: you know all the bindings upfront, you apply them as a group.

```
; Build the intent
intent = {
  type: "system/transaction/intent",
  data: {
    prefix: "app/config/"
    bindings: [
      { path: "app/config/cache",     hash: hash_of_new_cache_config },
      { path: "app/config/database",  hash: hash_of_new_db_config },
      { path: "app/config/version",   hash: hash_of_version_bump }
    ],
    ; bindings MUST be sorted by path (they are above — cache, database, version)
    isolation: "read-committed",
    rollback_on_failure: true
  }
}

; Execute
EXECUTE system/transaction  operation: "execute"
  params: intent

; Result tells you what happened
result.status == "committed"   → all bindings applied, committed record written
result.status == "partial"     → some bindings failed, see result.failed
result.status == "rolled-back" → failure + compensation, see result.compensated
result.status == "rejected"    → CAS mismatch or capability denial, nothing applied
```

### 4.1 CAS guards for optimistic concurrency

Add `expected_hash` to any binding you want to protect from concurrent writes:

```
bindings: [
  { path: "app/config/database",
    hash: hash_of_new_config,
    expected_hash: hash_of_current_config    ; fail if someone changed it
  }
]
```

All CAS guards are checked **before** any binding is applied. If any guard fails, the entire transaction is rejected (status "rejected", no bindings applied). This is the optimistic concurrency model — read the current state, build your intent with CAS guards, submit. If something changed in between, retry.

---

## 5. Interactive transactions (read-then-write)

When your writes depend on reads:

```
; 1. Begin — capture a snapshot
EXECUTE system/transaction  operation: "begin"
  params: { prefix: "app/accounts/", isolation: "snapshot" }
→ { transaction_id: "tx-123", snapshot_root: "sha256:abc..." }

; 2. Read through the transaction — sees the snapshot, not the live tree
EXECUTE system/transaction  operation: "read"
  params: { transaction_id: "tx-123", path: "app/accounts/alice/balance" }
→ { hash: "sha256:def...", source: "snapshot" }

; 3. Application logic: check balance, compute transfer
alice_balance = content_store.get("sha256:def...")
; ... decide what to write ...

; 4. Stage writes
EXECUTE system/transaction  operation: "write"
  params: { transaction_id: "tx-123", binding: { path: "app/accounts/alice/balance", hash: new_alice_hash } }

EXECUTE system/transaction  operation: "write"
  params: { transaction_id: "tx-123", binding: { path: "app/accounts/bob/balance", hash: new_bob_hash } }

; 5. Commit — applies all staged bindings
EXECUTE system/transaction  operation: "commit"
  params: { transaction_id: "tx-123" }
→ result.status == "committed" | "rejected" | "partial"
```

For serializable isolation (prevents write skew), use `isolation: "serializable"`. The transaction tracks every path you read. At commit, it checks that none of those paths changed since your snapshot. If they did, the commit is rejected and you retry from step 1.

---

## 6. Versioning with transactions

The transaction extension doesn't know about versioning. But a common question: "How do I get one version entry per transaction instead of one per binding?"

### 6.1 The problem

Auto-version (EXTENSION-REVISION §6.1) fires at emit position 7 on every tree.put. A transaction with 10 bindings produces 10 intermediate version entries. For batch operations, you want one version capturing the final state.

### 6.2 The pattern: continuation-pipeline versioning

1. **Don't use auto-version for this prefix.** Set `auto_version: false` in the revision config (or don't configure revision for this prefix).

2. **Set up a standing continuation** at `system/transaction/committed/*`:

```
; system/continuation/on-transaction-commit
type: system/continuation
data:
  target: "system/revision"
  operation: "commit"
  params:
    prefix: "app/"                    ; the prefix to version
    message: "Transaction commit"     ; optional commit message
  result_field: "trigger"             ; the committed record arrives here
  remaining_executions: null          ; standing — fires on every committed
```

3. **When a transaction commits**, the committed record lands at `system/transaction/committed/{id}`. The standing continuation fires, dispatching `revision/commit` for the prefix. One version entry is created capturing the post-transaction tree state.

This composition uses: EXTENSION-TRANSACTION (writes the committed entity) + EXTENSION-CONTINUATION (triggers the workflow) + EXTENSION-REVISION (creates the version). Each extension does its own job; the continuation pipeline connects them.

### 6.3 When to use which versioning strategy

| Scenario | Strategy | Why |
|---|---|---|
| Collaborative editing (real-time) | Auto-version (per-write) | Every keystroke / save matters |
| Batch config update | Continuation-pipeline (per-transaction) | Only the final state should be versioned |
| Database migration | Continuation-pipeline (per-transaction) | The migration is one logical change |
| Manual checkpoints | Direct `revision/commit` (no auto-version) | User decides when to version |
| Mixed workload | Auto-version for some prefixes, transaction-versioning for others | Different prefixes, different needs |

---

## 7. Rollback and what it actually means

Rollback in the transaction extension is **compensating writes** — not undo.

When `rollback_on_failure: true` and a binding fails mid-transaction:

1. Bindings 1..K have been applied to the tree (K tree.puts completed).
2. Binding K+1 failed.
3. The transaction handler writes the **previous** value back to paths 1..K.

Raw-tree consumers see three state transitions per path: original → new → reverted. Transaction-layer consumers (watching `system/transaction/committed/*`) see only the rolled-back record — they never saw the intermediate state.

**Compensating writes are best-effort.** If a compensating write fails (disk full, capability issue), the path remains at its committed value. The result tells you which paths were compensated and which weren't. This is honest about the guarantee.

**If you need stronger rollback**, use snapshot isolation: read from the snapshot, stage writes, and only commit when you're sure. If commit is rejected, nothing was applied — there's nothing to roll back.

---

## 8. Patterns and anti-patterns

### Do:

- **Sort bindings by path.** The spec requires it. Build your intent with sorted bindings.
- **Use CAS guards on paths you care about.** Optimistic concurrency prevents lost writes.
- **Read from trie snapshots** when you need cross-path consistency outside a transaction.
- **Keep transactions small.** 5-50 bindings is typical. 1000 is the default limit. Larger transactions hold intermediate state longer.
- **Use single-shot `execute` when you can.** Simpler, less state to manage.

### Don't:

- **Don't read the raw tree and expect transactional consistency.** The raw tree shows individual bindings as they land. Use Strategy A or B from §3.
- **Don't wrap single-path writes in transactions.** tree.put + CAS is sufficient. The transaction adds overhead without benefit.
- **Don't leave interactive transactions open.** They hold state at `system/transaction/pending/{id}`. There's a timeout (default 300s), but don't rely on it.
- **Don't assume rollback is atomic.** It's compensating writes. Best-effort. Use snapshot isolation if you need zero intermediate visibility for other transaction-layer consumers.
- **Don't nest transactions.** The extension doesn't support nested transactions. If an operation within a transaction needs its own atomicity, it writes to the transaction's intent, not to the tree.

---

## 9. What transactions don't solve

- **Cross-peer consistency.** Transactions are local to one peer. For multi-peer coordination, use the version DAG (revision push/merge) or a future cluster-transaction extension.
- **Before-write validation on arbitrary paths.** Transactions validate the paths IN the transaction. They don't prevent other writers from tree.put-ing to those paths concurrently. Capability scoping (restricting who can write which paths) is the mechanism for that.
- **Real-time consistency for raw-tree readers.** Consumers that read the raw tree (emit consumers, direct tree.get) see intermediate states. This is by design — persistence, query indexing, and history recording all need per-binding visibility.
