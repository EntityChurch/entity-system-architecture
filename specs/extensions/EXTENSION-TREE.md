# System Tree Extension

**Version**: 4.0.2

**Status**: Active
**Depends**: ENTITY-CORE-PROTOCOL.md (v7.3+)
**Encoding**: ENTITY-CBOR-ENCODING.md (ECF)

---

> **Path notation.** Paths in this document use peer-relative notation (without leading `/{peer_id}/`). All peer-relative paths resolve to the local peer's namespace: `system/tree` means `/{local_peer_id}/system/tree`. Every path in the entity tree is absolute at rest — rooted at a peer identity. See ENTITY-CORE-PROTOCOL.md §1.4 for the path model. Cross-peer examples use absolute paths with explicit peer identities.

## 1. Overview

This extension builds on the entity tree defined in ENTITY-CORE-PROTOCOL.md §1.7 and §6.3. Core protocol provides the tree handler with two operations (`get` and `put`) and optional `tree_id` for non-default trees. `get` reads entities or lists entries (trailing-slash path); `put` stores entities or removes bindings (null entity). This extension provides operations to work with trees as units: capturing state, comparing states, combining states, managing multiple trees, and scoping tree access for handlers.

### 1.1 Scope

This extension covers:

- Snapshots — content-addressable captures of tree state
- Diffs — structured comparison between two states
- Merges — combining one state into another
- Extraction — bundling a subtree as a transferable envelope
- Non-default trees — creating and destroying tree instances
- View trees — capability-scoped projections for handler isolation

This extension does **not** cover placement rules (how entity types map to tree paths). Placement/convention systems build on top of tree operations and are specified separately.

### 1.2 Three Structural Representations

The entity system has three ways to represent collections of entities:

**Entity** — the atomic unit. `{type, data}` with content hash.

**Envelope** — a flat collection. `{root, included: {hash → entity}}`. No structure beyond "root + supporting entities." The wire transport format.

**Tree** — a structured collection. `{bindings: {path → hash}}`. Entities have locations determined by the tree's organization. The naming layer.

Tree operations bridge between these representations:
- **Extract**: tree → envelope (snapshot a subtree, bundle its entities)
- **Merge**: snapshot → tree (apply captured state into a tree)
- Core protocol `get`/`put`: entity ↔ tree (single entity at a time)

---

## 2. The Tree

### 2.1 Definition

A tree consists of:

```
tree = {
  bindings:        path → hash          // the location index
  root_structure:  peer-namespaced | relaxed
  context:         { namespace }        // whose perspective
}
```

**Bindings** are the tree's data — a flat mutable map from paths (strings) to content hashes. A path may be simultaneously bound to an entity and serve as a prefix for child paths (ENTITY-CORE-PROTOCOL.md §1.7). For example, `system/type/system/handler` is bound to a type definition entity while `system/type/system/handler/operation-spec` exists as a child path. The listing response reflects both dimensions independently: `hash` indicates entity binding, `has_children` indicates child path existence. These are not mutually exclusive — implementations MUST NOT impose file-or-directory constraints on the location index.

**Root structure** determines path organization. Core protocol trees are `peer-namespaced` — paths begin with a peer_id. Relaxed trees (for internal computation, staging) may use different root conventions.

**Context** identifies the tree's perspective — whose namespace this tree represents.

### 2.2 The Two Index Operations

The location index provides two primitive operations:

| Operation | Signature | Description |
|-----------|-----------|-------------|
| `get` | `(path) → entity? \| listing` | Path ending with `/` or empty: listing. Otherwise: read entity at path. |
| `put` | `(path, entity?) → ()` | Entity present: store and bind. Entity absent/null: remove binding. |

These are defined in ENTITY-CORE-PROTOCOL.md §6.3 as system tree handler operations. Both accept an optional `tree_id` parameter — without it, they target the default tree; with it, they target the specified non-default tree.

Everything this extension provides is composition of these primitives.

`put` is a data-plane primitive. It writes a binding and fires the emit cascade (SYSTEM-COMPOSITION.md §1). It does not go through other handlers, does not validate against other extensions' entity types, and does not perform custom logic. Extensions that need validation, coordination, or any processing before a write lands SHOULD expose a named handler operation (e.g., `revision/config`, `compute/install`) and gate the underlying path via capability grants so callers route through the operation rather than calling `put` directly. See SYSTEM-COMPOSITION.md §2.9 for the rubric on when a named operation is appropriate vs direct `put`.

### 2.3 All Trees Are Trees

There is no fundamental distinction between the default tree and any other tree. A tree is a set of `path → hash` bindings with configuration. The default tree is structurally special only because core protocol says so:

- Handler dispatch operates on it
- The system tree handler targets it by default (when `tree_id` is absent)
- It exists before any extensions

Non-default trees are the same structure. A staging tree, a sub-peer tree, a view tree — all are location indexes with the same two operations. This extension treats them uniformly.

---

## 3. Snapshot

A snapshot captures tree state as a content-addressable entity. Same bindings **MUST** produce the same snapshot on any peer.

### 3.1 Type

```
system/tree/snapshot := {
  fields: {
    root: {type_ref: "system/hash"}           ; hash of the root trie node
  }
}
```

The snapshot is a typed marker around a content-addressed trie root. The trie root hash is the snapshot's identity — same content at any tree location produces the same snapshot hash. The snapshot does not carry location metadata (prefix); location is operational context provided by the operation that creates or consumes the snapshot.

This separation ensures snapshots are purely content-addressed. Two peers with identical content under different prefixes produce the same trie root hash, the same snapshot entity, and the same snapshot content_hash. Versions (revision extension) that reference the same content are structurally comparable regardless of where they were created.

The snapshot entity is small (~35 bytes — root hash). The tree state lives in trie node entities in the content store. The snapshot entity's hash changes when any binding changes (because the root node hash changes through the chain of parent hashes).

```
system/tree/snapshot/node := {
  fields: {
    map:  {type_ref: "primitive/bytes", length: 4}
      ; 32-bit bitmap (4 bytes) of occupied positions in this node, encoded as a
      ; K-bit unsigned integer (LSB-indexed: position p is bit p of the integer)
      ; serialized big-endian (most-significant byte first).
      ; Position 0 → integer 0x00000001 → 4-byte serialization 00 00 00 01.
      ; Position 28 → integer 0x10000000 → 4-byte serialization 10 00 00 00.
    data: {array_of: "Entry"}
      ; dense array of entries; length = popcount(map)
      ; Entry discriminated by CBOR major type at decode time:
      ;   CBOR major type 4 (array)        → Bucket: [[key, value_hash], ...]
      ;                                         length ≤ bucketSize=3, sorted lex by key
      ;   CBOR major type 2 (byte string)  → Link: 33-byte system/hash of a sub-node entity
  }
}
```

A trie node is an IPLD HashMap node (algorithm reference: [go-hamt-ipld v3.4.1](https://github.com/filecoin-project/go-hamt-ipld/tree/v3.4.1); see also [IPLD HashMap spec](https://ipld.io/specs/advanced-data-layouts/hamt/spec/)). Routing is by 5-bit slices of `SHA-256(UTF-8-bytes(canonical-normalize(relative_key)))` per §3.3. The `map` bitmap indicates which of K=32 positions are occupied; `data` is the dense popcount-compressed array of entries at occupied positions. Each entry is either a bucket of up to 3 `[key, value_hash]` tuples (for leaf-level storage) or a link to a sub-node entity (when a bucket overflowed and split). **Wire format is ours (ECF + `system/hash`); we adopt the IPLD HashMap algorithm + parameters as reference only, not byte-wire-compat. See `proposals/implemented/PROPOSAL-TREE-NODE-SHAPE-BOUNDED-FANOUT.md` §6 / §8.4.**

**Parameters (MUST, pinned in spec; not exposed on wire):**

- `bitWidth = 5` → K = 32 buckets per node, bitmap = 4 bytes
- `bucketSize = 3` — maximum tuples per bucket before recursing to a sub-node
- Hash function: SHA-256, input is `UTF-8-bytes(canonical-normalize(relative_key))` per §3.3

Implementations MUST NOT expose these as per-tree configuration. Drift on bitWidth, bucketSize, or hash function produces silently divergent root hashes — the v2-class bug Stage 7 exists to eliminate.

**Canonical form (MUST, per IPLD HashMap spec).** No non-root node may contain, either directly or via links through child nodes, fewer than `bucketSize + 1 = 4` reachable entries. On deletion, when a non-root node would violate this, it MUST be collapsed and its contents inlined into the parent bucket (preserving the bucket-sort invariant). This is the CHAMP-equivalent canonicalization property that guarantees byte-identical-output for the same binding set under arbitrary insert/delete history; without it, two peers building "the same" tree by different histories produce different root hashes and `convergent_mirror` breaks.

**Bucket-sort invariant (MUST).** Within any bucket entry, the `[key, value_hash]` tuples MUST be sorted lex by `key` (UTF-8 lexicographic order), on both insertion and deletion. Per IPLD HashMap spec.

**Determinism** is guaranteed by:

1. Map keys and bucket-tuple keys within a node MUST follow the canonical orderings above (CBOR map key ordering per ECF; bucket tuples lex by key).
2. Hash values are binary `system/hash` — no formatting ambiguity.
3. No timestamp — a snapshot is pure structural data.
4. The canonical-form invariant + bucket-sort invariant MUST be maintained after every operation (insert, delete, merge).
5. Empty trees produce a canonical empty-root node (literal hex below). Empty nodes other than the root MUST NOT exist (collapsed per canonical-form rule).
6. Node hashing uses the **standard ECF entity hash** for `system/tree/snapshot/node` entities, per `ENTITY-CBOR-ENCODING.md` (ECF). Implementations MUST use their existing ECF entity-hash routine — i.e. `SHA-256(canonical-ECF-encoding({type: "system/tree/snapshot/node", data}))`. Implementations MUST NOT implement this as literal string concatenation of the type-name bytes with raw CBOR-encoded `data`; the literal-hex examples below show resulting byte sequences end-to-end, not the algorithm.
7. The root node MAY have fewer than `bucketSize+1` entries (it is exempt from the canonical-form lower bound). All other nodes MUST satisfy the invariant.

Same bindings produce the same trie nodes, which produce the same root hash, which produces the same snapshot content hash. The snapshot's content hash serves as the identity of the tree state. Cross-impl byte-identical-output is the conformance proof point (see §12.1 + conformance fuzzer pattern).

**Literal CBOR encoding of the empty-root node** (no bindings):

```
A2 63 6D6170 44 00000000 64 64617461 80
```

Where `A2` = map(2), `63 6D6170` = text(3) "map", `44 00000000` = bytes(4) zero bitmap, `64 64617461` = text(4) "data", `80` = array(0). 17 bytes. `content_hash` is the standard ECF entity hash of the typed entity `{type: "system/tree/snapshot/node", data: <those 17 bytes interpreted as CBOR>}` per #6 above — computed via the implementation's existing ECF entity-hash routine, not literal string concat. **All implementations MUST produce this exact byte sequence for the empty-root node.** Any deviation breaks cross-peer trie root comparison. This is conformance fixture #1.

**Literal CBOR encoding of a single-binding root node** (one binding at `relative_key = ""` with value_hash `H`):

`SHA-256(UTF-8(""))` = `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855` (well-known empty-string SHA-256). First 5 bits of byte 0 (`0xe3` = `0b11100011`, MSB-first) = `0b11100` = position 28. Bitmap = `0x10000000` → 4-byte big-endian `10 00 00 00`.

```
A2 63 6D6170 44 10000000 64 64617461 81 81 82 60 58 21 <H>
```

Where: `A2` map(2); `63 6D6170` "map"; `44 10000000` bytes(4) bitmap; `64 64617461` "data"; `81` array(1) (one entry in data); `81` array(1) (bucket with one tuple); `82` array(2) (`[key, value_hash]`); `60` empty text string ""; `58 21` bytes(0x21 = 33); `<H>` 33-byte value hash. **All implementations MUST produce this exact byte sequence given identical canonical-normalize on the empty-string relative_key.** This is conformance fixture #2 — the canonical fuzzer seed for catching SHA-256-input ambiguity (relative-key vs absolute-path) and bitmap-convention ambiguity at fuzzer-touch time.

**Trie bindings use prefix-relative keys.** Bindings in the trie are keyed by path segments relative to the prefix used when the snapshot was created. These are not paths — they are structural keys within a subtree produced by trimming a known prefix from the stored path: `relative_key = trim_prefix(path, "/" + peer_id + "/" + operation_prefix)`. This is standard prefix removal where both the peer ID and the operation prefix are known. The prefix is an operational parameter to `snapshot`, `extract`, and `merge` — not stored in the snapshot entity. Full paths are reconstructed by the consumer: `prefix + relative_key`. A non-empty prefix **MUST** end with `/`.

**The SHA-256 input is the relative_key, not the absolute path.** Two impls hashing different forms of the path produce different routing positions and silently divergent root hashes. Canonical-normalize follows the existing rules in the spec (see §5.4 and ENTITY-CORE-PROTOCOL.md §5.4); the resulting `UTF-8-bytes(canonical-normalize(relative_key))` is the SHA-256 input for routing per §3.3.

Cross-peer comparison is natural: snapshot of `/alice_id/local/files/` and `/bob_id/local/files/` both produce bindings keyed by the same relative paths — and the same trie root hash if content is identical. Diff and merge operate on these relative paths.

A snapshot is static — it captures the tree's state when the operation runs. Subsequent writes are not reflected.

**Example.** Tree state (4 bindings under prefix `/{peerA}/`):

```
/{peerA}/system/handler/files    -> hash_F
/{peerA}/system/handler/version  -> hash_V
/{peerA}/system/type/file        -> hash_T
/{peerA}/data/project/readme     -> hash_R
```

Relative keys (after prefix-strip): `system/handler/files`, `system/handler/version`, `system/type/file`, `data/project/readme`. Under hash-keyed routing, each key's SHA-256 determines its bit-slice path through the HAMT. The trie structure is determined by hash bits, not by path-segment locality. With only 4 bindings (≤ bucketSize+1 reachable through root), the entire trie collapses into a single root bucket holding all 4 tuples sorted lex by key.

The trie shape under hash-keyed routing is no longer visually-aligned with the path hierarchy. To reason about "bindings under prefix X," consumers use LocationIndex (path-keyed prefix scan; preserved unchanged) rather than walking trie subtree structure. To get a deterministic hash over the bindings under prefix X, consumers build a fresh HAMT over the filtered set per §3.7. The trie's role is content-addressed cross-peer convergence; the LocationIndex's role is path-keyed prefix scan. Division of labor is explicit.

Update `/{peerA}/system/handler/files` to `hash_F2`: the change affects HAMT positions along the SHA-256 path of `system/handler/files` (~4 positions for K=32, log_K(N)). Other bindings' positions are unchanged. New nodes are created along the modified path; unchanged sub-nodes are shared via content-addressed reference. Structural sharing across versions remains automatic — proportional to changes, not total bindings.

**Storage.** The snapshot root entity is returned as the operation result. The trie node entities are stored in the content store. To reference a snapshot by content hash in subsequent operations (diff, merge, extract), the snapshot and its trie nodes must be in the content store. This is not mandated — a snapshot computed and sent over the wire without being stored locally is valid; it simply will not be available in the local content store for later operations.

**Trie node persistence.** Trie nodes are content-addressed entities in the content store, subject to the same persistence and GC policies as any other entity. Specifically:

- Trie nodes referenced by active version entries (EXTENSION-REVISION.md) SHOULD be retained — they are needed for diff, merge, and transfer operations.
- Trie nodes referenced only by a tracked root (§3.4) and not by any version entry MAY be garbage-collected — they can be rebuilt deterministically from the current bindings at O(N) cost.
- Trie nodes not referenced by any version entry or tracked root MAY be garbage-collected freely.

**Restart behavior.** After a restart, the tracked trie root (if stored in operational state) is lost. The implementation must either rebuild the trie from current bindings (O(N) one-time cost) or re-derive it from the latest version entry's trie root plus any uncommitted changes. If the root is stored at a tree path (§3.4.1), it persists with the tree and survives restarts.

**Deduplication.** Content addressing provides natural deduplication for trie nodes. Two trees with overlapping subtrees share trie nodes automatically. Structural sharing across versions means the storage cost of N versions is proportional to the total changes across all versions, not N × tree size.

### 3.2 Operation

```
EXECUTE system/tree  operation: "snapshot"  resource: {targets: [<prefix or "system/tree">]}
```

**Parameters:**

```
system/tree/snapshot-request := {
  fields: {
    prefix:    {type_ref: "system/tree/path", optional: true}
                                    ; Default: "" (full tree)
    tree_id:   {type_ref: "primitive/string", optional: true}
                                    ; Default: default tree
  }
}
```

**Returns:** `system/tree/snapshot`

### 3.3 Algorithm

```
compute_snapshot(tree, prefix):
  if prefix != "" and not prefix.ends_with("/"):
    return error("invalid_prefix")

  bindings = []
  for (path, hash) in tree:
    if path starts with prefix:
      relative = path[len(prefix):]
      bindings.append((relative, hash))

  bindings.sort()

  return {
    type: "system/tree/snapshot",
    data: {
      root: build_trie(bindings)
    }
  }

build_trie(bindings):
  ; bindings: sorted list of (relative_key, entity_hash) pairs (sort is by relative_key UTF-8 lex)
  ; returns: content hash of the root trie node (IPLD HashMap node per §3.1)

  ; Start with empty root and incrementally insert each binding via trie_put (§3.4.2).
  ; Equivalent result: build directly by hashing all relative_keys and grouping into
  ; per-bit-slice buckets, recursing on overflow. Both produce identical canonical-form
  ; trie nodes because the CHAMP-equivalent invariant is enforced after every operation.
  root = empty_root_node()              ; canonical empty-root per §3.1 literal hex
  for (relative_key, value_hash) in bindings:
    root = trie_put(root, relative_key, value_hash)
  return content_hash(root)
```

**Implementations MAY use any algorithm** that produces the same trie structure as a canonical-form IPLD HashMap built from the binding set. Byte-identical-output is the cross-impl invariant; algorithm choice is implementation freedom.

**"Same trie structure" means precisely:** byte-identical CBOR encoding of every node. The §3.1 empty-root and 1-binding test vectors are the byte-level conformance anchors for the simplest cases; the cross-impl byte-identical-output fuzzer extends this to arbitrary binding sets. If two impls produce different bytes for the same binding set under the same parameters, one of them is non-conformant — the test vectors plus fuzzer narrow down which.

Reference implementation: [go-hamt-ipld v3.4.1](https://github.com/filecoin-project/go-hamt-ipld/tree/v3.4.1) (algorithm reference; wire encoding differs per §3.1).

### 3.4 Trie Root Tracking

The `build_trie` algorithm (§3.3) constructs a trie from a set of bindings. The algorithm is deterministic: same bindings produce the same trie root hash. Two implementation strategies exist:

**On-demand construction.** Build the trie when a snapshot is requested. Each snapshot is O(N) where N is the number of bindings under the prefix. Trie nodes are created in the content store during construction. The root hash is returned and discarded — subsequent snapshots rebuild.

**Incremental maintenance.** Maintain the trie continuously. Each `tree.put` updates the trie from the changed leaf to the root — O(depth) new nodes per write. The current root hash is tracked. Snapshot is O(1) — return the tracked root.

Implementations SHOULD use incremental maintenance for prefixes where:
- Auto-versioning is enabled (EXTENSION-REVISION.md §6)
- Frequent snapshot/diff/merge operations are expected
- Concurrent operations need consistent multi-path reads (see §3.6)

Implementations MAY use on-demand construction for prefixes where:
- Snapshots are infrequent (manual commits only)
- The binding set is small (O(N) rebuild is cheap)
- Write throughput is the priority (avoiding O(depth) per write)

Both strategies produce identical trie nodes and root hashes. The choice is a performance trade-off, not a correctness one.

#### 3.4.1 Root Tracking Location

Implementations that track trie roots SHOULD store the current root hash at a well-known location. Two approaches:

**Tree path.** Store the root hash as an entity at `system/tree/root/{prefix}`. This makes the root discoverable, subscribable, and syncable.

**Path substitution.** The stored path for the tracked root is derived from the config's `prefix` field as follows:

1. Strip any leading `/` and any trailing `/` from `prefix`. Call the result the "canonical prefix form" `P`.
2. If `P` is empty (the config's `prefix` was `"/"`, representing the universal tree root — a valid prefix since it ends with `/` per §3.4.1a), the stored path is `system/tree/root`.
3. Otherwise, the stored path is `system/tree/root/` + `P`.

Examples:

| Config `prefix` | Canonical `P` | Storage path | Notes |
|---|---|---|---|
| `"/"` | `""` | `system/tree/root` | Universal tree (all paths in peer's tree). |
| `"project/"` | `"project"` | `system/tree/root/project` | Peer-relative subtree. |
| `"project/src/"` | `"project/src"` | `system/tree/root/project/src` | Peer-relative subtree (nested). |
| `"/alice/data/"` | `"alice/data"` | `system/tree/root/alice/data` | Peer-qualified (alice's namespace, distinct from universal `/`). |

Note the distinction between `"/"` (universal tree — the empty canonical form) and `"/alice/data/"` (a peer-qualified path under alice's namespace — the leading `/` here scopes to a peer ID, not to the universal root). Strip-leading-and-trailing-slash canonicalization handles both correctly: the universal case collapses to the empty string; peer-qualified cases retain the peer ID segment.

Stripping leading and trailing slashes matches the leaf-binding convention (the binding value is a content_hash pointing at an entity, not a directory marker), aligns with standard path normalization, and produces a canonical binding path that subscribers across peers can predict.

The binding at `system/tree/root/{P}` is a direct pointer: its value is the content_hash of the root trie node entity (type `system/tree/snapshot/node`, defined in §3.3). Consumers read the tracked root with a single lookup — `tree.get(storage_path)` returns the hash, `content_store.get(hash)` returns the trie node entity with `entries` and optional `binding` fields. No wrapper entity is interposed.

This matches the single-pointer convention used elsewhere in the system (`system/revision/head/{prefix}` points directly at a `system/revision/entry`; `system/revision/branches/**` and `system/revision/tags/**` point directly at version entities). Implementations MUST NOT interpose a `system/hash`-typed wrapper entity.

**Operational state.** Track the root in peer-local state (not in the entity tree). This avoids the tree write for the root update but makes the root non-discoverable and non-syncable.

For prefixes with active versioning, the tree path approach is RECOMMENDED — the root is useful for other extensions and for debugging. For internal/ephemeral tries, operational state is sufficient.

**Invalidation.** The tracked root MUST reflect the current tree state. After a `tree.put` that affects a tracked prefix, the root must be updated (incremental) or invalidated (on-demand). Stale roots produce incorrect snapshots.

**History tracking.** Root tracking paths (`system/tree/root/*`) are subject to normal history recording. Implementations SHOULD NOT exclude them by default. The history chain preserves prior trie roots that would otherwise be unrecoverable — each transition records the previous root hash, the replacement root hash, the timestamp, and the chain_id of the write that triggered the update. This provides inter-commit rollback granularity: the version DAG (EXTENSION-REVISION.md) captures trie roots at commit points, while history captures every intermediate root between commits. Users MAY exclude these paths via history configuration if write volume is a concern.

**Revision exclude.** Root tracking paths `system/tree/root/**` MUST be excluded from versioned bindings when the root tracking path falls under a versioned prefix. The trie root hash depends on all bindings under the prefix — including it in the versioned bindings creates a circular dependency (the root hash would include itself). This exclusion is not a loss: the version entry's `root` field already captures the trie root at each commit point. Implementations using the tree-path approach SHOULD include `system/tree/root/**` in the revision configuration's `exclude` patterns (EXTENSION-REVISION.md §2.4). Alternatively, use operational state storage to avoid the circularity entirely.

#### 3.4.1a Configuration

> **Note:** `system/tree/config` is already used for non-default tree instances (§7.1). The tracking config uses `system/tree/tracking-config` to avoid type name collision.

```
system/tree/tracking-config := {
  fields: {
    prefix:   {type_ref: "system/tree/path"}
              ; Subtree prefix to maintain an incremental trie root for.
              ; Must end with "/". The value "/" is valid and designates
              ; the universal-tree root (all paths in the peer's tree).
              ; All other valid prefix values are non-empty paths ending
              ; with "/". The empty string is NOT a valid prefix.
    enabled:  {type_ref: "primitive/bool"}
              ; When false, tracking is suspended. The tracked root
              ; becomes stale and MUST be invalidated or removed.
  }
}
```

Tracking configs are stored at `system/tree/tracking-config/{name}`. The name is an opaque identifier chosen by the creator (typically derived from the prefix).

**Hot-reload.** The structural summary consumer (SYSTEM-COMPOSITION.md §2.2, position 6) MUST watch for changes to `system/tree/tracking-config/*` and update its tracked prefix set accordingly. Adding a config triggers an initial trie build for the prefix (O(N) one-time cost). Removing or disabling a config stops tracking — the root at `system/tree/root/{prefix}` becomes stale and MUST be removed.

**Startup discovery.** On peer startup, the structural summary consumer MUST scan `system/tree/tracking-config/*` to discover existing configs and rebuild tries for enabled prefixes. This follows the same bootstrap pattern as the history extension's config discovery on startup.

**Rebuild-wins on divergence.** If the persisted tracked-root hash diverges from the freshly rebuilt trie root on startup, the rebuilt root wins — the tracked root is derived state and the tree bindings are authoritative. Implementations SHOULD log the divergence at warning level (useful for catching silent corruption or missed persistence from a prior shutdown).

**Initial build is async.** When a tracking-config with `enabled: true` is written, the initial O(N) trie build SHOULD NOT block the config put. The config write MUST succeed even if the initial build has not yet completed; failures during the initial build are logged and retried asynchronously. Implementations MAY expose build progress via an operational-state field (e.g., `build_status: pending | complete | failed`) at the tracking-config path, so that consumers can distinguish a not-yet-built root from a stale one.

**Self-guard.** The structural summary consumer MUST skip events at paths matching `system/tree/root/*` to prevent recursive trie updates when the root hash itself is written to the tree.

Tracking configs can be written via standard tree `put` — no dedicated operation required. This is consistent with history configs, which are also managed via tree puts.

#### 3.4.2 Incremental Update Algorithm

The incremental trie update for a single `put(path, hash)` uses IPLD HashMap routing (bit-slice descent) with CHAMP-equivalent canonical-form maintenance:

```
trie_put(current_root_hash, relative_key, value_hash):
  ; 1. Compute hash_bytes = SHA-256(UTF-8-bytes(canonical-normalize(relative_key)))
  ;    Produces 32 bytes.
  ;
  ; 2. Walk levels by consuming bitWidth=5 bits from hash_bytes
  ;    (big-endian per byte, MSB first). Level 0 = bits 0-4 of byte 0;
  ;    level 1 = bits 5-9 spanning bytes 0-1; etc.
  ;
  ; 3. At each level, position p = next 5 bits; check map bitmap at bit p:
  ;    - If clear (bit not set): set bit p; insert [key, value_hash] tuple
  ;      into a new single-entry bucket at the appropriate data position
  ;      (position = popcount(map & ((1 << p) - 1))); done.
  ;    - If set: locate existing entry at popcount position in data:
  ;      - Bucket and len(bucket) < bucketSize=3 and key not present:
  ;          insert tuple into bucket (maintain lex sort by key); done.
  ;      - Bucket and key already present: replace value_hash in tuple; done.
  ;      - Bucket and len(bucket) == bucketSize and key not present:
  ;          convert bucket → sub-node; recurse all bucketSize+1 entries
  ;          (existing 3 + new 1) into the sub-node by their next-level bits.
  ;      - Link (sub-node hash): descend into sub-node entity; recurse.
  ;
  ; 4. On ascent: rehash each modified node, write new node entity to content
  ;    store, parent link updated to new hash. Produce new root hash.

  root = content_store.get(current_root_hash)
  new_root = put_at_node(root, hash_bytes_of(relative_key), 0, relative_key, value_hash)
  return content_store.put(new_root)
```

Per-Delete:

```
trie_remove(current_root_hash, relative_key):
  ; 1. Walk via SHA-256(canonical(relative_key)) bits to the binding's leaf bucket.
  ; 2. Remove the [key, value_hash] tuple from the bucket (maintain lex sort).
  ; 3. On ascent, enforce canonical-form invariant for non-root nodes:
  ;    - If a sub-node now has branchSize < bucketSize+1 = 4 (counting reachable
  ;      entries through links AND inline buckets), collapse it: take ALL its
  ;      reachable [key, value_hash] tuples and inline them into the parent's
  ;      bucket at the position that linked to the sub-node (maintain lex sort);
  ;      remove the link entry from data and clear the bit in map; replace with
  ;      the inlined bucket at the same position.
  ;    - If a parent's bucket now has len(bucket) == 0: clear bit in parent's map;
  ;      remove from data.
  ; 4. The root node MAY have fewer than bucketSize+1 entries (exempt from the
  ;    canonical-form lower bound). Collapse rule applies to all non-root nodes.
  ; 5. Rehash modified nodes on ascent; produce new root hash.

  root = content_store.get(current_root_hash)
  new_root = remove_at_node(root, hash_bytes_of(relative_key), 0, relative_key)
  return content_store.put(new_root)
```

Reference implementation: [go-hamt-ipld v3.4.1](https://github.com/filecoin-project/go-hamt-ipld/tree/v3.4.1) (algorithm reference). The CHAMP paper [Steindorfer & Vinju, OOPSLA 2015](https://michael.steindorfer.name/publications/oopsla15.pdf) §4 provides the theoretical basis for the canonical-form invariant; the IPLD HashMap form (`bucketSize+1`) is what we adopt normatively. Both are operationally equivalent for our parameter choices.

**Implementations MAY use any algorithm** that produces the same trie structure as a canonical-form IPLD HashMap built from the full updated binding set. Byte-identical-output (verified by §3.1 test vectors + cross-impl fuzzer per §12) is the cross-impl invariant; algorithm choice is implementation freedom.

**CHAMP-on-delete is a silent-bug class.** Insert-only tests do not exercise the canonical-form collapse-and-inline logic. Implementations MUST verify the invariant with the cross-impl byte-identical-output fuzzer (random insert/delete sequence; root hash compared across all impls under M seeds). Without this, two peers building "the same" tree by different histories produce different root hashes — `convergent_mirror` breaks at the substrate.

### 3.5 Prefix Nesting

A single tree path may fall under multiple tracked prefixes. For example, a write to `project/src/main.rs` falls under both `project/` and `project/src/` if both are tracked.

**Independent tries.** Each prefix has its own independent trie built from the bindings under that prefix. The `project/` trie contains all bindings under `project/` (including those under `project/src/`). The `project/src/` trie contains only bindings under `project/src/`. They are independent projections of the same underlying location index.

**Write fan-out.** A write to a path that falls under N tracked prefixes requires N trie updates (one per prefix). For incremental maintenance, each update is O(depth) — the total cost per write is O(N × depth) where N is the number of enclosing tracked prefixes.

**No structural dependency.** The `project/src/` trie is NOT a subtree of the `project/` trie. They are independent tries with independent root hashes. The `project/` trie compresses `project/src/main.rs` relative to the `project/` prefix; the `project/src/` trie compresses `main.rs` relative to the `project/src/` prefix. Different relative paths produce different trie structures and different root hashes.

**Nesting is unusual.** Most deployments track non-overlapping prefixes. Nesting arises when a broad prefix (full tree backup) coexists with a narrow prefix (project-level versioning). Implementations SHOULD document the write fan-out cost when N > 1.

**Version independence.** Versions at different prefixes are independent. A commit at `project/` creates a version with the `project/` trie root. A commit at `project/src/` creates a version with the `project/src/` trie root. Neither references the other. Cross-prefix version relationships are application-level concerns, not structural ones.

### 3.6 Consistency

A snapshot captures the tree's state at a point in time. When concurrent writes are possible, the snapshot's consistency depends on the implementation strategy:

**Incremental trie maintenance.** If the trie root is maintained incrementally, any reader that captures the current root hash has a consistent, immutable view of the tree at that moment. The trie nodes are in the content store (immutable). Concurrent writes produce new roots without affecting the captured view. This provides snapshot isolation for free — the trie root IS the consistency boundary.

**On-demand construction.** If the trie is built by scanning the location index, concurrent writes during the scan may produce a snapshot that reflects a state that never existed as a whole (some paths from before a write, some from after). This is a scan anomaly — standard in systems without snapshot isolation.

**Recommendations:**
- Operations that require consistent multi-path reads (snapshot for versioning, diff base, merge inputs) SHOULD use a trie root captured before the operation begins.
- The emit pathway (ENTITY-CORE-PROTOCOL.md §6.8) serializes writes per-path. If all writes go through the emit pathway, an incrementally-maintained trie root advances monotonically — each root represents a valid sequential state.
- For operations that need stronger isolation (read-modify-write across multiple paths), use a non-default tree as a staging area (§7) and merge the result.

This extension does not mandate an isolation level. It provides the trie structure that enables consistent reads; implementations choose when to use it. See EXPLORATION-CONCURRENCY-AND-ISOLATION.md for extended analysis.

### 3.7 Math contract — what the hash-keyed trie cryptographically commits to

This section was added in v4.0 alongside the substrate fork to make the cryptographic guarantees explicit. Math (what the structure proves) is separate from trust (who signed what); operations needing trust-layer accountability use existing trust mechanisms (peer identity, cap chains, signed protocol envelopes) — not the trie.

**Math properties under the v4.0 hash-keyed structure:**

| Property | Guaranteed? |
|---|---|
| Each binding in the trie is verifiable via inclusion proof against snapshot root | YES (walk SHA-256 bits, hash check at each level) |
| Same binding set produces the same snapshot root across peers | YES (CHAMP canonical-form invariant) |
| Same binding set produces the same root under arbitrary insert/delete history | YES (CHAMP-on-delete; v3.x path-keyed did NOT have this property) |
| Receiver of any subset can construct a deterministic hash over that subset | YES (build a fresh HAMT over the subset; see §3.7.1) |
| Snapshot root cryptographically commits to "subset under prefix X is exactly H_X" as a derivable property | **NO** (this is the single property the v4.0 fork gives up; see §3.7.2 for what depended on it and the recovery path if any future use case needs it) |

**Local prefix-scoped reads — unchanged.** `extract(prefix)` continues to use LocationIndex (B-tree range scan / equivalent) — O(prefix-subtree) bindings; the trie is not descended for prefix narrowing. `revision:fetch-diff(prefix, base)` computes full-trie diff via §4.3 (bounded by actual differences via hash-equality early-exit), then filters by `path.starts_with(prefix)` post-hoc. Both unchanged from v3.x at the API contract layer.

#### 3.7.1 Subtree-hash by reconstruction (the recovery mechanism for prefix-bounded subset commitments)

Given any subset S' of bindings under prefix X, an implementation can build a fresh HAMT over S' and obtain a deterministic root hash R_{S'}. Two impls building the same S' produce byte-identical R_{S'} because CHAMP canonicalization guarantees identical structure for identical input sets — this is the invariant the v4.0 fork adopts and the entire reason for adopting it.

**Cost:** O(|S'| × log_K |S'|) — for typical workloads (filesystem subdir = 100-1000 entries; subscription prefix = thousands), microseconds to low-ms. Cheap.

**Precondition:** all peers reconstructing R_{S'} MUST apply identical canonical-normalize semantics to relative-key strings. The §3.1 SHA-256 input rule (`UTF-8-bytes(canonical-normalize(relative_key))`) pins this; the same rule applies when filtering bindings by prefix. Two peers using different Unicode normalization upstream would filter to different subsets and reconstruct to different roots.

**What this gives you:** subtree-hash equality survives — it's *constructed* rather than *extracted as a structural subtree*. Operations that need a stable hash for a subset compute one cheaply; cross-peer comparison of "do we have the same set under X" works by comparing reconstructed R_{S'} values.

#### 3.7.2 What was lost (verified-unused, with extension recovery path)

The one cryptographic property genuinely lost in the v4.0 fork: the original snapshot root no longer commits — derivably from the root hash alone — to "the set under prefix X is exactly H_X." Under v3.x path-keyed routing, that derivation came for free from the structural subtree's content hash; under v4.0 hash-keyed routing, bindings under prefix X are scattered across hash-determined HAMT positions, and no subtree-hash corresponds to a path-prefix subset.

**Audit of current consumers (Stage 7 sign-off, four-impl independent grep):** no current operation in any impl uses this property as a verification step. Walkers traverse the trie to collect entities; they do not anchor at "subtree hash at path X equals expected H_X." Merge applies bindings; it does not verify subtree equivalence at a path. Subscription, capability, identity, content, history extensions do not consume trie internals at all. The property was a side-effect of v3.x's data structure shape, not a load-bearing verification primitive.

**Five enumerated speculative use cases** (per `proposals/implemented/PROPOSAL-TREE-NODE-SHAPE-BOUNDED-FANOUT.md` §4.7) that *could* in principle depend on the lost property — all verified-unused at Stage 7 sign-off:

1. Cryptographically-verifiable prefix-bounded subscription scope (no consumer in any impl)
2. Light-client prefix proofs. **We ARE actively building light-client-class consumers** — egui-WASM browser peer per `PROPOSAL-EXTENSION-BRIDGE-HTTP`; browser-peer + IndexedDB target per `PROPOSAL-PERSISTENCE-AS-EMIT-CONSUMER` §4.5. Their current pattern is per-blob hash-verify (preserved under hash-keyed: each binding gets an inclusion proof against snapshot root regardless of position). Compact prefix proofs would be additional, not foundational; the §3.7.2 sidecar extension restores them if/when per-blob pattern proves insufficient. This MIGHT-WANT case has a viable per-blob answer today; the sidecar path is open. (v4.4 errata corrects the v4.3 overclaim "we do not ship light clients" — we do, or will.)
3. External audit / regulatory commitment to prefix subsets (no such requirement)
4. Operational debugging treating "subtree under X" as a structural object with its own hash (adapts to §3.7.1 reconstruction)
5. Incremental-sync short-circuit via cached prefix-subtree hash (workbench-go retired the tree:extract-by-subtree recipe in favor of `revision:fetch-diff → tree:merge` which uses version-DAG diff; the version-DAG approach already supersedes this use case)

**Tree-only-vs-tree+revision composition.** Use case (5)'s "version-DAG supersedes" verdict presumes the revision extension is available. A consumer using ONLY the tree extension (no revision) does not have version-DAG diff as an alternative. But: even in that tree-only mode, the receiver does not today verify "this bundle is the complete subtree under X in snapshot S" as a code path. Today's tree:extract receivers walk bundles and trust the sender for "you sent me what I asked for"; under hash-keyed v4.0, same trust posture, with the added math benefit of per-binding inclusion proofs against snapshot root. Tree-only consumers don't suddenly start needing the lost property.

**Extension recovery path — if any future use case truly needs cryptographic prefix-subset commitments.** The v4.0 substrate does NOT foreclose restoring the property via a parallel path-prefix Merkle index entity (sidecar extension). Possible shapes:

- **Parallel path-prefix Merkle index** (`system/tree/prefix-index`) — opt-in per tree; O(N) storage overhead + O(log N) per write; restores O(1) prefix-subtree hash lookup. Same pattern Cosmos IAVL+, IPFS tile reads, Ethereum stateless clients all use. Production-validated approach.
- **Application-layer Merkle accumulator** over (relative_key, value_hash) pairs maintained by an extension when prefix-bounded cryptographic commitments are needed in a specific domain.
- **Light-client proofs via per-binding inclusion** (already provided by v4.0 math) plus a sender attestation at the cap layer (signed by sender peer — accountability, not verification, but acceptable in a trust-anchor model).

**Reversibility cascade for "what if we find out we need it":**

1. First check whether §3.7.1 reconstruction is fast enough. For typical workload sizes, yes — µs to low-ms. **If yes, no action needed.**
2. If reconstruction is too slow at the use case's frequency, propose the path-prefix sidecar extension. Self-contained, opt-in, doesn't touch substrate.
3. If the sidecar isn't expressive enough (light-client-class proofs), real architectural conversation with real requirements.
4. Worst case: substrate rollback. Pre-1.0; viable.

All four paths open. The v4.0 fork is not painting the system into a corner.

---

## 4. Diff

A diff compares two snapshots. It is deterministic: same two snapshots produce the same diff on any peer.

**Note:** Diff is pure computation on two snapshot entities. It does not require tree access. Implementations MAY compute diffs client-side from snapshot entities. The handler operation provides convenience and discoverability.

### 4.1 Types

```
system/tree/diff := {
  fields: {
    base:       {type_ref: "system/hash"}              ; Content hash of base snapshot
    target:     {type_ref: "system/hash"}              ; Content hash of target snapshot
    added:      {map_of: {type_ref: "system/hash"}}    ; path → hash (in target, not in base)
    removed:    {map_of: {type_ref: "system/hash"}}    ; path → hash (in base, not in target)
    changed:    {map_of: {type_ref: "system/tree/diff/change"}}
    unchanged:  {type_ref: "primitive/uint"}            ; Count of identical-hash paths
  }
}

system/tree/diff/change := {
  fields: {
    base_hash:   {type_ref: "system/hash"}
    target_hash: {type_ref: "system/hash"}
  }
}
```

**Diff keys are prefix-relative.** The keys in `added`, `removed`, and `changed` maps are paths relative to the diff's scope — the same prefix-relative format as snapshot trie binding keys (§3.1). These are not paths (not peer-namespaced). Full paths are reconstructed by the consumer: `"/" + peer_id + "/" + scope_prefix + relative_key`.

### 4.2 Operation

```
EXECUTE system/tree  operation: "diff"
```

Diff operates on stored snapshots — no path-level authorization. The `resource` field is optional; when omitted, handler-scope authorization (§11) suffices.

**Parameters:**

```
system/tree/diff-request := {
  fields: {
    base:   {type_ref: "system/hash"}     ; Content hash of base snapshot
    target: {type_ref: "system/hash"}     ; Content hash of target snapshot
  }
}
```

Both snapshots **MUST** be in the content store or the envelope's `included` map.

**Returns:** `system/tree/diff`

### 4.3 Algorithm

The diff algorithm walks the trie recursively, skipping entire unchanged subtrees when their root hashes match. Path compression may produce different trie shapes for different tree states, so the algorithm decomposes compressed keys by first segment and resolves mismatches at the divergence point.

**Path construction helper.** All path construction uses `join_path` to avoid leading slashes when the prefix is empty and trailing-slash artifacts in recursive calls:

```
join_path(prefix, suffix):
  if prefix == "": return suffix
  if suffix == "": return prefix
  return prefix + "/" + suffix
```

**Entry decomposition.** Path compression guarantees that no two entries in a single node share a first path segment. (If they did, the first segment would be a separate node, not compressed.) This means entries can be matched unambiguously by their first segment.

```
decompose_entries(entries):
  ; Split each compressed key into (first_segment, remaining_path, child_hash)
  ; "type/file" → ("type", "file", hash)
  ; "type"      → ("type", "",     hash)
  ; "data/project/readme" → ("data", "project/readme", hash)

  result = {}   ; first_segment → (remaining_path, child_hash)
  for (key, hash) in entries:
    segments = key.split("/")
    first = segments[0]
    remaining = "/".join(segments[1:])     ; "" if key has no "/"
    result[first] = (remaining, hash)
  return result
```

Since no two entries share a first segment, each first segment maps to exactly one (remaining, hash) pair.

**Main diff algorithm:**

```
compute_diff(base_snapshot, target_snapshot):
  base_root = content_store.get(base_snapshot.data.root)
  target_root = content_store.get(target_snapshot.data.root)

  changes = diff(base_root, target_root, "")

  ; Assemble into diff entity
  added = {}
  removed = {}
  changed = {}
  unchanged_count = 0

  for change in changes:
    if change.kind == "added":
      added[change.path] = change.hash
    elif change.kind == "removed":
      removed[change.path] = change.hash
    elif change.kind == "changed":
      changed[change.path] = {base_hash: change.base_hash, target_hash: change.target_hash}

  ; Count unchanged by walking one trie and subtracting
  unchanged_count = count_leaf_bindings(base_root) - len(removed) - len(changed)

  return {
    type: "system/tree/diff",
    data: {
      base: base_snapshot.content_hash,
      target: target_snapshot.content_hash,
      added: added,
      removed: removed,
      changed: changed,
      unchanged: unchanged_count
    }
  }

diff(node_a, node_b):
  ; Walk two HAMT roots in parallel by bitmap position.
  ; Early-exit on hash equality at every node (entire subtree identical).
  ; Recurse where positions differ.
  ; Emit changes as we encounter them; output collected and sorted at the end.

  ; Hash-equality early exit (works at root AND at every recursion site)
  if content_hash(node_a) == content_hash(node_b):
    return []

  changes = []
  combined_map = node_a.map | node_b.map         ; positions occupied in either trie

  for p in iterate_set_bits(combined_map):
    bit_a = (node_a.map >> p) & 1
    bit_b = (node_b.map >> p) & 1

    entry_a = entry_at_position(node_a, p) if bit_a else null
    entry_b = entry_at_position(node_b, p) if bit_b else null

    if entry_a == null:
      ; Position present only in B → all bindings under it are ADDED
      changes.append_all(walk_entry_collect("added", entry_b))
      continue

    if entry_b == null:
      ; Position present only in A → all bindings under it are REMOVED
      changes.append_all(walk_entry_collect("removed", entry_a))
      continue

    ; Both have entry at this position; compare by entry type
    if is_bucket(entry_a) and is_bucket(entry_b):
      changes.append_all(diff_buckets(entry_a, entry_b))
    elif is_link(entry_a) and is_link(entry_b):
      ; Hash-equality early-exit applies recursively
      if entry_a.hash == entry_b.hash:
        continue
      child_a = content_store.get(entry_a.hash)
      child_b = content_store.get(entry_b.hash)
      changes.append_all(diff(child_a, child_b))
    elif is_bucket(entry_a) and is_link(entry_b):
      ; A has bucket, B has sub-node — flatten B's subtree to bindings, diff with bucket
      bindings_b = walk_entry_collect_bindings(entry_b)   ; list of (key, value_hash)
      changes.append_all(diff_bucket_vs_bindings(entry_a, bindings_b))
    elif is_link(entry_a) and is_bucket(entry_b):
      bindings_a = walk_entry_collect_bindings(entry_a)
      changes.append_all(diff_bindings_vs_bucket(bindings_a, entry_b))

  return changes

diff_buckets(bucket_a, bucket_b):
  ; Both buckets are sorted lex by key. Merge-walk.
  changes = []
  keys_a = {tuple.key: tuple.value_hash for tuple in bucket_a}
  keys_b = {tuple.key: tuple.value_hash for tuple in bucket_b}
  for k in keys(keys_a) | keys(keys_b):
    if k not in keys_b: changes.append(removed(k, keys_a[k]))
    elif k not in keys_a: changes.append(added(k, keys_b[k]))
    elif keys_a[k] != keys_b[k]: changes.append(changed(k, keys_a[k], keys_b[k]))
  return changes

walk_entry_collect(kind, entry):
  ; Walk an entry (bucket or link) and emit `kind` for every (key, value_hash) reachable
  changes = []
  if is_bucket(entry):
    for tuple in entry:
      changes.append({kind: kind, key: tuple.key, hash: tuple.value_hash})
  else:  ; link
    sub_node = content_store.get(entry.hash)
    for p in iterate_set_bits(sub_node.map):
      changes.append_all(walk_entry_collect(kind, entry_at_position(sub_node, p)))
  return changes

count_leaf_bindings(node):
  ; Count all entity bindings in the trie (for unchanged count computation)
  count = 0
  for p in iterate_set_bits(node.map):
    entry = entry_at_position(node, p)
    if is_bucket(entry):
      count += len(entry)
    else:  ; link
      count += count_leaf_bindings(content_store.get(entry.hash))
  return count
```

**Output ordering (MUST).** The diff output's `added`, `removed`, and `changed` arrays MUST be sorted lex by `key` (UTF-8 lexicographic) at output time, regardless of the trie's internal hash-keyed traversal order. The trie traversal visits positions in hash-bit order (effectively random with respect to keys); the output is explicitly re-sorted before return. This makes diff outputs comparable across peers regardless of internal traversal implementation.

**Properties.**

**Diff return type unchanged.** The `system/tree/diff` type still has `added`, `removed`, `changed`, `unchanged` fields with the same semantics. The diff is key-level — the trie structure is an implementation detail of how diffs are computed efficiently.

**Determinism.** The diff algorithm is deterministic: same two trie roots produce the same diff (with the mandated output sort).

**Cost.** O(changes) — the hash-equality early-exit at every node skips entire unchanged subtrees. For one binding change in a 10,000-binding tree with HAMT depth ~4, the diff visits ~4 positions (the SHA-256 bit path of the changed key) and skips everything else via hash-equality. **The compression-mismatch machinery from prior versions (`LongestCommonPrefix`, `ResolveAtDivergence`, `DecomposeEntries`, virtual nodes) is removed** — IPLD HashMap nodes have fixed-position structure indexed by `map` bitmap position; there is no compressed-key divergence to resolve. This is a substantial code simplification (net code reduction in the diff path).

---

## 5. Merge

Merge applies a snapshot's bindings into a target tree, with conflict handling.

### 5.1 Types

```
system/tree/merge-result := {
  fields: {
    applied:    {type_ref: "primitive/uint"}
    skipped:    {type_ref: "primitive/uint"}
    conflicts:  {map_of: {type_ref: "system/tree/merge-result/conflict"}}
    strategy:   {type_ref: "primitive/string"}
  }
}

system/tree/merge-result/conflict := {
  fields: {
    existing_hash: {type_ref: "system/hash"}
    incoming_hash: {type_ref: "system/hash"}
    resolution:    {type_ref: "primitive/string"}
                   ; "kept-existing" | "used-incoming" | "unresolved"
  }
}
```

### 5.2 Operation

```
EXECUTE system/tree  operation: "merge"  resource: {targets: [<target_prefix or paths>]}
```

**Parameters:**

```
system/tree/merge-request := {
  fields: {
    source:           {type_ref: "system/hash", optional: true}
                      ; Content hash of source snapshot (mutually exclusive with source_envelope)
    source_envelope:  {type_ref: "primitive/any", optional: true}
                      ; Inline envelope entity from extract result (continuation chains).
                      ; The handler ingests included entities and uses the root as source.
    target_tree:      {type_ref: "primitive/string", optional: true}
                      ; Tree to merge into. Default: default tree
    strategy:         {type_ref: "primitive/string", optional: true}
                      ; Default: "no-overwrite"
    source_prefix:    {type_ref: "system/tree/path", optional: true}
                      ; Tree path where the snapshot's content logically resides.
                      ; Used when target_prefix is absent — bindings placed at source_prefix + relative_path.
    target_prefix:    {type_ref: "system/tree/path", optional: true}
                      ; Tree path where merged bindings should be placed.
                      ; When provided, all bindings written to target_prefix + relative_path.
    dry_run:          {type_ref: "primitive/bool", optional: true}
                      ; Default: false
  }
}
```

**source vs source_envelope.** Exactly one of `source` or `source_envelope` must be provided. `source` is a hash referencing a snapshot already in the content store. `source_envelope` is the extract result entity — either a raw envelope `{root, included}` or an inline entity wrapping an envelope. When `source_envelope` is provided, the merge handler ingests all included entities into the content store and uses the root snapshot's hash as the source. The `source_envelope` form exists for continuation chains where the extract result flows directly into merge params via `result_field` injection, eliminating the need for an intermediate content ingest step.

**Prefix placement**: The snapshot contains only a trie root — no prefix. The `target_prefix` parameter specifies where to place merged bindings. When `target_prefix` is provided, bindings are written to `target_prefix + relative_path`. When only `source_prefix` is provided, bindings are written to `source_prefix + relative_path`. When neither is provided, bindings use relative paths as-is.

**Cross-peer merge.** When merging a snapshot from another peer, the snapshot carries no prefix — it is pure content. The `target_prefix` parameter specifies where to place the bindings in the local tree.

Example: merging peer B's files into peer A's tree:
1. Peer A extracts from peer B: `EXECUTE system/tree operation: "extract" resource: {targets: ["/{peer_B_id}/data/"]}` → returns snapshot `{root: hash}`
2. Peer A merges locally: `EXECUTE system/tree operation: "merge" params: {snapshot: hash, target_prefix: "/{peer_A_id}/data/"}`
3. Merge applies: for each binding `(relative_path, hash)` in the trie, write to `/{peer_A_id}/data/{relative_path}`

No prefix mismatch is possible — the snapshot has no prefix to mismatch.

**Returns:** `system/tree/merge-result`

### 5.3 Strategies

| Strategy | Behavior |
|----------|----------|
| `no-overwrite` | Place new paths. Report conflicts for existing paths with different content. **(Default)** |
| `source-wins` | Place all paths. Overwrite existing. |
| `target-wins` | Place new paths only. Keep existing on conflict. |

### 5.4 Algorithm

```
execute_merge(params, target_tree, capability, local_peer_id):
  ; Resolve source: either direct hash or from envelope
  if params.source_envelope is not null:
    envelope = params.source_envelope
    ; source_envelope may be entity-wrapped {type, data, content_hash} or raw {root, included}.
    if envelope.type is not null and envelope.data is not null:
      envelope = envelope.data                     ; entity-wrapped — unwrap
    for (hash, entity) in envelope.included:
      content_store.put(entity)
    content_store.put(envelope.root)
    source_snapshot = envelope.root
  else if params.source is not null:
    source_snapshot = content_store.get(params.source)
  else:
    return error("invalid_params: source or source_envelope required")

  strategy = params.strategy or "no-overwrite"
  source_prefix = params.source_prefix
  target_prefix = params.target_prefix
  dry_run = params.dry_run or false

  ; Merge operates on the location index (path → hash mappings).
  ; This is below the protocol get/put level — no entity resolution needed,
  ; only hash bindings from the snapshot.
  applied = 0
  skipped = 0
  conflicts = {}

  ; Collect all bindings by walking the trie from the snapshot root
  bindings = collect_all_bindings(content_store.get(source_snapshot.data.root), "")

  ; Pre-check: verify put authorization on all target paths (atomic)
  for (relative_path, hash) in bindings:
    target_path = apply_prefix(relative_path, source_prefix, target_prefix)
    if check_path_permission("put", target_path, capability, "system/tree", local_peer_id) == DENY:
      return error("capability_denied", 403)

  for (relative_path, hash) in bindings:
    target_path = apply_prefix(relative_path, source_prefix, target_prefix)

    existing = target_tree.index.get(target_path)   ; location index lookup → hash or null

    if existing is null:
      if not dry_run: target_tree.index.put(target_path, hash)
      applied += 1

    else if hash_equals(existing, hash):
      skipped += 1

    else:
      ; Conflict: existing hash differs from incoming hash.
      ; Always record the conflict; resolution depends on strategy.
      if strategy == "source-wins":
        if not dry_run: target_tree.index.put(target_path, hash)
        applied += 1
        conflicts[target_path] = {
          existing_hash: existing, incoming_hash: hash,
          resolution: "used-incoming"
        }
      else if strategy == "target-wins":
        skipped += 1
        conflicts[target_path] = {
          existing_hash: existing, incoming_hash: hash,
          resolution: "kept-existing"
        }
      else:  ; "no-overwrite"
        skipped += 1
        conflicts[target_path] = {
          existing_hash: existing, incoming_hash: hash,
          resolution: "unresolved"
        }

  return {
    type: "system/tree/merge-result",
    data: {applied, skipped, conflicts, strategy}
  }
```

**`applied` and `skipped` semantics under `dry_run` (normative).** The `applied` counter increments on every binding that would be written under the chosen `strategy` — i.e., the `!exists` branch and the `source-wins` overwrite branch. The `skipped` counter increments on every binding that would NOT be written — i.e., the `hash_equals` branch (no-op) and the `target-wins` / `no-overwrite` conflict branches. These counts are independent of `dry_run`; setting `dry_run = true` suppresses the actual `target_tree.index.put` writes but does NOT change which counter increments for any given binding. `dry_run` produces an honest preview of what a real merge would do, and `applied + skipped == len(bindings)` always holds. Conformance: implementations MUST report counts consistent with this rule; a `dry_run` invocation against any input MUST yield the same `(applied, skipped, conflicts)` triple as a non-`dry_run` invocation against the same inputs (modulo the actual binding writes).

```
apply_prefix(relative_path, source_prefix, target_prefix):
  if target_prefix is not null:
    return target_prefix + relative_path
  if source_prefix is not null:
    return source_prefix + relative_path
  return relative_path

collect_all_bindings(node, _path_prefix_unused):
  ; Walk the HAMT and collect all (key, value_hash) tuples reachable from this node.
  ; Under hash-keyed routing, keys live directly in leaf-level buckets — there is no
  ; per-node path-prefix to accumulate. The `key` field of each tuple is the full
  ; relative_key (the value originally inserted via trie_put). Output ordering is
  ; hash-bit-traversal order; callers that need lex-sorted output MUST sort at output.
  result = []
  for p in iterate_set_bits(node.map):
    entry = entry_at_position(node, p)
    if is_bucket(entry):
      for tuple in entry:
        result.append((tuple.key, tuple.value_hash))
    else:  ; link to sub-node
      sub = content_store.get(entry.hash)
      result.append_all(collect_all_bindings(sub, null))
  return result
```

**Prefix remapping is translation, not filtering.** Paths that do not match `source_prefix` pass through unchanged. To control which bindings are included in a merge, use a scoped snapshot prefix or the extract `paths` filter (§6) — not remapping.

**Merge is additive.** Merge does not remove paths from the target tree that are absent in the source snapshot. It only adds new bindings and (depending on strategy) updates conflicting ones. To fully synchronize two trees — making the target match the source — use diff to identify paths present in the target but absent in the source, then remove them separately via `put` with null entity.

**Trie-aware optimization.** When the source snapshot uses a trie format, implementations SHOULD use recursive trie traversal to skip subtrees where the source trie node hash matches the target tree's state at the corresponding prefix. This reduces merge cost from O(total_paths) to O(changed_paths * depth).

The tree merge operation is simpler than the version merge — it compares source snapshot against live tree state (not two snapshots). The trie optimization applies to the source side; the target side is the live location index (flat path → hash).

---

## 6. Extract

Extract produces a transferable envelope: a snapshot as root, with referenced entities in `included`.

By default, extract captures all bindings under the prefix. An optional `paths` filter narrows extraction to specific relative paths — for targeted transfer after a diff, where you know exactly which bindings you need.

**Math contract under v4.0 hash-keyed routing (see §3.7 for full detail).** Each binding the receiver gets is verifiable via inclusion proof against the source snapshot root (per-binding math, preserved). The receiver can build a fresh HAMT over the extracted bindings to obtain a deterministic subset root for comparison (subtree-hash by reconstruction; §3.7.1). The original snapshot root does NOT cryptographically commit to "this bundle is the complete set under the requested prefix" derivably — exhaustiveness is a trust-layer concern (sender identity, cap chain, protocol context), unchanged from v3.x trust posture. Per §3.7.2's four-impl audit, no current consumer in any impl uses the lost derivability property as a verification step; if any future use case ever needs it, the §3.7.2 extension recovery path (parallel path-prefix Merkle sidecar entity) restores it without re-opening the substrate decision.

### 6.1 Operation

```
EXECUTE system/tree  operation: "extract"  resource: {targets: [<prefix>]}
```

**Parameters:**

```
system/tree/extract-request := {
  fields: {
    prefix:  {type_ref: "system/tree/path"}
    tree_id: {type_ref: "primitive/string", optional: true}
    paths:   {array_of: {type_ref: "system/tree/path"}, optional: true}
                                    ; Specific relative paths to include.
                                    ; Default: all paths under prefix.
  }
}
```

When `paths` is provided, only those relative paths are included in the snapshot and envelope. Paths are relative to `prefix` — the same relative paths that appear in snapshot bindings and diff results.

> **Incremental "since" transport lives in the revision layer.** Transporting only what changed between two versions is a *revision* concern (its inputs are version hashes, and version → trie-root dereference is the revision extension's knowledge). See `EXTENSION-REVISION.md` §4.4.19 `fetch-diff`. A `since` parameter was briefly added to `tree:extract` (v3.14) and **withdrawn in v3.15** — placing it here forced a tree op to dereference revision version entries, a layering violation. `tree:extract` filters by `paths` only.

**Returns:** `system/envelope`

### 6.2 Algorithm

```
execute_extract(tree, content_store, prefix, paths):
  if prefix != "" and not prefix.ends_with("/"):
    return error("invalid_prefix")

  ; Collect bindings — only what's needed
  bindings = []
  if paths is not null:
    ; Filtered: read specific paths directly
    for path in paths:
      hash = tree.get(prefix + path)
      if hash is not null:
        bindings.append((path, hash))
  else:
    ; Full prefix: all bindings under prefix
    for (full_path, hash) in tree:
      if full_path starts with prefix:
        bindings.append((full_path[len(prefix):], hash))

  bindings.sort()

  ; Build trie and snapshot (§3.3)
  root_hash = build_trie(bindings)
  snapshot = {
    type: "system/tree/snapshot",
    data: {root: root_hash}
  }

  ; Bundle trie nodes + data entities
  included = {}

  ; Include all trie nodes reachable from root
  include_trie_nodes(root_hash, included):
    node = content_store.get(root_hash)
    included[root_hash] = node
    for (key, child_hash) in node.entries:
      if child_hash not in included:
        include_trie_nodes(child_hash, included)

  include_trie_nodes(root_hash, included)

  ; Include data entities at leaf bindings
  for (path, hash) in bindings:
    entity = content_store.get(hash)
    if entity is not null:
      included[hash] = entity

  return {
    type: "system/envelope",
    data: {root: snapshot, included: included}
  }
```

If a data entity referenced by a leaf binding is not in the local content store, the binding appears in the trie but the entity is absent from `included`. The snapshot and trie nodes accurately represent tree structure; `included` contains what is locally available. Receivers can request missing entities separately.

**Trie nodes in extract envelopes.** The extract envelope's `included` map MUST contain:
- The snapshot root entity.
- All trie node entities reachable from the snapshot root.
- All data entities referenced by leaf bindings in the trie.

When the `paths` filter is specified, the envelope includes only the trie nodes along the paths from the filtered paths to the root, plus the data entities at the filtered paths. Unfiltered subtree nodes MAY be omitted.

This means a full extract includes the complete trie plus all data entities. A filtered extract includes only the relevant trie branches.

---

## 7. Non-Default Trees

### 7.1 Config Type

```
system/tree/config := {
  fields: {
    tree_id:        {type_ref: "primitive/string"}
    root_structure: {type_ref: "primitive/string"}
                    ; "peer-namespaced" | "relaxed"
    purpose:        {type_ref: "primitive/string", optional: true}
                    ; "staging" | "translation" | "sub-peer" | "view" | custom
    ephemeral:      {type_ref: "primitive/bool", optional: true}
                    ; Default: false. If true, not persisted across restarts.
    source:         {type_ref: "primitive/string", optional: true}
                    ; Source tree_id. When present, this is a view tree (§8).
    capability:     {type_ref: "system/hash", optional: true}
                    ; Capability hash that defines the view filter (§8).
                    ; Required when source is present.
  }
}
```

Tree configs are stored at `system/tree/instances/{tree_id}`.

When `source` is present, the tree is a **view tree** — its bindings are derived from the source tree, filtered by the capability's grants. See §8.

### 7.2 Create

```
EXECUTE system/tree  operation: "create"
  params: {type: "system/tree/config", data: {tree_id: "staging-01", ...}}
```

Create MUST validate:
- `tree_id` is not already in use → `tree_exists` (409)
- If `source` is present, `capability` MUST also be present → `invalid_config` (400)
- If `capability` is present, it MUST reference a valid capability in the content store → `capability_not_found` (404)

**Returns:** `system/tree/config`

### 7.3 Destroy

```
EXECUTE system/tree  operation: "destroy"
  params: {type: "primitive/string", data: "staging-01"}
```

Removes the tree and all its bindings. The default tree MUST NOT be destroyed — attempts return `default_tree` (400).

**Returns:** `primitive/bool`

### 7.4 Use Patterns

**Staging**: Create an ephemeral tree to absorb incoming data in isolation. Snapshot the staging tree, dry-run merge against the default tree to preview, then merge for real.

**Sub-peering**: Each sub-peer gets its own tree with its own identity context. Tree operations (snapshot, merge, extract) manage the sub-peer's namespace.

**Translation**: Create a tree with different root structure for internal computation. Extract to move results back into the protocol-namespaced default tree.

**View**: Create a capability-scoped projection of a source tree for handler isolation. See §8.

### 7.5 Emit Behavior

Extensions operate on the tree provided in their handler context (`ctx.entity_tree`). No extension has `tree_id` in its operation parameters — extensions are tree-agnostic by construction. The tree identity is determined by the runtime wiring: which location index generates events, and which tree reference the handler context provides.

The default tree's emit chain is wired at peer startup (SYSTEM-COMPOSITION.md §2.2). Non-view non-default trees have their own location indexes but do not participate in the default tree's emit chain. Writes to a non-default tree do not trigger system extension consumers (history, clock, compute, subscription, structural summaries, query indexing) unless the implementation has explicitly registered consumers for that tree.

View trees are not affected — writes propagate to the source tree and events fire there (§8.3).

A non-default tree with its own registered consumers is functionally a peer-within-a-peer: same content store, same identity, independent emit chain with its own consumer instances. Each consumer instance operates through its own `ctx.entity_tree` reference, writing to paths like `system/history/head/{path}` that naturally live in whichever tree the consumer is wired to. The extension interfaces require no modification — multi-tree support is a runtime wiring concern, not an API concern.

The infrastructure for per-tree consumer registration (creating emit chains, instantiating consumer sets, managing per-tree config namespaces) is implementation-defined. This version of the spec does not define a normative mechanism for wiring emit consumers to non-default trees.

---

## 8. View Trees

A view tree is a filtered projection of a source tree, scoped by the effective grants for a request. Implementations MUST ensure that view tree filtering prevents access to paths outside the effective grants, regardless of handler implementation.

### 8.1 Concept

A capability defines a scope. That scope IS a tree. When a request is dispatched to a handler, the system creates (or reuses) a view tree filtered to the effective grants. The handler receives the view tree's `tree_id` and interacts with it using standard tree operations. The handler cannot tell it's operating on a view.

The **effective grants** are determined by:
- If the handler has no `max_scope` (ENTITY-CORE-PROTOCOL.md §3.7): the request capability's grants.
- If the handler has `max_scope`: the intersection of the request capability's grants and the handler's `max_scope`. The handler cannot exceed either bound.

```
Request arrives with capability:
  grants: [{handlers: {include: ["system/tree"]}, resources: {include: ["peer/local/files/*"]}, operations: {include: ["get", "put"]}}]

Handler manifest:
  max_scope: [{handlers: {include: ["system/tree"]}, resources: {include: ["peer/local/files/*"]}, operations: {include: ["get"]}}]

Effective grants (intersection):
  [{handlers: {include: ["system/tree"]}, resources: {include: ["peer/local/files/*"]}, operations: {include: ["get"]}}]
  ; Request allowed get+put, handler only allows get → effective is get only.

Dispatch creates a view tree:
  tree_id:     "view-{scope_hash}"
  source:      default tree (or another tree)
  scope:       effective grants
  ephemeral:   true

Handler receives tree_id in its context.
All tree operations target this tree_id.
```

### 8.2 Read Behavior

Reads delegate to the source tree with path filtering:

```
view_tree.get(path):
  if path ends with "/" or path == "":
    ; Listing: delegate and filter by scope
    listing = source_tree.get(path)
    filtered_entries = {}
    for (name, value) in listing.data.entries:
      if scope.can_get(path + name):
        filtered_entries[name] = value
    return listing with entries = filtered_entries, count = len(filtered_entries)
  else:
    ; Entity read: check path access
    if not scope.can_get(path):
      return not_found                        ; Invisible, not forbidden
    return source_tree.get(path)
```

Out-of-scope reads MUST return `not_found`. The response MUST be indistinguishable from a genuinely non-existent path — implementations MUST NOT leak whether out-of-scope paths exist. Listings return only entries the capability grants `get` access to. The listing `count` field MUST reflect the filtered count (visible entries), not the source tree's total count — a discrepancy between `count` and the number of returned entries would leak the existence of hidden paths.

### 8.3 Write Behavior

Writes check scope then propagate to the source tree:

```
view_tree.put(path, entity):
  if not scope.can_put(path):
    return capability_denied
  return source_tree.put(path, entity)
  ; entity is null → remove binding (core protocol §6.3)
```

Writes propagate immediately to the source tree. Events (subscriptions, change notifications) fire on the source tree — all views reflect the current source state.

### 8.4 Bulk Operations on View Trees

All bulk operations (§3-§6) work on view trees unchanged:

| Operation | Behavior on View Tree |
|-----------|-----------------------|
| `snapshot` | Captures only bindings visible through the filter |
| `diff` | Pure computation — no tree access, works on any snapshots |
| `merge` | Each write path checked against the view's write scope. If any path denied, entire merge fails (atomic). |
| `extract` | Bundles only visible entities |

A snapshot of a view tree captures exactly what the handler is authorized to see. A merge into a view tree enforces write scope per-path (each path checked against `put` grants; atomic failure on denial). No changes to the operation definitions — the view tree's filtering is the only difference.

### 8.5 Scope Compilation

At dispatch time, the system compiles the effective scope for the two tree operations:

```
compile_scope(capability, handler, local_peer_id):
  ; Compile request capability grants.
  request_entries = compile_grant_entries(capability.data.grants, handler.data.pattern, local_peer_id)

  ; Compile handler max_scope if present.
  handler_entries = null
  if handler.data.max_scope is not null:
    handler_entries = compile_grant_entries(handler.data.max_scope, handler.data.pattern, local_peer_id)

  ; Scope checks BOTH grant sets. A path is allowed only if the request
  ; capability allows it AND (if max_scope exists) the handler allows it.
  return scope {
    can_get(path):
      p = canonicalize(path, local_peer_id)
      if not check_scope(p, "get", request_entries, local_peer_id): return false
      if handler_entries is not null and not check_scope(p, "get", handler_entries, local_peer_id): return false
      return true
    can_put(path):
      p = canonicalize(path, local_peer_id)
      if not check_scope(p, "put", request_entries, local_peer_id): return false
      if handler_entries is not null and not check_scope(p, "put", handler_entries, local_peer_id): return false
      return true
  }

compile_grant_entries(grants, handler_pattern, local_peer_id):
  ; Build per-grant scope entries preserving exclude association.
  ; This mirrors the per-scope exclude semantics of core protocol
  ; check_path_permission (§6.3) — an exclude within one grant's
  ; resources scope does not affect a different grant.
  ; Only grants whose `handlers` scope matches this handler are included.
  ; Grant dimensions use system/capability/scope ({include, exclude}).
  entries = []
  for grant in grants:
    ; Filter by handler — skip grants that don't apply to this handler
    if not matches_scope(handler_pattern, grant.handlers, local_peer_id):
      continue
    resources = []
    excludes = []
    for resource in grant.resources.include:
      resources.add(canonicalize(resource, local_peer_id))
    if grant.resources.exclude is not null:
      for exclusion in grant.resources.exclude:
        excludes.add(canonicalize(exclusion, local_peer_id))
    entries.add({
      resources: resources,
      excludes: excludes,
      operations: grant.operations
    })
  return entries

check_scope(canonical_path, operation, grant_entries, local_peer_id):
  for entry in grant_entries:
    if not matches_scope(operation, entry.operations, local_peer_id):
      continue
    matched = false
    for resource in entry.resources:
      if matches_pattern(canonical_path, resource):
        matched = true
        break
    if not matched: continue
    excluded = false
    for pattern in entry.excludes:
      if matches_pattern(canonical_path, pattern):
        excluded = true
        break
    if not excluded: return true
  return false
```

`get` covers both entity reads and listings; `put` covers both writes and removals. Pattern matching uses ENTITY-CORE-PROTOCOL.md §5.4 rules. All paths and patterns are canonicalized (§5.4) before matching. Compilation happens once per request dispatch. The compiled scope is a fast check (prefix matching) on every tree operation.

When `max_scope` is absent, the dual-check reduces to the single request capability check — no overhead for handlers without max_scope.

Handler filtering in `compile_grant_entries` ensures that only grants relevant to the tree handler (matching the `handlers` field) contribute to the compiled scope. Grants for other handlers are skipped.

### 8.6 Content Store Access

View trees scope the **naming layer** (paths). The **content layer** (hashes) is a separate security concern.

Having a content hash does not imply authorization to read the content. Content store access is scoped by the content extension (EXTENSION-CONTENT.md), which can limit which hashes a handler can resolve. View trees and content scoping are complementary:

| Layer | Mechanism | Scopes |
|-------|-----------|--------|
| Path (naming) | View trees | Which paths a handler can read/write |
| Content (storage) | Content extension | Which hashes a handler can resolve |

View trees prevent discovery of which hashes exist at out-of-scope paths (because `get(path)` is filtered). Content scoping prevents resolution of hashes obtained through other means. Together, they provide defense in depth.

How handlers access the content store (hash-based reads) is implementation-defined in the absence of the content extension.

### 8.7 Handler Security Model

View trees provide structural enforcement for tree access. They do not replace handler-level security.

Domain handlers remain capability-aware: they validate domain-specific constraints, enforce capability caveats with domain semantics, and make authorization decisions specific to their domain. View trees ensure that tree access cannot exceed the request's authorization — handlers cannot accidentally (or intentionally) read or write paths outside scope. Domain semantics within that scope remain the handler's responsibility.

- **Domain handlers**: receive a view tree `tree_id` — tree access scoped to request capability. Handler still checks domain constraints.
- **Privileged system handlers**: the tree handler and handlers handler need broad tree access because they implement the infrastructure that provides scoping and handler management. The capability handler needs access to capability storage paths. These handlers receive the default tree or appropriately broad grants.
- **Non-privileged system handlers**: not all system handlers need unscoped access. The types handler only needs to read type definitions at `system/type/*`; the inbox handler only needs access to inbox paths. Being under `system/*` is organizational — it does not imply full tree access.

The principle of least privilege (ENTITY-CORE-PROTOCOL.md §6.8) applies uniformly: every handler, system or domain, SHOULD receive only the grants it needs. The `system/*` path reservation (ENTITY-CORE-PROTOCOL.md §6.2) prevents user-installed handlers from registering there — it does not grant special privilege to handlers that are registered there.

### 8.8 Lifecycle

View trees are ephemeral by default. They are created at dispatch time and destroyed when the request completes.

For long-lived sessions (a peer with an ongoing connection and a stable capability), the view tree can persist across requests with the same capability. The view tree's identity is determined by `(source_tree, capability_hash)` — same capability against the same source produces the same view, so implementations MAY cache and reuse view trees.

If the capability is revoked or expires, the view tree MUST be invalidated. Operations on an invalid view tree MUST return `view_tree_invalid` (403). This ties capability revocation directly to tree access.

### 8.9 View of a View

A handler operating on a view tree that dispatches to another handler with a narrower capability produces a sub-view. The sub-view's filter is the intersection of both capabilities — further narrowed, never widened. Attenuation composes at the tree level.

---

## 9. Handler Registration

```
system/handler := {
  data: {
    name:       "tree"
    pattern:    "system/tree"
    operations: {
      snapshot:  {input_type: "system/tree/snapshot-request",  output_type: "system/tree/snapshot"}
      diff:      {input_type: "system/tree/diff-request",     output_type: "system/tree/diff"}
      merge:     {input_type: "system/tree/merge-request",    output_type: "system/tree/merge-result"}
      extract:   {input_type: "system/tree/extract-request",  output_type: "system/envelope"}
      create:    {input_type: "system/tree/config",           output_type: "system/tree/config"}
      destroy:   {input_type: "primitive/string",             output_type: "primitive/bool"}
    }
  }
}
```

Manifest at pattern path `system/tree`. Index entry at `system/handler/system/tree`.

The core protocol tree handler (§6.3) provides `get` and `put`. This extension adds the operations above to the same handler. All operations target `system/tree`.

**Registered types.** The following types are registered by this extension:

| Type | Bootstrap ID | Purpose |
|------|-------------|---------|
| `system/tree/snapshot` | (existing) | Snapshot root entity |
| `system/tree/snapshot/node` | (needs assignment) | Trie node entity for structural sharing |
| `system/tree/diff` | (existing) | Diff result |
| `system/tree/diff/change` | (existing) | Individual change entry |
| `system/tree/merge-result` | (existing) | Merge result |
| `system/tree/merge-result/conflict` | (existing) | Conflict entry |
| `system/tree/config` | (existing) | Non-default tree config |
| `system/tree/snapshot-request` | (existing) | Snapshot operation params |
| `system/tree/diff-request` | (existing) | Diff operation params |
| `system/tree/merge-request` | (existing) | Merge operation params |
| `system/tree/extract-request` | (existing) | Extract operation params |
| `system/tree/tracking-config` | (needs assignment) | Trie root tracking configuration (§3.4.1a) |

Note: `system/tree/snapshot/node` and `system/tree/tracking-config` require bootstrap type ID assignments in the core protocol's bootstrap type table.

---

## 10. Relationship to Other Extensions

Tree operations write bindings. How those writes interact with other extensions (subscriptions, inbox delivery, compute) is not specified here — those extensions define their own wiring.

Write operations (`put` and `merge`) modify the same location index regardless of which tree they target. For view trees, writes propagate to the source tree — events fire on the source tree. This extension makes no requirements about event behavior — that is the subscription extension's concern, or the implementation's choice.

---

## 11. Capability Requirements

ENTITY-CORE-PROTOCOL.md §6.3 defines the two-level capability model for tree operations using structured grants (ENTITY-CORE-PROTOCOL.md §5.4). This extension follows the same model. All extension operations require two checks:

1. **Handler scope**: The dispatch chain (ENTITY-CORE-PROTOCOL.md §6.5) checks the EXECUTE's literal operation name against `operations` in grants where `handlers` matches the tree handler pattern. When the EXECUTE includes a `resource` field (`system/protocol/resource-target`), `check_permission` also verifies the resource targets against the grant's `resources` scope. Capabilities MUST list the specific operation names they authorize — e.g., `{handlers: {include: ["system/tree"]}, operations: {include: ["get", "snapshot", "extract"]}, resources: {include: [...]}}`.

2. **Path scope** (defense-in-depth): The tree handler maps extension operations to two base permissions (`get` and `put`) for path-level checks via `check_path_permission` (ENTITY-CORE-PROTOCOL.md §6.3). When `resource` is present on the EXECUTE, `check_path_permission` is defense-in-depth (dispatch already verified resource scope). When `resource` is absent, `check_path_permission` is the sole path-level enforcement. The `resources` scope in matching grants determines which paths are accessible.

| Operation | Path Permission | Scope |
|-----------|----------------|-------|
| `snapshot` | `get` | Snapshot prefix |
| `extract` | `get` | Extract prefix |
| `diff` | — | Operates on stored snapshots; no path-level check |
| `merge` | `put` | Each target path (after remapping) |
| `create`/`destroy` | — | Handler scope only; tree handler manages its own internal paths |

**Operation mapping.** The tree handler maps extension operation names to base permissions before calling `check_path_permission`. This mapping is performed by the handler, not by `check_permission` or `check_path_permission` — those functions receive the already-mapped permission name.

```
tree_handler_path_permission(operation, path, capability, local_peer_id):
  ; Map extension operation to base permission for path-level check
  base_permission = map_operation(operation)
  if base_permission is null:
    return ALLOW                  ; No path-level check needed (diff, create, destroy)
  return check_path_permission(base_permission, path, capability, "system/tree", local_peer_id)

map_operation(operation):
  if operation in ["get", "snapshot", "extract"]:  return "get"
  if operation in ["put", "merge"]:                return "put"
  return null                                      ; diff, create, destroy — handler scope only
```

The grant's `operations.include` array lists the literal extension operation names (e.g., `"snapshot"`, `"merge"`). `check_permission` (ENTITY-CORE-PROTOCOL.md §5.2) uses `matches_scope` to check these at dispatch time. The handler then maps to base permissions for `check_path_permission`. Both checks use the same grant entries — `check_permission` uses `operations` and `handlers` scopes, `check_path_permission` uses the mapped permission against the `resources` scope (including `resources.exclude`).

Merge requires `put` authorization on every path it writes. The handler **MUST** verify authorization before applying any writes. If any path is denied, the entire operation **MUST** fail with `capability_denied` (403) — merges are atomic. Implementations MAY optimize by checking a covering prefix when one exists (e.g., `target_prefix` when remapping is active), falling back to per-path checks otherwise.

**View trees and capabilities**: When a handler operates on a view tree, the capability checks described above are performed against the view tree's filtered state. The view tree's scope filter is compiled from the same capability — so the checks are structurally enforced. Operations on a view tree cannot access paths outside the view's scope.

---

## 12. Conformance

### 12.1 MUST

- **HAMT node shape (§3.1)** — `{map: bytes(4), data: [Entry]}`; Entry discriminated by CBOR major type (array = bucket of [key, value_hash] tuples sorted lex; byte string = 33-byte link to sub-node). No `binding` field. Field names `map` / `data` per spec text (chosen for terseness; we are NOT wire-compatible with IPLD HashMap tooling).
- **Bitmap convention (§3.1)** — K-bit unsigned integer where position p is bit p (LSB-indexed); serialized as K/8 bytes big-endian.
- **Parameters pinned (§3.1)** — `bitWidth=5` (K=32), `bucketSize=3`, hash=SHA-256, hash input = `UTF-8-bytes(canonical-normalize(relative_key))`. Implementations MUST NOT expose these as per-tree configuration; MUST NOT carry parameter values on wire.
- **Canonical-form invariant (§3.1)** — IPLD HashMap form: no non-root node may contain (directly or via links) fewer than `bucketSize+1 = 4` reachable entries; on deletion, violations MUST be collapsed and inlined into parent (CHAMP-equivalent). The root node MAY be exempt from the lower bound.
- **Bucket-sort invariant (§3.1)** — `[key, value_hash]` tuples within any bucket MUST be sorted lex by key on both insertion and deletion.
- **Empty-root literal hex (§3.1)** — implementations MUST produce `A2 63 6D6170 44 00000000 64 64617461 80` for the empty-root node. Conformance fixture #1.
- **Single-binding literal hex (§3.1)** — implementations MUST produce the exact byte sequence specified in §3.1 for a single binding at `relative_key = ""` with value_hash `H`. Conformance fixture #2 — the canonical fuzzer seed for SHA-256-input and bitmap-convention conformance.
- **Cross-impl byte-identical-output fuzzer** — implementations MUST verify CHAMP-on-delete correctness via a fuzzer that runs random insert/delete sequences across multiple impls and compares root hashes under M seeds. CHAMP-on-delete bugs are silent under insert-only tests (§3.4.2).
- Deterministic snapshots: same bindings → same trie → same root hash → same snapshot content hash (§3.1, §3.3). Trie node hashing uses ECF.
- Diff computation (§4.3) including output sort (added/removed/changed arrays sorted lex by key)
- Merge algorithm with all three strategies (§5.4)
- Merge is additive — does not remove target paths absent from source (§5.4)
- `dry_run` support on merge
- Prefix placement on merge via `source_prefix` and `target_prefix` parameters
- Snapshot entities contain only `root` — no `prefix` field
- Non-empty prefix validation (must end with `/`)
- Error codes as specified (Appendix A)
- ECF deterministic encoding for snapshot content hashing
- Handler-level capability checks using `get` and `put` grants (§11)
- Atomic merge failure on capability denial (403)

### 12.2 SHOULD

- Non-default trees: create/destroy operations (§7)
- Merge `source_envelope` parameter for direct extract→merge continuation chains (§5.2)
- Extract operation with `paths` filter support (§6)
- View trees for handler isolation (§8). When view trees are implemented, the following are MUST:
  - Scope compilation from capability grants (§8.5)
  - Handler `max_scope` enforcement via dual-check in scope compilation (§8.5)
  - Information hiding: out-of-scope reads return `not_found`, indistinguishable from non-existent paths (§8.2)

### 12.3 MAY

- Ephemeral trees (not persisted across restarts)
- Client-side diff computation (bypassing handler)
- View tree caching by `(source_tree, capability_hash)` pair

### 12.4 Implementation-Defined

- Storage backend for non-default tree bindings
- Maximum number of non-default trees
- View tree lifecycle management (per-request vs session-cached)
- Content store access mechanism for handlers (hash-based reads)

---

## 13. Related Specifications

| Document | Relationship |
|----------|-------------|
| ENTITY-CORE-PROTOCOL.md | Core protocol: entity model, tree handler (`get`/`put`), capabilities |
| ENTITY-CBOR-ENCODING.md | Wire encoding: ECF, deterministic CBOR |
| EXTENSION-SUBSCRIPTION.md | Change subscriptions: tree path watching |
| EXTENSION-CONTENT.md | Content chunking: large snapshots |
| EXTENSION-CONVENTION.md | Placement rules: entity type → tree path mapping (future) |

---

## Appendix A: Error Codes

| Operation | Error Code | Status | Description |
|-----------|-----------|--------|-------------|
| `snapshot` | `invalid_prefix` | 400 | Non-empty prefix doesn't end with `/` |
| `snapshot` | `tree_not_found` | 404 | Referenced tree_id doesn't exist |
| `diff` | `snapshot_not_found` | 404 | Referenced snapshot hash not in content store |
| `merge` | `snapshot_not_found` | 404 | Source snapshot hash not in content store |
| `merge` | `tree_not_found` | 404 | Target tree doesn't exist |
| `merge` | `capability_denied` | 403 | Capability does not grant `put` on target path |
| `extract` | `invalid_prefix` | 400 | Non-empty prefix doesn't end with `/` |
| `extract` | `tree_not_found` | 404 | Referenced tree_id doesn't exist |
| `create` | `tree_exists` | 409 | Tree with this tree_id already exists |
| `create` | `invalid_config` | 400 | Missing `capability` when `source` is present |
| `create` | `capability_not_found` | 404 | Referenced capability hash not in content store |
| `destroy` | `tree_not_found` | 404 | No tree with this tree_id |
| `destroy` | `default_tree` | 400 | Cannot destroy the default tree |
| (any) | `view_tree_invalid` | 403 | View tree's capability has been revoked or is expired |

---

## Appendix B: Structural Sharing via Hash-Keyed HAMT (v4.0)

The snapshot format uses a content-addressed hash-keyed HAMT (IPLD HashMap algorithm, see §3.1) for structural sharing. Each trie node is an entity in the content store. Changing one binding creates new nodes along the SHA-256-bit path from root to the affected leaf bucket — O(log_K N) new entities per change for K=32. All unchanged sub-nodes are shared by hash reference.

This provides root identity (snapshot hash changes if any binding changes) plus efficient change propagation (per-version cost proportional to changes, not to total bindings). Under CHAMP-equivalent canonicalization, the same binding set produces the same root regardless of insertion or deletion history — the property that makes cross-peer convergence work via byte-identical root comparison.

**Prefix queries do NOT navigate the trie under v4.0.** The trie is hash-keyed; bindings under a path prefix are scattered across HAMT positions, not co-located in a structural subtree. Prefix-scoped reads go through LocationIndex (the path-keyed B-tree / BTreeMap layer); the trie's role is Merkle hash structure for content-addressed cross-peer convergence. This division of labor is explicit in §3.7.

**Prior versions (v3.x) used a path-keyed compressed trie** where prefix queries DID navigate the trie. That structure produced O(N) per-Put encoding cost in wide-flat workloads (the cliff Stage 7 fixed) and structurally privileged path locality at the cost of depth-with-path-segment-count. The v4.0 fork trades path-locality for bounded-fanout / bounded-depth / canonicalization-under-history-reorder. See `proposals/implemented/PROPOSAL-TREE-NODE-SHAPE-BOUNDED-FANOUT.md` §3 (why IPLD HashMap, not JMT or other alternatives) + §3.7 (math contract — what's preserved, lost, recoverable) for the full rationale.

The location index remains a flat path → hash map. The trie is a content-addressed projection — built when snapshots are needed (or maintained incrementally per §3.4), stored in the content store, used for diff/merge/transfer. Both structures co-exist; each has a clear role.
