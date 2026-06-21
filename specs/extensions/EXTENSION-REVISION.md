# System Revision Extension

**Version**: 3.8

**Status**: Active
**Depends**: ENTITY-CORE-PROTOCOL.md (v7.26+), EXTENSION-TREE.md (v3.3+), SYSTEM-COMPOSITION.md (v1.5+)
**Optional**: EXTENSION-HISTORY.md (v1.0+) — enriches versioning with per-path detail
**Optional**: EXTENSION-SUBSCRIPTION.md (v3.4+) — cross-peer version following (see §6.3)
**Optional**: EXTENSION-CONTINUATION.md (v1.2+) — cross-peer version following (see §6.3)
**Encoding**: ENTITY-CBOR-ENCODING.md (ECF)

---

> **Path notation.** Paths in this document use peer-relative notation (without leading `/{peer_id}/`). All peer-relative paths resolve to the local peer's namespace: `system/tree` means `/{local_peer_id}/system/tree`. Every path in the entity tree is absolute at rest — rooted at a peer identity. See ENTITY-CORE-PROTOCOL.md §1.4 for the path model. Cross-peer examples use absolute paths with explicit peer identities.
>
> **Prefix convention.** The `prefix` field in revision config entities stores absolute paths (e.g., `/{peerA}/project/`). All revision operations that accept a `prefix` parameter resolve it to absolute via standard path resolution (V7 §5.4) before computing the prefix hash: a prefix without a leading `/` resolves to the local peer's namespace; a prefix with a leading `/` is used as-is. To operate on any prefix outside the local peer's namespace, callers MUST provide the full absolute path. Peer-relative notation in this document is a documentation convenience; stored and hashed prefixes are always absolute.

## 1. Overview

This extension provides versioning, merge, and peer-to-peer synchronization for entity trees. It builds on the tree extension's structural primitives (snapshot, diff, merge, extract) and adds version semantics: a directed acyclic graph (DAG) of versions, a merge strategy framework, and a sync protocol for coordinating state between peers.

### 1.1 Scope

This extension covers:

- **Versions** — content-addressed trie roots with parent lineage
- **Version DAG** — history traversal, common ancestor finding, divergence detection
- **Merge framework** — per-path configurable merge strategies with handler delegation
- **Conflict representation** — conflicts as entities, not blocking state
- **Sync protocol** — version negotiation, delta transfer, integration between peers
- **Auto-versioning** — automatic version creation on writes to configured prefixes

This extension does **not** cover:

- Per-path transition recording (history extension)
- Content chunking for large entities (content extension)
- Real-time collaborative editing (application-level, uses merge framework)
- Working copy / filesystem integration (implementation-specific)

### 1.2 Design Principles

**Build from entity primitives.** Versions are entities. The version DAG is entities referencing entities by hash. Snapshots are tree extension entities. No new primitives — only new entity types and a handler.

**Snapshot-based, not patch-based.** Each version references a complete trie root (tree state). Diffs are computed, not stored. This matches the entity tree's architecture (bindings are current state, not change logs).

**Conflicts as side-channel metadata.** When merge cannot resolve a path, the local entity stays at the path and a conflict entity is stored at a system path (`system/revision/conflicts/`). The document never disappears — handlers reading the path get the real entity, not conflict metadata. Resolution replaces the entity at the path and removes the conflict entry.

**Prefix-scoped versioning.** Versions cover a subtree prefix, not the entire tree. Different prefixes have independent version histories. You version what matters.

**Three-way merge as default.** The most widely understood merge algorithm. CRDT and custom merge strategies plug into the same framework.

**CRDT is a merge algorithm, not a storage format.** Entities are regular entities. CRDT logic is invoked during merge to resolve conflicts. No persistent CRDT metadata.

**Metadata is tree content.** The version entry is purely structural — trie root + sorted parents. Commit metadata (author, timestamp, message) is stored as entities at paths in the tree, managed by application convention. The version system is agnostic to what's in the trie.

**Immutable once shared.** Local versions can be rewritten (creates a new version). Once a version has been synced to another peer, it's effectively immutable — the version hash has been observed externally.

### 1.3 Relationship to Tree Extension

The revision extension uses four tree extension operations:

| Tree Operation | Version Usage |
|----------------|-----------|
| `snapshot` / `build_trie` | Captures tree state as a trie root — the version's content reference |
| `diff` | Computes what changed between two versions' trie roots |
| `extract` | Bundles entities for transfer between peers |
| `merge` | Applies incoming trie content into the local tree |

The version workflow: `build_trie → (version entity) → diff → extract → merge`

View trees (tree extension §8) provide capability-scoped versioning: a peer sees only what their capability allows them to see.

---

## 2. Types

### 2.1 Version Entity

The fundamental unit of the version DAG:

```
system/revision/entry := {
  fields: {
    root:    {type_ref: "system/hash"}
              ; Content hash of the trie root node (system/tree/snapshot/node)
    parents: {array_of: {type_ref: "system/hash"}}
              ; Parent version hashes, sorted by hash value.
              ; 0 = initial. 1 = linear. 2+ = merge.
  }
}
```

Two fields. The version entry is the minimal structural DAG node: what's in the tree (trie root) + what came before (parents).

**Content-addressed.** The version's `content_hash` is computed per ENTITY-CORE-PROTOCOL.md §1.2 using the peer's configured `content_hash_format` over the canonical encoding of `{type: "system/revision/entry", data: ...}`. Parents are in data, so they're included in the hash. The version DAG has cryptographic integrity — a version's ancestors cannot be altered without changing its hash. With only two fields — both deterministic — two peers using the same `content_hash_format` and performing the same merge produce the exact same version entry hash (two peers on different formats produce different hashes for the same logical version — the v7.66 §5.3 expectation, not a bug).

**Location-independent.** The version entry contains no prefix, author, timestamp, or message. The same content at different mount points (`A/data/project/` and `B/projects/myproject/`) shares version entries and version history. The prefix lives externally in the head pointer (see §3.1). This is the git model — the commit hash does not include the working directory path.

**Metadata is tree content.** Commit metadata (author, timestamp, message) is stored as entities at paths in the tree, managed by application convention. The version system is agnostic to what's in the trie. See §1.2.

**Sorted parents.** The `parents` array MUST be sorted by hash value (lexicographic comparison of binary hash representation). Without sorting, parent order reflects which side was "local" vs "remote" — a per-peer distinction that would produce different hashes for the same merge. The sort is deterministic, simple, and consistent with CBOR array encoding.

**Parent semantics:**

| Parents | Meaning |
|---------|---------|
| `[]` (empty) | Initial version — first commit |
| `[V]` (one) | Linear update — no sorting needed (one element) |
| `[V_a, V_b]` (two, sorted) | Merge — reconciling two divergent branches |
| `[V_a, V_b, V_c, ...]` (n, sorted) | N-way merge — reconciling multiple branches |

### 2.2 Conflict Entity

Stored at a **system path** when merge cannot automatically resolve divergence. The entity at the conflicting path is NOT replaced — the document stays visible.

```
system/revision/conflict := {
  fields: {
    path:     {type_ref: "system/tree/path"}
               ; Trie-relative path within the prefix where the conflict exists
    base:     {type_ref: "system/hash", optional: true}
               ; Common ancestor entity hash (absent for create/create conflicts)
    local:    {type_ref: "system/hash", optional: true}
               ; Local entity hash (what's currently visible at the path).
               ; Absent when local deleted the path.
    remote:   {type_ref: "system/hash", optional: true}
               ; Remote entity hash (what the remote peer has).
               ; Absent when remote deleted the path.
    strategy: {type_ref: "primitive/string"}
               ; The merge strategy that was attempted
    version_local:  {type_ref: "system/hash"}
                     ; Local version hash
    version_remote: {type_ref: "system/hash"}
                     ; Remote version hash
    supersedes: {type_ref: "system/hash", optional: true}
                 ; Content hash of the conflict entity this replaces.
                 ; Present when a new merge produces a conflict at a path
                 ; that already has an unresolved conflict. The prior
                 ; conflict entity remains in the content store.
  }
}
```

**Side-channel storage.** Conflict entities are stored at `system/revision/{H}/conflicts/{relative_path}` (where `{H}` is the prefix hash per §3.1), NOT at the conflicting path itself. The entity at the original path remains the local version — the document never disappears.

```
; Before merge:
project/config.toml                              → hash_of_local_config

; After merge with conflict:
project/config.toml                              → hash_of_local_config  (unchanged!)
system/revision/{H}/conflicts/config.toml             → hash_of_conflict_entity
```

Handlers reading `project/config.toml` get the config entity — they work normally. Only version-aware handlers/UIs check `system/revision/{H}/conflicts/` to discover and present unresolved conflicts.

**Resolution.** Put the resolved entity at the path (`project/config.toml`), then remove the conflict entry from `system/revision/{H}/conflicts/`. The resolve operation (§4.4.5) handles both steps atomically.

**Conflicts are peer-local.** Conflict entities at `system/revision/conflicts/` are stored under the `system/` prefix, outside the version's prefix scope. They are NOT included in version snapshots and are NOT synced to other peers. Conflicts are local working state — the version DAG records the merge (with both parents), but the conflict metadata is for local resolution only. If a peer receives a merge version, it gets the merged snapshot (with local versions at conflicted paths) but not the conflict entities.

**Stacking conflicts.** When a merge produces a conflict at a path that already has an unresolved conflict (from a prior merge), the new conflict supersedes the old one. The `supersedes` field links the new conflict to the old conflict entity (which remains in the content store). The conflict at the system path always reflects the most recent merge's divergence. Prior conflict entities are reachable via the `supersedes` chain.

### 2.3 Merge Configuration

Per-path-pattern configuration for merge strategies:

```
system/revision/merge-config := {
  fields: {
    pattern:              {type_ref: "system/tree/path"}
                           ; Path pattern (glob)
    strategy:             {type_ref: "primitive/string"}
                           ; Strategy name: "three-way", "source-wins", "target-wins",
                           ;   "lww", "keep-both", "manual", or a handler path for custom
    handler:              {type_ref: "system/tree/path", optional: true}
                           ; Custom merge handler path (for strategy = handler path)
    deletion_resolution:  {type_ref: "primitive/string", optional: true}
                           ; Deletion-vs-entity resolution strategy (v3.1, A4).
                           ; Default "preserve-on-conflict"; also "deletion-wins",
                           ;   "three-way-fallthrough", "deterministic", or a handler path.
                           ; "lww" and "keep-both" MUST be rejected at config-write time.
  }
}
```

Stored at `system/revision/config/merge/path/{name}` (per-path) or `system/revision/config/merge/type/{type_name}` (per-type). See §5.1 for the merge resolution cascade. See §3.1.1 for the complete path inventory.

**Built-in strategies:**

| Strategy | Behavior |
|----------|---------|
| `three-way` | Diff3 with common ancestor. Auto-merge non-overlapping changes. Conflict on overlap. |
| `source-wins` | Always take the incoming (remote) version for conflicting paths |
| `target-wins` | Always keep the local version for conflicting paths |
| `lww` | Last-write-wins — compare version timestamps, take the newer one |
| `keep-both` | Store both versions (local at path, remote at `{path}.keep-both-{hash_prefix}`) |
| `manual` | Always create a conflict entity, even for auto-resolvable changes |

**KeepBoth path naming.** The `keep-both` strategy creates a second binding at `{path}.keep-both-{hash_prefix}` where `hash_prefix` is the first 8 hex characters of the second entity's content hash. The hash prefix provides a deterministic, unique suffix independent of peer identity. Under deterministic merge ordering, "local" and "remote" are assigned by hash comparison, not by peer — using peer IDs in the path name would be misleading. The `.keep-both-` infix distinguishes these entries from conflict entities (which use `system/revision/conflicts/`). KeepBoth only applies to edit-vs-edit conflicts (both sides non-null). Delete-vs-edit conflicts fall through to conflict entities — "keeping both" when one side is a deletion is not meaningful.

**KeepBoth bindings are versioned.** The additional binding at the `.keep-both-` path falls under the prefix and is included in the version snapshot. Subsequent merges see it as a normal binding. If a future merge also produces keep-both at the same original path, a new `.keep-both-{different_hash_prefix}` binding is created — keep-both paths accumulate until manually cleaned up or until a different strategy resolves the divergence.

**Custom strategies.** When `strategy` is a handler path (e.g., `app/merge/text-handler`), the revision handler delegates content-level merge to that handler. The custom handler receives base, local, and remote entities and returns a merged entity or conflict.

**Deletion-vs-entity resolution (v3.1, Amendment 4).** When the three-way merge classifies a (local, remote) pair as "both changed differently" and exactly one side is a deletion-marker binding (per §4.4.4 and ENTITY-NATIVE-TYPE-SYSTEM.md §4.9 `system/deletion-marker`), the `deletion_resolution` strategy applies. Default is `preserve-on-conflict`. Strategies:

| `deletion_resolution` | Behavior |
|---|---|
| `preserve-on-conflict` (default) | The entity supersedes the deletion marker; the delete is silently discarded; no conflict entity written. Recommended for collaborative-edit workflows where edit preservation is the priority. |
| `deletion-wins` | The deletion marker supersedes the entity; sticky delete; no conflict entity written. Recommended for security-sensitive workflows where delete intent MUST be respected (access-control revocation, etc.). |
| `three-way-fallthrough` | Falls through to the existing conflict-entity behavior per §2.2: local-by-deterministic-hash entity stays at the path (or path unbinds if local-by-deterministic-hash is the marker side); a conflict entity is written at `system/revision/{H}/conflicts/{path}` recording base/local/remote hashes, the strategy attempted, and version inputs. For workflows that want explicit conflict signaling. |
| `deterministic` | Content-hash ordering between the entity hash and `CANONICAL_DELETION_MARKER_HASH`. **Direction (v3.3):** the **lower** of `entity_hash` and `CANONICAL_DELETION_MARKER_HASH` under byte-wise lexicographic comparison wins. Convergent but arbitrary — the winner depends on byte-ordering, not semantics. No conflict entity written. |
| Custom handler path | Application-defined. Same framework as the `strategy` field. |

**Honest framing.** Neither `preserve-on-conflict` nor `deletion-wins` is "lossless" — both silently drop one operator's signal in concurrent edit-vs-delete: `preserve-on-conflict` drops the DELETE signal (the delete vanishes after the concurrent edit syncs back); `deletion-wins` drops the EDIT signal (the edit vanishes after the concurrent delete syncs in). The default is chosen for collaborative-edit workloads (users actively editing; surprise edit-loss is more disruptive than surprise resurrection). Operators with different priorities pick a different strategy explicitly.

**Rejected values.** `keep-both` is NOT a valid value for `deletion_resolution`. Per §193 of this spec, KeepBoth only applies to edit-vs-edit conflicts; delete-vs-edit falls through to conflict entities — "keeping both" when one side is a deletion is not meaningful. Implementations encountering `deletion_resolution: keep-both` MUST reject with `invalid_strategy` at config-write time. `three-way-fallthrough` provides the conflict-entity semantic operators might have wanted from `keep-both`.

`lww` is NOT a valid value for `deletion_resolution`. Real LWW requires commit-metadata not currently spec'd; canonical deletion markers (ENTITY-NATIVE-TYPE-SYSTEM.md §4.9) cannot carry timestamps without breaking the DAG convergence property. Implementations encountering `deletion_resolution: lww` MUST reject with `invalid_strategy` at config-write time. Application-layer LWW remains available via custom-handler.

**Deletion-vs-deletion at the same path is not a divergent case** — both sides produce the same canonical hash (the marker is canonical); the three-way classification is "same on both sides," not "both changed differently."

**Handler-owned namespace (v3.3, D1).** `system/revision/config/merge/{path,type}/*` is a **handler-owned namespace**. The canonical write path is the `merge-config` handler operation (§4.4.18) — that's where the strategy-rejection contract above (`lww` / `keep-both` rejected at config-write time with `400 invalid_strategy`) is enforced, where field-shape validation runs, and where audit / future-evolution hooks live. Operator capabilities granting `system/tree:put` on this namespace are a **deployment misconfiguration** rather than a protocol-supported write path — they bypass the handler op and therefore bypass the strategy-rejection contract, allowing invalid persisted values that read-time defensive validation can only collapse (silently dropping the operator's intent) rather than reject (returning a clear error). Default operator-capability scopes SHOULD NOT include `system/tree:put` on this namespace; see `GUIDE-CAPABILITIES.md` §8.1 ("Broad `tree:put` on a system handler namespace") and §5 ("Per-extension coherent capability checklist") for the deployment posture. This declaration anchors the existing guide guidance to a specific spec namespace.

### 2.4 Version Configuration

Per-prefix version behavior:

```
system/revision/config := {
  fields: {
    prefix:       {type_ref: "system/tree/path"}
                   ; Absolute tree path of the subtree to version.
                   ; Peer-relative input is resolved to absolute on config
                   ; write via standard path resolution (V7 §5.4).
    exclude:      {array_of: {type_ref: "primitive/string"}, optional: true}
                   ; Glob-style path patterns relative to `prefix`.
                   ; Examples: `ephemeral/**`, `**/*.cache`.
    exclude_types: {array_of: {type_ref: "primitive/string"}, optional: true}
                   ; Entity types to exclude from versioned tries and transfer.
                   ; E.g., ["app/temp/scratch", "system/protocol/*"].
                   ; NOTE: `exclude_types` filters entities stored at tracked tree
                   ; paths. It does NOT prevent reentrancy from the revision
                   ; extension's own metadata writes — reentrancy is addressed by
                   ; path-pattern `exclude` (see §6.1 "Reentrancy"). Setting
                   ; `exclude_types: ["system/revision/entry"]` has no effect,
                   ; because version entries are stored in the content store and
                   ; referenced by hash from `system/revision/head/**`, not bound
                   ; at tracked paths.
    auto_version: {type_ref: "primitive/bool", optional: true}
                   ; Create versions automatically on writes. Default: false.
    merge_order:  {type_ref: "primitive/string", optional: true}
                   ; "deterministic" (default) | "caller-perspective"
                   ; Controls local/remote assignment in merge.
                   ; "deterministic" produces convergent DAGs across peers without
                   ; coordination — required for p2p auto-versioning and the default
                   ; for all new configurations. "caller-perspective" preserves
                   ; git-style caller-centric history; appropriate for
                   ; single-authority deployments or when history from one peer's
                   ; perspective is more important than cross-peer convergence.
                   ; Operators choosing "caller-perspective" with auto_version: true
                   ; MUST also raise oscillation_depth to at least 2N where N is the
                   ; mesh size.
    oscillation_depth: {type_ref: "primitive/uint", optional: true}
                   ; Default: 4. Minimum: 2.
                   ; Controls how many DAG levels the BFS checks for oscillation.
                   ; Depth needed scales with cycle length:
                   ;   2 peers oscillating: cycle length 2 → depth 4
                   ;   3 peers in ring: cycle length ~3 → depth 6
                   ;   N peers in cycle: depth ~2N
                   ; Larger values catch longer merge cycles at the cost of DAG-walk time.
                   ; Not needed with deterministic ordering — no oscillation occurs.
  }
}
```

**Overlapping prefixes.** When a tree write matches multiple revision configs (e.g., a nested prefix within a broader one), auto-version fires **once per matching config**, producing one version entry in each config's DAG (one head per config, see §3.1). Each DAG is independent; the operator sees parallel version histories at different granularities. If this is not the desired behavior, operators SHOULD configure non-overlapping prefixes.

Rationale: firing once per config is the only option that preserves each config's stated contract ("every write under this prefix produces a version"). Picking a single "winning" prefix would silently drop versions from other configs' DAGs. Operators who want single-DAG coverage should use a single prefix; operators who want multi-granularity coverage should accept the per-config firing. Cross-config merge strategy and exclude composition are each config-scoped — the inner config's settings apply to its DAG, the outer's to its DAG.

Stored at `system/revision/{H}/config` where `{H}` is the prefix hash (§3.1). See §3.1.1 for the complete path inventory.

**Exclude patterns.** When computing bindings for a version, paths matching any `exclude` pattern are omitted. When syncing, excluded paths are not transferred. This is the entity equivalent of `.gitignore` — but stored as typed configuration, not a text file.

```
; Example: version a project, excluding ephemeral artifacts
; Example: version a project (H = prefix_hash("/{peerA}/project/"))
system/revision/{H}/config := {
  type: "system/revision/config"
  data: {
    prefix:       "/{peerA}/project/"
    exclude:      ["ephemeral/**", "**/*.cache"]
    exclude_types: ["app/temp/scratch"]
    auto_version: false
  }
}
```

**Type-based exclude.** `exclude_types` filters by entity type rather than path. When computing bindings, entities whose type matches any `exclude_types` pattern are omitted regardless of path. This handles transient entity types that shouldn't be versioned regardless of where they appear.

**Version metadata exclude.** When `auto_version` is enabled for a prefix that encompasses the revision extension's metadata paths (`system/revision/**`), `exclude` MUST include `"system/revision/**"` along with the other required excludes enumerated in §6.1 "Reentrancy". The `revision/config` operation (§4.4.17) rejects invalid configs before writing — see §4.4.17 V2 for the exclude-validation rule.

**Config writes go through the handler.** Prefix config writes (`system/revision/{H}/config`) MUST go through the `revision/config` operation (§4.4.17), not direct tree.put. The operation validates the config entity (§4.4.17 V1-V5), resolves the prefix to absolute, computes the prefix hash, and coordinates tracking-config writes (§6.1) within a single handler invocation. Direct tree.put to `system/revision/*/config` from external callers is gated by capability — the revision handler's grant holds write access; external callers do not. See §4.3.

**Trie root exclude.** When a versioned prefix encompasses `system/tree/root/`, the exclude patterns SHOULD include `system/tree/root/**`. Trie root hashes are derived state — including them in versioned bindings creates a circular dependency (the trie root hash depends on all bindings under the prefix, including itself). The version entry's `root` field already captures the trie root at each commit point, making the tracked root path redundant for version purposes. See EXTENSION-TREE.md §3.4.1 for the full rationale.

**Exclude applies to trie building.** The `compute_versioned_bindings` call during `commit` applies the exclude filters:

```
compute_versioned_bindings(tree, prefix, config):
  bindings = {}
  for (path, hash) in tree:
    if path starts with prefix:
      relative = path[len(prefix):]
      if any(glob_match(pattern, relative) for pattern in config.exclude):
        continue    ; excluded by path
      if config.exclude_types:
        entity = content_store.get(hash)
        if any(glob_match(pattern, entity.type) for pattern in config.exclude_types):
          continue  ; excluded by type
      bindings[relative] = hash
  return sorted(bindings.items())
```

### 2.5 Version Status

Current version state for a prefix:

```
system/revision/status := {
  fields: {
    prefix:   {type_ref: "system/tree/path"}
    head:     {type_ref: "system/hash", optional: true}
               ; Current local version hash. Absent if no versions exist.
    remotes:  {map_of: {type_ref: "system/hash"}, optional: true}
               ; Remote peer ID → their last known version hash
    conflicts: {type_ref: "primitive/uint"}
               ; Number of unresolved conflict entities in the tree
    pending:   {type_ref: "primitive/uint"}
               ; Number of path changes since last version (if history extension present)
  }
}
```

---

## 3. Version DAG

### 3.1 Storage

Version entities are stored in the content store (content-addressed, immutable). All per-prefix metadata is stored under a hash-addressed subtree:

```
system/revision/{prefix_hash}/...
```

Where `prefix_hash` is the hex-encoded ECF content hash of the absolute prefix path:

```
prefix_hash(prefix) = hex(content_hash(type="system/tree/path", data=prefix))
```

This produces a 66-character lowercase hex string (2 chars `00` format code + 64 chars SHA-256 digest), matching the signature storage invariant pointer convention (ENTITY-CORE-PROTOCOL.md §7.4). The hash segment is structurally distinct from metadata names — no collision is possible (see §3.1.1).

Version heads, branches, tags, conflicts, remotes, and config all live under the same prefix subtree:

```
system/revision/{H}/head                →  hash of current version entity
system/revision/{H}/active-branch       →  branch name (e.g., "main")
system/revision/{H}/branches/{name}     →  hash of branch version entity
system/revision/{H}/tags/{name}         →  hash of tag version entity
system/revision/{H}/conflicts/{path}    →  conflict entity hash
system/revision/{H}/remotes/{peer_id}   →  hash of their last known version
system/revision/{H}/config              →  revision config entity
```

Where `{H}` is `prefix_hash(prefix)` throughout.

All version entities referenced by the DAG must be in the content store for traversal.

**Branches.** A branch is a named version pointer. The working head (`system/revision/{H}/head`) tracks whatever branch is currently active. Branch pointers track the tip of each branch's version history.

```
# prefix: "/{peerA}/project/"  →  H = prefix_hash("/{peerA}/project/")
system/revision/{H}/head                       → V7 (current working version)
system/revision/{H}/branches/main              → V7 (main branch, currently active)
system/revision/{H}/branches/feature-login     → V4_x (feature branch)
```

Committing advances the current head AND the active branch pointer. Checkout switches the working head to a different branch's version (applying its trie content to the tree).

**Subscribing to all revision state.** A single subscription on `system/revision/{H}/**` tracks all state changes for a prefix — head advances, branch creation/deletion, tag creation, conflict state, remote updates.

### 3.1.1 Path Inventory

All `system/revision/` tree paths used by this extension. `{H}` denotes `prefix_hash(prefix)` — the 66-character hex-encoded ECF content hash of the absolute prefix path (see §3.1).

**Per-prefix metadata** (under `system/revision/{H}/`):

| Path | Value | Defined in |
|------|-------|------------|
| `system/revision/{H}/head` | version hash (current head) | §3.1 |
| `system/revision/{H}/active-branch` | branch name string | §3.1 |
| `system/revision/{H}/branches/{name}` | version hash | §3.1, §4.4.13 |
| `system/revision/{H}/tags/{name}` | version hash | §4.4.14 |
| `system/revision/{H}/conflicts/{path}` | conflict entity hash | §2.2, §4.4.4 |
| `system/revision/{H}/remotes/{peer_id}` | version hash | §3.1, §4.4.8 |
| `system/revision/{H}/config` | revision config entity (§2.4) | §2.4, §4.4.17 |

**Global configuration** (not prefix-scoped):

| Path | Value | Defined in |
|------|-------|------------|
| `system/revision/config/merge/path/{name}` | merge-config entity (§2.3) | §2.3, §5.1 |
| `system/revision/config/merge/type/{type_name}` | merge-config entity (§2.3) | §2.3, §5.1 |

**Version entities** (`system/revision/entry`) are stored in the content store, not at tree paths. They are referenced by hash from head, branch, tag, and remote pointers listed above.

**Namespace reservation.** The `system/revision/config/` global namespace cannot collide with hash-addressed prefix subtrees: hash segments are 66-character hex strings (`[0-9a-f]+`), while `config` contains non-hex characters and is 6 characters long. No validation rule is needed — the reservation holds by construction. This extends to any future global namespace added under `system/revision/`.

### 3.2 DAG Traversal

Walking the version DAG from a head. The algorithm uses BFS (breadth-first search) for consistent presentation ordering (closer versions first), but implementations MAY use any DAG traversal that visits each version at most once and respects the `limit` parameter. The traversal order does not affect the version data — only the order in which versions appear in the result.

```
walk_history(version_hash, limit):
  result = []
  queue = [version_hash]     ; BFS shown as reference traversal
  visited = {}

  while queue is not empty and len(result) < limit:
    current_hash = queue.pop_front()
    if current_hash in visited: continue
    visited.add(current_hash)

    version = content_store.get(current_hash)
    if version is null: break
    result.append(version)

    for parent_hash in version.data.parents:
      if parent_hash not in visited:
        queue.append(parent_hash)

  return result
```

### 3.3 Common Ancestor Finding

Finding the lowest common ancestor (LCA) of two versions — essential for merge:

```
find_common_ancestor(version_a, version_b):
  ancestors_a = {}     ; hash → depth
  ancestors_b = {}

  queue_a = [(version_a, 0)]
  queue_b = [(version_b, 0)]

  while queue_a or queue_b:
    ; Expand A
    if queue_a:
      (current, depth) = queue_a.pop_front()
      if current in ancestors_b:
        return current    ; found LCA
      if current not in ancestors_a:
        ancestors_a[current] = depth
        version = content_store.get(current)
        if version is not null:
          for parent in version.data.parents:
            queue_a.append((parent, depth + 1))

    ; Expand B
    if queue_b:
      (current, depth) = queue_b.pop_front()
      if current in ancestors_a:
        return current    ; found LCA
      if current not in ancestors_b:
        ancestors_b[current] = depth
        version = content_store.get(current)
        if version is not null:
          for parent in version.data.parents:
            queue_b.append((parent, depth + 1))

  return null    ; no common ancestor (independent histories)
```

**Non-uniqueness.** The bilateral BFS finds *a* lowest common ancestor, not necessarily *the* LCA. When the version DAG contains multiple common ancestors at equal depth, different implementations (or different execution orderings of the bilateral expansion) may find different ones. This does not affect correctness:

- Any common ancestor produces a valid three-way merge. The merge result may differ with different ancestors, but the resulting version entry records the specific merge (trie root + parents), and the DAG converges on subsequent merges.
- Deterministic ancestor selection is not required for convergence. The version DAG records outcomes (trie roots), not intermediate choices (which ancestor was used).
- Implementations that require deterministic ancestor selection for testing or reproducibility MAY break BFS ties by preferring the ancestor with the smaller hash value.

### 3.4 Divergence Detection

Two versions have diverged when neither is an ancestor of the other:

```
check_relationship(local_head, remote_head):
  if local_head == remote_head:
    return "in_sync"          ; includes both-null case (neither has versions)

  if remote_head is null:
    return "ahead"            ; remote has no versions, local does — we are ahead

  if local_head is null:
    return "behind"           ; local has no versions, remote does — we are behind

  if is_ancestor(local_head, remote_head):
    return "behind"           ; local is behind remote, fast-forward

  if is_ancestor(remote_head, local_head):
    return "ahead"            ; local is ahead of remote

  return "diverged"           ; both have changes since common ancestor
```

```
is_ancestor(potential_ancestor, descendant):
  ; Walk descendant's parent chain, check if potential_ancestor appears
  visited = {}
  queue = [descendant]
  while queue:
    current = queue.pop_front()
    if current == potential_ancestor: return true
    if current in visited: continue
    visited.add(current)
    version = content_store.get(current)
    if version is not null:
      queue.extend(version.data.parents)
  return false
```

---

## 4. Handler

**Result path convention.** All path fields in revision result types and conflict entities are trie-relative — they are paths within the version trie, not absolute tree paths. The caller's `prefix` parameter provides the absolute context. This ensures result paths are meaningful across peers regardless of prefix mapping (two peers versioning the same content at different absolute prefixes share the same trie-relative paths).

### 4.1 Handler Manifest

```
system/handler := {
  data: {
    pattern:    "system/revision"
    name:       "version"
    operations: {
      commit:          {input_type: "system/revision/commit-params",          output_type: "system/revision/commit-result"}
      log:             {input_type: "system/revision/log-params",             output_type: "system/revision/log-result"}
      status:          {input_type: "system/revision/status-params",          output_type: "system/revision/status"}
      merge:           {input_type: "system/revision/merge-params",           output_type: "system/revision/merge-result"}
      resolve:         {input_type: "system/revision/resolve-params",         output_type: "system/revision/resolve-result"}
      fetch:           {input_type: "system/revision/fetch-params",           output_type: "system/revision/fetch-result"}
      fetch-entities:  {input_type: "system/revision/fetch-entities-params",  output_type: "system/revision/fetch-entities-result"}
      push:            {input_type: "system/revision/push-params",            output_type: "system/revision/push-result"}
      pull:            {input_type: "system/revision/fetch-params",           output_type: "system/revision/merge-result"}
      find-ancestor:   {input_type: "system/revision/ancestor-params",        output_type: "system/revision/ancestor-result"}
      branch:          {input_type: "system/revision/branch-params",          output_type: "system/revision/branch-result"}
      checkout:        {input_type: "system/revision/checkout-params",        output_type: "system/revision/checkout-result"}
      tag:             {input_type: "system/revision/tag-params",             output_type: "system/revision/tag-result"}
      diff:            {input_type: "system/revision/diff-params",            output_type: "system/tree/diff"}
      fetch-diff:      {input_type: "system/revision/fetch-diff-params",      output_type: "system/envelope"}
      cherry-pick:     {input_type: "system/revision/cherry-pick-params",     output_type: "system/revision/cherry-pick-result"}
      revert:          {input_type: "system/revision/revert-params",          output_type: "system/revision/revert-result"}
      config:          {input_type: "system/revision/config-params",          output_type: "system/revision/config-result"}
      merge-config:    {input_type: "system/revision/merge-config-params",    output_type: "system/revision/merge-config-result"}
    }
  }
}
```

Manifest at pattern path `system/revision`. Index entry at `system/handler/system/revision`.

### 4.2 Core and Convenience Operations

Operations are classified as **core** (structural mechanism, MUST implement) or **convenience** (composable from core + tree operations, SHOULD implement).

**Core operations** provide the structural mechanism. They cannot be composed from simpler primitives. Implementations MUST support all core operations.

| Operation | Purpose |
|-----------|---------|
| `commit` | Build trie from current tree state, create version entry, update HEAD |
| `merge` | Three-way merge with type-aware conflict dispatch |
| `resolve` | Put resolved entity at path, remove conflict entry |
| `fetch` | Get version DAG (entries + root trie nodes) from remote |
| `fetch-entities` | Get entities by hash, validated against version DAG |
| `log` | Walk DAG from HEAD (read-only) |
| `status` | Current HEAD, conflict count, pending changes (read-only) |
| `config` | Validate and write prefix config with tracking-config coordination |

**Convenience operations** compose from core operations + tree operations. Implementations SHOULD support them but MAY expose them through alternative interfaces (CLI, domain handler, library).

| Operation | Decomposition |
|-----------|---------------|
| `push` | Remote status + local trie diff + send envelope + remote merge |
| `branch` | `tree put` / `tree get` on `system/revision/{H}/branches/` paths |
| `tag` | `tree put` / `tree get` on `system/revision/{H}/tags/` paths |
| `checkout` | Get trie root from version entry, batch `tree put` for each binding |
| `diff` | Delegates to tree extension trie diff with two version trie roots |
| `fetch-diff` | Current head + caller's base version → trie roots → tree `compute_trie_diff` + closure bundle (returns `system/envelope`) |
| `cherry-pick` | Diff version against parent + apply delta + commit |
| `revert` | Inverse diff + apply + commit |
| `pull` | `fetch` + incremental `fetch-entities` (trie walk, multiple rounds) + `merge` |
| `find-ancestor` | Bilateral BFS on DAG (pure algorithm, no tree modification) |

**Note on `find-ancestor`:** The `merge` core operation MUST determine the common ancestor version entry to identify the base trie root for three-way merge. The ancestor-finding algorithm (bilateral BFS on the version DAG) is therefore required internally by any implementation that supports `merge`. Classifying `find-ancestor` as convenience means the *handler operation* (exposing it to external callers) is optional — the algorithm itself is not. Implementations that support `merge` inherently implement the ancestor-finding algorithm; the convenience operation simply exposes it.

**Implementation sequencing.** Implementations MAY implement core operations first and add convenience operations later. The recommended sequence:

1. **Phase 1 (core versioning):** config, commit, log, status, merge, resolve. Local version control.
2. **Phase 2 (core transfer):** fetch, fetch-entities. Cross-peer sync.
3. **Phase 3 (convenience):** push, branch, tag, checkout, diff, cherry-pick, revert, pull, find-ancestor.

Phase 1 is usable without Phase 2 (local-only versioning). Phase 2 requires Phase 1. Phase 3 requires Phase 1 and benefits from Phase 2 but is independently implementable.

### 4.3 Capability Model

Version operations require capability for the `system/revision` handler plus capability for the prefixes being operated on. For operations that modify the tree (merge, resolve, pull), the caller needs `put` permission on the affected paths.

For cross-peer operations (fetch, pull, push), the caller also needs capability for the remote peer connection.

**Config path ownership.** The revision handler's grant includes `put` access to `system/revision/**`. External callers SHOULD NOT hold direct `put` grants for `system/revision/*/config` — config writes route through the `revision/config` operation (§4.4.17), which validates before writing. If an implementation's capability model cannot restrict direct tree.put to handler-owned paths, the §6.1 emit-consumer check serves as defense-in-depth.

### 4.4 Operations

#### 4.4.1 commit

Create a version from the current tree state at a prefix.

```
EXECUTE system/revision  operation: "commit"
  resource: {targets: ["system/revision"]}
  params: {
    type: "system/revision/commit-params"
    data: {
      prefix:  "project/"
      message: "Add login feature"
    }
  }
```

```
system/revision/commit-params := {
  fields: {
    prefix:   {type_ref: "system/tree/path"}
    message:  {type_ref: "primitive/string", optional: true}
               ; Optional commit message. Not stored in the version entry.
               ; The handler MAY store this as a metadata entity in the tree
               ; (application convention, see §1.2).
  }
}
```

**Algorithm:**

```
handle_commit(ctx, params):
  prefix = resolve(params.data.prefix, local_peer_id)
  prefix_hash = prefix_hash(prefix)

  ; Build trie from current tree state at prefix.
  ; Apply exclude filters from version configuration if present.
  version_config = find_version_config(prefix)
  if version_config is not null:
    trie_root_hash = build_trie(compute_versioned_bindings(tree, prefix, version_config))
  else:
    trie_root_hash = build_trie(compute_bindings(tree, prefix))

  ; Get current head (parent of new version)
  current_head = tree.get("system/revision/" + prefix_hash + "/head")
  parents = []
  if current_head is not null:
    parents = [current_head]

  ; Create version entity (structural — root + sorted parents only)
  version = {
    type: "system/revision/entry"
    data: {
      root:    trie_root_hash
      parents: sorted(parents)
    }
  }
  version_hash = content_store.put(version)

  ; Update head pointer
  tree.put("system/revision/" + prefix_hash + "/head", version_hash)

  ; Advance active branch pointer (if on a branch)
  active_branch = tree.get("system/revision/" + prefix_hash + "/active-branch")
  if active_branch is not null:
    tree.put("system/revision/" + prefix_hash + "/branches/" + active_branch, version_hash)

  return {type: "system/revision/commit-result", data: {version: version_hash, root: trie_root_hash}}
```

#### 4.4.2 log

Walk version history for a prefix.

```
EXECUTE system/revision  operation: "log"
  params: {
    type: "system/revision/log-params"
    data: {
      prefix: "project/"
      limit:  20
    }
  }
```

```
system/revision/log-params := {
  fields: {
    prefix: {type_ref: "system/tree/path"}
    limit:  {type_ref: "primitive/uint", optional: true}    ; Default: 50
    since:  {type_ref: "system/hash", optional: true}       ; Start after this version
  }
}
```

Returns a `system/envelope` whose root is a `system/revision/log-result` containing version content hashes in reverse chronological order. The envelope's `included` map SHOULD contain the corresponding version entities, keyed by content hash.

#### 4.4.3 status

Current version state for a prefix.

```
EXECUTE system/revision  operation: "status"
  params: { type: "system/revision/status-params", data: { prefix: "project/" } }
```

```
system/revision/status-params := {
  fields: {
    prefix: {type_ref: "system/tree/path"}
  }
}
```

Returns `system/revision/status` with head, remote heads, conflict count, and pending change count.

#### 4.4.4 merge

Merge a remote version into the current tree state.

**Deletion-marker semantics (v3.1, Amendment 3 — applies to this operation, to fast-forward, and to all version-transcription operations enumerated below).**

*Three-way merge classification.* The merge classifies each path's (ancestor, local, remote) hash triple. Deletion-marker hashes (i.e., `CANONICAL_DELETION_MARKER_HASH` per ENTITY-NATIVE-TYPE-SYSTEM.md §4.9) are treated as ordinary entity hashes in this classification. The case "path present in ancestor's trie, absent from local's or remote's trie" MUST classify the absent side as NOT having an opinion — the path is preserved from the side that has it. **Deletion requires the deleting side to bind the path to the canonical deletion marker in its trie; mere absence is not deletion.**

*Both-absent fallback.* If a path is present in the ancestor's trie but absent from BOTH local's and remote's tries, the path MUST be treated as preserved-unbound — equivalently, neither side had an opinion, so the merge does not assert anything about the path. Under post-v3.1 operation this case cannot arise, because §6.1 (Amendment 2) ensures every in-scope path in a version trie has an explicit entry (real binding or canonical marker). The fallback is normative because the case CAN arise in cross-version merges with pre-v3.1 versions (which expressed deletes via absence) and with non-conforming implementations.

Because deletion markers are canonical (same hash on every peer), **deletion-vs-deletion is NOT a divergent case** — both sides produce the same hash, classified as "same on both sides," not as "both changed differently."

*Deletion-marker translation at apply.* Version-transcription operations apply merged bindings to the live tree. When the bound entity at a path is `system/deletion-marker`, the operation MUST translate the binding to a live-tree unbind (`tree:put(path, null)`), not a live-tree set. **This preserves the invariant that deletion markers never appear in the live location index.**

*Version-transcription operations in scope (the enumeration is normative; implementations adding new operations MUST extend it when amending the spec):* `merge` (three-way and fast-forward — this operation), `checkout` (§4.4.12), `revision:cherry-pick` (§4.4.15), `revision:revert` (§4.4.16). Implementations that fold push semantics into `merge` (e.g., via a `SourceEnvelope` parameter rather than a standalone `push` handler) satisfy this requirement transitively — the apply-translation invariant applies wherever a version's bindings get transcribed to the live tree, regardless of which operation initiates it.

**Explicit non-site (v3.3, P1):** `revision:push` (§4.4.9) is **not** a version-transcription site. Push transfers version metadata + content-store entries and updates the remote-head pointer on the recipient; it does NOT transcribe a version's bindings into the recipient's live tree. Recipient-side transcription happens later when the recipient calls `merge` / fast-forward / `checkout` to integrate the pushed version (one of the enumerated sites above, which then carries this deletion-marker apply-translation requirement). This non-site declaration was originally implied by listing "`revision:push` recipient-side apply" in this enumeration; cross-impl absorption (Go's `handlePush` only binds `remotePath = localHead`) confirmed push has no transcription step. Pattern parallels ENTITY-CORE-PROTOCOL.md §5.8 v7.46's explicit non-site treatment of EXTENSION-COMPUTE reactive re-eval in the cross-peer chain-construction registry.

*Deletion-marker identification.* Determining whether a hash refers to a deletion marker SHOULD use direct hash equality against `CANONICAL_DELETION_MARKER_HASH` — O(1), no I/O. **`CANONICAL_DELETION_MARKER_HASH` is format-relative (V7 v7.70):** it is `content_hash(deletion_marker_entity)` under the **trie's own `content_hash_format`** (ENTITY-NATIVE-TYPE-SYSTEM.md §4.9); `ecf-sha256:689ae4…` is its instance in the SHA-256 address space. Within a single-format network — every trie in one home format, the recommended deployment — the O(1) equality and the "deletion markers are canonical / same hash on every peer" property below hold exactly as before; an implementation computes the marker hash once under its home format and compares. When classifying a **foreign-format** trie (the experimental cross-format case, ENTITY-CORE-PROTOCOL.md §1.2a / §1.5), markers differ like all content and identification MUST fall back to entity-layer recognition (load entity from content store, check `entity.type == "system/deletion-marker"`). The same entity-layer fallback is the path for any implementation supporting non-canonical markers, which this spec version does not introduce.

*Conflict resolution.* When the classification surfaces a deletion-vs-entity divergence (exactly one of local/remote is the canonical marker), the resolution strategy is governed by `merge-config.deletion_resolution` per §2.3 (Amendment 4); default `preserve-on-conflict`.

**Version-transcription invariants (v3.2, A.3 — normative; applies to this operation and to every operation in the enumerated writer list below).**

Three invariants govern any operation that transcribes a version's bindings into the local tree. They land together because F10's six-round investigation surfaced them as one recurring code-pattern (asserting an invariant without enumerating writers/inferrers); each clause's writer enumeration is **load-bearing** — F10 repeatedly bit reviewers who verified the invariant against one writer while a different writer was the actual culprit. This is the same registry pattern ENTITY-CORE-PROTOCOL.md §5.8 v7.46 set for cross-peer chain-construction sites.

*(1) Version-transcription invariant.* Operations transcribing a version's bindings into the local tree (`merge` in any form — this operation; `checkout` §4.4.12; fast-forward — the `relationship == "behind"` branch of this operation; `revision:cherry-pick` §4.4.15; `revision:revert` §4.4.16; **future analogs**) MUST NOT remove paths from the live tree based on the live tree's contents alone.

**Explicit non-site (v3.3, P1):** `revision:push` (§4.4.9) is **not** in this enumeration. Push transfers version metadata + content-store entries and updates the remote-head pointer on the recipient; it does NOT transcribe a version's bindings into the recipient's live tree (the recipient transcribes later via `merge` / fast-forward / `checkout`, which are enumerated above and therefore carry this invariant). Push's apparent listing in earlier drafts was an over-broad enumeration corrected here in v3.3 after cross-impl absorption (Go's `handlePush` only binds `remotePath = localHead`). Same non-site treatment pattern ENTITY-CORE-PROTOCOL.md §5.8 v7.46 applied to EXTENSION-COMPUTE reactive re-eval in the cross-peer chain-construction registry. The set of paths an operation may remove is **exactly** those:

- present in the **source version's trie** (the operation's "from" version — for FF, the local head's version; for checkout, the committed local head's version; for three-way merge, the ancestor version), **AND**
- absent from the **target version's trie** (the operation's "to" version), **AND**
- for which a **deletion marker** (per v3.1 §4.4.4 Amendment 3 + ENTITY-NATIVE-TYPE-SYSTEM.md §4.9) exists in the target version's history (equivalently: the target side bound the canonical deletion marker in its trie, not merely "the path is missing").

Live-tree paths not present in **either** version (in-flight writes pending auto-version capture, untracked paths, prior application state) MUST be preserved through the operation. The live tree is **not** a valid diff baseline for any of these operations.

*(2) Head-write atomicity.* All writers to the head pointer `system/revision/{prefix-hash}/head` MUST coordinate via either compare-and-swap against the prior head value, OR a mutex shared by ALL writers. **Writers in scope (normative enumeration):** auto-version emits (§6.1); `merge` three-way and fast-forward (this operation); `checkout` (§4.4.12); `commit` (§4.4.1); `cherry-pick` (§4.4.15); `revert` (§4.4.16); `revision:push` recipient-side integrate (§4.4.9); future version-transcription operations.

Implementations using CAS MUST handle the conflict case by returning a **retry status** BEFORE applying any binding changes to the local tree. Applying bindings under a stale head produces orphaned writes (the bindings land in the live tree but no version captures them descending from the head the operation thought it advanced); the orphan then looks like an in-flight write to the next transcription operation, which under invariant (1) is now preserved — but only if invariant (1) is honored. Stale-head application is the failure mode invariant (2) exists to prevent.

*(3) Oscillation detection.* When evaluating whether a proposed merge result would re-create an existing version (the standard "did this merge converge to a prior state?" check), implementations MUST compare the candidate's **full identity** — `{root, sorted_parents}` — against recent ancestors, **not just the root hash**.

Same root with different parents is a **legitimate cross-link version**: the standard CRDT case where two peers reach the same content state via different lineage (parent histories) and need a cross-link merge to converge their DAGs. Treating it as oscillation aborts the merge and leaves heads stuck at divergent terminals — convergence becomes impossible. Implementations encountering same-root-different-parents MUST proceed with the merge as a structural cross-link, producing a new version whose `parents` is the sorted union of both sides' parents (per v2.1 structural-entries semantics).

**Spec house-style commitment (ratified with v3.1 / A.8; extended here).** Any spec amendment introducing a new operation that writes to revision-tracked prefix state MUST extend the writer enumeration in invariants (1) and (2). Implementations adding extension-specific write operations to revision-tracked prefixes MUST surface them for the next cross-impl audit. Reviewers reviewing such amendments MUST verify the enumeration extension is present. This commitment closes the F10 recurring race-class pattern at the spec-process level, parallel to ENTITY-CORE-PROTOCOL.md §5.8 v7.46's cross-peer chain-construction registry.

```
EXECUTE system/revision  operation: "merge"
  params: {
    type: "system/revision/merge-params"
    data: {
      prefix:         "project/"
      remote_version: <hash of remote version to merge>
      strategy:       "three-way"     ; optional override
      dry_run:        false           ; optional: preview without applying
    }
  }
```

```
system/revision/merge-params := {
  fields: {
    prefix:         {type_ref: "system/tree/path"}
    remote_version: {type_ref: "system/hash"}
    strategy:       {type_ref: "primitive/string", optional: true}
    dry_run:        {type_ref: "primitive/bool", optional: true}
  }
}
```

**Algorithm:**

```
handle_merge(ctx, params):
  prefix = resolve(params.data.prefix, local_peer_id)
  prefix_hash = prefix_hash(prefix)
  remote_version_hash = params.data.remote_version

  local_head = tree.get("system/revision/" + prefix_hash + "/head")

  ; Check relationship
  relationship = check_relationship(local_head, remote_version_hash)

  if relationship == "in_sync":
    return {status: "already_in_sync"}

  if relationship == "behind":
    ; Fast-forward: local is ancestor of remote.
    ;
    ; Diff baseline is the committed local head's trie root, NOT the live tree
    ; (per the Version-transcription invariants block below — v3.2 / A.3).
    ; In-flight live-tree writes pending auto-version capture MUST be
    ; preserved through the operation.
    remote_version = content_store.get(remote_version_hash)
    if not params.data.dry_run:
      ; Compute the local trie baseline.
      if local_head is null:
        ; First-ever sync: local trie is empty, only "set" operations apply.
        local_trie_root = empty_trie_root()
      else:
        local_version = content_store.get(local_head)
        local_trie_root = local_version.data.root

      ; Apply the diff between local and remote tries:
      ;   - set: paths added/changed in remote relative to local
      ;   - remove: paths present in local trie ∧ absent from remote trie
      ;             ∧ have a deletion marker in target version's history
      ;             (per v3.1 deletion-marker semantics — §6.6 retention,
      ;             §4.4.4 Amendment 3 apply translation)
      ; Paths present in the live tree but absent from BOTH local and remote
      ; tries are in-flight writes / untracked paths and MUST be preserved.
      apply_version_transcription(local_trie_root, remote_version.data.root, prefix)
      tree.put("system/revision/" + prefix_hash + "/head", remote_version_hash)
    return {status: "fast_forward", version: remote_version_hash}

  if relationship == "ahead":
    return {status: "already_ahead"}

  ; Diverged — need actual merge

  ; Content-identity check (defense-in-depth, SHOULD)
  ; With structural entries, same merge inputs produce the same version hash,
  ; so check_relationship returns in_sync before reaching merge.
  ; This check catches edge cases: corrupted content store, implementation bugs,
  ; or mixed-version peers where one side has non-structural entries.
  local_version = content_store.get(local_head)
  remote_version = content_store.get(remote_version_hash)
  if local_version.data.root == remote_version.data.root:
    winner = max(local_head, remote_version_hash)    ; deterministic tie-break
    update_head(prefix, winner)
    return {status: "converged_identical"}

  ; Normalize merge sides based on configured merge ordering
  version_config = find_version_config(prefix)
  merge_order = "deterministic"
  if version_config is not null and version_config.data.merge_order is not null:
    merge_order = version_config.data.merge_order
  (local_head, remote_version_hash) = normalize_merge_sides(local_head, remote_version_hash, merge_order)
  local_version = content_store.get(local_head)
  remote_version = content_store.get(remote_version_hash)

  ancestor_hash = find_common_ancestor(local_head, remote_version_hash)
  ancestor_version = content_store.get(ancestor_hash) if ancestor_hash else null

  ; Get trie root nodes
  local_root = content_store.get(local_version.data.root)
  remote_root = content_store.get(remote_version.data.root)
  ancestor_root = content_store.get(ancestor_version.data.root) if ancestor_version else null

  ; Merge path by path
  ;
  ; Merge tracks two outputs:
  ;   merged_bindings — paths with live entities (for the trie)
  ;   deletions       — paths to remove from the tree (not in the trie)
  ;
  ; A path absent from a trie means it was deleted (or never existed).
  ; Deleted paths are simply absent from the trie.
  ;
  ; Note: Implementations SHOULD use the trie-aware version merge algorithm
  ; (§5.5) for O(changes * depth) performance. This flat algorithm shows
  ; the logical merge; it is equivalent but O(total_paths).

  ; Walk tries to collect all bindings as path → hash maps
  local_bindings = bindings_map(collect_all_bindings(local_root, ""))
  remote_bindings = bindings_map(collect_all_bindings(remote_root, ""))
  ancestor_bindings = bindings_map(collect_all_bindings(ancestor_root, "")) if ancestor_root else {}

  merged_bindings = {}
  deletions = []
  conflicts = []

  all_paths = union(keys(local_bindings),
                    keys(remote_bindings),
                    keys(ancestor_bindings))

  for path in all_paths:
    local_hash = local_bindings.get(path)       ; null = absent (deleted or never existed)
    remote_hash = remote_bindings.get(path)     ; null = absent
    ancestor_hash = ancestor_bindings.get(path)

    if local_hash == remote_hash:
      ; Same on both sides — no conflict (includes both-null = both deleted)
      if local_hash is not null:
        merged_bindings[path] = local_hash
      else:
        deletions.append(path)

    elif local_hash == ancestor_hash:
      ; Only remote changed — take remote (may be a deletion)
      if remote_hash is not null:
        merged_bindings[path] = remote_hash
      else:
        deletions.append(path)

    elif remote_hash == ancestor_hash:
      ; Only local changed — keep local (may be a deletion)
      if local_hash is not null:
        merged_bindings[path] = local_hash
      else:
        deletions.append(path)

    else:
      ; Both changed differently — apply merge strategy
      ;
      ; This includes delete-vs-edit conflicts:
      ;   local_hash is null, remote_hash is not  → local deleted, remote edited
      ;   remote_hash is null, local_hash is not  → remote deleted, local edited
      ;   both non-null but different              → edit-vs-edit conflict
      ;
      ; The merge strategy receives null for deleted sides.
      ; Built-in strategies: delete-vs-edit always produces a conflict.
      ; Custom merge handlers decide based on domain semantics.

      strategy = find_merge_strategy(prefix + path, params.data.strategy)
      result = apply_merge_strategy(strategy, ancestor_hash, local_hash, remote_hash, path)

      if result is not null and result.resolved:
        if result.hash is not null:
          merged_bindings[path] = result.hash
        else:
          deletions.append(path)       ; strategy resolved to "delete"
        ; Handle strategies that produce additional bindings (e.g., keep-both)
        if result.additional_bindings is not null:
          for (additional_path, additional_hash) in result.additional_bindings:
            merged_bindings[additional_path] = additional_hash
      else:
        ; Unresolved conflict
        ; Keep the local state at the path: local entity if it exists,
        ; otherwise the path stays deleted (conflict still recorded)
        if local_hash is not null:
          merged_bindings[path] = local_hash

        ; Store conflict as side-channel metadata.
        ; If an existing conflict exists at this path (from a prior merge),
        ; the new conflict supersedes it — linking to the old via `supersedes`.
        existing_conflict = tree.get("system/revision/" + prefix_hash + "/conflicts/" + path)

        conflict = {
          type: "system/revision/conflict"
          data: {
            path: path, base: ancestor_hash,
            local: local_hash, remote: remote_hash,
            strategy: strategy.name,
            version_local: local_head, version_remote: remote_version_hash,
            supersedes: existing_conflict   ; null if no prior conflict
          }
        }
        conflict_hash = content_store.put(conflict)
        tree.put("system/revision/" + prefix_hash + "/conflicts/" + path, conflict_hash)
        conflicts.append(path)

  if params.data.dry_run:
    dry_status = "would_merge" if len(conflicts) == 0 else "would_conflict"
    return {status: dry_status, conflicts: conflicts,
            merged_count: len(merged_bindings), deleted_count: len(deletions)}

  ; Create merge version
  ; The trie contains only live bindings — deletions are absent
  ; Build trie from merged bindings (EXTENSION-TREE.md §3.3)
  trie_root_hash = build_trie(sorted(merged_bindings.items()))

  ; Oscillation detection (MUST) — check if proposed trie root has appeared
  ; in recent ancestry. If so, the merge is producing no forward progress.
  oscillation_depth = 4    ; default
  if version_config is not null and version_config.data.oscillation_depth is not null:
    oscillation_depth = max(2, version_config.data.oscillation_depth)

  if detect_oscillation(trie_root_hash, local_head, oscillation_depth):
    ; Oscillation detected — create conflict entities for all divergent paths
    ; instead of committing a cycling version.
    for path in all_paths:
      local_hash = local_bindings.get(path)
      remote_hash = remote_bindings.get(path)
      if local_hash != remote_hash:
        conflict = {
          type: "system/revision/conflict"
          data: {
            path: path, base: ancestor_bindings.get(path),
            local: local_hash, remote: remote_hash,
            strategy: "oscillation_detected",
            version_local: local_head, version_remote: remote_version_hash
          }
        }
        conflict_hash = content_store.put(conflict)
        tree.put("system/revision/" + prefix_hash + "/conflicts/" + path, conflict_hash)
        conflicts.append(path)
    return {
      status: "oscillation_detected",
      conflicts: conflicts
    }

  merge_version = {
    type: "system/revision/entry"
    data: {
      root:    trie_root_hash,
      parents: sorted([local_head, remote_version_hash])
    }
  }
  merge_version_hash = content_store.put(merge_version)

  ; IMPORTANT: Advance head to merge_version BEFORE applying bindings.
  ; Under auto-version (§6.1), per-write consumers fire on each binding application
  ; and create version entries chained from the current head. If bindings are
  ; applied first and head advanced last, the auto-version intermediates become
  ; orphans (no head points to them). Advancing head first ensures the intermediate
  ; chain descends from merge_version: merge_version → V_1 → ... → V_N.
  ; See §6.1 "Multi-parent versions" for the walk-through.
  tree.put("system/revision/" + prefix_hash + "/head", merge_version_hash)

  ; Advance active branch pointer. This was a gap in the pre-amendment merge op —
  ; cherry-pick and revert advance branches, merge didn't. Under auto-version OFF,
  ; this matters for correctness (branches/main was lagging head). Under auto-version
  ; ON, auto-version will further advance the branch through intermediates, but the
  ; initial advance to merge_version is still needed so the branch doesn't appear to
  ; skip the merge commit.
  active_branch = tree.get("system/revision/" + prefix_hash + "/active-branch")
  if active_branch is not null:
    tree.put("system/revision/" + prefix_hash + "/branches/" + active_branch, merge_version_hash)

  ; Apply merged state to tree. Under auto-version, each write creates an
  ; intermediate version chained from the current head (which is merge_version
  ; at the start of this loop, then the most recent intermediate).
  for (path, hash) in merged_bindings:
    tree.put(prefix + path, hash)
  for path in deletions:
    tree.put(prefix + path, null)

  return {
    status: "merged" if len(conflicts) == 0 else "merged_with_conflicts",
    version: merge_version_hash,
    conflicts: conflicts
  }

normalize_merge_sides(head_a, head_b, mode):
  if mode == "caller-perspective":
    return (local: head_a, remote: head_b)    ; head_a is the caller's head
  else:  ; "deterministic"
    if head_a <= head_b:
      return (local: head_a, remote: head_b)
    else:
      return (local: head_b, remote: head_a)

detect_oscillation(proposed_root, local_head, depth_limit):
  ; BFS through recent ancestors — checks all parents, not just first
  visited = {}
  queue = [local_head]
  depth = 0

  while queue and depth < depth_limit:
    next_queue = []
    for current in queue:
      if current is null or current in visited: continue
      visited.add(current)
      version = content_store.get(current)
      if version is null: continue
      if version.data.root == proposed_root:
        return true       ; oscillation — this trie root appeared recently
      next_queue.extend(version.data.parents)
    queue = next_queue
    depth += 1

  return false
```

**Traversal flexibility.** The oscillation detection requires a bounded walk of the version DAG to collect recent trie roots. BFS is shown as the reference traversal, but any traversal that examines versions within `depth_limit` steps of the starting version is acceptable. The essential property is the depth bound — the oscillation detection checks whether the proposed trie root appeared in recent history, not the order in which history is traversed.

**Merge side ordering.** The `normalize_merge_sides` function controls the assignment of "local" and "remote" sides in the merge. Two modes are defined:

- **Deterministic ordering** (default): `local` = the version with the lower hash value, `remote` = the higher. All strategies produce the same result regardless of which peer performs the merge. Required for full p2p convergence without an authoritative peer.
- **Caller perspective**: `local` = the peer's current head, `remote` = the incoming version. `source-wins` means "incoming wins," `target-wins` means "keep mine." Intuitive for human reasoning. Produces different results on different peers for asymmetric strategies.

| Mode | Symmetric strategies (field-level, LWW, CRDT) | Asymmetric strategies (source-wins, target-wins) |
|------|-----------------------------------------------|--------------------------------------------------|
| Deterministic | Convergent | Convergent in all topologies |
| Caller perspective | Convergent | Convergent only with authoritative peer |

**Guidance:** The default `deterministic` mode is appropriate for p2p meshes and auto-versioned prefixes. Deployments with an authoritative peer MAY use `caller-perspective` for git-style caller-centric history. Symmetric strategies converge in both modes — the choice only affects asymmetric strategies.

**Oscillation detection.** Before committing a merge version, implementations MUST check whether the proposed trie root has appeared in recent ancestry. If it has, the merge is producing no forward progress (oscillation). The merge MUST fall through to conflict entities instead of committing a cycling version.

When oscillation is detected, the merge creates conflict entities for all divergent paths and returns `{status: "oscillation_detected"}`. The conflict entities surface the disagreement for resolution (human or strategy change). The cycle stops.

**Precise cycling conditions.** All four must be true simultaneously: (1) bidirectional sync via subscription+continuation, (2) genuine conflict (same path changed by both peers), (3) asymmetric resolution strategy (source-wins or target-wins), (4) caller-perspective merge ordering (not deterministic). Remove any one condition and the cycle does not occur. Note that the default `merge_order` is `"deterministic"` (§2.4), so this concern only arises under explicit operator opt-in.

**Oscillation recovery.** When oscillation is detected, recovery follows the standard conflict resolution path:

1. Inspect the conflict entities at `system/revision/{H}/conflicts/{path}` (the `strategy` field reads `"oscillation_detected"`).
2. Resolve each conflict via `revision/resolve` with a chosen entity hash.
3. Commit to record the resolved state.

The oscillation stops because the resolved state is a new trie root that has not appeared in recent ancestry — the merge loop cannot reproduce it.

The recommended recovery is to switch to `deterministic` merge ordering — this eliminates the structural cause. If caller-perspective is operationally required, resolve the conflicts and consider raising `oscillation_depth` to catch longer cycles earlier.

Implementations MAY provide a convenience operation that detects oscillation-state conflicts (conflicts where `strategy` is `"oscillation_detected"`), resolves them using a specified fallback strategy (e.g., `"target-wins"` to stabilize on the local state), and commits. This is a compositional convenience, not a new primitive — it sequences resolve + commit.

**Transient head-vs-tree inconsistency during merge.** `merge_version.root` claims `trie_root_hash` (the target state) before the bindings have been applied. During the binding application loop, the tree is in intermediate partial-merge states.

Observers fall into two categories:

- **External RPC callers** observe only settled state (SYSTEM-COMPOSITION.md §1.3). They never see the partial merge states — by the time the merge op returns, all bindings have been applied. Cascade consumers may or may not have all completed: if any binding's cascade halted (207), the binding is committed but some consumers were skipped. The caller sees the merge result (including `cascade_warnings` if any halts occurred), not the intermediate partial states.
- **Emit-pipeline consumers firing within the cascade** — including auto-version itself — observe each intermediate state. Auto-version is designed to handle this (the intermediate states are what it captures as V_1..V_N). Subscription (at position 8 per SYSTEM-COMPOSITION.md §2.2) fires on each binding write after auto-version has produced the corresponding intermediate version. Subscribers on `system/revision/{H}/head` observe head advancing through V_merge → V_1 → … → V_N during the merge. Subscribers on `system/tree/root/{prefix}` observe the tracked root advancing through R_old → R_1 → … → R_N (where R_N = R_new). V_merge.root equals R_new as a claim, but the actual tracked root never equals V_merge.root until the final binding write lands — the tree state only reaches R_new after all N bindings have been applied. During the window between the head advance to V_merge (which claims root = R_new) and the final binding write, `head.root` diverges from `tracked_root`; after write N completes, they realign. This is observable and expected under per-write auto-version semantics.

Implementations with subscribers that require strict head-matches-tree consistency must either disable auto-version for the prefix during the merge (and re-enable after) or filter out the intermediate notifications at the subscriber layer.

**Partial application under mid-op failure.** When a binding write during merge returns a non-success status, merge distinguishes two cases:

- **Pre-write rejection** (4xx/5xx from tree.put): the binding did NOT land. The path was not applied. Merge terminates with `partial_applied` status and a list of unapplied paths (this path and all subsequent).
- **Cascade halt** (207 from tree.put): the binding DID land, but some cascade consumer (e.g., auto-version at position 7) returned non-200, halting the cascade. The path IS applied — the location index reflects the new binding. Merge continues applying remaining paths. The 207 detail is collected in `cascade_warnings`.

**`partial_applied` vs `cascade_warnings`.** These are orthogonal. `partial_applied` means some tree bindings were not written (4xx/5xx). `cascade_warnings` means some tree bindings were written but their cascade consumers halted (207). A merge result may have neither, either, or both.

The head pointer is NOT rolled back in either case. State after partial application:

- `system/revision/{H}/head` points at V_k (the last auto-version intermediate, or V_merge if no intermediates were created).
- The location index has bindings for writes 1..k but not k+1..N.
- `current_tracked_root(prefix)` computes to some R_k (partial merge state).
- Auto-version intermediates V_1..V_k are in the DAG, reachable via V_merge → V_1 → … → V_k.
- V_{k+1}..V_N were never created.

Recovery is application-level: the caller re-issues the remaining writes (auto-version continues the chain from V_k), or performs a revert/checkout to a known-good ancestor. The DAG remains internally consistent; no orphans are produced (V_1..V_k are reachable from head). The tree is simply at R_k rather than R_new.

Implementations MAY expose the `partial_applied` status in the operation result so callers can distinguish full success from partial application. The `conflicts` field in the merge result captures merge-strategy conflicts; `partial_applied` status is orthogonal and reflects infrastructure-level write failures.

**DAG size under independent merges across peers.** Content addressing makes V_merge reproducible: same parents (`[A_head, B_head]`) and same root (`R_new`) produce the same hash regardless of which peer performs the merge. V_1..V_N, however, are produced by the local merge handler's binding-application order. Implementations MAY order binding application deterministically (e.g., sorted by path) or non-deterministically; the former produces reproducible V_1..V_N hashes across peers, the latter does not.

When peer A and peer B each independently perform the same merge (e.g., both peers observed the same diverged state and both ran `revision/merge`), their V_1..V_N chains may differ. After sync, each peer holds:

- V_merge (shared — same hash on both peers).
- V_1_A..V_N_A (A's local intermediates).
- V_1_B..V_N_B (B's intermediates, fetched from A).

Both chains descend from V_merge. The peer's head remains on its own chain; the other peer's chain is visible via DAG traversal but not the active head lineage.

DAG size roughly doubles for any merge performed independently on multiple peers. This is not a correctness issue (convergence holds, no orphans), but implementations should size storage accordingly. Implementations that want deterministic DAG convergence across independent merges SHOULD sort binding application by path hash before applying — this produces matching V_1..V_N chains and avoids the size-asymmetry problem for merges that are otherwise independent.

#### 4.4.5 resolve

Resolve a conflict at a path.

```
EXECUTE system/revision  operation: "resolve"
  params: {
    type: "system/revision/resolve-params"
    data: {
      prefix:   "project/"
      path:     "config.toml"
      resolved: <hash of resolved entity>
    }
  }
```

```
system/revision/resolve-params := {
  fields: {
    prefix:   {type_ref: "system/tree/path"}
               ; Subtree prefix
    path:     {type_ref: "system/tree/path"}
               ; Relative path within prefix where the conflict exists
    resolved: {type_ref: "system/hash", optional: true}
               ; Content hash of the resolved entity to place at the path.
               ; Null means "delete the path" (resolve by removal — neither
               ; side's entity should exist at this path). Valid for
               ; create/delete conflicts where the user decides neither side wins.
  }
}
```

**Algorithm:**

```
handle_resolve(ctx, params):
  prefix = resolve(params.data.prefix, local_peer_id)
  prefix_hash = prefix_hash(prefix)
  path = params.data.path
  full_path = prefix + path
  conflict_path = "system/revision/" + prefix_hash + "/conflicts/" + relative_path

  ; Verify conflict exists
  conflict_hash = tree.get(conflict_path)
  if conflict_hash is null:
    return error(404, "no_conflict", "No conflict at " + full_path)

  ; Verify resolved entity exists in content store (when non-null)
  if params.data.resolved is not null:
    if not content_store.has(params.data.resolved):
      return error(404, "resolved_not_found",
        "Resolved entity hash not found in content store")

  ; Put resolved entity at the original path (null = unbind/delete)
  tree.put(full_path, params.data.resolved)

  ; Remove conflict entry
  tree.put(conflict_path, null)    ; remove binding

  ; Count remaining conflicts for this prefix
  remaining = len(tree.list("system/revision/" + prefix_hash + "/conflicts/"))

  return {
    type: "system/revision/resolve-result",
    data: {path: path, resolved: params.data.resolved, remaining_conflicts: remaining}
  }
```

The resolve operation:
1. Verifies a `system/revision/conflict` entry exists at the system conflict path
2. Verifies the resolved entity hash exists in the content store (when non-null)
3. Puts the resolved entity at the original path (replacing the local version), or unbinds the path (when resolved is null)
4. Removes the conflict entry from `system/revision/conflicts/`
5. Returns the remaining conflict count for the prefix
6. Does NOT create a new version — the caller creates a version when all conflicts are resolved (or whenever they choose)

**Post-resolution commit.** The resolve operation does not create a version. When `remaining_conflicts` reaches zero, the caller SHOULD commit to record the fully-resolved state. Under `auto_version: true`, the tree.put from resolve triggers auto-version — the resolved state is captured automatically. Under `auto_version: false`, the caller is responsible for committing.

#### 4.4.6 fetch

Get version DAG metadata from a remote peer without applying them. The revision handler mediates its own transfer — no content handler dependency.

```
EXECUTE system/revision  operation: "fetch"
  params: {
    type: "system/revision/fetch-params"
    data: {
      prefix:  "/{peerD}/"
      since:   V_common_hash         ; latest known version (null for full DAG)
      depth:   50                    ; max versions to return (optional)
    }
  }
```

```
system/revision/fetch-params := {
  fields: {
    prefix:        {type_ref: "system/tree/path"}
                    ; Local prefix — identifies the local revision config
                    ; and metadata subtree
    remote_prefix: {type_ref: "system/tree/path", optional: true}
                    ; Remote peer's prefix for the same content.
                    ; Default: same as `prefix` (both peers use the same prefix).
                    ; When set, the handler uses this prefix when querying
                    ; the remote peer's revision handler (status, fetch-entities).
    since:         {type_ref: "system/hash", optional: true}
             ; Latest known version hash. DAG walk stops here.
             ; Null for full DAG from HEAD.
    depth:  {type_ref: "primitive/uint", optional: true}
             ; Maximum number of versions to fetch from the remote head.
             ; Default: unlimited (fetch entire history). When set, the
             ; DAG walk stops after this many versions. The oldest fetched
             ; version becomes a shallow boundary — its parents may not
             ; be locally available. Subsequent operations (find-ancestor,
             ; merge) that walk past the boundary will see a truncated DAG.
  }
}
```

**Protocol:**

The remote revision handler:
1. Checks capability (does requester have version access for this prefix?).
2. Walks DAG from HEAD back to `since` (or root if null).
3. Collects version entries + the root trie node entity for each version (the node itself, not its children — child node hashes are visible in the root's entries but not resolved).
4. Returns a `system/envelope` result: root is a `system/revision/fetch-result`, included contains the version entries and root trie node entities.

```
Response:
  result: {
    type: "system/envelope"
    data: {
      root: {
        type: "system/revision/fetch-result"
        data: {
          head:     V_head_hash
          versions: [V5_hash, V4_hash, V3_hash]    ; DAG order
          has_more: false                           ; pagination
        }
      }
      included: {
        V5_hash: version_entry_entity
        V4_hash: version_entry_entity
        V3_hash: version_entry_entity
        root_V5_hash: trie_root_node               ; top-level entries + child hashes
        root_V4_hash: trie_root_node
        root_V3_hash: trie_root_node
      }
    }
  }
```

After fetch, the client has:
- The full version DAG structure (version entries with trie root + sorted parents).
- Root trie nodes (depth 1 only) — enough to see which top-level subtrees changed between versions.

The client does NOT yet have the full trie or the data entities. It can see which subtrees changed by comparing root trie node child hashes across versions. The client controls transfer granularity by walking deeper via `fetch-entities` — the server never preemptively sends deeper trie levels.

#### 4.4.7 fetch-entities

Get entities by hash, validated against the version DAG. The client identifies missing entities by walking the trie incrementally:

```
EXECUTE system/revision  operation: "fetch-entities"
  params: {
    type: "system/revision/fetch-entities-params"
    data: {
      prefix:   "/{peerD}/"
      snapshot: root_V5_hash                    ; trie root hash from version.data.root
      hashes:   [hash1, hash2, hash3, ...]     ; entity hashes to retrieve
    }
  }
```

```
system/revision/fetch-entities-params := {
  fields: {
    prefix:   {type_ref: "system/tree/path"}
               ; Subtree prefix (for capability check)
    snapshot: {type_ref: "system/hash"}
               ; Trie root hash (from version.data.root) these entities belong to
    hashes:   {array_of: {type_ref: "system/hash"}}
               ; Entity hashes to retrieve
  }
}
```

The remote revision handler:
1. Checks capability for this prefix.
2. Validates `snapshot` is a trie root referenced by a version in this prefix's DAG.
3. Validates each requested hash appears in the trie (at any depth).
4. Returns a `system/envelope` result: root is a `system/revision/fetch-entities-result`, included contains the retrieved entities.

```
Response:
  result: {
    type: "system/envelope"
    data: {
      root: {
        type: "system/revision/fetch-entities-result"
        data: {
          found:   [hash1, hash2]
          missing: [hash3]                ; not in content store (GC'd)
        }
      }
      included: {
        hash1: data_entity_1
        hash2: data_entity_2
      }
    }
  }
```

**Hash validation.** The revision handler MUST validate that each requested hash appears in the specified trie root's trie. This prevents the revision handler from becoming an open proxy to the content store. The `snapshot` parameter (trie root hash from `version.data.root`) scopes the validation — like the content extension's blob hash scoping chunk requests.

The `hashes` field can contain both trie node hashes (for walking deeper into the trie) and data entity hashes (leaf bindings). This allows the client to incrementally walk the trie:

1. Fetch gives root trie nodes.
2. Client compares root nodes, identifies changed subtrees.
3. fetch-entities with trie node hashes for changed subtrees — gets deeper trie nodes.
4. Client finds leaf bindings, identifies missing data entities.
5. fetch-entities with data entity hashes — gets actual content.

Steps 3-5 repeat until the client has everything it needs.

`fetch-entities` SHOULD accept up to 1,000 hashes per request. Clients MAY issue multiple `fetch-entities` requests for the same trie root.

#### 4.4.8 pull

Fetch + incremental fetch-entities (trie walk, multiple rounds) + merge in one operation.

```
handle_pull(ctx, params):
  fetch_result = handle_fetch(ctx, params)
  if fetch_result.status == "up_to_date":
    return fetch_result

  ; Incrementally fetch entities via trie walk
  ; (implementation walks trie nodes, identifies missing data entities,
  ;  issues fetch-entities requests until all needed entities are local)

  merge_result = handle_merge(ctx, {
    prefix: params.data.prefix,
    remote_version: fetch_result.remote_head,
    strategy: params.data.strategy
  })

  return {status: merge_result.status, version: merge_result.version, conflicts: merge_result.conflicts}
```

#### 4.4.9 push

Send local versions to a remote peer. Push uses the same trie diff mechanism as fetch — the sender computes the delta between its head and the remote's head, then transfers only the missing entities.

**Capability:** The caller needs local revision read access for the prefix (to read the DAG and trie) and remote revision write access (to send version entries and trigger integration). The remote peer validates the incoming version entries and entities against its own capability and DAG constraints.

```
EXECUTE system/revision  operation: "push"
  params: {
    type: "system/revision/push-params"
    data: {
      prefix:   "/{peerD}/"
    }
  }
```

**Protocol:**

1. Get remote's current head (status call — small, just the head hash).
2. Determine relationship (ahead/behind/diverged/in_sync).
3. If ahead: diff local head's trie against remote head's trie to identify missing entities.
4. Send missing version entries + changed trie nodes + changed data entities in one envelope.
5. Remote integrates (fast-forward or merge).

```
handle_push(ctx, params):
  prefix = resolve(params.data.prefix, local_peer_id)
  prefix_hash = prefix_hash(prefix)
  remote_pfx = params.data.remote_prefix or prefix
  remote_uri = params.data.remote

  local_head = tree.get("system/revision/" + prefix_hash + "/head")
  if local_head is null:
    return {status: "nothing_to_push"}

  ; Step 1: Get remote's current state
  remote_status = execute_remote(remote_uri, "system/revision", "status", {prefix: remote_pfx})
  remote_head = remote_status.data.head

  ; Step 2: Determine relationship
  relationship = check_relationship(local_head, remote_head)

  if relationship == "in_sync":
    return {status: "up_to_date"}

  if relationship == "behind":
    return {status: "behind", message: "Local is behind remote. Pull first."}

  if relationship == "diverged":
    return {status: "diverged", message: "Pull first to resolve divergence."}

  ; Step 3: relationship == "ahead" — compute delta
  ; Collect missing version entries (DAG walk from remote_head to local_head)
  missing_versions = versions_between(remote_head, local_head)

  ; Diff tries to find missing trie nodes and data entities
  local_version = content_store.get(local_head)
  local_root = content_store.get(local_version.data.root)

  if remote_head is not null:
    remote_version = content_store.get(remote_head)
    remote_root = content_store.get(remote_version.data.root)
  else:
    remote_root = null

  ; Collect entities the remote needs:
  ;   - All missing version entries
  ;   - Trie nodes that differ between remote's root and local's root
  ;   - Data entities at changed leaf bindings
  included = {}

  ; Version entries (small — always include all missing)
  for v_hash in missing_versions:
    v = content_store.get(v_hash)
    included[v_hash] = v

  ; Trie diff: walk both tries, collect nodes/entities that exist in local
  ; but not in remote. Same trie walk logic as diff (§4.3 of EXTENSION-TREE.md)
  ; but collecting entities instead of computing a change list.
  collect_missing_entities(local_root, remote_root, included)

  ; Step 4: Send envelope to remote
  execute_remote(remote_uri, "system/revision", "integrate", {
    type: "system/revision/push-params"
    data: {
      prefix: prefix,
      head: local_head,
      versions: missing_versions
    }
    included: included
  })

  ; Update remote tracking pointer
  tree.put("system/revision/" + prefix_hash + "/remotes/" + remote_peer_id, local_head)

  return {status: "pushed", versions: len(missing_versions)}

collect_missing_entities(local_node, remote_node, included):
  ; If both are real nodes with same hash, entire subtree is shared — skip
  if local_node is not null and remote_node is not null:
    if is_real(local_node) and is_real(remote_node):
      if hash(local_node) == hash(remote_node):
        return

  ; Include the local node itself (if real and not already included)
  if local_node is not null and is_real(local_node):
    node_hash = hash(local_node)
    if node_hash not in included:
      included[node_hash] = local_node

  ; Include data entity at this node's binding
  if local_node is not null and local_node.binding is not null:
    if local_node.binding not in included:
      entity = content_store.get(local_node.binding)
      if entity is not null:
        included[local_node.binding] = entity

  ; Recurse into children using the same decompose/match logic as trie diff
  if local_node is null:
    return

  local_entries = decompose_entries(local_node.entries)
  remote_entries = decompose_entries(remote_node.entries) if remote_node is not null else {}

  for seg in keys(local_entries):
    (local_rem, local_hash) = local_entries[seg]

    if seg not in remote_entries:
      ; Entire subtree is new — collect all nodes and entities
      collect_missing_entities(content_store.get(local_hash), null, included)
    else:
      (remote_rem, remote_hash) = remote_entries[seg]
      if local_rem == remote_rem:
        if local_hash != remote_hash:
          collect_missing_entities(
            content_store.get(local_hash),
            content_store.get(remote_hash),
            included)
      else:
        ; Compression mismatch — resolve at divergence point and recurse
        ; (same resolve_mismatch logic as diff, but collecting entities)
        collect_missing_entities(
          content_store.get(local_hash),
          content_store.get(remote_hash) if remote_hash else null,
          included)
```

**Remote integration.** The remote peer receives the envelope containing version entries, trie nodes, and data entities. The remote stores the entities in its content store, validates the version DAG chain, and integrates the new head — fast-forward if the remote's current head is an ancestor of the pushed head, or merge if the remote has diverged (concurrent push from another peer).

The remote does not need a separate `integrate` operation — the existing `merge` operation handles both fast-forward and three-way merge. The push envelope provides the entities; the merge operation provides the integration logic.

**Diverged state handling.** When the relationship between the local head and the remote head is `"diverged"` (neither is an ancestor of the other), the push operation MUST NOT send entities to the remote. It MUST return `{status: "diverged"}` and the caller MUST resolve the divergence locally (via pull/fetch + merge) before pushing. Implementations that silently proceed with a diverged push create DAG inconsistency at the remote — the remote may fast-forward to the pushed head, losing its own divergent work.

#### 4.4.10 find-ancestor

Find the common ancestor of two versions.

```
EXECUTE system/revision  operation: "find-ancestor"
  params: {
    type: "system/revision/ancestor-params"
    data: {
      version_a: <hash>,
      version_b: <hash>
    }
  }
```

```
system/revision/ancestor-params := {
  fields: {
    version_a: {type_ref: "system/hash"}
    version_b: {type_ref: "system/hash"}
  }
}
```

Returns the common ancestor version hash, or null if the versions have no common ancestor.

#### 4.4.11 branch

Create, list, or delete branches.

```
EXECUTE system/revision  operation: "branch"
  params: {
    type: "system/revision/branch-params"
    data: {
      prefix: "project/"
      action: "create"          ; "create", "list", "delete"
      name:   "feature-login"   ; branch name (for create/delete)
      from:   <version hash>    ; optional: create branch from this version (default: current head)
    }
  }
```

```
system/revision/branch-params := {
  fields: {
    prefix: {type_ref: "system/tree/path"}
    action: {type_ref: "primitive/string"}
    name:   {type_ref: "primitive/string", optional: true}
    from:   {type_ref: "system/hash", optional: true}
  }
}
```

**Create:**
```
handle_branch_create(ctx, params):
  prefix = resolve(params.data.prefix, local_peer_id)
  prefix_hash = prefix_hash(prefix)
  name = params.data.name
  from = params.data.from or tree.get("system/revision/" + prefix_hash + "/head")

  ; Store branch pointer
  tree.put("system/revision/" + prefix_hash + "/branches/" + name, from)

  return {status: "created", branch: name, version: from}
```

**List:**
```
handle_branch_list(ctx, params):
  prefix = resolve(params.data.prefix, local_peer_id)
  prefix_hash = prefix_hash(prefix)
  branches = tree.list("system/revision/" + prefix_hash + "/branches/")
  active = tree.get("system/revision/" + prefix_hash + "/active-branch")
  return {branches: branches, active: active}
```

**Delete:**
```
handle_branch_delete(ctx, params):
  prefix = resolve(params.data.prefix, local_peer_id)
  prefix_hash = prefix_hash(prefix)
  name = params.data.name

  ; Cannot delete the active branch
  active = tree.get("system/revision/" + prefix_hash + "/active-branch")
  if active == name:
    return error(400, "active_branch", "Cannot delete the active branch. Checkout a different branch first.")

  ; Verify branch exists
  if tree.get("system/revision/" + prefix_hash + "/branches/" + name) is null:
    return error(404, "branch_not_found")

  tree.put("system/revision/" + prefix_hash + "/branches/" + name, null)
  return {status: "deleted", branch: name}
```

#### 4.4.12 checkout

Switch the working tree to a different branch or version.

```
EXECUTE system/revision  operation: "checkout"
  params: {
    type: "system/revision/checkout-params"
    data: {
      prefix:  "project/"
      branch:  "feature-login"     ; checkout by branch name
      ; OR
      version: <version hash>      ; checkout by specific version
    }
  }
```

```
system/revision/checkout-params := {
  fields: {
    prefix:  {type_ref: "system/tree/path"}
    branch:  {type_ref: "primitive/string", optional: true}
    version: {type_ref: "system/hash", optional: true}
  }
}
```

**Algorithm:**

```
handle_checkout(ctx, params):
  prefix = resolve(params.data.prefix, local_peer_id)
  prefix_hash = prefix_hash(prefix)

  ; Resolve target version
  if params.data.branch is not null:
    branch_name = params.data.branch
    target_version_hash = tree.get("system/revision/" + prefix_hash + "/branches/" + branch_name)
    if target_version_hash is null:
      return error(404, "branch_not_found")
  elif params.data.version is not null:
    target_version_hash = params.data.version
    branch_name = null
  else:
    return error(400, "must specify branch or version")

  target_version = content_store.get(target_version_hash)

  ; Get trie root nodes
  target_root = content_store.get(target_version.data.root)

  ; Diff baseline (v3.2 / A.3 — Version-transcription invariant): the committed
  ; local head's trie root, NOT the live tree. Auto-version intermediates that
  ; descend from the committed head represent state captured between commits and
  ; are not relevant to the checkout diff. Live-tree paths not present in either
  ; the committed head's trie or target_version's trie are in-flight writes /
  ; untracked paths and MUST be preserved through the operation.
  local_head = tree.get("system/revision/" + prefix_hash + "/head")
  if local_head is null:
    ; No prior commits — empty baseline; only "set" operations apply.
    current_trie_root_hash = empty_trie_root()
    current_root = empty_trie_node()
  else:
    current_version = content_store.get(local_head)
    current_trie_root_hash = current_version.data.root
    current_root = content_store.get(current_trie_root_hash)

  ; Walk tries to collect bindings as path → hash maps
  current_bindings = bindings_map(collect_all_bindings(current_root, ""))
  target_bindings = bindings_map(collect_all_bindings(target_root, ""))

  ; Advance head and active-branch BEFORE applying bindings (§6.1 structural rule).
  ; Head moves to target_version; under auto-version ON, intermediate writes during
  ; the diff apply will create versions chained from target_version. Under auto-version
  ; OFF, the tree writes don't create versions and head remains at target_version.
  tree.put("system/revision/" + prefix_hash + "/head", target_version_hash)

  ; Update active-branch:
  ;   - Checkout by branch: set active-branch to the branch name.
  ;   - Checkout by version: clear active-branch to null (detached mode per §8.3).
  ; This closes a pre-existing gap where checking out a specific version left
  ; active-branch pointing at the previously-active branch.
  if branch_name is not null:
    tree.put("system/revision/" + prefix_hash + "/active-branch", branch_name)
  else:
    tree.put("system/revision/" + prefix_hash + "/active-branch", null)    ; detached

  ; Compute what needs to change and apply.
  ;
  ; Under auto-version ON: each write creates an intermediate version chained from
  ; the current head. Post-operation head is the final intermediate V_N whose root
  ; matches target_version.root. target_version is reachable via ancestor traversal.
  ; If checkout is by branch, auto-version also advances branches/{name} through the
  ; intermediates — the branch tip ends at V_N, not target_version. This is correct:
  ; the collaborative branch has evolved to match target_version's content.
  ;
  ; Under auto-version OFF: tree writes don't create versions. Head stays at
  ; target_version; if by branch, branches/{name} is also unchanged (target is
  ; already its tip).

  ; Paths in current but not in target → remove
  for (path, hash) in current_bindings:
    if path not in target_bindings:
      tree.put(prefix + path, null)

  ; Paths in target → set (creates or updates)
  for (path, hash) in target_bindings:
    current_hash = current_bindings.get(path)
    if current_hash != hash:
      tree.put(prefix + path, hash)

  ; Read final head — differs from target_version under auto-version ON.
  final_head = tree.get("system/revision/" + prefix_hash + "/head")

  return {
    status:         "checked_out",
    target_version: target_version_hash,    ; what was requested
    head:           final_head,             ; actual post-op head
    branch:         branch_name
  }
```

**What checkout does to the tree.** Checkout computes the diff between the current tree state and the target version's trie, then applies it:
- Paths that exist in the current tree but not the target → removed (binding deleted)
- Paths that exist in the target but not the current tree → created (binding set)
- Paths that differ → updated (binding changed)
- Paths that match → unchanged (no write, no history entry)

Each of these writes goes through the normal emit pathway — history records every change, subscriptions fire. The checkout is not "magical" — it's just a batch of puts and deletes on the tree, driven by the diff between current and target state.

**Auto-version OFF vs ON semantics.** Checkout has two distinct semantics depending on the auto-version mode. Both are internally correct; they reflect the different contracts of the two modes.

**Under auto-version OFF (manual commit mode),** checkout is a working-copy switch: head moves to `target_version`, tree bindings are adjusted to match, no new versions are created. This matches classical VCS checkout semantics (git, mercurial).

**Under auto-version ON (CRDT collaborative editing mode),** checkout is a state-restoration operation: the system applies changes to the shared tree state until it matches `target_version`'s content. Each binding change produces a new version entry (per §6.1). The post-operation head is a NEW version descending from `target_version` with matching root — representing "we collectively decided to restore our shared state to match this template." This is semantically equivalent to an auto-generated revert-of-everything-since, which is what "go back" means in a CRDT — there is no time travel, only forward evolution that happens to match a prior state.

Both semantics follow from the same underlying algorithm. The ordering fix (head advance before bindings) is required in both cases to satisfy the no-orphan invariant when auto-version is on.

**Result structure.** The result reports both `target_version` (the requested checkout target) and `head` (the actual post-operation head pointer). Under auto-version OFF they are equal. Under auto-version ON, `head` is a new descendant of `target_version` with matching root. Applications should use `head` to display current position and `target_version` to record what the user asked for.

**Cross-peer propagation under auto-version ON.** The intermediate versions V_1...V_N produced during checkout are regular DAG entries. They propagate via normal sync and are observed by subscribers like any other change. Remote peers receive them as additive edits evolving the shared state toward `target_version`'s content. No special cross-peer signaling is needed — checkout under auto-version is just a sequence of ordinary changes.

**When checkout-by-branch is used under auto-version ON:** the branch pointer advances through the intermediates to V_N. The branch tip ends at V_N (a new version), not at the original branch tip. This is correct: under collaborative CRDT semantics, "switching to feature and restoring its content" IS an evolution of the feature branch.

**Read-only alternatives to checkout under auto-version.** Operators who want to inspect an old state without mutating the tree SHOULD NOT use `checkout` under auto-version ON — that operation is always mutating. Use these read-only paths instead:

- **`revision/diff`** (§4.4.14) to compare the current tree against an older version. Returns the set of bindings that differ, without applying changes.
- **`revision/log`** (§4.4.2) to walk version history and inspect ancestor metadata.
- **`revision/fetch`** (§4.4.6) to download V_old's version entry and trie nodes into the local content store, then walk the trie manually to inspect bindings. The fetched trie root (type `system/tree/snapshot/node`) can be traversed recursively to materialize V_old's bindings without affecting the default tree.

Checkout is a state-mutating operation. Under auto-version ON, mutating operations produce versions — there is no read-only mode of checkout that escapes this. If an operator needs to "undo recent changes to match an old state," they are performing a state-restoration operation, which under CRDT semantics IS a set of forward edits (the generated intermediates) regardless of how the operation is named.

**`checkout_under_auto_version` policy.** Revision config MAY include an optional field `checkout_under_auto_version: "allow" | "warn" | "deny"` (default `"warn"`) to surface this distinction at operation time:

- `"allow"` — checkout proceeds silently under auto-version ON; operator is assumed to understand the semantics.
- `"warn"` (default) — checkout proceeds but the operation result includes an informational field indicating intermediate versions were created.
- `"deny"` — checkout is rejected when auto-version is ON for the prefix; the operator must use read-only alternatives (see above) or disable auto-version first.

Implementations that support policy enforcement MUST use the field name `checkout_under_auto_version` exactly. Implementations that don't support this policy MAY ignore the field — the absence of enforcement does not make the implementation non-conformant; it just means the implementation always behaves as if `"allow"` were set. Specifying the field name here prevents per-implementation divergence (`checkout_policy`, `auto_version_checkout_mode`, etc.) that would make operator config non-portable across peers.

**Partial application.** Partial-application semantics identical to merge (see §4.4.4). Pre-write rejection (4xx/5xx) triggers `partial_applied` with unapplied paths listed. Cascade halt (207) is treated as success — the binding landed; the operation continues. Cascade-halt details are surfaced via `cascade_warnings` in the result. Head is not rolled back in either case.

**Uncommitted changes.** Checkout applies the target version's trie regardless of whether the current state has been committed. If there are uncommitted changes (the tree differs from the current head's trie), those changes are overwritten. The caller should commit first to preserve them. The revision handler MAY warn about uncommitted changes (by comparing current tree state to the head's trie root) but MUST NOT refuse the checkout — the caller's intent is explicit.

#### 4.4.13 tag

Create, list, or delete tags. Tags are immutable named version pointers — once created, a tag cannot be moved to a different version (unlike branches, which advance on commit).

```
EXECUTE system/revision  operation: "tag"
  params: {
    type: "system/revision/tag-params"
    data: {
      prefix:  "project/"
      action:  "create"
      name:    "v1.0"
      version: <hash>
    }
  }
```

```
system/revision/tag-params := {
  fields: {
    prefix:  {type_ref: "system/tree/path"}
    action:  {type_ref: "primitive/string"}
              ; "create", "list", "delete"
    name:    {type_ref: "primitive/string", optional: true}
              ; Tag name (required for create/delete)
    version: {type_ref: "system/hash", optional: true}
              ; Version to tag (for create). Default: current head.
  }
}
```

Tags are stored at:

```
system/revision/{H}/tags/{tag_name}  →  version hash
```

**Create:**

```
handle_tag_create(ctx, params):
  prefix = resolve(params.data.prefix, local_peer_id)
  prefix_hash = prefix_hash(prefix)
  name = params.data.name
  version = params.data.version or tree.get("system/revision/" + prefix_hash + "/head")

  if version is null:
    return error(400, "no_version", "No version to tag")

  tag_path = "system/revision/" + prefix_hash + "/tags/" + name

  ; Tags are immutable — cannot overwrite
  if tree.get(tag_path) is not null:
    return error(409, "tag_exists", "Tag '" + name + "' already exists")

  tree.put(tag_path, version)
  return {status: "created", tag: name, version: version}
```

**List:**

```
handle_tag_list(ctx, params):
  prefix = resolve(params.data.prefix, local_peer_id)
  prefix_hash = prefix_hash(prefix)
  tags = tree.list("system/revision/" + prefix_hash + "/tags/")
  return {tags: tags}
```

**Delete:**

```
handle_tag_delete(ctx, params):
  prefix = resolve(params.data.prefix, local_peer_id)
  prefix_hash = prefix_hash(prefix)
  name = params.data.name

  if tree.get("system/revision/" + prefix_hash + "/tags/" + name) is null:
    return error(404, "tag_not_found")

  tree.put("system/revision/" + prefix_hash + "/tags/" + name, null)
  return {status: "deleted", tag: name}
```

#### 4.4.14 diff

Compare two versions' trie roots. Returns a tree diff showing what changed between two points in the version DAG.

```
EXECUTE system/revision  operation: "diff"
  params: {
    type: "system/revision/diff-params"
    data: {
      prefix: "project/"
      base:   <hash>
      target: <hash>
    }
  }
```

```
system/revision/diff-params := {
  fields: {
    prefix: {type_ref: "system/tree/path"}
    base:   {type_ref: "system/hash"}
    target: {type_ref: "system/hash"}
  }
}
```

**Algorithm:**

```
handle_diff(ctx, params):
  base_version = content_store.get(params.data.base)
  target_version = content_store.get(params.data.target)

  if base_version is null or target_version is null:
    return error(404, "version_not_found")

  base_root = content_store.get(base_version.data.root)
  target_root = content_store.get(target_version.data.root)

  return compute_trie_diff(base_root, target_root)    ; Returns system/tree/diff
```

Returns `system/tree/diff` — the same diff type used by the tree extension. The revision handler delegates to the tree's trie diff computation. This is a convenience operation: callers could retrieve both versions' trie roots and call tree diff directly, but the version-level operation provides discoverability.

#### 4.4.15 cherry-pick

Apply a specific version's changes to the current tree state. Computes what the version changed relative to its parent, then applies those changes as a three-way merge against the current state.

```
EXECUTE system/revision  operation: "cherry-pick"
  params: {
    type: "system/revision/cherry-pick-params"
    data: {
      prefix:  "project/"
      version: <hash of version to cherry-pick>
    }
  }
```

```
system/revision/cherry-pick-params := {
  fields: {
    prefix:  {type_ref: "system/tree/path"}
    version: {type_ref: "system/hash"}
              ; Version whose changes to apply
    parent:  {type_ref: "system/hash", optional: true}
              ; Parent version hash to use as the merge ancestor.
              ; MUST be one of version.data.parents.
              ; Default: version.data.parents[0] (first in sorted order).
              ; Required when cherry-picking a merge version (2+ parents).
  }
}
```

**Algorithm:**

Cherry-pick reuses the path-by-path merge logic from §4.4.4 with different trie root assignments:

| Merge role | Cherry-pick assignment |
|------------|----------------------|
| ancestor | Cherry-picked version's parent trie root |
| local | Current trie root (what we have now) |
| remote | Cherry-picked version's trie root (what to apply) |

This three-way merge applies "what the version changed relative to its parent" on top of the current state. Paths the version didn't touch are unaffected. Paths the version changed that haven't diverged locally are cleanly applied. Paths that conflict go through the merge strategy framework (§5).

```
handle_cherry_pick(ctx, params):
  prefix = resolve(params.data.prefix, local_peer_id)
  prefix_hash = prefix_hash(prefix)
  target = content_store.get(params.data.version)

  ; Validate
  if target is null: return error(404, "version_not_found")
  if len(target.data.parents) == 0:
    return error(400, "no_parent", "Cannot cherry-pick an initial version")

  if params.data.parent is not null:
    if params.data.parent not in target.data.parents:
      return error(400, "invalid_parent", "Specified parent is not in version's parent list")
    parent_hash = params.data.parent
  elif len(target.data.parents) > 1:
    return error(400, "ambiguous_parent",
      "Merge version has multiple parents — specify which parent to diff against")
  else:
    parent_hash = target.data.parents[0]

  parent = content_store.get(parent_hash)
  if parent is null: return error(404, "parent_not_found", "Parent version not available locally")

  local_head = tree.get("system/revision/" + prefix_hash + "/head")
  if local_head is null: return error(400, "no_head", "No versions exist for this prefix")
  local_version = content_store.get(local_head)

  ; Three-way merge: parent is ancestor, current is local, cherry-picked is remote
  ancestor_root = content_store.get(parent.data.root)
  local_root = content_store.get(local_version.data.root)
  remote_root = content_store.get(target.data.root)

  ; Execute the path-by-path merge algorithm (same logic as §4.4.4 merge)
  (merged_bindings, deletions, conflicts) =
    merge_tries(ancestor_root, local_root, remote_root,
                prefix, local_head, params.data.version)

  ; Build trie from computed merged_bindings
  trie_root_hash = build_trie(sorted(merged_bindings.items()))

  ; Create cherry-pick version entity. Parent is current head — NOT the cherry-picked
  ; version. Cherry-pick is a new commit on the current branch that carries the
  ; same changes as the source.
  version = {
    type: "system/revision/entry"
    data: {
      root:    trie_root_hash,
      parents: sorted([local_head])
    }
  }
  version_hash = content_store.put(version)

  ; Advance head BEFORE applying bindings (§6.1 structural rule).
  tree.put("system/revision/" + prefix_hash + "/head", version_hash)

  active_branch = tree.get("system/revision/" + prefix_hash + "/active-branch")
  if active_branch is not null:
    tree.put("system/revision/" + prefix_hash + "/branches/" + active_branch, version_hash)

  ; Apply merged state to tree
  for (path, hash) in merged_bindings:
    tree.put(prefix + path, hash)
  for path in deletions:
    tree.put(prefix + path, null)

  return {
    status: "cherry_picked" if len(conflicts) == 0 else "cherry_picked_with_conflicts",
    version: version_hash, source: params.data.version, conflicts: conflicts
  }
```

**Partial application.** Partial-application semantics identical to merge (see §4.4.4). Pre-write rejection (4xx/5xx) triggers `partial_applied` with unapplied paths listed. Cascade halt (207) is treated as success — the binding landed; the operation continues. Cascade-halt details are surfaced via `cascade_warnings` in the result. Head is not rolled back in either case.

**Merge versions.** When cherry-picking a merge version (2+ parents), the caller MUST specify which parent to use as the ancestor via the `parent` parameter. Unlike Git's `-m 1` default, the entity system does not assign semantic ordering to hash-sorted parents. Requiring explicit selection avoids silent semantic confusion.

#### 4.4.16 revert

Undo a specific version's changes. Computes the inverse of what the version changed relative to its parent, then applies that inverse as a three-way merge against the current state.

```
EXECUTE system/revision  operation: "revert"
  params: {
    type: "system/revision/revert-params"
    data: {
      prefix:  "project/"
      version: <hash of version to revert>
    }
  }
```

```
system/revision/revert-params := {
  fields: {
    prefix:  {type_ref: "system/tree/path"}
    version: {type_ref: "system/hash"}
              ; Version whose changes to undo
    parent:  {type_ref: "system/hash", optional: true}
              ; Parent version hash to use as the merge ancestor.
              ; MUST be one of version.data.parents.
              ; Default: version.data.parents[0] (first in sorted order).
              ; Required when reverting a merge version (2+ parents).
  }
}
```

**Algorithm:**

Revert is the inverse of cherry-pick. The trie root role assignments are swapped:

| Merge role | Revert assignment |
|------------|-------------------|
| ancestor | Version's trie root (what we're undoing) |
| local | Current trie root (what we have now) |
| remote | Version's parent trie root (what it was before) |

This three-way merge "un-applies" the version's changes. Paths the version changed that haven't diverged locally are cleanly reverted. Paths that have been further modified since the version conflict go through the merge strategy framework.

```
handle_revert(ctx, params):
  prefix = resolve(params.data.prefix, local_peer_id)
  prefix_hash = prefix_hash(prefix)
  target = content_store.get(params.data.version)

  ; Validate
  if target is null: return error(404, "version_not_found")
  if len(target.data.parents) == 0:
    return error(400, "no_parent", "Cannot revert an initial version")

  if params.data.parent is not null:
    if params.data.parent not in target.data.parents:
      return error(400, "invalid_parent", "Specified parent is not in version's parent list")
    parent_hash = params.data.parent
  elif len(target.data.parents) > 1:
    return error(400, "ambiguous_parent",
      "Merge version has multiple parents — specify which parent to diff against")
  else:
    parent_hash = target.data.parents[0]

  parent = content_store.get(parent_hash)
  if parent is null: return error(404, "parent_not_found", "Parent version not available locally")

  local_head = tree.get("system/revision/" + prefix_hash + "/head")
  if local_head is null: return error(400, "no_head")
  local_version = content_store.get(local_head)

  ; Inverse three-way merge: version is ancestor, current is local, parent is remote
  ancestor_root = content_store.get(target.data.root)
  local_root = content_store.get(local_version.data.root)
  remote_root = content_store.get(parent.data.root)

  ; Execute the path-by-path merge algorithm (same logic as §4.4.4 merge)
  (merged_bindings, deletions, conflicts) =
    merge_tries(ancestor_root, local_root, remote_root,
                prefix, local_head, params.data.version)

  ; Build trie from computed merged_bindings
  trie_root_hash = build_trie(sorted(merged_bindings.items()))

  ; Create revert version entity. Parent is current head, not the reverted version.
  version = {
    type: "system/revision/entry"
    data: {
      root:    trie_root_hash,
      parents: sorted([local_head])
    }
  }
  version_hash = content_store.put(version)

  ; Advance head BEFORE applying bindings (§6.1 structural rule).
  tree.put("system/revision/" + prefix_hash + "/head", version_hash)

  active_branch = tree.get("system/revision/" + prefix_hash + "/active-branch")
  if active_branch is not null:
    tree.put("system/revision/" + prefix_hash + "/branches/" + active_branch, version_hash)

  ; Apply merged state to tree
  for (path, hash) in merged_bindings:
    tree.put(prefix + path, hash)
  for path in deletions:
    tree.put(prefix + path, null)

  return {
    status: "reverted" if len(conflicts) == 0 else "reverted_with_conflicts",
    version: version_hash, reverted: params.data.version, conflicts: conflicts
  }
```

**Partial application.** Partial-application semantics identical to merge (see §4.4.4). Pre-write rejection (4xx/5xx) triggers `partial_applied` with unapplied paths listed. Cascade halt (207) is treated as success — the binding landed; the operation continues. Cascade-halt details are surfaced via `cascade_warnings` in the result. Head is not rolled back in either case.

**Merge versions.** When reverting a merge version (2+ parents), the caller MUST specify which parent to use as the ancestor via the `parent` parameter. Unlike Git's default of reversing against the first parent, the entity system does not assign semantic ordering to hash-sorted parents. Requiring explicit selection avoids silent semantic confusion. This mirrors the cherry-pick requirement (§4.4.15).

**Revert vs rollback.** Revert (revision extension) undoes a version's changes across all paths it touched, creating a new version in the DAG. Rollback (history extension) restores a single path to a previous state. Revert is version-level undo; rollback is path-level undo.

#### 4.4.17 config

Validate and write prefix configuration with tracking-config coordination.

```
EXECUTE system/revision  operation: "config"
  params: {
    type: "system/revision/config-params"
    data: {
      name:           "my-project"
      action:         "set"
      config: {
        type: "system/revision/config"
        data: {
          prefix:       "project/"
          exclude:      ["system/**"]
          auto_version: true
          merge_order:  "deterministic"
        }
      }
    }
  }
```

**Input/output types:**

```
system/revision/config-params := {
  fields: {
    name:           {type_ref: "primitive/string"}
                     ; Config name — used as a human-readable identifier for this config
    action:         {type_ref: "primitive/string"}
                     ; "set" | "delete"
    config:         {type_ref: "system/revision/config", optional: true}
                     ; Required when action is "set". Omit for "delete".
    expected_hash:  {type_ref: "system/hash", optional: true}
                     ; CAS guard on the existing config binding.
  }
}

system/revision/config-result := {
  fields: {
    config_path:              {type_ref: "system/tree/path"}
    config_hash:              {type_ref: "system/hash", optional: true}
    previous_hash:            {type_ref: "system/hash", optional: true}
    tracking_config_path:     {type_ref: "system/tree/path", optional: true}
    tracking_config_action:   {type_ref: "primitive/string", optional: true}
                               ; "created" | "updated" | "deleted" | null
  }
}
```

**Validation rules (set action):**

| ID | Rule | Error code | Status |
|----|------|------------|--------|
| V1 | `prefix` is a valid absolute path | `config/invalid-prefix` | 400 |
| V2 | Auto-version excludes include all §6.1 required patterns | `config/missing-required-exclude` | 400 |
| V3 | Auto-version excludes include `system/tree/root/**` when prefix encompasses it | `config/missing-trie-root-exclude` | 400 |
| V4 | `merge_order` is `"deterministic"` or `"caller-perspective"` | `config/invalid-merge-order` | 400 |
| V5 | `oscillation_depth` >= 2 | `config/oscillation-depth-below-minimum` | 400 |
| CAS | `expected_hash` matches current binding | `config/concurrent-modification` | 409 |

**Tracking-config coordination (set action):**

When enabling auto-version (`auto_version: true` and not previously enabled):
1. Write tracking-config FIRST — `ctx.entity_tree.put("system/tree/tracking-config/" + normalize_prefix_key(prefix), tracking_entity)`.
2. Write revision config — `ctx.entity_tree.put("system/revision/" + prefix_hash + "/config", config)`.

When disabling auto-version (`auto_version` false/absent and previously enabled):
1. Write revision config FIRST (disables auto-version).
2. Delete tracking-config.

Write ordering minimizes the inconsistent window: enable tracking FIRST so reads during the window find a populated tracked root; disable revision config FIRST so auto-version stops reading before tracking is removed.

**Delete action:** deletes the config at `system/revision/{H}/config`. If auto-version was enabled, cleans up the tracking-config.

**Nested cascade handling:** the operation issues up to three tree.put calls. Status 200 and 207 from sub-writes are treated as success (the binding landed). Status 4xx/5xx returns 500 with `config/tracking-config-write-failed` or `config/config-write-failed`.

#### 4.4.18 merge-config

Canonical write path for the merge-config namespace (`system/revision/config/merge/{path,type}/*`). Validates and writes per-path or per-type merge-config entries, enforcing the §2.3 strategy-rejection contract at config-write time.

```
EXECUTE system/revision  operation: "merge-config"
  params: {
    type: "system/revision/merge-config-params"
    data: {
      scope:          "path"                          ; "path" | "type"
      name:           "drafts/*"                      ; pattern (scope=path) or type name (scope=type)
      action:         "set"                           ; "set" | "delete"
      config: {                                       ; required when action=set
        type: "system/revision/merge-config"
        data: {
          pattern:               "drafts/*"
          strategy:              "three-way"
          deletion_resolution:   "preserve-on-conflict"   ; optional
        }
      }
      expected_hash:  <hash>                          ; optional CAS guard
    }
  }
```

```
system/revision/merge-config-params := {
  fields: {
    scope:          {type_ref: "primitive/string"}     ; "path" | "type"
    name:           {type_ref: "primitive/string"}     ; pattern or type name
    action:         {type_ref: "primitive/string"}     ; "set" | "delete"
    config:         {type_ref: "system/revision/merge-config", optional: true}
    expected_hash:  {type_ref: "system/hash", optional: true}
  }
}

system/revision/merge-config-result := {
  fields: {
    path:    {type_ref: "system/tree/path"}            ; binding path that was written or deleted
    hash:    {type_ref: "system/hash", optional: true} ; new entity hash (action=set); absent on delete
    status:  {type_ref: "primitive/string"}            ; "set" | "deleted" | "no_change"
  }
}
```

**Algorithm:**

```
handle_merge_config(ctx, params):
  ; Validate scope
  if params.data.scope not in ("path", "type"):
    return error(400, "invalid_scope")

  ; Compute the canonical write path for this (scope, name).
  ; Merge configs are GLOBAL (not prefix-scoped) per §3.1.1 + §5.1 —
  ; their `pattern` field matches trie-relative paths under any prefix,
  ; so the storage path itself is unscoped.
  if params.data.scope == "path":
    write_path = "system/revision/config/merge/path/" + params.data.name
  else:
    write_path = "system/revision/config/merge/type/" + params.data.name

  ; CAS guard
  if params.data.expected_hash is not null:
    current = tree.get(write_path)
    if hash(current) != params.data.expected_hash:
      return error(409, "stale_expected_hash")

  if params.data.action == "delete":
    tree.put(write_path, null)
    return {path: write_path, status: "deleted"}

  if params.data.action != "set":
    return error(400, "invalid_action")

  ; Validate the config entity
  config = params.data.config
  if config is null:
    return error(400, "missing_config")
  if config.data.deletion_resolution in ("lww", "keep-both"):
    ; §2.3 strategy-rejection contract — these are explicitly invalid for deletion_resolution
    return error(400, "invalid_strategy",
                 reason = "deletion_resolution " + config.data.deletion_resolution
                          + " is not a valid value (see §2.3); use one of "
                          + "preserve-on-conflict | deletion-wins | three-way-fallthrough "
                          + "| deterministic | <handler-path>")
  ; Other field-shape validation (strategy field, pattern shape, etc.) per §2.3.

  ; Idempotent: re-writing the same content_hash is a no-op
  config_hash = content_store.put(config)
  current_hash = hash(tree.get(write_path))
  if config_hash == current_hash:
    return {path: write_path, hash: config_hash, status: "no_change"}

  tree.put(write_path, config_hash)
  return {path: write_path, hash: config_hash, status: "set"}
```

**Why a dedicated op (v3.3, D1).** This op exists for a different reason than §4.4.17 `config`. §4.4.17 was driven by real *coordination* need — multi-entity coordinated writes (revision-config + `system/tree/tracking-config/{prefix}` in ordered phases per the auto-version-enable/disable path), V1–V5 validation interacting with §6.1 reentrancy rules, cascade-halt handling. Merge-config has none of those (single-entity write, no tracking-config sibling, no auto-version cascade interaction — merge-config is a passive configuration read by the merge algorithm at merge time). The original v2.6 design correctly didn't create an op for merge-config: there was no coordination need, and validation could happen at resolve time (the lazy / read-defensive pattern). v3.1 / Amendment 4 then added the **write-time strategy-rejection contract** to §2.3 (`lww` and `keep-both` MUST be rejected at config-write time with `400 invalid_strategy`); that contract retroactively *requires* a handler op as the enforcement site, because raw `tree:put` cannot reject at write time — only a handler op can. Amendment 4 focused on the strategy semantics, not on the enforcement mechanism; the contract landed without the op that makes it enforceable. v3.3 closes that gap.

Prior to v3.3 the spec mandated the strategy-rejection outcome but did not pin the mechanism. Implementations diverged: Rust authored a dedicated `merge-config` op; Go and Python wired a read-time defensive `ValidateDeletionResolution()` (collapsing invalid persisted values silently at resolve time rather than rejecting at write time). The §2.3 contract is only enforceable at write time if there IS a canonical write path; otherwise an operator capability granting `system/tree:put` on the merge-config namespace bypasses the rejection check. Anchoring the namespace as handler-owned (§2.3 "Handler-owned namespace" paragraph) + providing this op as the canonical write path gives all three impls the same enforcement site. See `GUIDE-CAPABILITIES.md` §8.1 ("Broad `tree:put` on a system handler namespace") for the operator-cap deployment posture.

**Conformance vectors (cross-impl):**

- `merge_config_set_rejects_deletion_resolution_lww` — action=set, config.deletion_resolution="lww" → `400 invalid_strategy`; no binding lands.
- `merge_config_set_rejects_deletion_resolution_keep_both` — same shape; `400 invalid_strategy`.
- `merge_config_set_accepts_valid_deletion_resolution` — all four valid strategies (`preserve-on-conflict`, `deletion-wins`, `three-way-fallthrough`, `deterministic`) bind successfully.
- `merge_config_set_idempotent` — re-issuing identical config returns `no_change`; no new content-store entry.
- `merge_config_cas_guard` — stale `expected_hash` → `409 stale_expected_hash`; no binding written.
- `merge_config_delete` — action=delete unbinds the path; returns `status: "deleted"`.

#### 4.4.19 fetch-diff

Incremental content transport: bundle the **content closure** of what changed between a caller-supplied base version and the handler peer's **current head**, in one round trip. This is the bandwidth-efficient core of cross-peer revision-follow — the canonical 2-step follow chain is `subscribe head → revision:fetch-diff(prefix, base) → tree:merge`.

`fetch-diff` is the content-returning sibling of `diff` (§4.4.14): where `diff` returns `system/tree/diff` *metadata* between two explicit versions, `fetch-diff` returns the `system/envelope` *closure* between the caller's base and the handler's current head. The target is **implicit** (the handler peer's current head) — this is what keeps the operation chain-expressible: a standing continuation chain threads a single dynamic field (`base`, from the subscription notification's `previous_hash`) into otherwise-static params. (A two-explicit-version shape would require multi-field injection that the continuation `transform_ops` vocab cannot assemble today.)

```
EXECUTE system/revision  operation: "fetch-diff"
  params: {
    type: "system/revision/fetch-diff-params"
    data: {
      prefix: "project/"
      base:   <version hash the caller already has; zero hash for full closure>
    }
  }
```

```
system/revision/fetch-diff-params := {
  fields: {
    prefix: {type_ref: "system/tree/path"}
    base:   {type_ref: "system/hash"}
             ; Version hash the caller already has. The diff is computed
             ; from this version's trie root to the handler peer's current
             ; head trie root. Zero hash = diff against the empty trie
             ; (full closure — first-time / bootstrap follower).
             ; `target` is NOT a parameter: it is always the handler peer's
             ; current head, which keeps the op single-dynamic-field for
             ; chain expressibility.
  }
}
```

**Algorithm:**

```
handle_fetch_diff(ctx, params):
  prefix = resolve(params.data.prefix, local_peer_id)
  prefix_hash = prefix_hash(prefix)

  ; target = current head (implicit)
  head_hash = tree.get("system/revision/" + prefix_hash + "/head")
  if head_hash is null:
    return error(404, "no_local_state", "No version history at prefix")
  target_version = content_store.get(head_hash)
  target_root    = content_store.get(target_version.data.root)

  ; base — caller-supplied version (or zero for full closure)
  if params.data.base == ZERO_HASH:
    base_root = EMPTY_TRIE_ROOT          ; diff-against-empty → full closure
  else:
    base_version = content_store.get(params.data.base)
    if base_version is null:
      return error(404, "base_not_found", "Base version not in content store")
    if not is_version_entry(base_version):
      return error(400, "base_not_a_version", "Base hash does not resolve to a version entry")
    base_root = content_store.get(base_version.data.root)

  ; Diff + bundle the changed closure. compute_trie_diff and the closure-bundle
  ; primitives live in the tree extension (§4.3, §6); the revision handler calls
  ; them downward (revision → tree). No tree-layer op reaches into revision.
  diff = compute_trie_diff(base_root, target_root)        ; tree §4.3
  return build_diff_envelope(target_root, diff)           ; tree §6 closure-bundle:
                                                          ;   root = snapshot(target_root)
                                                          ;   included = added/changed leaves
                                                          ;     + trie nodes from each change to root
```

**Returns:** `system/envelope` — root is a `system/tree/snapshot` of the current-head trie root; `included` carries the added/changed data entities and the trie nodes along the paths from each change to the root. The follower applies it with `tree:merge` (`source_envelope`). Wire-format-identical to a filtered `tree:extract` envelope; only the producer differs.

**Layering.** `fetch-diff` is a revision operation because its inputs are *versions* (revision-layer objects): the caller supplies a version hash, the handler reads its own version head, and dereferencing version → trie root is the revision extension's own knowledge. The diff and closure-bundle computation are tree-layer primitives (`compute_trie_diff` §4.3; the reachable-hash / trie-entity collectors §6) that the revision handler invokes downward. This is the correct dependency direction (revision → tree). The earlier `tree:extract.since` design placed this in the tree layer and was withdrawn (see EXTENSION-TREE.md v3.15 and `proposals/PROPOSAL-TREE-EXTRACT-SINCE.md` Amendment 1) precisely because it forced a tree op to dereference revision version entries.

**Cross-peer dispatch semantics.** `revision:fetch-diff` is **unambiguously executor-local**: under any dispatch shape (in-process L0, local dispatch, or cross-peer wire dispatch), the implicit `target` is the executor peer's current head and the base lookup occurs in the executor peer's content store. This contract is single-valued, so cross-peer invocation is well-defined. Two canonical call patterns:

1. **Local follow chain (push-shaped).** Runs on the follower peer; cross-peer subscribes to leader's head, then cross-peer dispatches `revision:fetch-diff` to the leader, then local `tree:merge`. Recipe: `subscribe head@leader → revision:fetch-diff@leader(prefix, base=$notification.previous_hash) → tree:merge@follower`.

2. **Cross-peer pull reconcile.** Caller (A) directly asks the remote executor (B) for B's diff: `peer_at(B).revision_fetch_diff(prefix, base=A.last_seen_head_for(B))`. A then applies the returned envelope locally. Used by `ReconcileSinceLastSeen`-style SDK helpers (e.g., `entity-workbench-go/entitysdk/reconcile.go`).

Implementations **MUST NOT** reject cross-peer dispatch of `fetch-diff` (no `400 invalid_dispatch` or analog). Only the error codes in the table below apply. The earlier `PROPOSAL-REVISION-DIFF-SINCE-LOCAL-HEAD` framing about "reads local state ambiguously cross-peer" applied to a *different*, deferred op (in `proposals/deferred/`) with ambiguous-local semantics — `fetch-diff` is not that op; its target-implicit shape was chosen precisely to keep semantics single-valued.

Capability requirement under cross-peer dispatch: as with all read ops, caller MUST hold `revision:fetch-diff` (or sufficient revision-read) on the executor's prefix (the `capability_denied/403` row below applies identically under local and cross-peer dispatch). Capability is not affected by call pattern.

**Apply step is content-only.** `fetch-diff` + `tree:merge` mirrors *content* into the follower's namespace (`/{follower}/{prefix}/...`). It does **not** advance the follower's local version DAG. A follower that also wants its DAG to converge calls `revision:merge` (or uses the `RevisionConverge` composition) — a deliberately separate concern. Keeping the canonical follow recipe at `tree:merge` is what keeps content-mirroring and DAG-mirroring from re-entangling.

**Classification:** convenience (composes from a head read + version-entry deref + tree's `compute_trie_diff` + closure bundle). Implementations SHOULD support it; cross-peer revision-follow at scale depends on it (the alternative, full `tree:extract` per commit, is O(workspace) bandwidth).

**Errors:**

| Error Code | Status | Condition |
|-----------|--------|-----------|
| `invalid_params` | 400 | Malformed params (missing `prefix`, etc.) |
| `capability_denied` | 403 | Caller lacks revision-read on the prefix |
| `no_local_state` | 404 | No version head at the prefix |
| `base_not_found` | 404 | `base` (non-zero) not in the handler's content store; caller may retry with a different base or zero |
| `base_not_a_version` | 400 | `base` resolves to a non-version entity |

**Chain-dispatch observability.** When `revision:fetch-diff` is dispatched as a chain step and returns a non-2xx response, the continuation engine binds a chain-error marker per EXTENSION-CONTINUATION.md §3.10.1 with `{reason}` = the `code` value from the errors table above **verbatim** (one of `no_local_state`, `base_not_found`, `base_not_a_version`, `capability_denied`, `invalid_params`). `chain_trace` consumers find the marker at the canonical path `system/runtime/chain-errors/lost/{chain_id}/{step_index}/{code}/{marker_hash}`.

Note that `fetch-diff` produces no executor-side state binding — the diff lands on the *follower* side after the subsequent `tree:merge` chain step applies the returned envelope. The §9 #8 completion contract (per `GUIDE-INSPECTABILITY.md` v1.1 §9 #8) for `fetch-diff`-rooted chains uses the follower-side mirror as the SUCCESS sentinel:

- **SUCCESS** = follower-side tree mutation under `/{follower}/{prefix}/...` after the subsequent `tree:merge` chain step lands.
- **FAILURE** = chain-error marker bound at the canonical path above.
- **ANOMALY** = `fetch-diff` EXECUTE dispatched (visible on dispatch tap) but neither follower-side tree state advanced (no new entities under prefix) nor a chain-error marker bound under the chain's `(chain_id, step_index)`.

Rationale: F-CIMP-7 (Cohort B) was a Python `base_not_a_version` rejection that went undiagnosed because the marker was not at a predictable path for `chain_trace` to find. Silent failure looked identical to "the chain never ran." This paragraph closes that gap by pinning the `code → {reason}` propagation rule and naming the follower-side mirror as the structural SUCCESS sentinel for revision-rooted chains.

**Clarification (v1.2.1, per Core Rust §9 item 2 + synthesis §4.3):** `fetch-diff` itself does NOT emit explicit success-sentinel entities. The SUCCESS condition described above (follower-side tree mutation under `/{follower}/{prefix}/...` after the subsequent `tree:merge` chain step lands) is observable *evidence* that the chain succeeded, captured by inspect tooling walking the follower's tree after the chain completes — it is NOT something `fetch-diff` produces. The handler's contract is: return non-2xx with chain-error marker propagation per the table above on failure; return 2xx with the envelope on success. The SUCCESS sentinel is observed downstream (after `tree:merge` lands), not emitted by `fetch-diff`.

### 4.5 Result Types

```
system/revision/commit-result := {
  fields: {
    version:  {type_ref: "system/hash"}          ; Content hash of the created version entity
    root:     {type_ref: "system/hash"}           ; Content hash of the trie root node
  }
}

system/revision/log-result := {
  fields: {
    prefix:    {type_ref: "system/tree/path"}
    versions:  {array_of: {type_ref: "system/hash"}}
                ; Version content hashes in reverse chronological order (newest first).
                ; Implementations SHOULD include the corresponding version entities
                ; in the result envelope's included map (system/envelope, not outer protocol envelope).
    has_more:  {type_ref: "primitive/bool"}
                ; Whether more versions exist beyond the returned set
  }
}

system/revision/merge-result := {
  fields: {
    status:    {type_ref: "primitive/string"}
                ; "already_in_sync", "fast_forward", "already_ahead",
                ; "merged", "merged_with_conflicts", "would_merge", "would_conflict",
                ; "converged_identical", "oscillation_detected"
    version:   {type_ref: "system/hash", optional: true}
                ; Content hash of the merge version (absent for already_in_sync/already_ahead)
    conflicts: {array_of: {type_ref: "system/tree/path"}, optional: true}
                ; Trie-relative paths with unresolved conflicts
    merged_count:  {type_ref: "primitive/uint", optional: true}
                    ; Number of merged bindings (present for preview)
    deleted_count: {type_ref: "primitive/uint", optional: true}
                    ; Number of deleted bindings (present for preview)
    cascade_warnings: {array_of: {type_ref: "system/revision/cascade-warning"}, optional: true}
                    ; Bindings whose tree.put returned 207 (cascade halted).
                    ; Absent or empty when all cascades completed normally.
  }
}

system/revision/cascade-warning := {
  fields: {
    path:               {type_ref: "system/tree/path"}
                          ; Trie-relative path within the prefix where the cascade halted
    consumer_halted:    {type_ref: "primitive/string"}    ; e.g., "revision/auto-version"
    error_code:         {type_ref: "primitive/string"}    ; from system/protocol/error
  }
}

system/revision/resolve-result := {
  fields: {
    path:               {type_ref: "system/tree/path"}
                          ; Trie-relative path within the prefix that was resolved
    resolved:           {type_ref: "system/hash", optional: true}
                          ; Content hash of the resolved entity.
                          ; Null when resolved by deletion (input was null).
    remaining_conflicts: {type_ref: "primitive/uint"}
                          ; Number of unresolved conflicts remaining for this prefix
                          ; after this resolution. Zero = all conflicts resolved.
  }
}

system/revision/fetch-result := {
  fields: {
    head:      {type_ref: "system/hash", optional: true}
                ; Remote's current HEAD version hash. Absent if no versions.
    versions:  {array_of: {type_ref: "system/hash"}, optional: true}
                ; Version entry hashes in DAG order.
                ; Corresponding entities in the result envelope's included map (system/envelope).
    has_more:  {type_ref: "primitive/bool", optional: true}
                ; Whether more versions exist beyond the depth limit
  }
}

system/revision/fetch-entities-result := {
  fields: {
    found:   {array_of: {type_ref: "system/hash"}}
              ; Entity hashes that were successfully retrieved.
              ; Corresponding entities in the result envelope's included map (system/envelope).
    missing: {array_of: {type_ref: "system/hash"}, optional: true}
              ; Entity hashes not found in content store (GC'd or invalid)
  }
}

system/revision/push-params := {
  fields: {
    remote:         {type_ref: "primitive/string"}
                     ; Remote peer URI
    prefix:         {type_ref: "system/tree/path"}
                     ; Local prefix — identifies the local revision config
                     ; and metadata subtree
    remote_prefix:  {type_ref: "system/tree/path", optional: true}
                     ; Remote peer's prefix for the same content.
                     ; Default: same as `prefix`.
    versions: {array_of: {type_ref: "system/hash"}, optional: true}
               ; Version entry hashes to push. Default: HEAD.
               ; Envelope includes version entries + trie nodes + data entities.
  }
}

system/revision/push-result := {
  fields: {
    status:   {type_ref: "primitive/string"}
               ; "up_to_date", "pushed", "behind", "diverged", "nothing_to_push"
    versions: {type_ref: "primitive/uint", optional: true}
               ; Number of versions pushed
    message:  {type_ref: "primitive/string", optional: true}
               ; Explanation (for behind/diverged)
  }
}

system/revision/ancestor-result := {
  fields: {
    ancestor: {type_ref: "system/hash", optional: true}
               ; Common ancestor version hash. Absent if no common ancestor.
  }
}

system/revision/branch-result := {
  fields: {
    status:   {type_ref: "primitive/string", optional: true}
               ; "created", "deleted" (for create/delete actions)
    branch:   {type_ref: "primitive/string", optional: true}
               ; Branch name
    version:  {type_ref: "system/hash", optional: true}
               ; Branch head version (for create)
    branches: {map_of: {type_ref: "system/hash"}, optional: true}
               ; Branch name → version hash (for list action)
    active:   {type_ref: "primitive/string", optional: true}
               ; Active branch name (for list action)
  }
}

system/revision/checkout-result := {
  fields: {
    status:         {type_ref: "primitive/string"}       ; "checked_out"
    target_version: {type_ref: "system/hash"}            ; Version hash requested
    head:           {type_ref: "system/hash"}            ; Actual post-operation head pointer.
                                                         ; Under auto-version OFF, equals target_version.
                                                         ; Under auto-version ON, a new descendant
                                                         ; of target_version with matching root.
    branch:         {type_ref: "primitive/string", optional: true}
                     ; Branch name (absent for detached HEAD checkout)
  }
}

system/revision/tag-result := {
  fields: {
    status:  {type_ref: "primitive/string", optional: true}
              ; "created", "deleted" (for create/delete actions)
    tag:     {type_ref: "primitive/string", optional: true}
              ; Tag name
    version: {type_ref: "system/hash", optional: true}
              ; Tagged version hash (for create)
    tags:    {map_of: {type_ref: "system/hash"}, optional: true}
              ; Tag name → version hash (for list action)
  }
}

system/revision/cherry-pick-result := {
  fields: {
    status:    {type_ref: "primitive/string"}
                ; "cherry_picked", "cherry_picked_with_conflicts"
    version:   {type_ref: "system/hash"}
                ; Content hash of the new version created
    source:    {type_ref: "system/hash"}
                ; Content hash of the cherry-picked source version
    conflicts: {array_of: {type_ref: "system/tree/path"}, optional: true}
                ; Trie-relative paths with unresolved conflicts
  }
}

system/revision/revert-result := {
  fields: {
    status:    {type_ref: "primitive/string"}
                ; "reverted", "reverted_with_conflicts"
    version:   {type_ref: "system/hash"}
                ; Content hash of the new version created
    reverted:  {type_ref: "system/hash"}
                ; Content hash of the reverted version
    conflicts: {array_of: {type_ref: "system/tree/path"}, optional: true}
                ; Trie-relative paths with unresolved conflicts
  }
}
```

---

## 5. Merge Strategy Framework

### 5.1 Merge Resolution Cascade

When the version merge encounters a leaf conflict (same path, both sides changed, different hashes), resolution follows this cascade:

1. **Per-type merge config.** Check `system/revision/config/merge/type/{type_name}` for a type-based strategy. The config can specify a built-in strategy or dispatch to a custom merge handler.

2. **Per-path merge config.** Check `system/revision/config/merge/path/{name}` for a path-pattern-based strategy. Same strategy options.

3. **Default strategy.** Three-way diff3 on entity data (§5.2).

4. **Conflict entity.** If nothing resolves it, create a `system/revision/conflict` entity.

Merge behavior is configured, not auto-discovered. There is no implicit type → handler lookup. To enable custom merge for a type, an explicit merge config entry must exist. This keeps the merge framework extensible without requiring infrastructure for type-handler discovery.

**Path argument scope.** The `path` argument throughout the merge cascade is trie-relative — it is the path within the version trie, not the absolute tree path. Merge configs stored at `system/revision/config/merge/path/{name}` are global (not prefix-scoped), but their `pattern` field matches against trie-relative paths. A config with `pattern: "*"` matches all paths within any merge, regardless of prefix. A config with `pattern: "docs/**"` matches `docs/` subtrees under any prefix. This is by design — merge strategies are typically path-pattern concerns (e.g., "all `.lock` files use source-wins"), not prefix-specific. Per-type config (step 1) is inherently path-independent.

**Wildcard scope is a peer-wide change — review accordingly (v7.70 Amendment 1; guidance).** The global-vs-prefix design above was reasoned for the *path-pattern* case; the safety of broad wildcards was not. A wildcard config (`pattern: "*"` or `"**"`) with a conflict-suppressing strategy (`keep-both`, `source-wins`, `target-wins`) **silently rewires conflict resolution for every prefix and every future merge on the peer, including prefixes that do not exist yet.** Two consequences operators must account for: (1) installing such a config is a **peer-wide configuration change** and SHOULD be reviewed with the same care as any other peer-config write (a left-behind config behaves identically to an operator-introduced silent regression); (2) **there is no audit signal today** — a config that resolves a conflict via a non-default strategy produces a merge result byte-identical to a genuinely conflict-free merge (`status: merged`, empty `conflicts`), so a subscriber / sync chain / downstream verifier cannot tell a clean merge from a config-suppressed one. Making config-resolved conflicts observable (a merge-result field reporting which configs resolved which paths) is the intended direction but is **not yet specified** — tracked for a holistic pass (WORKSTREAMS W2). No behavior change in this version; this paragraph documents the footgun and the audit-signal gap so implementers and operators are aware while the fix is designed. Wildcard patterns are **not** rejected (a fully-automated peer using `pattern: "*"` + a single strategy is legitimate); the gap is observability, not the knob.

When the config specifies `strategy: "handler"`, the revision handler dispatches to the named handler with a `system/revision/merge-request` and expects a `system/revision/merge-response` (§5.3). The handler can implement any algorithm — field-level merge, text diff3, CRDT, or domain-specific semantics.

**Type matching.** The per-type config lookup checks both sides when types differ — local type first (existing entity), then remote type (incoming). Either config's handler may understand cross-type merge. The handler receives both entity hashes and can inspect types directly. When both entities have the same type, only one lookup occurs. If neither type has a config, per-path config (step 2) handles it — per-path config is type-agnostic and covers cases where type-based dispatch is insufficient.

**Default three-way is also type-agnostic.** The field-level merge (§5.2) compares entity data fields regardless of type name. Two entities with different types but overlapping fields can still merge — fields present in both are compared, fields unique to one side are kept. This handles structural typing and type evolution without requiring explicit cross-type configuration.

**Strategy lookup algorithm:**

```
resolve_leaf_conflict(base_hash, local_hash, remote_hash, path):
  local = content_store.get(local_hash) if local_hash else null
  remote = content_store.get(remote_hash) if remote_hash else null

  ; Step 1: Per-type merge config
  ; Check local type first (existing entity), then remote type (incoming entity).
  ; When types differ, both configs get a chance — either side's handler may
  ; understand cross-type merge. When types are the same, only one lookup occurs.
  types_to_check = []
  if local is not null:
    types_to_check.append(local.type)
  if remote is not null and (local is null or remote.type != local.type):
    types_to_check.append(remote.type)

  for type_name in types_to_check:
    type_config = tree.get("system/revision/config/merge/type/" + type_name)
    if type_config is not null:
      result = apply_strategy(type_config.data.strategy, base_hash, local_hash, remote_hash, path,
                              type_config.data.handler)
      if result is not null:
        return result

  ; Step 2: Per-path merge config
  configs = list_entities("system/revision/config/merge/path/")
  best_match = null
  best_specificity = -1

  for config in configs:
    if glob_match(config.data.pattern, path):
      specificity = pattern_specificity(config.data.pattern)
      if specificity > best_specificity:
        best_match = config
        best_specificity = specificity

  if best_match is not null:
    result = apply_strategy(best_match.data.strategy, base_hash, local_hash, remote_hash, path,
                            best_match.data.handler)
    if result is not null:
      return result

  ; Step 3: Default three-way
  result = three_way_merge(base_hash, local_hash, remote_hash, path)
  if result.resolved:
    return {resolved: true, hash: result.hash}

  ; Step 4: Conflict entity
  return null    ; caller creates conflict entity

apply_strategy(strategy, base_hash, local_hash, remote_hash, path, handler_path):
  if strategy == "source-wins":
    return {resolved: true, hash: remote_hash}
  elif strategy == "target-wins":
    return {resolved: true, hash: local_hash}
  elif strategy == "lww":
    return {resolved: true, hash: lww_resolve(local_hash, remote_hash)}
  elif strategy == "field-level":
    result = three_way_merge(base_hash, local_hash, remote_hash, path)
    if result.resolved:
      return {resolved: true, hash: result.hash}
    return null
  elif strategy == "keep-both":
    ; KeepBoth only applies to edit-vs-edit conflicts (both non-null).
    ; Delete-vs-edit conflicts (one side null) fall through to conflict —
    ; "keeping both" when one side is a deletion is not meaningful.
    if local_hash is null or remote_hash is null:
      return null    ; fall through to conflict entity
    hash_prefix = hex(remote_hash)[0:8]
    return {
      resolved: true,
      hash: local_hash,
      additional_bindings: [(path + ".keep-both-" + hash_prefix, remote_hash)]
    }
  elif strategy == "handler":
    handler_result = dispatch_merge_handler(handler_path, base_hash, local_hash, remote_hash)
    if handler_result is not null:
      return {resolved: true, hash: handler_result}
    return null
  else:
    return null    ; unknown strategy — fall through to conflict

dispatch_merge_handler(handler_path, base_hash, local_hash, remote_hash):
  result = dispatch(handler_path, "merge", {
    type: "system/revision/merge-request"
    data: { base: base_hash, local: local_hash, remote: remote_hash }
  })
  if result.data.resolved:
    return result.data.entity
  return null
```

**Strategy result shape.** `apply_strategy` returns a structured result `{resolved, hash, additional_bindings}` or null. Most strategies return a single binding (`additional_bindings` absent). The `keep-both` strategy returns additional bindings. The `additional_bindings` field propagates to every site that calls `apply_strategy` — implementations typically have multiple dispatch sites (binding-at-node, delete-vs-modify, flat snapshot merge). All sites MUST handle `additional_bindings` when present.

**Per-type merge config.** Stored at `system/revision/config/merge/type/{type_name}`:

```
system/revision/config/merge/type/{type_name} := {
  type: "system/revision/merge-config"
  data: {
    strategy: "field-level" | "source-wins" | "target-wins" | "lww" | "keep-both" | "handler"
    handler:  system/tree/path (optional, when strategy is "handler")
  }
}
```

This complements the per-path merge config. Type-based config takes priority over path-based config (step 1 before step 2 in the cascade).

**Custom merge handlers are not limited to same-type conflicts.** The handler receives entity hashes and can inspect the entities directly — including their types. A handler configured for one type can handle cross-type conflicts, structural type compatibility, or any domain-specific logic. The configuration maps type → handler; the handler decides what it can resolve.

**Merge handlers can maintain their own state.** The `merge-request` provides base, local, and remote entity hashes. A handler can store operation logs, merge metadata, or convergence state as separate entities in the tree — the merge handler has full tree access through its dispatch context. This supports both stateless merge (compute result from three inputs) and stateful merge (maintain state across merges, e.g., CRDT convergence data, text diff history, schema migration records).

**Commutativity requirement for custom handlers.** Custom merge handlers registered via `system/revision/config/merge/type/**` or `system/revision/config/merge/path/**` MUST satisfy one of: (a) commutativity — `merge(A, B, ancestor) == merge(B, A, ancestor)` for all valid inputs; OR (b) be used only on prefixes configured with `merge_order: "deterministic"`. Non-commutative handlers under `caller-perspective` ordering produce cross-peer divergence and are non-conformant. Implementations SHOULD detect and reject configurations that pair a known non-commutative handler type with `caller-perspective` ordering; operators are responsible for declaring commutativity of their custom handlers.

Built-in merge strategies (three-way, last-write-wins, concat, etc.) are commutative by construction; the requirement applies only to user-registered handlers.

### 5.2 Built-in Strategy: Three-Way Merge

For structured entities (entities with typed `data` fields):

```
three_way_merge(ancestor_hash, local_hash, remote_hash, path):
  ; Handle delete-vs-edit conflicts.
  ; The merge algorithm (§4.4.4) calls this when both sides changed differently.
  ; Either side may be null (deleted).
  if local_hash is null or remote_hash is null:
    ; One side deleted, other side edited (or both deleted differently — impossible,
    ; both-null is caught by the "same on both sides" check in the merge algorithm).
    ; Delete-vs-edit always produces a conflict in three-way merge.
    return {resolved: false}

  ancestor = content_store.get(ancestor_hash)
  local = content_store.get(local_hash)
  remote = content_store.get(remote_hash)

  if ancestor is null:
    ; Create/create conflict — no common ancestor for these entities
    return {resolved: false}

  ; Compare field by field
  merged_data = {}
  conflicts = []

  all_keys = union(keys(local.data), keys(remote.data), keys(ancestor.data))
  for key in all_keys:
    ancestor_val = ancestor.data.get(key)
    local_val = local.data.get(key)
    remote_val = remote.data.get(key)

    if local_val == remote_val:
      merged_data[key] = local_val
    elif local_val == ancestor_val:
      merged_data[key] = remote_val      ; only remote changed
    elif remote_val == ancestor_val:
      merged_data[key] = local_val       ; only local changed
    else:
      conflicts.append(key)              ; both changed differently

  if len(conflicts) > 0:
    return {resolved: false}

  merged_entity = {type: local.type, data: merged_data}
  merged_hash = content_store.put(merged_entity)
  return {resolved: true, hash: merged_hash}
```

### 5.3 Custom Merge Handlers

When the merge strategy names a handler path (e.g., `app/merge/text`), the revision handler delegates:

```
EXECUTE <handler_path>  operation: "merge"
  params: {
    type: "system/revision/merge-request"
    data: {
      path:     <conflicting path>
      base:     <ancestor entity hash>
      local:    <local entity hash>
      remote:   <remote entity hash>
    }
  }
```

```
system/revision/merge-request := {
  fields: {
    base:   {type_ref: "system/hash", optional: true}  ; common ancestor (null if no ancestor)
    local:  {type_ref: "system/hash"}                   ; local entity hash
    remote: {type_ref: "system/hash"}                   ; remote entity hash
  }
}

system/revision/merge-response := {
  fields: {
    resolved: {type_ref: "primitive/bool"}
    entity:   {type_ref: "system/hash", optional: true}  ; merged entity hash (if resolved)
    reason:   {type_ref: "primitive/string", optional: true}  ; why not resolved
  }
}
```

A custom merge handler can implement any algorithm: text diff3, CRDT merge, semantic merge for domain-specific entity types. The revision extension provides the framework; the handler provides the intelligence.

### 5.4 CRDT as Merge Strategy

A CRDT merge handler receives base, local, and remote entities. It:

1. Computes operations from base → local (local changes)
2. Computes operations from base → remote (remote changes)
3. Applies both operation sets to a CRDT instance
4. Extracts the merged state
5. Returns a regular entity (not CRDT state)

No CRDT metadata persists. The merged entity is a normal entity. This follows the Eg-walker principle: CRDT is a computational artifact during merge, not a storage format.

### 5.5 Trie-Aware Version Merge

The full version merge algorithm combines trie merge with the type-aware cascade (§5.1). The trie merge extends the tree extension's diff algorithm's compressed key handling to three-way comparison.

**Null node semantics.** A null node represents a subtree that doesn't exist on one side of the merge. The three-way merge must handle all combinations of present/absent nodes across base, local, and remote:

| base | local | remote | result |
|------|-------|--------|--------|
| null | null | exists | take remote (added by remote) |
| null | exists | null | take local (added by local) |
| null | exists | exists | both added — recurse (may conflict at leaves) |
| exists | null | null | both deleted — omit |
| exists | null | exists | local deleted, remote kept — conflict |
| exists | exists | null | local kept, remote deleted — conflict |
| exists | exists | exists | normal three-way — recurse |

Delete/modify conflicts (rows 5-6) are reported at the subtree root path. The implementation MAY report per-leaf conflicts instead by walking the deleted side's subtree.

**Node comparison.** Hash comparison is only valid between two real (stored) nodes. Virtual nodes (constructed during compression mismatch resolution) have no canonical hash. When either node in a comparison is virtual, the algorithm MUST recurse into children rather than comparing hashes.

```
nodes_equal(a, b):
  if a is null and b is null: return true
  if a is null or b is null:  return false
  if is_real(a) AND is_real(b): return hash(a) == hash(b)
  return false   ; virtual nodes — must recurse
```

```
version_merge(base_root, local_root, remote_root):
  return trie_merge(
    base_root,
    local_root,
    remote_root,
    prefix = ""
  )

trie_merge(base, local, remote, path_prefix):
  ; Handle null nodes (subtree absent on one or more sides)
  if local is null and remote is null:
    return null                              ; both deleted — omit subtree
  if base is null and local is null:
    return remote                            ; added by remote only
  if base is null and remote is null:
    return local                             ; added by local only
  if local is null:
    ; local deleted, remote exists — delete/modify conflict
    report_subtree_conflict("local_deleted", path_prefix, base, remote)
    return remote                            ; default: keep remote (configurable)
  if remote is null:
    ; remote deleted, local exists — modify/delete conflict
    report_subtree_conflict("remote_deleted", path_prefix, base, local)
    return local                             ; default: keep local (configurable)

  ; Both sides exist — early exit on equal nodes (real nodes only)
  if nodes_equal(local, remote):  return local     ; both same
  if nodes_equal(local, base):    return remote    ; only remote changed
  if nodes_equal(remote, base):   return local     ; only local changed

  ; Both changed — recurse into children
  result = new_node()

  ; Merge bindings at this node
  result.binding = merge_leaf(
    base.binding if base else null,
    local.binding,
    remote.binding,
    path_prefix
  )

  ; Decompose all three entry maps by first segment
  decomp_base  = decompose_entries(base.entries if base else {})
  decomp_local = decompose_entries(local.entries)
  decomp_remote = decompose_entries(remote.entries)

  all_segments = union(keys(decomp_base), keys(decomp_local), keys(decomp_remote))

  for seg in sorted(all_segments):
    entry_base  = decomp_base.get(seg)      ; (remaining, hash) or null
    entry_local = decomp_local.get(seg)
    entry_remote = decomp_remote.get(seg)

    ; Resolve compression mismatches across all three sides
    ; When remaining paths differ, find divergence point and construct virtual nodes
    child_base, child_local, child_remote, child_prefix =
      resolve_three_way(seg, entry_base, entry_local, entry_remote, path_prefix)

    ; Apply three-way logic on resolved children (using nodes_equal for comparison)
    if nodes_equal(child_local, child_remote):
      if child_local is not null:
        merged_key, merged_hash = store_child(child_local, path_prefix, child_prefix)
        result.entries[merged_key] = merged_hash
    elif nodes_equal(child_local, child_base):
      if child_remote is not null:
        merged_key, merged_hash = store_child(child_remote, path_prefix, child_prefix)
        result.entries[merged_key] = merged_hash
    elif nodes_equal(child_remote, child_base):
      if child_local is not null:
        merged_key, merged_hash = store_child(child_local, path_prefix, child_prefix)
        result.entries[merged_key] = merged_hash
    else:
      ; Both changed — recurse
      merged = trie_merge(child_base, child_local, child_remote, child_prefix)
      if merged is not null:
        merged_key = relative_key(path_prefix, child_prefix)
        ; Recompress at call site — returns (key, node) with key extended
        ; through any collapsed single-child-no-binding chain
        (final_key, final_node) = recompress(merged_key, merged)
        result.entries[final_key] = content_store.put(final_node)

  ; Eliminate empty nodes
  if len(result.entries) == 0 and result.binding is null:
    return null

  ; Return raw result — the caller applies recompress with the correct entry key.
  ; The root node (called from version_merge) is never compressed.
  return result

merge_leaf(base_hash, local_hash, remote_hash, path):
  if local_hash == remote_hash: return local_hash
  if local_hash == base_hash:   return remote_hash
  if remote_hash == base_hash:  return local_hash

  ; Both changed — apply type-aware merge cascade (§5.1)
  return resolve_leaf_conflict(base_hash, local_hash, remote_hash, path)

store_child(node, parent_prefix, child_prefix):
  ; Store a resolved child node and return (key, hash) for parent's entries map
  ; relative_key strips parent_prefix from child_prefix to get the entry key
  key = relative_key(parent_prefix, child_prefix)
  if is_real(node):
    return (key, hash(node))
  else:
    ; Virtual node — store it (it's now a real merge result)
    return (key, content_store.put(node))
```

**Recompression algorithm.** After merging children, a result node may have gained or lost children, changing which entries should be compressed. The result MUST be recompressed before storing to maintain the path compression invariant.

`recompress` takes a `(key, node)` pair and returns a `(key, node)` pair. When a node is compressible (one child entry, no binding), the node is eliminated and its child's segment is concatenated into the key. The recursion continues until a non-compressible node is reached.

```
recompress(key, node):
  ; If the node has exactly one child entry and no binding, collapse it
  if len(node.entries) == 1 AND node.binding is null:
    child_key = the single key in node.entries
    child_hash = node.entries[child_key]
    child_node = content_store.get(child_hash)
    ; Recurse — the child may also be compressible
    return recompress(key + "/" + child_key, child_node)
  ; Node has multiple children or a binding — not compressible
  return (key, node)
```

The caller uses the returned key as the entry in the parent's entries map and stores the returned node. This ensures the parent's entry key correctly reflects any collapsed intermediate nodes.

Recompression is applied at the recursive call site in `trie_merge`, not at the end of `trie_merge` itself. This is because the entry key (needed for correct compression) is known by the caller, not by the node being returned. The root node (returned to `version_merge`) is never compressed.

**Compressed key handling in three-way merge.** The `resolve_three_way` function extends the tree extension's two-way `resolve_mismatch` to three inputs:

```
resolve_three_way(seg, entry_base, entry_local, entry_remote, path_prefix):
  ; entry_base, entry_local, entry_remote are each (remaining, hash) or null
  ; Returns: (child_base, child_local, child_remote, child_prefix)
  ;   where each child is a node (real or virtual) or null
  ;   and child_prefix is the path prefix for recursion

  ; Collect non-null remaining paths
  remainders = {}
  if entry_base  is not null: remainders["base"]  = entry_base.remaining
  if entry_local is not null: remainders["local"] = entry_local.remaining
  if entry_remote is not null: remainders["remote"] = entry_remote.remaining

  ; If all non-null remainders are the same, no mismatch to resolve
  unique_rems = unique(values(remainders))
  if len(unique_rems) <= 1:
    rem = unique_rems[0] if len(unique_rems) == 1 else ""
    full_key = seg if rem == "" else seg + "/" + rem
    child_prefix = join_path(path_prefix, full_key)
    return (
      content_store.get(entry_base.hash)  if entry_base  else null,
      content_store.get(entry_local.hash) if entry_local else null,
      content_store.get(entry_remote.hash) if entry_remote else null,
      child_prefix
    )

  ; Remainders differ — find longest common prefix across all non-null remainders
  ; Empty remainders become empty lists (side has real node at seg level)
  ; This matches resolve_mismatch which maps "" to []
  all_segs = [rem.split("/") if rem != "" else [] for rem in values(remainders)]
  common_len = 0
  if len(all_segs) > 0:
    min_len = min(len(s) for s in all_segs)
    while common_len < min_len:
      if all(s[common_len] == all_segs[0][common_len] for s in all_segs):
        common_len += 1
      else:
        break

  ; Build divergence path
  if common_len > 0 and len(all_segs) > 0:
    common = "/".join(all_segs[0][:common_len])
    diverge_prefix = join_path(join_path(path_prefix, seg), common)
  else:
    diverge_prefix = join_path(path_prefix, seg)

  ; For each side: resolve to a node at the divergence point
  ; If remaining goes past divergence → virtual node with suffix entry
  ; If remaining stops at divergence → real node from content store
  ; If absent → null (subtree doesn't exist on this side)

  resolve_side(entry, common_len):
    if entry is null: return null
    rem_segs = entry.remaining.split("/") if entry.remaining != "" else []
    suffix_segs = rem_segs[common_len:]
    suffix = "/".join(suffix_segs)
    if suffix == "":
      return content_store.get(entry.hash)           ; real node at divergence point
    else:
      return virtual_node(entries: {suffix: entry.hash}, binding: null)

  return (
    resolve_side(entry_base, common_len),
    resolve_side(entry_local, common_len),
    resolve_side(entry_remote, common_len),
    diverge_prefix
  )
```

This is a direct extension of the two-way mismatch resolution to three inputs. The same virtual node technique applies — when a side compressed past the divergence point, a virtual intermediate node is constructed so the three-way merge can proceed recursively.

The three `nodes_equal` comparisons at the top of `trie_merge` skip entire subtrees — this is what makes merge O(changes) instead of O(total_paths). The null handling ensures subtree additions and deletions are handled correctly without special-casing throughout the algorithm.

---

## 6. Auto-Versioning

### 6.1 Mechanism

Auto-version's contract is CRDT-style collaborative editing. When `auto_version` is enabled for a prefix, every tree write to a matching path produces a version entry. The version DAG converges naturally across peers under entity exchange.

**Per-write creation.** For each tree write event that matches a tracked prefix (and does not match the `exclude` patterns in the prefix's `system/revision/config`), auto-version produces one `system/revision/entry` capturing the tree state after the write.

**Deletion-marker emission at commit (v3.1, Amendment 2).** When auto-version (or explicit `commit`) emits a new version, the new version's trie MUST include explicit entries for every path that was bound in the parent version's trie. For paths bound in parent and bound in current live state, the entry carries the live binding (or the parent binding if unchanged). For paths bound in parent but unbound in current live state — the path was deleted between the parent and this commit — the entry binds the path to `CANONICAL_DELETION_MARKER_HASH` (ENTITY-NATIVE-TYPE-SYSTEM.md §4.9). This ensures version tries are explicit about every in-scope path; there is no ambiguous absence.

*Efficient diff path.* Implementations SHOULD compute the deletion set via the trie-diff primitive (EXTENSION-TREE.md §5) returning (`added`, `removed`, `changed`) between the parent version's trie root and the current live-tree trie root. The `removed` set is exactly the deletion-marker emission candidate set. Cost is O(changes × depth), not O(total paths in scope).

*Marker-augmentation precedes dedup (v3.3, D3).* Implementations MUST perform the marker-augmentation step (this paragraph and "Efficient diff path" above) BEFORE the dedup-against-prior-head check (the algorithm's `if current_head.data.root == new_trie_root: return current_head without creating a new entry` step). Dedup MUST observe the **post-augmentation** root — otherwise a commit whose only change is a deletion (and therefore whose pre-augmentation diff might appear empty) would be incorrectly suppressed, dropping the explicit deletion-marker entry the new version's trie requires. Go and Rust already perform augment-then-dedup; pinning prevents drift in new impls or refactors.

*Deletion-marker carry-forward.* Because deletion markers are canonical (same hash every time), a path marked deleted in the parent version remains marked deleted in the new version automatically — the trie entry stays the same; the content store dedups any re-put; no new deletion-marker entity is emitted. Storage is bounded by delete *events*, not delete-times-commits.

*Idempotency on already-unbound paths.* `tree:put(path, null)` when `path` is already unbound is a true no-op. The diff does not classify the path as `removed` (it was not in the parent's trie either); no deletion marker is emitted; no event fires.

*Single-event semantics.* The deletion-marker emission at commit time is a commit-internal trie update; it does NOT fire a separate EXTENSION-HISTORY event. The single `deleted` event for a given logical delete fires at `tree:put(path, null)` time, carrying the `prev_hash` of the entity that was unbound. Subscriptions on `deleted` events therefore fire once per logical delete, not twice. Implementations MUST NOT emit a second `deleted` event when the auto-version commit writes the deletion marker into the new version's trie.

*Synchronous auto-version emit (normative; per-prefix scope).* Auto-version emit MUST be synchronous with the tree write that triggers it, within the prefix's serialization context. Specifically: for a given revision-tracked prefix, implementations MUST ensure that no version-transcription operation can interleave between an unbind on that prefix and the corresponding deletion-marker commit. Operations on different prefixes proceed concurrently — the invariant is per-prefix, not global. A per-prefix mutex satisfies the requirement; a global write-lock satisfies it but is not required and would over-constrain concurrency.

*Nested-prefix coordination.* When multiple revision-tracked prefix configs cover a single path (e.g., a config at `archives/` and a nested config at `archives/notes/`), each config's serialization context is independent. A version-transcription operation holds the coordination context for ITS target prefix; concurrent auto-version emits for COVERING parent configs may execute against partial-state live-tree snapshots and may produce intermediate version entries that don't represent stable tree states. Each config's DAG converges independently and eventually given finite operations and reliable replication. The intermediate partial-state entries are NOT data-corrupting under canonical-deletion-marker semantics (preserve-on-absence handles them correctly during downstream merges).

*Cross-peer-sync visibility window.* During a nested-config in-progress merge, cross-peer syncs of the parent config's DAG may transport an intermediate partial-state version to other peers. The receiving peer's live tree may briefly reflect this partial state until the inner config's DAG separately syncs and propagates the full merge. Both states converge to the same eventual configuration; no manual operator action is required. Implementations and operators should be aware that this window exists.

*Implementation freedom.* Implementations MAY acquire coordination state for all covering prefixes in deterministic order to eliminate partial-state intermediates entirely. This is a quality-of-implementation choice, not a correctness requirement under deletion-marker semantics. Single-threaded implementations satisfy the per-prefix-coordination invariant trivially via their single-threaded execution; the spec's requirement is the *invariant* (per-prefix serialization context exists and is honored), not any particular mechanism.

**Interaction with version-transcription invariants (v3.2, A.3).** Under auto-version ON, in-flight live-tree writes that have not yet been captured by an auto-version emit at the moment a version-transcription operation runs (merge, checkout, fast-forward, push-integrate, cherry-pick, revert) are explicitly preserved by §4.4.4's Version-transcription invariant (1) — they are not in the source version's trie, so they are not in the removable set. After the transcription operation lands and head advances to `target_version`, the next auto-version emit captures the in-flight writes as an intermediate version that descends from `target_version`. The result is a normal CRDT merge state — `target_version → auto_intermediate_with_in_flight_writes` — not a wipe. The head-write atomicity invariant (§4.4.4 invariant (2)) ensures the auto-version emit's CAS sees the head at `target_version`, not at the prior head, so the auto-intermediate correctly chains from `target_version`. The two invariants compose: (2) keeps the auto-version emit from racing the transcription's head advance; (1) keeps the transcription from wiping the writes (2) is protecting.

**Algorithm.**

```
; Invoked once per matching tree write event, after structural summaries
; have updated system/tree/root/{prefix} (see "Emit ordering" below).
auto_version_on_write(event, prefix, version_config):
  if event.path matches version_config.exclude:
    return

  root = current_tracked_root(prefix)          ; from system/tree/root/{prefix}
  current_head = tree.get("system/revision/" + prefix_hash + "/head")

  if current_head is not null:
    current_version = content_store.get(current_head)
    if current_version.data.root == root:
      return    ; no structural change — no-op suppression

  version = {
    type: "system/revision/entry",
    data: {
      root:    root,
      parents: [current_head] if current_head else [],
    }
  }
  version_hash = content_store.put(version)
  advance_head(prefix, current_head, version_hash)    ; see "Contention handling"

  ; Advance active branch pointer if configured
  active_branch = tree.get("system/revision/" + prefix_hash + "/active-branch")
  if active_branch is not null:
    tree.put("system/revision/" + prefix_hash + "/branches/" + active_branch, version_hash)
```

Auto-version does not trigger push to remotes. Cross-peer propagation is not the revision extension's concern — see §6.3 for the subscription+continuation composition.

Version entries carry only structural identity: `root` and `parents`. Per-version timing (when the version was created, by whom, under what capability) is recoverable from the history transition for the write that produced the version — history already records execution context (`clock`, `author`, `capability`, `chain_id`, `parent_chain_id`) per transition. Adding these fields to the version entry itself would cause structurally-identical commits from different peers to produce different content hashes (two peers doing the "same commit" at different clocks would create DAG duplicates). Version identity is therefore kept purely structural; observability metadata lives in history.

**Reentrancy (structural).** Auto-version's own writes target `system/revision/entry/...` (via content_store) and `system/revision/head/**`, `system/revision/branches/**`, `system/revision/active-branch/**` (via tree.put). Configurations MUST include these paths in `exclude` when they would otherwise fall under a tracked prefix, OR auto-version MUST be scoped to prefixes that do not encompass `system/revision/**`. The most common case — auto-version enabled on application prefixes distinct from `system/**` — is self-excluding by construction.

When auto-version is configured for a prefix that encompasses system-owned engine paths (e.g., `"/"` for universal-tree versioning, per EXTENSION-TREE.md §3.4.1a), `exclude` MUST include the following patterns to prevent reentrancy cascades and meaningless versioning of engine state:

- `system/revision/**` — auto-version's own entries, head, branches, active-branch, tags, remotes, conflicts, and config paths.
- `system/tree/root/**` — structural-summary consumer's tracked-root outputs (written on every tree write to a tracked prefix; versioning these creates a self-feeding loop).
- `system/tree/tracking-config/**` — tracking config entities (meta-config; should not be versioned as data).
- `system/history/**` — history transition entities (versioning every transition is wasteful and creates compounding growth).
- `system/clock/**` — clock state advances (high-frequency, meaningless to version).

**SHOULD excludes (in addition to MUST).** When the corresponding extension is active on the peer, these paths SHOULD be added to the exclude list:

- `system/inbox/**` — incoming deliveries from subscription fan-out, continuation chains, and async results. High frequency; content-non-unique (many entities per path).
- `system/subscription/**` — subscription configs and token refresh writes.
- `system/continuation/**` — continuation entities, join state, advance-request entities.
- `system/compute/**` — compute process state and reactive result paths.

These are engine-owned state paths written at high frequency during normal operation. Versioning them produces DAG growth without user-meaningful content. They don't create auto-version feedback loops (inbox writes don't trigger auto-version's own work), but they do produce tens or hundreds of entries per minute on an active peer. The `system/**` shorthand already covers these; the explicit SHOULD list helps operators who use finer-grained exclude configurations.

The rule of thumb: exclude any path written by a system extension's engine as part of its normal operation. User-application data under `system/` (rare by convention) may be kept included if the operator explicitly wants it versioned.

A shorthand RECOMMENDED exclude for universal-tree auto-version: `system/**`. This over-excludes slightly (versioning opportunities in custom application paths under `system/` are lost) but is safe by construction. Implementations **MUST reject at config-write time** any `system/revision/config` with `auto_version: true` whose `prefix` encompasses the required-exclude paths (enumerated above) when those paths are NOT covered by the config's `exclude` list. Rejection is a fail-closed validator, not a runtime check — config writes that would cause cascades are denied at the handler boundary. Implementations MAY offer an explicit operator override flag (warning-with-acceptance) but the default MUST be rejection. This replaces the runtime reentrancy guard from prior drafts — structural exclusion is strictly simpler and has no failure modes.

**Contention handling.** Auto-version's head advance (`tree.put("system/revision/" + prefix_hash + "/head", ...)`) may contend under concurrent writes. The data model stores a single head hash per prefix at `system/revision/{H}/head`, so two unmitigated concurrent advances would overwrite each other — producing an orphan (a version entry in the content store that no head references). Under the no-orphan invariant this is a conformance failure.

Implementations MUST use one of the following mechanisms to satisfy the no-orphan invariant:

- **CAS+retry (unbounded).** Pass `expected_hash` on the `system/tree/put-request` for head advance. On conflict (409), re-read the head, rebuild parents from the new head, retry. Retry is unbounded — implementations MUST NOT bound the retry count and silently drop the version entry on exhaustion. Implementations SHOULD use exponential backoff with jitter to avoid thundering-herd contention. Retry location is implementer choice (caller thread, background worker, dedicated retry queue); implementations SHOULD document the choice because it affects tail-latency characteristics of tree writes under contention. Persistent contention that prevents retry convergence within reasonable operator expectations is observable via telemetry and is the operator's signal to reduce concurrency, not the implementation's signal to skip versions.
- **Single-writer serialization per prefix.** Serialize auto-version head advances per prefix through a single writer. Trivially safe; no CAS needed. Most applicable to single-threaded implementations or implementations that already serialize per-prefix processing.

What is NOT a conformant tactic: no mechanism at all; OR any tactic that may terminate without creating the version entry. Each matching tree write MUST produce a version entry. If the tree write has already landed in the location index (i.e., auto-version has been invoked), the version entry MUST eventually land in the DAG. "Bounded retry then give up" violates this invariant — the tree write is already committed and cannot be rolled back by auto-version, so abandoning version creation leaves the tree in a state where the write has happened but no version records it.

"Siblings are valid under DAG semantics" is a claim about the DAG data model, not about the single-head-binding data model currently defined — siblings can only be expressed with a multi-head data model extension (not part of this amendment). Under the current data model, the head is single-valued and contention MUST be handled via one of the two mechanisms above.

**Internal failure handling.** Beyond CAS contention, auto-version's internal operations (`content_store.put` for the version entry, `tree.put` for the head advance) may fail for non-contention reasons — storage I/O errors, disk-full, content-store unavailability. The "for each write, produce a version" invariant still applies: the tree write has landed, so some DAG entry MUST be produced.

Implementations MUST:
- Retry transient failures (I/O, temporary unavailability) with the same unbounded-retry discipline as CAS.
- On persistent infrastructure failure (disk full, permanent content-store failure), return a non-200 status from the auto-version consumer. Per SYSTEM-COMPOSITION.md §2.7, this halts the cascade — subsequent consumers for this event are skipped. The originating tree.put returns 207 with `system/tree/partial-result` naming `"revision/auto-version"` in `consumers_halted`. This is a consistency-vs-availability trade-off: halting the cascade prevents subsequent writes from compounding the inconsistency; the operator addresses the underlying failure via the peer's audit pathway (SYSTEM-COMPOSITION.md §2.7).
- Implementations MUST NOT silently drop the version entry on failure. The per-write invariant is that each tree write has a corresponding DAG entry; violating this would make the version DAG incomplete without any signal to the operator.

**Consumer name.** Auto-version registers with the stable consumer name `"revision/auto-version"` per SYSTEM-COMPOSITION.md §2.7A. Implementations MUST use this name in cascade-halt responses and peer audit.

**Cascade-halt scope.** "Halts the containing emit cascade" has the semantics defined in SYSTEM-COMPOSITION.md §2.7: subsequent Phase 1 consumers for this event are skipped; the binding update is NOT reversed; the originating tree.put returns status 207 with a `system/tree/partial-result` identifying the halting consumer. In-flight events at other cascade depths are unaffected. See SYSTEM-COMPOSITION.md §2.7-2.7A for the full model.

**Originating tree.put return semantics.** The tree write has already landed in the location index by the time auto-version fires (auto-version is a position-7 consumer, after the write completes). When auto-version's emit consumer returns a non-200 status, the originating tree.put returns status 207 (Multi-Status) with a `system/tree/partial-result` (SYSTEM-COMPOSITION.md §2.7A). The `consumers_halted` field names `"revision/auto-version"` and carries the error detail. Callers can distinguish:

- **200** — tree write landed AND all cascade consumers (including auto-version) completed successfully.
- **207** — tree write landed, but the cascade halted. `consumers_halted` identifies which consumer(s) halted and why. If `"revision/auto-version"` is in the halted list, the tree write has no corresponding DAG entry.
- **4xx/5xx** — tree write was rejected (binding did not land).

**Multi-parent versions.** Auto-version creates single-parent version entries only. Multi-parent versions (merges) are produced exclusively by the `merge` operation (§4.4.4).

A merge of N non-no-op paths, under per-write auto-version, produces **one multi-parent commit (V_merge) plus N single-parent intermediate commits** capturing partial-merge states. Trace:

1. Merge op creates V_merge with `parents=[A_head, B_head]` and `root=R_new`, advances head to V_merge.
2. Merge op applies N path writes to realize R_new in the location index. Each write fires the auto-version consumer.
3. At each write i (including the final one): the tracked root reflects partial state R_i. Dedup check compares `current_head.data.root` against the new tracked root. At write 1, `current_head = V_merge`, `V_merge.root = R_new`, new tracked root = R_1 (partial). `R_new == R_1`? No → create V_1 with `parents=[V_merge]`, advance head. At write i>1, `current_head = V_{i-1}` with `root = R_{i-1}` (not R_new), new tracked root = R_i. `R_{i-1} == R_i`? No → create V_i. This continues through write N.
4. After write N completes, `tracked_root = R_new` and `current_head = V_N` with `V_N.root = R_new`.

Result: head-reachable chain is V_merge → V_1 → V_2 → ... → V_N. V_N has the same root as V_merge (both = R_new) but a different hash (different parents — V_N has one parent V_{N-1}, V_merge has two parents). This is a cosmetic duplicate at the tail of the chain representing the final settled state, and is intentional under the CRDT contract — intermediate states are real states the tree passed through and are preserved in the DAG. Peers converge correctly under exchange; the parent chains are all valid.

**Dedup does not eliminate the final intermediate.** The dedup check suppresses a version entry only when the tree write produced no change to the tracked root (i.e., a no-op write that tree-level no-op suppression in SYSTEM-COMPOSITION.md §1.3 would typically catch before the emit event fires). It does NOT fire on the last write of a merge, because the current head at that moment (V_{N-1}) has root = R_{N-1}, not R_new — so the check sees a real state change.

Presentation layers that want to collapse merge-intermediate chains for display MAY do so by walking the parent chain from V_N back to a multi-parent ancestor V_merge; the structural filter is "linear single-parent descendants of a multi-parent version produced during one merge operation, terminating in a descendant whose root matches the multi-parent ancestor's root." This is a display concern, not a DAG-structure concern.

Implementations concerned about DAG growth from large merges may (a) use the `merge` operation only when not under auto-version for that prefix, or (b) disable auto-version for prefixes with frequent large-scale merges and rely on manual commits.

**Emit ordering.** The auto-version consumer reads `system/tree/root/{prefix}`, which is maintained by the structural-summaries consumer at SYSTEM-COMPOSITION.md §2.2 position 6. Auto-version MUST fire after position 6; reading the tracked root before position 6 runs yields the pre-write root paired with a post-write head pointer, producing inconsistent version entries.

Within the emit pipeline, auto-version is assigned dedicated position 7, and subscription shifts to position 8. The ordering matters: subscribers listening to `system/revision/head/**` or to paths under tracked prefixes would see ambiguous observations if subscription fired before auto-version — a head change without a corresponding DAG entry yet visible, or a path change with stale head. Placing auto-version strictly before subscription eliminates this race: when subscription fires at position 8, the version entry already exists in content store and head has already been advanced.

Implementations MUST NOT register auto-version at positions ≤ 6 or at the same position as subscription.

**Trie root tracking coordination.** This is a precondition, not a best-effort coordination: the algorithm above reads the tracked root via `current_tracked_root(prefix)` from `system/tree/root/{prefix}`, which is populated only when a `system/tree/tracking-config` exists for the prefix with `enabled: true` (EXTENSION-TREE.md §3.4.1a).

The revision extension MUST ensure a tracking config exists whenever auto-version is enabled for a prefix, and MUST remove or disable the tracking config when auto-version is disabled. Implementations MUST enforce this coordination through the `revision/config` operation (§4.4.17). The operation writes the tracking-config before the revision config when enabling auto-version, and removes the tracking-config after writing the revision config when disabling. See §4.4.17 for the write-ordering rationale.

This is sequenced coordination, not ACID-atomic: the tree layer has no multi-path atomic primitive. The `revision/config` operation orders the writes to minimize the inconsistent window — enable the tracking config FIRST (so reads during the window find a populated tracked root, even if auto-version hasn't been enabled yet), disable the revision config LAST (so auto-version remains disabled once tracking is removed).

Without the tracking config, `current_tracked_root(prefix)` has no binding to read — the algorithm cannot correctly produce version entries. Implementations MUST NOT silently fall back to O(N) full trie rebuilds per write (prohibitively expensive and masks misconfiguration). Implementations **MUST** return an error from auto-version's emit consumer if the coordination invariant is violated (auto-version enabled for a prefix but tracking-config absent or disabled). The consumer returns non-200, halting the cascade per SYSTEM-COMPOSITION.md §2.7. This is a defense-in-depth check: when config writes route through `revision/config` (§4.4.17), the invariant is maintained by construction. The emit-consumer check catches configuration drift from direct tree.put bypass (capability misconfiguration, migration, operator override). Silent degradation — either to no-op or to full-trie rebuild — is non-conformant.

**No chain-completion hook.** Prior drafts referenced an `on_chain_complete` mechanism. That mechanism is not defined in the spec set and is not required by this algorithm. Auto-version fires per tree-write event, inheriting the standard emit pathway lifecycle. No new hook, consumer class, or composition primitive is introduced.

### 6.2 Debounce Interaction

Auto-version fires per tree-write event. It does not itself debounce. Controlling version creation frequency is the responsibility of whatever upstream mechanism produces the tree writes.

**File watcher debounce.** When a file watcher (e.g., DOMAIN-LOCAL-FILES.md §6) produces tree writes, the watcher's own debounce window controls how many tree writes a burst of filesystem events produces. If the watcher coalesces a 2-second burst of saves into one tree write, auto-version creates one version. If the watcher produces one write per save, auto-version creates one version per save.

**Rate limiting.** Implementations concerned about version DAG growth under high write throughput MAY add a rate limiter at the tree write layer, at the auto-version consumer layer, or at a higher-level batching handler (see §6.5 for the higher-layer transactional pattern). This is an operational concern, not part of the auto-version contract.

**Users wanting coarse checkpoints** should use manual `revision/commit` rather than auto-version. Auto-version's contract is per-write; coarse commits are the explicit-commit path. The two modes do not blend — enabling `auto_version: true` commits the prefix to CRDT-style per-write entries regardless of whether callers also issue manual commits.

**`revision/commit` called while auto-version is ON.** The two operations both produce version entries with `root = current trie root, parents = [current_head]`. Under auto-version ON, the current head already reflects the latest write (because auto-version fired on it). Calling `commit` immediately after creates a redundant entry: same root as current_head (the auto-version entry), different parents (commit's has `[current_head]` where current_head IS the auto-version entry). The redundant entry is valid under CRDT but adds no information.

Implementations MUST apply a dedup check in `revision/commit`: if `current_head.data.root == freshly_computed_trie_root`, return `current_head` without creating a new entry. This mirrors the dedup in auto-version's algorithm (§6.1) and prevents trivial no-op commits under auto-version. Callers wanting a "mark this moment" entry distinct from the latest auto-version entry should add metadata via a higher layer (e.g., emit a named-checkpoint entity at a separate path) — modifying commit's entry shape to distinguish user-vs-auto commits is not part of this amendment.

### 6.3 Following another peer's versions (informative)

The revision extension does not manage cross-peer synchronization. Peers that want to follow another peer's version DAG use the subscription + continuation composition described here. This section is informative; implementations are free to compose differently.

**Recommended pattern.** Peer B wants to follow Peer A's `/project/` versions:

1. B creates a subscription on A at `system/revision/{H}/head` (where `{H}` is the prefix hash for A's project prefix) with delivery to B's inbox at `system/inbox/follow-a-project`.
2. A's subscription consumer fires on every head advance (emit position 8 per SYSTEM-COMPOSITION.md §2.2). Notifications land in B's inbox.
3. A standing continuation on B's inbox dispatches `revision/fetch` against A to pull the new version entry and trie nodes into B's content store.
4. After fetch, the continuation dispatches `revision/merge` (or equivalent) against B's local DAG to integrate the new version. B's head advances per its own `merge_order` setting.

Under this composition:

- B decides what it tracks (the subscription's `pattern`) and at what granularity.
- B absorbs its own backpressure — if B is overwhelmed, it can unsubscribe, rate-limit the continuation, or drop to periodic manual fetch.
- A doesn't know or care who's watching. Subscription's bounded-fanout dissemination trees (EXTENSION-SUBSCRIPTION v3.5) handle any number of followers.
- No revision-extension knob controls the propagation rate — the subscription and continuation configs do.

**Bidirectional sync.** If A and B both want to follow each other, each peer sets up its own subscription. The content-addressed DAG ensures convergence: each peer fetches what it doesn't have, merges per its local strategy, and advances head. Loops terminate because re-fetching a version already in the content store is a no-op (content-addressed dedup). See the user guide for worked examples.

**One-shot sync.** For CI pipelines, manual catchup, or scheduled sync, operators call `revision/push` or `revision/fetch` directly. No standing subscription needed.

### 6.4 Interaction with History Extension

When both history and auto-versioning are enabled for a path, each tree write produces:

1. One history transition (per-path, with execution context).
2. One version entry (per-write, with root and parents).

History and auto-version grow at the same per-write rate. Neither supplants the other — history provides per-path detail with full execution context (capability, author, chain_id, clock); the version DAG provides content-addressed multi-path checkpoints suitable for cross-peer exchange and merge.

Prior drafts described history as "per-write detail" and the version DAG as "multi-path checkpoint," implying different granularities. Under the CRDT contract, both are per-write. The distinction is dimensional: history records the transition (what changed at a path); the version DAG records the resulting state (what the whole prefix looks like now, hash-addressable).

### 6.5 Higher-layer batch-complete pattern (informative)

Handlers that perform multi-path batch operations (merge, revert, cherry-pick, compute cascade) can signal batch completion to interested subscribers using existing primitives — a completion entity written to a convention path (e.g., `system/tree/batch-complete/{chain_id}`). Batch-aware subscribers observe the completion entity and take coarse actions; per-write consumers continue firing during the batch.

The batch-complete pattern is **additive, not substitutive**. Operating modes are effectively binary: (a) per-write CRDT auto-version ON — fine-grained DAG, every change captured, intended for collaborative editing; or (b) auto-version OFF and manual/batch commits only — coarse-grained DAG, user controls granularity. A user wanting "only batch-scoped versions with no per-write intermediates" MUST disable auto-version on the tracked prefix and rely exclusively on explicit `revision/commit` plus the batch-complete subscriber. The two granularities do not blend within a single prefix.

This pattern is NOT normative. Implementations MAY adopt a different convention or omit batch signaling entirely.

### 6.6 Deletion-marker retention (v3.1, Amendment 5)

Deletion markers in version tries are pruned when their containing versions fall out of standard revision retention (per `system/revision/config` and EXTENSION-HISTORY's `max_depth`). **No separate deletion-marker compaction operation is required or specified — retention at the version level handles deletion-marker storage growth uniformly.**

Implementations MUST NOT attempt to compact or rewrite deletion-marker entries within a retained version's trie. Tries are content-addressed; rewriting trie nodes changes the version's root hash and breaks DAG ancestry. **The only valid mechanism for removing deletion markers from storage is pruning the containing version.**

The live location index is NEVER affected by deletion-marker retention — deletion markers never appear there (per §4.4.4 Amendment 3 apply translation).

---

## 7. Transfer Protocol

### 7.1 Phases

Sync between two peers proceeds in four phases:

**Phase 1: Status Exchange.** Peers exchange version heads for prefixes of interest.

```
Peer A → Peer B: status(prefix: "project/")
Peer B → Peer A: {head: V_b, ...}
Peer A → Peer B: {head: V_a, ...}
```

**Phase 2: Relationship Check.** Each peer determines the relationship between its head and the remote's head (in_sync, ahead, behind, diverged).

**Phase 3: Transfer.** Transfer has two parts, both mediated by the revision handler:

- **Version DAG transfer.** The peer with missing versions uses `fetch` (§4.4.6) to download version entries and root trie nodes from the remote DAG. This is lightweight — version entries are two fields (root + sorted parents).
- **Entity transfer.** For integration (merge, fast-forward, checkout), the actual entities referenced by the version's trie root must be available locally. The client uses `fetch-entities` (§4.4.7) to incrementally walk the trie and retrieve missing entities, validated against the version DAG. Content MAY be fetched lazily — a peer can hold version entries (understanding DAG structure) without immediately fetching all referenced content. Content is transferred when needed for integration or application access. No content handler dependency — the revision handler mediates all transfer.

**Phase 4: Integration.** Fast-forward updates the head pointer. Diverged states invoke the merge operation. Auto-versioning configurations may trigger push of the merge result.

### 7.2 Convergence

The version DAG provides convergence guarantees that subscriptions alone cannot:

- **Content-addressed versions**: if both peers have the same version hash, they're converged. No further sync needed.
- **Ancestor detection**: if the received version is an ancestor of the local head, it's already integrated. No action needed.
- **Merge creates a descendant**: the merge version has both divergent versions as parents. The next sync sees a fast-forward opportunity and converges.

**Non-concurrent case.** When only one side has new changes, sync converges in a single round: the behind peer fast-forwards.

**Concurrent merges converge in O(1).** With structural version entries (root + sorted parents), concurrent merges of the same inputs produce the exact same version entry hash. The second-order divergence that previously required an additional merge round does not occur:

```
1. A commits V_a (parent V0). B commits V_b (parent V0).
2. Both peers follow each other via the §6.3 composition and exchange concurrently.
3. A merges → V_m = {root: trie_merged, parents: sorted([V_a, V_b])}
4. B merges → V_m = {root: trie_merged, parents: sorted([V_a, V_b])}
5. V_m is the same entity on both peers (same hash). check_relationship returns in_sync.
6. Converged. No additional round needed.
```

This works because the version entry contains only deterministic content: the trie root hash (deterministic merge of the same inputs) and sorted parent hashes. No author, timestamp, or message fields differ between peers.

**Convergence is guaranteed** because:
1. Structural entries eliminate second-order divergence — same merge inputs produce the same version hash
2. Each merge round produces a version that is a descendant of all input versions
3. The number of distinct heads across the cluster is monotonically non-increasing
4. Content addressing provides O(1) convergence detection (`check_relationship` compares hashes)

**Asymmetric strategy caveat.** When asymmetric merge strategies (`source-wins`, `target-wins`) are used with `caller-perspective` ordering, different peers may produce different merge results (different trie roots). This is the one case where structural entries do not eliminate divergence. Use `deterministic` merge ordering (§4.4.4) for full convergence in p2p topologies with asymmetric strategies. Oscillation detection (§4.4.4) catches and halts any cycling that occurs.

### 7.3 Cross-Prefix Import

Version entries are location-independent — the same version entry can be used at any prefix. To import content from one prefix into a different location (analogous to cloning a git repository to a new path):

1. Retrieve the version's trie root from the content store
2. Use tree merge with `source_prefix` / `target_prefix` to place the content at the new location (tree merge is additive — it does not delete existing content at the target)
3. Set the head pointer at the new prefix to the imported version hash — the version entry is shared, only the head pointer path differs
4. Subsequent commits at the new prefix share the DAG with the original

This is the git model: the commit hash does not include the working directory path. The same version entry, the same DAG, different mount points.

---

## 8. Operational Notes

### 8.1 Concurrent Commits

Version-creating operations (commit, merge, cherry-pick, revert, auto-version) that run concurrently against the same prefix contend on `system/revision/{H}/head`. Implementations MUST use one of the mechanisms specified in §6.1 "Contention handling" (CAS+retry, single-writer serialization per prefix, or equivalent) to satisfy the no-orphan invariant: every version entry created MUST remain reachable by parent-pointer traversal from some head pointer.

Implementations that allow concurrent head advances to overwrite each other — orphaning the losing writer's version entry — are non-conformant.

The version hash disambiguates concurrent lineage: even after CAS retry, the retrying writer creates a version entry with the newly-advanced head as parent, so both operations' work lands in the DAG. Implementations MAY surface CAS retries in observability but are not required to.

### 8.2 Nested Prefixes

If `project/` is versioned and `project/frontend/` is independently versioned, their version DAGs are independent. A commit to `project/frontend/` does NOT automatically create a version for `project/`. The parent prefix's trie is only captured when someone explicitly commits at `project/`.

This is the expected behavior — different prefixes represent different versioning granularity. A change under a child prefix makes the parent's trie stale, but staleness is resolved at commit time (the next commit to `project/` captures the current tree state, including any changes under `project/frontend/`).

### 8.3 Detached HEAD

Checking out a specific version (not a branch) results in a detached HEAD — `system/revision/{H}/active-branch` has no value, and the head points to the checked-out version. Subsequent commits create versions that no branch tracks.

The revision handler SHOULD warn (but MUST NOT refuse) when committing in a detached HEAD state. Implementations MAY automatically create a branch from a detached HEAD commit to prevent orphaned versions.

### 8.4 Resolve Without Commit

The resolve operation (§4.4.5) modifies the tree (places the resolved entity, removes the conflict entry) but does NOT create a new version. Resolved-but-uncommitted state is working copy — it is not captured in the version DAG until a commit. If the peer crashes after resolving but before committing, the resolved entities are in the tree but no version records the resolution.

### 8.5 LWW and Clock Skew

The `lww` (last-write-wins) merge strategy compares timestamps from metadata entities in the trie (if the application convention includes them) or timestamps from the entity data itself (if the conflicting entities have timestamp fields). With structural version entries, timestamps are no longer in the version entry. When no timestamp is available, LWW falls through to the next cascade step. Deployments that configure LWW should ensure their entities or tree conventions include comparable timestamps. For multi-peer scenarios where ordering matters, applications should use a domain-specific merge handler with logical clocks or vector clocks rather than relying on `lww`.

### 8.6 Version Retention

Version entities are stored in the content store and subject to normal GC policies. Version entities reachable from any head, branch, tag, or remote head pointer MUST NOT be garbage collected. Version entities not reachable from any pointer MAY be collected.

Long-running auto-versioned prefixes accumulate version entities over time. The content-addressed DAG model means that versions cannot be "compacted" (changing parents would change the hash, invalidating all descendants). Storage pressure is addressed by:

- Shallow sync (§4.4.6 `depth` parameter) — new peers fetch only recent history
- GC of unreachable versions (from deleted branches or superseded merges)
- Implementation-specific retention policies

Detailed GC policies are implementation-specific and outside the scope of this extension.

### 8.7 Versioning Scope in Multi-Peer Environments

#### 8.7.1 The Universal Namespace

The entity tree is a universal namespace: `/{peer_id}/{path}`. A peer's tree contains its own namespace (where it has authority) and local copies of other peers' namespaces (acquired through sync, subscription, or direct fetch).

Versioning applies to prefixes within this namespace. The choice of prefix determines the merge characteristics and operational complexity.

#### 8.7.2 Recommended Versioning Scopes

| Scope | Prefix example | Characteristics |
|-------|---------------|----------------|
| Own namespace | `/{self}/` | Local authority. Linear history unless collaborating. |
| Single foreign namespace | `/{peerD}/` | Tracks local view of one peer's data. May aggregate from multiple sources. |
| Domain subtree | `/{self}/project/` | Focused versioning for a specific concern. |
| Full tree | `"/"` | Captures entire tree state. Requires `system/revision/**` exclude. |

**Per-peer-namespace versioning is the recommended default.** Each peer namespace has its own version DAG. Version metadata is naturally scoped — the head pointer for `/{peerD}/` lives at `/{self}/system/revision/{H}/head` (where `{H}` is the hash of the prefix `/{peerD}/`), outside the versioned prefix.

**Full-tree versioning** is appropriate for milestone snapshots (explicit commits at known-good states) but not recommended for continuous auto-versioning due to reentrancy risk (§6.1 exclude requirement) and cross-namespace merge complexity.

Note: with the trie snapshot format (EXTENSION-TREE.md §3), snapshot creation cost is O(changed_paths * depth) regardless of prefix size. The cost concern that previously applied to large prefixes is resolved. The scope guidance now focuses on merge semantics, not snapshot size.

#### 8.7.3 Independent DAG Merges

When a peer acquires versioned data about a namespace from two sources with independent version histories (no common ancestor), the first merge produces conflicts on every differing path. This is a one-time cost — the merge version permanently grafts the two DAGs together, providing a common ancestor for all subsequent syncs.

Strategies to reduce the graft cost:

- **CRDT merge strategies** (union merge, LWW) for the no-ancestor case.
- **Syncing from the authoritative source first.** If all peers sync from the namespace owner, subsequent cross-peer syncs share a common ancestor.

#### 8.7.4 Raw State Transfer (No Versioning)

Use the tree extension's `extract` and `merge` operations directly for bulk state transfer without version tracking. Appropriate for initial population before establishing version-tracked sync.

---

## 9. Capability Model

### 9.1 Version Handler Capability

Version operations require a grant covering `system/revision` with the requested operation:

```
{handlers: {include: ["system/revision"]}, operations: {include: ["commit", "log", "status"]}}
```

### 9.2 Prefix Scoping

Commit, log, and status operations access tree state at the requested prefix. The caller needs read capability for the prefix (for log/status) or write capability (for commit, merge, resolve).

### 9.3 View Tree Versioning

The revision extension operates on view trees (tree extension §8). A peer syncing with a remote only sees paths their capability allows. Snapshots, diffs, and merges are computed on the view tree — out-of-scope paths are invisible.

This provides natural sync boundaries:
- Grant Peer B capability for `project/docs/*` → B only syncs docs, not source code
- Grant Peer C capability for `config/public/*` → C only syncs public config, not secrets

### 9.4 Cross-Peer Authorization

Fetch, pull, and push operations require:
1. Local revision handler capability
2. Remote connection capability (from the connection/capability exchange)
3. Remote tree read/write capability (validated by the remote peer)

### 9.5 Write Authorization

Revision operations produce tree writes in two authorization modes. Writes to the caller's data namespace use the caller's capability. Writes to revision metadata paths use the handler's own grant.

| Operation | Write target | Authorization | Capability recorded |
|-----------|-------------|---------------|-------------------|
| commit (version entry) | content store | Handler grant | Handler grant |
| commit (head pointer) | `system/revision/{H}/head` | Handler grant | Handler grant |
| merge (bindings) | `{prefix}/{path}` per binding | Caller capability | Caller capability |
| merge (head pointer) | `system/revision/{H}/head` | Handler grant | Handler grant |
| merge (conflicts) | `system/revision/{H}/conflicts/{path}` | Handler grant | Handler grant |
| checkout (bindings) | `{prefix}/{path}` per restored binding | Caller capability | Caller capability |
| checkout (head pointer) | `system/revision/{H}/head` | Handler grant | Handler grant |
| branch | `system/revision/{H}/branches/{name}` | Handler grant | Handler grant |
| tag | `system/revision/{H}/tags/{name}` | Handler grant | Handler grant |
| push (local cache update) | `{prefix}/{path}` per binding (local tree) | Handler grant | Handler grant |
| resolve | `system/revision/{H}/conflicts/{path}` | Handler grant | Handler grant |
| cherry-pick (bindings) | `{prefix}/{path}` per applied binding | Caller capability | Caller capability |
| revert (bindings) | `{prefix}/{path}` per reverted binding | Caller capability | Caller capability |

**Caller-authorized writes** (binding writes to data namespace): The handler MUST verify the caller's capability covers each write path using `check_path_permission("put", path, capability)`. If any path is denied, the entire operation MUST fail — no partial writes (§4.4).

**Handler-authorized writes** (revision metadata at `system/revision/*`): The handler's own grant covers its managed namespace. These writes cannot fail authorization — the handler grant is issued at registration with scope covering `system/revision/*`.

See ENTITY-CORE-PROTOCOL.md §6.8 for the general write authorization model. See EXTENSION-TREE.md §11 for the tree handler's equivalent documentation.

---

## 10. External System Bridge

### 10.1 Git Mapping

The entity version model maps directly to Git's data model:

| Entity Concept | Git Equivalent |
|---------------|----------------|
| `system/revision/entry` | Git commit |
| `system/tree/snapshot` | Git tree (flattened) |
| Entity in content store | Git blob |
| Version parents | Commit parents |
| `system/revision/{H}/head` | Git ref (branch) |
| Version hash | Commit SHA |

**Import from Git:**
1. Walk Git commit DAG
2. For each commit: flatten the Git tree into path → hash bindings → trie root
3. Create version entity with trie root hash, sorted parents (mapped)
4. Store author, timestamp, message as metadata entities at application convention paths (e.g., `.meta/{version_hash}/{peer_id}/`)
5. Store in entity content store

**Export to Git:**
1. Walk entity version DAG
2. For each version: expand trie root bindings into Git tree objects (reconstruct hierarchy)
3. Read metadata from trie content (application convention) for author, timestamp, message
4. Create Git commit with tree, parents (mapped), author, timestamp, message
5. Push to Git repository

The translation is mechanical. The git commit tree hash maps to `root`. Author, timestamp, message are tree content (application convention), not version entry fields. Entity types (typed payloads) vs Git blobs (raw bytes) is the encoding question — the bridge handler decides how to serialize/deserialize entity data.

### 10.2 Bridge Handler Pattern

A Git bridge handler at `app/bridge/git` could provide:

| Operation | Description |
|-----------|-------------|
| `import` | Import Git repository into entity tree |
| `export` | Export entity version history to Git |
| `sync` | Bidirectional sync with a Git remote |

The bridge handler uses the revision extension's version DAG and the tree extension's snapshots. It translates between Git's hierarchical tree objects and entity's flat snapshot bindings.

---

## 11. Conformance

### 11.1 Conformance Levels

**Full conformance** requires all core operations (§4.2) and all convenience operations (§4.2).

**Core conformance** requires all core operations only. Implementations at core conformance level SHOULD document which convenience operations they provide through alternative interfaces.

The revision handler interface MUST declare which conformance level is implemented.

### 11.2 Core Requirements

| Requirement | Level |
|-------------|-------|
| Store version entities in content store | MUST |
| Store version heads at `system/revision/{H}/head` (hash-addressed, §3.1) | MUST |
| Version entries contain `root` and `parents` only | MUST |
| `parents` array sorted by hash value (lexicographic binary comparison) | MUST |
| Support core operations: `commit`, `log`, `status`, `merge`, `resolve`, `fetch`, `fetch-entities` | MUST |
| Implement three-way merge as default strategy | MUST |
| Support type-aware merge cascade (§5.1) | MUST |
| Support both `merge_order` modes (`caller-perspective` and `deterministic`) | MUST |
| Oscillation detection before committing merge versions | MUST |
| Support per-path merge configuration | SHOULD |
| Support per-type merge configuration | SHOULD |
| Support custom merge handler delegation | SHOULD |
| Compute prefix hash via `content_hash("system/tree/path", absolute_prefix)` | MUST |
| Resolve prefix to absolute via standard path resolution (ENTITY-CORE-PROTOCOL.md §5.4) in all operations | MUST |
| Content-identity check in merge (defense-in-depth) | SHOULD |
| Store conflicts at `system/revision/{H}/conflicts/` (side-channel, not at original path) | MUST |
| Original entity MUST remain at its path during conflict | MUST |
| Support `exclude` patterns in version configuration | SHOULD |
| Structural reentrancy exclusion via `exclude` config when auto_version enabled | MUST |
| Required-exclude patterns present (`system/revision/**`, `system/tree/root/**`, `system/tree/tracking-config/**`, `system/history/**`, `system/clock/**`) when tracked prefix encompasses those paths | MUST |
| Reject at config-write time any `auto_version: true` config missing the required excludes | MUST |
| Tracking-config coordination (create/enable when auto_version on; remove/disable when off) | MUST |
| Error from auto-version emit consumer when tracking-config is absent or disabled | MUST |
| Contention handling via CAS+retry unbounded, single-writer serialization per prefix, or equivalent | MUST |
| Support auto-versioning | MAY |
| Auto-version registered at SYSTEM-COMPOSITION.md §2.2 position 7 (not ≤ 6, not same as subscription) | MUST |
| Multi-path operations (merge, checkout, cherry-pick, revert) advance head BEFORE applying bindings | MUST |
| Default `merge_order` is `"deterministic"` | MUST |
| Custom merge handlers commutative OR restricted to `merge_order: "deterministic"` | MUST |
| Support `dry_run` on merge | SHOULD |
| Support `depth` parameter on fetch | SHOULD |
| Validate `fetch-entities` hashes against version trie | MUST |
| Version entities reachable from any ref MUST NOT be garbage collected | MUST |
| Tags are immutable — create MUST fail if tag already exists | MUST |
| Warn about detached HEAD on commit | MAY |

**Cascade degradation.** Implementations that do not support per-path or per-type merge configuration (steps 1-2 of §5.1) degrade gracefully: the cascade falls through to the default three-way merge (step 3) and conflict entity creation (step 4) for all conflicts. All merge strategies — including `keep-both`, `source-wins`, `target-wins`, `lww`, and `manual` — remain available via the `strategy` field on `system/revision/merge-params` (§4.4.4), which takes precedence over the cascade. Implementations MUST accept the `strategy` override regardless of which cascade steps they support. Implementations that skip cascade steps SHOULD document which steps are not supported.

### 11.3 Convenience Requirements

| Requirement | Level |
|-------------|-------|
| Support `push` operation | SHOULD |
| Support `branch` operation (create/list/delete) | SHOULD |
| Support `checkout` operation | SHOULD |
| Warn about uncommitted changes on checkout | MAY |
| Support `tag` operation (create/list/delete) | SHOULD |
| Support `diff` between versions | SHOULD |
| Support `cherry-pick` operation | SHOULD |
| Support `revert` operation | SHOULD |
| Support `pull` operation (fetch + fetch-entities + merge) | SHOULD |
| Support `find-ancestor` operation | SHOULD |

### 11.4 Types Installed

```
system/revision/entry
system/revision/conflict
system/revision/merge-config
system/revision/config
system/revision/status
system/revision/commit-params
system/revision/commit-result
system/revision/log-params
system/revision/log-result
system/revision/merge-params
system/revision/merge-result
system/revision/merge-request
system/revision/merge-response
system/revision/resolve-params
system/revision/resolve-result
system/revision/fetch-params
system/revision/fetch-result
system/revision/fetch-entities-params
system/revision/fetch-entities-result
system/revision/push-params
system/revision/push-result
system/revision/ancestor-params
system/revision/ancestor-result
system/revision/status-params
system/revision/branch-params
system/revision/branch-result
system/revision/checkout-params
system/revision/checkout-result
system/revision/tag-params
system/revision/tag-result
system/revision/diff-params
system/revision/cherry-pick-params
system/revision/cherry-pick-result
system/revision/revert-params
system/revision/revert-result
```

### 11.5 Handler Registered

```
system/handler/system/revision := system/handler {
  pattern:    "system/revision"
  name:       "version"
  operations: {
    ; Core (MUST)
    commit, log, status, merge, resolve,
    fetch, fetch-entities,
    ; Convenience (SHOULD)
    push, pull, find-ancestor,
    branch, checkout, tag, diff,
    cherry-pick, revert
  }
  ; See §4.1 for full operation-spec map with input/output types
}
```

---

## 12. Privacy + cross-peer observability

Per `GUIDE-INSPECTABILITY.md` v1.2 §9 #4:
- Version entries (`system/revision/entry`), head pointer (`system/revision/{H}/head`), active-branch pointer (`system/revision/{H}/active-branch`), branches (`system/revision/{H}/branches/{name}`), and tags (`system/revision/{H}/tags/{name}`) are **public-by-convention** — version entities are content-addressed and designed to travel cross-peer; the trie content they reference defaults to public unless the application overrides per its own §9 #4 declarations.
- Per-prefix `remotes` (`system/revision/{H}/remotes/{peer_id}`), `config` (`system/revision/{H}/config`), and `conflicts` (`system/revision/{H}/conflicts/{path}`) are **capability-controlled** — collaboration topology, operator configuration, and merge-failure state respectively.
- Global merge configuration at `system/revision/config/merge/path/{name}` and `system/revision/config/merge/type/{type_name}` is **capability-controlled** (handler-owned namespace per §2.3 v3.3 D1).
- Conflict side-channel entities (`system/revision/conflict`) are **capability-controlled** — bodies carry base/local/remote hashes + attempted strategy.

Per §9 #7:
- **Convergent:** version entities + trie state under the prefix + `head`/`active-branch`/`branches/*`/`tags/*` pointers propagate cross-peer via the revision transfer protocol (§7) and via cross-peer subscriptions on `system/revision/{H}/**`. Convergence model: `revision:fetch-diff` (§4.4.19) + `tree:merge` with CAS pinning (§3.4 divergence detection + §5 merge strategy framework + ENTITY-CORE-PROTOCOL.md §3.9 CAS).
- **Local-namespace:** `system/revision/{H}/conflicts/{path}` (already declared peer-local in §2.2: "Conflicts are peer-local. They are NOT included in version snapshots and are NOT synced to other peers." — this subsection lifts that to the formal §9 #7 declaration position); `system/revision/{H}/remotes/{peer_id}` (collaboration-topology metadata); `system/revision/{H}/config` (per-prefix operator configuration); `system/revision/config/merge/**` (global merge configuration). These MUST NOT be carried in version snapshots; subscription-based propagation MUST be refused at the subscription handler (subscription pattern `system/revision/**` matches MUST explicitly carve out these subpaths, or grant scope MUST enumerate the included subpaths).

Chain-error markers bound on `revision:fetch-diff` chain-dispatch failures per §4.4.19 are themselves **local-namespace** per EXTENSION-CONTINUATION.md §6.5 (the canonical home for chain-error marker locality).
