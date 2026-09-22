# System Tree Extension

**Version**: 4.12
**Status**: Active
**Depends**: ENTITY-CORE-PROTOCOL.md (v7.3+)
**Encoding**: ENTITY-CBOR-ENCODING.md (ECF)

**The dependency contract** (per `GUIDE-EXTENSION-DEVELOPMENT.md` §3.3 — *"an implementer scanning
the spec should be able to answer 'what does installing this extension touch' from the header
alone"*). Every entry below is derived from this document's own sections, cited beside it.

**Used by (informative):**
- `EXTENSION-REVISION` — the deepest consumer; builds its version graph on §3 snapshots and §3.4.1
  root tracking (cites §3, §3.3, §3.4.1, §3.4.1a, §11).
- `EXTENSION-REGISTRY` — its enumeration-completeness claim rests on **§3.8's walk contract**, and
  its signed-root browse path on §3.1 / §3.3a / §3.4.2.
- `EXTENSION-NETWORK` — serves and fetches **`system/peer/published-root`** (§3.3a) as the anchor of
  the walk-from-signed-root threat model.
- `EXTENSION-TRANSACTION` — §3.4 / §3.4.1a root tracking for transactional consistency.
- `EXTENSION-SUBSCRIPTION` — §2.2's two index operations as the event source.
- `EXTENSION-CONTINUATION` — `system/tree/path` as a resumption anchor.

**Owned namespaces:**
- `system/tree/` **(closed)** — the whole subtree. Occupants are the §9 registered types: `snapshot`
  and `snapshot/node` · `diff` and `diff/change` · `merge-result` and `merge-result/conflict` ·
  `config` · `tracking-config` · the four operation request types.
- **`system/peer/published-root` (§3.3a) — a single path OUTSIDE this extension's own subtree, and
  the one entry an implementer will not predict.** §3.3a is its normative home and supersedes any
  earlier definition. **The claim is that one path only; `system/peer/` is NOT owned by this
  extension.** The head pointer is bound at `{peer_id}/system/peer/published-root` and nowhere else,
  and it carries **no** peer-id segment — appending one double-qualifies (§3.3a `[MUST, v4.3]`).

**Owned `properties.kind` values:** none. This extension defines no `kind` and claims no row in the
kind-ownership table (`EXTENSION-ATTESTATION.md` §3.2).

**Owned handler ops** — all six are added to the **core protocol's** tree handler, not to a handler
this extension registers (§9):
- `system/tree:snapshot` (§3.2) · `system/tree:diff` (§4.2) · `system/tree:merge` (§5.2) ·
  `system/tree:extract` (§6.1) · `system/tree:create` (§7.2) · `system/tree:destroy` (§7.3).
- `get` and `put` are **core's** (`ENTITY-CORE-PROTOCOL.md` §6.3), not this extension's.

**Extension points exposed:** none. §10 is explicit that how tree writes interact with subscriptions,
inbox delivery and compute is **not specified here** — those extensions define their own wiring, and
this extension makes no requirements about event behaviour.

**Extension points consumed:**
- **The core protocol's tree handler at pattern `system/tree`** (§9) — this extension *extends* a
  handler every conformant peer already bootstraps (`ENTITY-CORE-PROTOCOL.md` §6.2), rather than
  registering its own. ⚠ **Installing it is therefore not an ordinary handler registration**, and
  an installer that treats it as one meets `SDK-OPERATIONS.md` §11.6's pattern-collision refusal.
  This is a known open seam, not a defect in this document.
- **The core protocol's bootstrap type table** — `system/tree/snapshot/node` and
  `system/tree/tracking-config` **require bootstrap type ID assignments in core** (§9). This is the
  first instance of an extension reaching into the core type table, and it is stated in §9 rather
  than assumed.

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

> **`get` does NOT require a `resource`, and the two empties are not the same empty `[MUST]`.** ENTITY-CORE-PROTOCOL.md §3.3 delegates *which* operations require one to each operation's own specification, and this table is `get`'s: an **empty** path is a specified input and its answer is the root listing. But a **genuinely absent** `resource` and a `resource` that is **present with an empty effective list** (§5.2 — the caller named a target and excluded it) are different requests, and only the first asks for a listing. The second **MUST** be refused **`400 path_required`**; serving it the root listing answers a request for one excluded path with a listing of the tree. `put` requires a resource and answers `path_required` for both.
>
> **`get` is a BROAD-RESULT operation and §2.2a declares the whole set `[MUST]` (v4.11; the classification is three-valued as of v4.12).** ENTITY-CORE-PROTOCOL.md §3.3 requires each resource-optional operation to state whether its absent case is a **broad result** (refuse the self-excluded case) or an **optional filter** (answer it empty). **This extension's eight operations are declared in §2.2a** — the field is not inferable from a handler's source, and three independent implementations inferred three different answers from this paragraph alone.

### 2.2a Resource requirement, per operation (normative, v4.12)

ENTITY-CORE-PROTOCOL.md §3.3 delegates *which* operations require a `resource` to each operation's own specification, and — since 0.8.2.25 — requires every resource-**optional** operation to declare which of two shapes it has. **This table is that declaration for all eight operations. It is normative, and it is the field an implementation cannot derive from its own source.**

**The classification is three-valued, because §3.3's test is** (v4.12): an operation **requires** a `resource` where the path is its subject; is **resource-optional** where the `resource` selects or narrows, which is the case §3.3 obliges to declare BROAD or OPTIONAL-FILTER; or **targets no entity binding at all**, which is §3.3's own carve-out and for which neither obligation arises.

| Operation | `resource` | Absent-case answer | `targets:[P] exclude:[P]` |
|---|---|---|---|
| `get` (§2.2) | optional | **BROAD** — the root listing | **400 `path_required`** |
| `snapshot` (§3.2) | optional | **BROAD** — prefix `""`, the **whole tree** | **400 `path_required`** |
| `extract` (§6) | optional | **BROAD** — an envelope of **every bound entity** under the prefix | **400 `path_required`** |
| `put` (§2.2) | **required** | — | 400 `path_required` (both empties, §3.3 unchanged) |
| `merge` (§5.2) | **required** | — | 400 `path_required` (both empties) |
| `diff` (§4.2) | **no path subject** | — | not read; proceeds on handler scope |
| `create` (§7.2) | **no path subject** | — | not read; proceeds on handler scope |
| `destroy` (§7.3) | **no path subject** | — | not read; proceeds on handler scope |

**`no path subject` is ENTITY-CORE-PROTOCOL.md §3.3's carve-out — the operation targets no entity binding, so a `resource` would have nothing to name.** Neither the `path_required` requirement nor §3.3's BROAD/OPTIONAL-FILTER declaration obligation arises for these three rows. **The test is not editorial: §11's `map_operation` already computes it** — an operation for which `map_operation` returns `null` has no path subject, and §11's `tree_handler_path_permission` returns `ALLOW` on exactly that branch. The two statements are one fact, and **§11's block is the authority; this row is a reading of it.** The handler **MUST NOT** refuse one of these three on `resource` grounds. Dispatch-level `check_permission` (ENTITY-CORE-PROTOCOL.md §5.2) is unchanged and unaffected — whatever it does with a `resource` it was handed, it has already done before the handler runs.

⚠ **`diff`'s §4.2 sentence answers a different question than this column asks.** *"The `resource` field is optional; when omitted, handler-scope authorization (§11) suffices"* is about **authorization**. This column classifies **subject selection**. `diff` binds nothing — its operands are two `system/hash` values in `params`, and its result is identical whether a `resource` is present or not. The same word answers both questions, which is why a two-valued column read the sentence as the wrong answer to the wrong one.

**The three BROAD rows are ordered by blast radius and `extract` is the widest** — it returns the entities themselves, where `get` returns a listing of paths and `snapshot` returns a root hash. A self-excluded `extract` served its absent case hands the caller every entity in the tree in response to a request naming one path the caller then excluded.

⚠ **`snapshot` is the row §8.4 exempts from the path-level check**, on the argument that a view-tree snapshot captures only what the handler is authorized to see. **That exemption's premise is falsified by a request the caller writes itself** if the self-excluded case is served as absent: the diff-against-empty then carries the excluded key and its content hash. The exemption is unchanged and remains correct; it is not a licence to skip this row.

⛔ **A guard keyed on *"this handler reads `resource`"* does not reach this set.** `snapshot`, `diff`, `merge`, `create` and `destroy` take their path from `params`, so a handler can ignore the field entirely and still owe the refusal — and a handler that reads it may still **fall back** to a `params` path when the effective set is empty, which is the same defect with an extra step. Guard at the handler's **entry**, ahead of the operation dispatch, so an operation added later inherits it.

⚠ **And the refusal is only reachable if the dispatch boundary preserved the discriminator** — ENTITY-CORE-PROTOCOL.md §3.3's non-lossy-projection rule. An implementation that narrows `resource.targets` to the effective set before the handler runs has already turned `targets:[P] exclude:[P]` into the absent case, and **every row in this table becomes dead code behind a green unit test**. Only a drive across a socket distinguishes the two.

Everything this extension provides is composition of these primitives.

`put` is a data-plane primitive. It writes a binding and fires the emit cascade (SYSTEM-COMPOSITION.md §1). It does not go through other handlers, does not validate against other extensions' entity types, and does not perform custom logic.

**"Does not validate" is about *semantics*, never about *structure*.** `put` still admits its argument as a `core/entity` and checks the carried hash before it writes anything — ENTITY-CORE-PROTOCOL.md §6.3's two-step admission, and Appendix A's `put` rows are its codes. What `put` declines to do is ask whether `data` is a well-formed instance of the type `type` names. A peer that reads this paragraph as licence to store an unadmitted value writes an entity no typed peer can decode, under a hash nobody agreed to. Extensions that need validation, coordination, or any processing before a write lands SHOULD expose a named handler operation (e.g., `revision/config`, `compute/install`) and gate the underlying path via capability grants so callers route through the operation rather than calling `put` directly. See SYSTEM-COMPOSITION.md §2.9 for the rubric on when a named operation is appropriate vs direct `put`.

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
      ;   CBOR major type 2 (byte string)  → Link: system/hash of a sub-node entity
      ;     (format byte + digest; length follows the format byte — 33 B under SHA-256,
      ;      49 B under SHA-384. NEVER a fixed width: SPECIFICATION-FORMAT.md 8.4.5.
      ;      The Entry discriminator is the CBOR MAJOR TYPE, not the length, so a
      ;      longer digest changes nothing about decoding.)
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

> **This fixture is `content_hash_format = 0x00` (ECFv1-SHA-256) `[qualification added 2026-08-10]`.** The `58 21` header below is bytes(33) and the routing digest is SHA-256 — **both are properties of this fixture's format, not of the node shape.** A SHA-384-home peer produces the identical *structure* with `58 31` (bytes(49)); the Entry discriminator is the CBOR **major type**, never the length (§3.1). The "**All** implementations MUST produce this exact byte sequence" below is therefore scoped to a SHA-256-home peer — read unqualified it would forbid a conformant SHA-384 peer from producing anything at all (`SPECIFICATION-FORMAT.md` §8.4.5).

Where: `A2` map(2); `63 6D6170` "map"; `44 10000000` bytes(4) bitmap; `64 64617461` "data"; `81` array(1) (one entry in data); `81` array(1) (bucket with one tuple); `82` array(2) (`[key, value_hash]`); `60` empty text string ""; `58 21` bytes(0x21 = 33); `<H>` 33-byte value hash. **All implementations MUST produce this exact byte sequence given identical canonical-normalize on the empty-string relative_key.** This is conformance fixture #2 — the canonical fuzzer seed for catching SHA-256-input ambiguity (relative-key vs absolute-path) and bitmap-convention ambiguity at fuzzer-touch time.

**Trie bindings use prefix-relative keys.** Bindings in the trie are keyed by path segments relative to the prefix the trie was built under. These are not paths — they are structural keys within a subtree, produced by trimming that prefix in **absolute form** from the stored path:

```
relative_key = trim_prefix(path, absolute_prefix)
```

`absolute_prefix` is the configured prefix (§3.4.1a) resolved to absolute form. **The three admissible shapes, and what each yields `[MUST; ruled 2026-08-08]`:**

| Configured `prefix` | `absolute_prefix` | `relative_key` for `/{peer_id}/system/attestation` |
|---|---|---|
| `"system/"` (peer-relative subtree) | `/{peer_id}/system/` | `attestation` |
| `"/{peer_id}/"` (peer-qualified — the peer's whole namespace) | `/{peer_id}/` | `system/attestation` |
| `"/"` (universal tree) | *(empty — the trim is a **no-op**)* | `/{peer_id}/system/attestation` |

**The universal case is a no-op trim and its keys are fully qualified.** The universal tree spans every peer's namespace (`ENTITY-CORE-PROTOCOL.md` §1.4 — a peer's root is the set of peer-ids it holds), so "the peer_id" is not a single value there: trimming the local peer's id would leave every other peer's keys qualified and produce a **mixed key space**, which is worse than either uniform answer.

> **Superseded formula (do not reintroduce).** This rule previously read `relative_key = trim_prefix(path, "/" + peer_id + "/" + operation_prefix)`. That form is **ill-defined for the universal tree** — it yields `"/" + peer_id + "/" + "/"`, i.e. `/{peer}//`, against which an implementation silently tracks nothing (reported by `entity-core-rust`, 2026-08-08) — and it double-qualifies the peer-qualified shape, whose prefix already contains the peer-id. §3.4.1a's *"the universal case collapses to the empty string"* governs the **storage path** in §3.4.1's substitution table, **not** this formula; the two were conflated.

Full paths are reconstructed by the consumer: `absolute_prefix + relative_key`. A non-empty prefix **MUST** end with `/`.

**Where the prefix comes from, and the one entity that carries it.** For `snapshot`, `extract`, and `merge` the prefix is an **operational parameter of the request** and is deliberately *not* stored in the snapshot entity (§12.1). A **published root** has no request — it is fetched by a consumer who was not present at publish time — so it **MUST** carry its prefix in the entity (§3.3a). Do not generalize the published-root field back onto the snapshot entity; the two differ precisely in whether a request channel exists.

**The SHA-256 input is the relative_key, not the absolute path.** Two impls hashing different forms of the path produce different routing positions and silently divergent root hashes. Canonical-normalize follows the existing rules in the spec (see §5.4 and ENTITY-CORE-PROTOCOL.md §5.4); the resulting `UTF-8-bytes(canonical-normalize(relative_key))` is the SHA-256 input for routing per §3.3.

> **This rule is unchanged by the universal case, and is not an exception to it.** Under `prefix: "/"` the `relative_key` **is** the absolute path — because the trim is a **no-op**, not because the rule was skipped. The hash input is still the relative_key in every case. Conformance fixture #2 (single binding at `relative_key = ""`) is unaffected.

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

> **`snapshot` is resource-OPTIONAL and BROAD-RESULT `[MUST]` (v4.11; §2.2a).** Its absent-case prefix is `""` — **the whole tree** — so a `resource` that is **present** with an empty effective list (`targets:[P] exclude:[P]`, ENTITY-CORE-PROTOCOL.md §5.2) **MUST** be refused **`400 path_required`** and **MUST NOT** fall back to the absent case or to a `params`-supplied prefix. An empty prefix is a valid *absent-case* input and is not a valid answer to a self-excluded request. **This is the widest form of the rule after `extract`**, and §8.4's view-tree exemption does not reach it (§2.2a).

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

### 3.3a The published root — `system/peer/published-root`

*This section is the normative home for `system/peer/published-root`; it supersedes any earlier definition. Rationale and provenance are in `PROPOSAL-PUBLISHED-ROOT-PREFIX-AND-REPUBLISH`.*

A **published root** is a signed, mutable pointer to a trie root that a publisher commits to serving. It is the anchor of the walk-from-signed-root threat model: a consumer fetches it, verifies the signature, and walks the hash-chain from `root_hash` — never trusting paths the host claims outside that chain.

```
system/peer/published-root := {
  fields: {
    peer_id:      {type_ref: "system/peer-id"}      ; whose root this is (pubkey IS identity, V7 §1.5)
    root_hash:    {type_ref: "system/hash"}         ; BARE — the committed trie root
    prefix:       {type_ref: "system/tree/path"}    ; REQUIRED. The prefix these trie keys are
                                                    ; relative to, per §3.3. MUST end with "/".
                                                    ; "/" designates the universal tree.
    seq:          {type_ref: "primitive/int"}       ; monotonic freshness; MUST increase per republish
    published_at: {type_ref: "primitive/int"}       ; ms since Unix epoch, UTC
    predecessor:  {type_ref: "system/hash", optional: true}
                                                    ; BARE — prior published-root content_hash;
                                                    ; absent only on the first publish
  }
}
```

**Head-pointer path — `{peer_id}/system/peer/published-root`, and it carries NO peer-id segment `[MUST, v4.3]`.** The peer's current published root is bound at that path and nowhere else. **The type path is the whole path; appending the peer-id to it double-qualifies**, because the peer namespace is already the first segment of every absolute path (`ENTITY-CORE-PROTOCOL.md` §1.4). `/{peer}/system/peer/published-root/{peer}` names the peer twice, and the stated reason for doing so — *locating the current root for peer X without enumerating a content-addressed type's siblings* — is what the peer prefix already provides.

> **This path was never pinned, and three implementations picked `[found by cross-impl publish/consume, v4.3]`.** Two landed the form above; one appended `/{base58_peer_id}` and recorded in its own source that doing so *"supersedes the legacy `signed_pointer` string"* — **a supersession this corpus never ratified and holds no record of.** The divergence is invisible locally and fatal at the seam: the extra segment turns the pointer's path into a **directory**, so a consumer reading the pointer as a file fails at hop 0, before reaching any of the surfaces that do agree. **An unpinned path is not a free choice — it is a divergence with a delay**, and this one sat behind three otherwise-aligned surfaces (content sharding, bare-hashable bodies, two-hop signature keying) that a cross-impl run only reached after the front door was fixed.

**Signature carriage.** Per `ENTITY-CORE-PROTOCOL.md` §5.2 target-matching at the invariant-pointer path `system/signature/{hex(published_root.content_hash)}`. The `MANIFEST_GET` envelope includes that signature entity under `envelope.included`. This entity carries **no `refs:` block** (V7 refless contract, as REGISTRY §3 and DISCOVERY §2.1).

**Verification.** The signature MUST verify against `peer_id`'s public key (derived locally from the Base58 form, V7 §1.5). `seq` monotonicity is the rollback defense: a consumer MUST reject `seq` lower than one it has already accepted for that `peer_id`.

**`published_at` is a signed lower bound on the artifact's age, and MUST NOT be read as evidence of origin freshness.** It says *the publisher signed this root no earlier than T*. It does not say the origin is serving the publisher's newest root, and no field on this entity can: **a publisher that has not republished and an origin withholding a newer root produce byte-identical results at the consumer** — same signature, same `seq`, same `published_at`. A consumer that treats a recent `published_at` as "this is current" has verified authenticity and inferred freshness, which is precisely the step this entity cannot support. What a verified root establishes is **authenticity and non-rollback as of the root received**; freshness against a *hostile* origin is not obtainable from the artifact, and against a *cooperating* origin it is bounded by `EXTENSION-NETWORK.md` §6.5.6's 30 s convergence MUST — which binds the publisher's republish cadence, not the origin's serving behavior. Consumer-facing display wording for the three honest states is `guides/GUIDE-SERVING-MODE.md`, not here.

> **One convention, stated here and cited elsewhere.** `EXTENSION-NETWORK` §6.5.1c's
> `system/peer/transport-set` is the other *signed statement a peer makes about itself*, and it
> deliberately uses **the same signature carriage** (`system/signature/{hex(content_hash)}`, verified
> against the peer id derived from its Base58 form) and **the same `published_at` rule**, verbatim as
> stated above. The two records answer different questions — *what is my tree* and *where am I* — and
> a peer that publishes both binds the transport-set into the tree, so **the root's `seq` is the one
> freshness authority and there is not a second `seq` stream.** Two records answering two questions
> are not a fork; forking would be two roots for one tree, which is what destroys rollback detection.

**`prefix` is REQUIRED, and this is the field the type was missing `[MUST; ruled 2026-08-08]`.** Without it a consumer holds `relative_key`s and not the operand needed to reconstruct a single absolute path — §3.3's reconstruction rule is unsatisfiable, and the hash-chain walk that is the whole security model is underspecified at its first step. It is required rather than optional-with-a-default because **any default would have to be one of §3.3's three shapes**, silently promoting one implementation's convention to "the answer you get for saying nothing" — the same asymmetry, in a form that is harder to see.

**It lives on the root, not on the manifest.** The root is the *signed artifact*, and `prefix` is what makes its contents interpretable. Putting the key convention in a separately-signed entity would mean a consumer that has verified the root's signature still cannot read it without verifying a second chain — and the two could disagree, with no rule for which wins. **Keep the interpretation of a signed artifact inside the signed artifact.**

**Two conformant publishers may legitimately publish different extents.** `prefix` is what distinguishes them: a publisher at `prefix: "system/"` bounds its publication to a strictly smaller region than one at `prefix: "/"`. A consumer MUST read that **bound** from `prefix` and MUST NOT infer it from the publisher's identity or from what it happens to find. **The bound is an upper limit on what the trie may contain, never a promise about what it does contain** — see the clarification below.

**`prefix` bounds the publication's scope; it is NOT a completeness claim `[clarified v4.2]`.** It says *nothing under this trie lies outside `prefix`*. It does **not** say every binding the publisher holds under `prefix` is in the trie, and a consumer MUST NOT read it that way — the negative-scoping MUST below is the operative rule, and it holds regardless of how broad `prefix` is. A publisher MAY declare `/{peer_id}/` and publish a small subset; that is a narrow claim honestly made, not a false one.

> **A completeness MUST landed here in v4.1 and is WITHDRAWN in full `[v4.2]`.** It read *"a publisher MUST publish completely under `prefix`; disjoint areas get separate roots."* It was refuted on four independent grounds, each verified against source before withdrawal.
>
> **It was unsatisfiable by construction, and this is the part worth keeping as an invariant.** Publishing binds two entities *under the peer prefix*: the head pointer at `/{peer_id}/system/peer/published-root` and the signature at `/{peer_id}/system/signature/{hex(H)}`. **H is the hash of the entity committing to `root_hash`, so putting either in the trie requires a hash fixed point** — committing to a trie that binds H changes `root_hash`, which changes H. **An anchor cannot sit inside the tree it anchors.** With `prefix: "/{peer_id}/"` — the default in every engine — the rule failed on every publisher's first publish, including the reference implementation cited when it was written.
>
> **Its justification was revoked by the paragraph directly below it, in the same commit.** The stated harm was *"a consumer receives a clean negative and is entitled to read it as authoritative."* The negative-scoping MUST forbids exactly that, normatively, to every consumer. **The harm it existed to prevent was already closed, four lines away, by the better mechanism** — so it imposed a restructure on every publisher and delivered no consumer anything. It also **contradicted** that paragraph's explicit *"a publisher MAY serve different subsets to different audiences from the same prefix"*, since a per-audience subset is incomplete under its prefix by definition. And *"disjoint areas"* was never defined: `sites/a/` and `sites/b/` are disjoint in the same sense as `sites/` and `system/registry/`, so any literal reading degenerates to one root per key.
>
> **The general form: a completeness property cannot be asserted by the artifact whose own anchor lives inside the region.** Signing what you published cannot prove you published everything you hold — which the paragraph below already said, and which is why nothing replaces this rule.

**A negative answer is scoped to the root that produced it `[MUST]`.** `not_found` against a published root means *"this root does not bind K"* — never *"the publisher does not bind K."* The distinction is not pedantic: a publisher MAY serve different subsets to different audiences from the same prefix, and **no field on this entity can tell a consumer which subset it received.** A consumer MUST NOT present a negative from a published root as authoritative non-existence, and a surface that reports "not bound" MUST scope it to the root it asked. This is a limit of static publication, not a defect to be closed by a mechanism — signing what you published cannot prove you published everything you hold.

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

This matches the single-pointer convention used elsewhere in the system (`system/revision/head/{prefix}` points directly at a `system/revision/entry`; `system/revision/branches/*` and `system/revision/tags/*` point directly at version entities). Implementations MUST NOT interpose a `system/hash`-typed wrapper entity.

**Operational state.** Track the root in peer-local state (not in the entity tree). This avoids the tree write for the root update but makes the root non-discoverable and non-syncable.

For prefixes with active versioning, the tree path approach is RECOMMENDED — the root is useful for other extensions and for debugging. For internal/ephemeral tries, operational state is sufficient.

**Invalidation.** The tracked root MUST reflect the current tree state. After a `tree.put` that affects a tracked prefix, the root must be updated (incremental) or invalidated (on-demand). Stale roots produce incorrect snapshots.

**History tracking.** Root tracking paths (`system/tree/root/*`) are subject to normal history recording. Implementations SHOULD NOT exclude them by default. The history chain preserves prior trie roots that would otherwise be unrecoverable — each transition records the previous root hash, the replacement root hash, the timestamp, and the chain_id of the write that triggered the update. This provides inter-commit rollback granularity: the version DAG (EXTENSION-REVISION.md) captures trie roots at commit points, while history captures every intermediate root between commits. Users MAY exclude these paths via history configuration if write volume is a concern.

**Revision exclude.** Root tracking paths `system/tree/root/*` MUST be excluded from versioned bindings when the root tracking path falls under a versioned prefix. The trie root hash depends on all bindings under the prefix — including it in the versioned bindings creates a circular dependency (the root hash would include itself). This exclusion is not a loss: the version entry's `root` field already captures the trie root at each commit point. Implementations using the tree-path approach SHOULD include `system/tree/root/*` in the revision configuration's `exclude` patterns (EXTENSION-REVISION.md §2.4). Alternatively, use operational state storage to avoid the circularity entirely.

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
2. Light-client prefix proofs. **We ARE actively building light-client-class consumers** — the entity-browser-rust peer per `PROPOSAL-EXTENSION-BRIDGE-HTTP`; browser-peer + IndexedDB target per `PROPOSAL-PERSISTENCE-AS-EMIT-CONSUMER` §4.5. Their current pattern is per-blob hash-verify (preserved under hash-keyed: each binding gets an inclusion proof against snapshot root regardless of position). Compact prefix proofs would be additional, not foundational; the §3.7.2 sidecar extension restores them if/when per-blob pattern proves insufficient. This MIGHT-WANT case has a viable per-blob answer today; the sidecar path is open. (v4.4 errata corrects the v4.3 overclaim "we do not ship light clients" — we do, or will.)
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

### 3.8 The walk contract — the absence of a node is never an answer

A trie fetched from a counterparty is a structure whose nodes arrive one at a time from a party that
chooses what to serve. Every operation that walks one — enumeration, lookup, diff, extract, a
registry browse — depends on the rule below. **It is one rule, not a mode set.**

**R1 — a node that did not resolve is a failure, never an answer `[MUST]`.** Every trie operation
MUST distinguish two outcomes and MUST NOT return the first when it observed the second:

- **(a)** a node that **resolved** and did not contain the sought key → `not_found`, *an answer*;
- **(b)** a node the walk needed and **could not resolve** → **`incomplete_walk`**, *a failure*.

This covers the seam at both depths. At **enumeration** depth a withheld interior node stops
silently shortening the result. At **lookup** depth — including a revocation check keyed by hash —
*"no such key"* and *"that key's node was not served"* stop being the same value.

**The reason it is a MUST and not an implementation concern:** the two cases are byte-identical at
the consumer, so an origin that omits a node produces a **correct, complete, shorter** answer that
verifies. The signature still checks out, because it commits to the root hash and the root hash is
intact; every binding returned is genuine. Nothing distinguishes an honest short answer from a
hostile one except this rule.

**R2 — completeness is structural, never heuristic `[MUST]`.** A walk MUST derive a node's children
by decoding the node (§3.1) and reading its declared entries — every bucket tuple's `value_hash` and
every link. It MUST NOT derive them by scanning node bytes for hash-shaped windows, and it MUST NOT
enqueue a child conditionally on that child already being present. **A byte scan cannot tell *not
declared* from *declared and withheld*, which is the entire distinction R1 rests on** — and a walk
that enqueues only children it can already load makes the missing set unreachable by construction,
so a guard written against R1 can never fire.

**R3 — the error names the parent, not only the child `[MUST]`.** `incomplete_walk` MUST carry both
the unresolvable child hash (`missing_hash`) and the hash of the node that declared it
(`declared_by`). The declaring node is the **cut point**, and the cut point is what bounds which keys
are unaccounted for; a consumer holding only the child hash cannot locate the cut, and so cannot say
what it failed to see.

**R4 — a partial walk is opt-in and MUST NOT be presented as complete `[MUST]`.** The tolerant
behaviour stays reachable for the case it is correct for: a walk over a store **the walker itself
populated** — a partial local store, a peer mid-sync, the `included` semantics of §6.2. Such a walk
is requested explicitly, returns a subset, and MUST NOT be reported as a complete enumeration.
**The discriminator is the trust boundary, not a flag:** a walk over bytes a counterparty chose to
serve has no partial mode.

**R5 — an incomplete walk is terminal, not retryable `[MUST]`.** `incomplete_walk` is the origin
failing to produce what its own signed root declares; retrying grants a withholding origin unbounded
attempts and turns it into a hang. A **transport** failure while fetching is a different condition
and is separately retryable. The discriminator: *the node resolved and its declared child did not
exist* is terminal; *the fetch itself failed* is retryable.

**What R1 asks of a consumer is reachable with what it already holds.** The walk needs the node bytes
it is already fetching — and their absence *is* the signal, observed locally — and the declaring
parent's hash, which the consumer holds because it just decoded that parent to learn the child
existed. No new index, no second request, no key material, and no cooperation from the origin.

**Scope — what this does not claim.** It does not prevent withholding; it makes withholding
**visible** where it was a silent short answer, which is strictly better and is not the same thing.
It does not make a full walk cheap: O(N) fetches against a remote origin is not a first-paint
primitive, and a served listing remains the right first-paint artifact (`EXTENSION-REGISTRY` §6a.3a).
And it does not close the key-space question — §3.3a's negative-scoping MUST is still the operative
rule for *"the publisher does not bind K."* **R1 is about nodes the root declares; §3.3a is about
keys it never carried.**

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

walk_entry_collect(kind, entry, declaring_node_hash):
  ; Walk an entry (bucket or link) and emit `kind` for every (key, value_hash) reachable.
  ; §3.8 R1/R2/R3: children come from the node's DECLARED entries, and a declared child
  ; that does not resolve is a failure carrying its cut point — never a shorter answer.
  changes = []
  if is_bucket(entry):
    for tuple in entry:
      changes.append({kind: kind, key: tuple.key, hash: tuple.value_hash})
  else:  ; link
    sub_node = content_store.get(entry.hash)
    if sub_node is null:
      return error("incomplete_walk", {
        missing_hash: entry.hash,
        declared_by:  declaring_node_hash
      })
    for p in iterate_set_bits(sub_node.map):
      changes.append_all(
        walk_entry_collect(kind, entry_at_position(sub_node, p), entry.hash))
  return changes

count_leaf_bindings(node, node_hash):
  ; Count all entity bindings in the trie (for unchanged count computation)
  count = 0
  for p in iterate_set_bits(node.map):
    entry = entry_at_position(node, p)
    if is_bucket(entry):
      count += len(entry)
    else:  ; link
      child = content_store.get(entry.hash)
      if child is null:
        return error("incomplete_walk", {
          missing_hash: entry.hash, declared_by: node_hash
        })
      count += count_leaf_bindings(child, entry.hash)
  return count
```

**The `null` branches above are the whole of §3.8 R1 in the diff path, and they are what three
engines each omitted.** A `continue`, an absent `else`, or an empty-node fallback at these sites
turns a withheld node into a shorter diff that reports success. The recursion also threads the
**declaring** node's hash rather than only the missing child's, because R3's cut point is the parent.

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

**A malformed `paths` entry is `400 invalid_path` for the whole request; an ABSENT one is silently omitted `[MUST]` (v4.9).** These are different inputs with different remedies and they must not be collapsed:

| the entry is | answer |
|---|---|
| **absent** — a well-formed relative path that binds nothing | **silently omitted** from the result. Unchanged, and it is what the filter is *for*: a caller extracting after a diff is asking which of these exist |
| **malformed** — a control character, an empty segment (`//`), a leading `/`, or anything that does not canonicalize to a valid path under `prefix` | **`400 invalid_path`**, and **no** partial result |

> **Why reject rather than omit, when both are safe.** Silent omission is defensible — a malformed path binds nothing, and the boundary already treats it as absent — but it makes the two cases **indistinguishable to the caller**: a request for ten paths comes back with seven and the caller cannot tell whether three are missing or three were garbage. *"Fix your path"* is a different instruction from *"that binding does not exist"*, and the code is what selects the remedy. It also matches the disposition of a malformed **resource** target, so one operation does not carry two rules for the same defect on two channels.
>
> **And it is a `[MUST]` rather than a preference because the two readings are cross-peer observable**: one conformant peer answers `400`, another answers `200` with a short result, and a client written against the second breaks against the first. Both readings are safe; only one can be the contract.
>
> ⚠ **The rejection is the HANDLER's answer, not the boundary's.** `ENTITY-CORE-PROTOCOL` §5.4 (0.8.2.21) requires the store boundary to stay **total** independently — a read at an invalid path is absent, never a panic or an assert — because `paths[]` is a caller-controlled array reaching a path boundary through `params`, which no resource-target pre-validator sees. **The 400 is what the caller is told; the total boundary is what makes the operation safe if some future call site forgets to tell them.** Neither substitutes for the other.

> **Incremental "since" transport lives in the revision layer.** Transporting only what changed between two versions is a *revision* concern (its inputs are version hashes, and version → trie-root dereference is the revision extension's knowledge). See `EXTENSION-REVISION.md` §4.4.19 `fetch-diff`. A `since` parameter was briefly added to `tree:extract` (v3.14) and **withdrawn in v3.15** — placing it here forced a tree op to dereference revision version entries, a layering violation. `tree:extract` filters by `paths` only.

**Returns:** `system/envelope`

> ⭐ **`extract` is resource-OPTIONAL and BROAD-RESULT, and it is the WIDEST instance of the rule `[MUST]` (v4.11; §2.2a).** A `resource` that is **present** with an empty effective list (`targets:[P] exclude:[P]`, ENTITY-CORE-PROTOCOL.md §5.2) **MUST** be refused **`400 path_required`**. It **MUST NOT** fall back to the absent case, and — the arm that actually bites — **MUST NOT fall back to the `params`-supplied `prefix`**, whose default is `""`: an empty prefix is a *valid* input in its own right, so a fallback silently succeeds and validates.
>
> **Where `get` leaks a listing of paths and `snapshot` leaks a root hash, `extract` returns the entities themselves** — every bound entity under the prefix, bundled, in response to a request naming one path the caller then excluded. The `paths` filter above does not save it: `paths` is an *intersection* applied after the prefix is chosen, and a caller who supplies no `paths` gets the whole closure.

### 6.2 Algorithm

```
execute_extract(tree, content_store, prefix, paths):
  if prefix != "" and not prefix.ends_with("/"):
    return error("invalid_prefix")

  ; Collect bindings — only what's needed
  bindings = []
  if paths is not null:
    ; Validate EVERY entry before reading ANY (v4.9). paths[] is a
    ; caller-controlled array arriving through params — a channel the
    ; resource-target pre-validator never sees — so this is the operation's
    ; own admission step (ENTITY-CORE-PROTOCOL §5.4).
    for path in paths:
      if not is_valid_relative_path(path):
        return error(400, "invalid_path",
          "extract paths[] entry is not a valid relative path")
    ; Filtered: read specific paths directly. A well-formed path that binds
    ; nothing is ABSENT and is silently omitted — that is what the filter is
    ; for, and it is NOT the malformed case handled above.
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

**An extract's envelope is complete against its own root `[MUST, v4.6]`.** This holds for a filtered
extract exactly as for a full one, and it is a property of the algorithm above rather than an extra
obligation: `root_hash = build_trie(bindings)` is built over **precisely the selected bindings**, and
`include_trie_nodes` then bundles every node reachable from that new root. A `paths` filter therefore
produces a **smaller trie**, not a partial view of a larger one — so there are no "unfiltered subtree
nodes" for it to leave out. **Walking the returned root MUST NOT reach an entity outside `included`.**

> **A sentence licensing the opposite stood here until v4.6 and is deleted.** It read *"unfiltered
> subtree nodes MAY be omitted"*, which describes navigating the original trie by path — the v3.x
> path-keyed model §13 records as replaced. Under hash-keyed routing paths do not navigate the trie
> and the algorithm four lines above **rebuilds**, so at best the sentence was vacuous. **At worst it
> licensed building `extract` as a view rather than a rebuild**, letting two conformant peers emit
> envelopes that disagree — the cross-peer `MAY` divergence this corpus pins rather than leaves. **It
> is also the sentence that made a tolerant walk look defensible:** if incompleteness can be
> legitimate, tolerating it is reasonable. It cannot be, so it is not (§3.8).

**Publishing a subset is re-rooting, not filtering `[v4.6]`.** The same property is what a publisher
relies on, and it is worth stating where a publisher will read it: a publisher serving part of its
tree does **not** serve a filtered view of one large trie. It builds a trie over exactly what it
publishes, with keys relative to the tracked prefix (§3.4) and that root signed and tracked per
§3.3a. **So there is no such thing as legitimate incompleteness *within* a published root** — which
is why §3.8 admits no tolerance at a trust boundary — while what the publisher chose to *put* in that
root remains unconstrained, per §3.3a's negative-scoping MUST.

A **partial walk** over a store the walker itself populated is the one tolerant case, and it is §3.8
R4: opt-in, returning a subset, and never presented as a complete enumeration.

This means a full extract includes the complete trie plus all data entities. A filtered extract includes a complete smaller trie.

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

> ⚠ **This exemption is about the caller's AUTHORIZATION, never about which request was made `[MUST]` (v4.11).** `snapshot`'s premise here — *it captures only what the handler may see, so a path-level check adds nothing* — holds for a genuinely absent `resource` and is false for a **self-excluded** one, because the caller wrote the exclusion themselves and the view filter knows nothing about it. §2.2a's refusal is therefore **not** covered by this row and MUST be applied before it: a self-excluded `snapshot` served as absent puts the excluded key and its content hash into a diff-against-empty, using the exemption as the carrier. **Two different questions — *may this caller see it* and *did this caller ask for it* — and only the first is authorization.**

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

> **This reduction is scoped to `max_scope`, and it does NOT reduce `ENTITY-CORE-PROTOCOL.md` §6.8 row 1 `[MUST]`.** That row's ceiling is **the executing handler's own grant**, which is mandatory at every dispatch (§6.8's dispatch-time grant validation: a missing or invalid grant is `permission_denied`, and the dispatcher MUST NOT fall back to the caller's capability). `max_scope` is the optional **view-tree** field; the handler grant is not optional, so the intersection §6.8 requires on a path serving a caller's request is always owed. A view tree is one mechanism for enforcing that intersection structurally — it is not the condition under which the intersection applies.

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

The principle of least privilege (ENTITY-CORE-PROTOCOL.md §6.8) applies uniformly: every handler, system or domain, SHOULD receive only the grants it needs. Being registered under `system/*` grants no special tree privilege: the prefix is organizational, and a handler's authority is its grant. Which callers may install a handler at a `system/*` path is decided by the standard capability check on the install path (ENTITY-CORE-PROTOCOL.md §6.2, §6.13), not by the prefix.

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

⭐ **This block is also the authority for §2.2a's third column value (v4.12).** An operation for which `map_operation` returns `null` **targets no entity binding** — ENTITY-CORE-PROTOCOL.md §3.3's carve-out — so it neither requires a `resource` nor owes a BROAD/OPTIONAL-FILTER declaration. §2.2a reads that classification off this function rather than restating it; **an operation added to or removed from this `null` arm changes §2.2a with it, and the two MUST NOT be edited independently.**

The grant's `operations.include` array lists the literal extension operation names (e.g., `"snapshot"`, `"merge"`). `check_permission` (ENTITY-CORE-PROTOCOL.md §5.2) uses `matches_scope` to check these at dispatch time. The handler then maps to base permissions for `check_path_permission`. Both checks use the same grant entries — `check_permission` uses `operations` and `handlers` scopes, `check_path_permission` uses the mapped permission against the `resources` scope (including `resources.exclude`).

Merge requires `put` authorization on every path it writes. The handler **MUST** verify authorization before applying any writes. If any path is denied, the entire operation **MUST** fail with `capability_denied` (403) — merges are atomic. Implementations MAY optimize by checking a covering prefix when one exists (e.g., `target_prefix` when remapping is active), falling back to per-path checks otherwise.

**View trees and capabilities**: When a handler operates on a view tree, the capability checks described above are performed against the view tree's filtered state. The view tree's scope filter is compiled from the same capability — so the checks are structurally enforced. Operations on a view tree cannot access paths outside the view's scope.

---

## 12. Conformance

### 12.1 MUST

- **HAMT node shape (§3.1)** — `{map: bytes(4), data: [Entry]}`; Entry discriminated by CBOR major type (array = bucket of [key, value_hash] tuples sorted lex; byte string = link to sub-node, its length per its format byte — §8.4.5). No `binding` field. Field names `map` / `data` per spec text (chosen for terseness; we are NOT wire-compatible with IPLD HashMap tooling).
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
- Snapshot entities contain only `root` — no `prefix` field. *(This is **not** in tension with §3.3a: a snapshot's prefix rides the request; a published root has no request. See §3.3's closing paragraph.)*
- Non-empty prefix validation (must end with `/`)
- **Absolute-prefix trim (§3.3)** — `relative_key = trim_prefix(path, absolute_prefix)` across all three prefix shapes, with the universal case (`prefix: "/"`) a **no-op** yielding fully-qualified keys. An implementation MUST NOT emit the superseded `"/" + peer_id + "/" + operation_prefix` form, which is ill-defined for the universal tree.
- **Published-root `prefix` (§3.3a)** — a published root MUST carry `prefix`; consumers MUST use it (never a default or an inference) to reconstruct paths and to determine the published extent.
- **Key-form assertion (§3.3a / §3.3)** — the conformance oracle MUST take an absolute path known to be bound in the published subtree, derive `relative_key` from the publisher's **declared `prefix`**, and assert the trie resolves *that* key. **A trie-rebuild equality check does not satisfy this**: a rebuild takes its keys from the trie and is therefore self-consistent by construction on key *form*, so an implementation keying by absolute path rebuilds to its own root and passes. Equality proves the routing algorithm; only this assertion proves the key convention. *(This distinction is why a three-way-green `published_root` category coexisted with a three-way key divergence for months — `entity-core-go`, 2026-08-08.)*
- **Reconstruction round-trip (§3.3)** — `absolute_prefix + relative_key` MUST reproduce the absolute path the publisher bound.
- **The walk contract (§3.8)** — R1 a declared node that does not resolve is `incomplete_walk`, never a shorter answer; R2 children are derived by decoding declared entries, never by byte-scanning and never conditionally on the child already being present; R3 the error carries `missing_hash` **and** `declared_by`; R4 a partial walk is opt-in and MUST NOT be presented as complete; R5 `incomplete_walk` is terminal.
- **Extract completeness (§6.2)** — an extract envelope is complete against its own root, filtered or not; walking the returned root MUST NOT reach an entity outside `included`.
- **Walk-contract vectors (§3.8)** — an implementation MUST exercise all six. **The assertion is the error, not the count**: asserting `len(keys) == N` also passes on an implementation that returns N by luck of which branch was cut.
  - `TREE-WALK-WITHHELD-1` — a root committing to N keys, every node served but one **interior** node; the walk MUST fail with `incomplete_walk`.
  - `TREE-WALK-CONTROL-1` — the same trie **fully** served MUST return exactly N keys and no error. **Without this a walk that always fails scores green on the vector above.**
  - `TREE-WALK-PARENT-1` — the error's `declared_by` equals the hash of the node holding the dangling link (R3).
  - `TREE-WALK-VALUE-1` — a withheld **bucket value** entity, not a link, also fails. **Cut the other half and an implementation that only checks links reports clean.**
  - `TREE-EXTRACT-COMPLETE-1` — a `paths`-filtered extract's envelope is complete against its own root: walking the returned root touches no entity outside `included`.
  - `TREE-WALK-PARTIAL-1` — an explicitly-requested partial walk returns the short list and no error, pinning that R4's opt-in exists and is reachable.
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
| `put` | `invalid_request` | 400 | **The submitted `entity` is not a `core/entity`** — the structural step of `ENTITY-CORE-PROTOCOL` §6.3's two-step admission. It is not an entity when it is not a map, or `type` is absent / empty / not a text string, or `data` is absent, or `content_hash` is absent or its length does not match its format code (§1.2). The generic structurally-invalid case — §3.3's 400 default, not a tree-specific code *(v4.4; predicate stated v4.5)* |
| `put` | `hash_mismatch` | 400 | A structurally valid entity whose carried `content_hash` does not equal `content_hash({type, data})`. **Distinct from the 409 below**, and the same code `EXTENSION-CONTENT` Appendix A uses for this failure. **Reached only after the row above passes** — a submission that is both malformed and mis-hashed is the row above *(v4.4; ordering stated v4.5)* |
| `put` | `hash_mismatch` | 409 | A CAS `expected_hash` precondition lost a race — another writer committed first. **A different failure from the 400 row**: 400 says *this entity is not what it claims to be* and is a defect in the submission; 409 says *someone else wrote first*, is nobody's defect, and is retryable *(v4.4 — tabulated; the behaviour is `ENTITY-CORE-PROTOCOL` §3.6 and `EXTENSION-SUBSCRIPTION` §2.2)* |
| `put` | `unsupported_content_hash_format` | 400 | The submitted entity's `content_hash` is well-formed but names a format code this peer does not support. **Not the `invalid_request` row** — the value is a structurally valid hash and the peer simply cannot verify it. `ENTITY-CORE-PROTOCOL` **§4.7 row 5** is the authority for this code; the row is restated here because `put` is one of its ingest surfaces *(v4.5)* |
| `snapshot` | `invalid_prefix` | 400 | Non-empty prefix doesn't end with `/` |
| `snapshot` | `tree_not_found` | 404 | Referenced tree_id doesn't exist |
| `diff` | `snapshot_not_found` | 404 | Referenced snapshot hash not in content store |
| `merge` | `snapshot_not_found` | 404 | Source snapshot hash not in content store |
| `merge` | `tree_not_found` | 404 | Target tree doesn't exist |
| `merge` | `capability_denied` | 403 | Capability does not grant `put` on target path |
| `extract` | `invalid_prefix` | 400 | Non-empty prefix doesn't end with `/` |
| `extract` | `invalid_path` | 400 | A `paths[]` entry is not a valid relative path (control character, empty segment, leading `/`). **The whole request is rejected; no partial result.** Distinct from a well-formed entry that binds nothing, which is silently omitted (§6.1) |
| `extract` | `tree_not_found` | 404 | Referenced tree_id doesn't exist |
| `create` | `tree_exists` | 409 | Tree with this tree_id already exists |
| `create` | `invalid_config` | 400 | Missing `capability` when `source` is present |
| `create` | `capability_not_found` | 404 | Referenced capability hash not in content store |
| `destroy` | `tree_not_found` | 404 | No tree with this tree_id |
| `destroy` | `default_tree` | 400 | Cannot destroy the default tree |
| (any) | `view_tree_invalid` | 403 | View tree's capability has been revoked or is expired |
| (any walk) | `incomplete_walk` | 502 | **A node the walk needed did not resolve, and the root declares it** (§3.8 R1). Carries `missing_hash` (the unresolvable child) and `declared_by` (the hash of the node that declared it — the cut point, R3). **502 because the failure is the origin's, not the caller's**: the request was well-formed and the answer is unobtainable from the party asked. **Terminal, never retried** (R5) — a transport failure while fetching is a different condition and is separately retryable. **Distinct from `not_found`**, which is the answer a node that *did* resolve gives *(v4.6)* |

---

## Appendix B: Structural Sharing via Hash-Keyed HAMT (v4.0)

The snapshot format uses a content-addressed hash-keyed HAMT (IPLD HashMap algorithm, see §3.1) for structural sharing. Each trie node is an entity in the content store. Changing one binding creates new nodes along the SHA-256-bit path from root to the affected leaf bucket — O(log_K N) new entities per change for K=32. All unchanged sub-nodes are shared by hash reference.

This provides root identity (snapshot hash changes if any binding changes) plus efficient change propagation (per-version cost proportional to changes, not to total bindings). Under CHAMP-equivalent canonicalization, the same binding set produces the same root regardless of insertion or deletion history — the property that makes cross-peer convergence work via byte-identical root comparison.

**Prefix queries do NOT navigate the trie under v4.0.** The trie is hash-keyed; bindings under a path prefix are scattered across HAMT positions, not co-located in a structural subtree. Prefix-scoped reads go through LocationIndex (the path-keyed B-tree / BTreeMap layer); the trie's role is Merkle hash structure for content-addressed cross-peer convergence. This division of labor is explicit in §3.7.

**Prior versions (v3.x) used a path-keyed compressed trie** where prefix queries DID navigate the trie. That structure produced O(N) per-Put encoding cost in wide-flat workloads (the cliff Stage 7 fixed) and structurally privileged path locality at the cost of depth-with-path-segment-count. The v4.0 fork trades path-locality for bounded-fanout / bounded-depth / canonicalization-under-history-reorder. See `proposals/implemented/PROPOSAL-TREE-NODE-SHAPE-BOUNDED-FANOUT.md` §3 (why IPLD HashMap, not JMT or other alternatives) + §3.7 (math contract — what's preserved, lost, recoverable) for the full rationale.

The location index remains a flat path → hash map. The trie is a content-addressed projection — built when snapshots are needed (or maintained incrementally per §3.4), stored in the content store, used for diff/merge/transfer. Both structures co-exist; each has a clear role.

---

## Document History

**v4.12:** §2.2a — the `resource` column becomes **three-valued**. `diff`, `create` and `destroy` move from `required` to **no path subject**: none of the three targets an entity binding, so `ENTITY-CORE-PROTOCOL` §3.3's carve-out applies and neither the `path_required` requirement nor the BROAD/OPTIONAL-FILTER declaration obligation arises. As written at v4.11 the column obliged a conformant peer to refuse `diff(base, target)` — **the only call shape §4.2 documents** — and it contradicted four normative homes in this document, two of them inside §11's own pseudocode, which names all three operations by name. **§11's `map_operation` is now cited as the authority rather than restated**, so the classification has one home. `merge` is re-ordered above `diff` to group the three states; `put` and `merge` are unchanged, and the three BROAD rows — the reason §2.2a exists — are untouched. **The withdrawn rows were never implementable and the correction removes an obligation rather than adding one.**

**v4.10 / v4.11** are absent from this section and their entries were not written at the time; v4.11 is the fold that introduced §2.2a, described above by the revision that corrects it.

**v4.9:** §6.1/§6.2 — **the disposition of a malformed `extract.paths[]` entry**, which was undefined and had two defensible readings. A malformed entry is now **`400 invalid_path` for the whole request**; a well-formed entry that binds nothing is **silently omitted**, unchanged. The two were collapsible precisely because both are safe — and a caller who gets seven results for ten paths cannot tell which case it hit. Ruled as a MUST because the readings are cross-peer observable. §6.2 validates every entry **before reading any**, and Appendix A gains the row. ⚠ Independently of this, the store boundary stays **total** per `ENTITY-CORE-PROTOCOL` §5.4 (0.8.2.21): `paths[]` reaches a path boundary through `params`, a channel no resource-target pre-validator sees, and a boundary that asserts there is a remote denial of service.

**v4.8:** §8.7 — the sentence no longer restates a `system/*` prefix rule at all. `ENTITY-CORE-PROTOCOL` 0.8.2.13 **withdrew** the reservation; install authorization at any path is the capability check on the install path. The least-privilege point this paragraph exists to make — the prefix is organizational and confers no tree privilege — is unchanged and is now stated without leaning on a rule that no longer exists. *(v4.7 restated the same rule at a narrower scope and is superseded; both it and the scope it described lasted one day.)*

**v4.6:** §3.8 — the walk contract, new. *The absence of a node is never an answer*: a declared node that does not resolve is `incomplete_walk` (502, terminal, carrying the cut point) and never a shorter result, children are derived structurally rather than by byte-scan, and a partial walk is opt-in and never presented as complete. §4.3's collectors gain the branch they never had and thread the declaring node's hash. §6.2 **deletes** the "unfiltered subtree nodes MAY be omitted" sentence — v3.x path-navigation residue that contradicted the rebuild directly above it — and states the property that is true: an extract is complete against its own root, filtered or not, because publishing a subset is **re-rooting, not filtering**. Six vectors, and the control case is required so that a walk which always fails cannot score green.

**v4.5:** Appendix A's `put` rows get the predicate they were missing. *"Does not decode"* is now stated — the submitted value is admitted as a `core/entity` (all three fields required) **before** its hash is compared, so a submission that is both malformed and mis-hashed is the structural row; the `unsupported_content_hash_format` arm is restated from `ENTITY-CORE-PROTOCOL` §4.7 row 5 because `put` is one of its ingest surfaces; and **`set` is dropped from the rows** — `ENTITY-CORE-PROTOCOL` §6.3 and §2.2 below both define exactly two index operations, and `set` was never one of them.

**v4.4:** Appendix A gains the `put` / `set` rows. The two core data operations the protocol runs on had no error-code row in the only table their extension has: a non-decoding entity is `400 invalid_request` (§3.3's generic case), a content-hash mismatch is `400 hash_mismatch` (`EXTENSION-CONTENT` §923's code for the same failure), and the CAS race that shares that token is tabulated beside it at 409 so the two are not collapsed.
