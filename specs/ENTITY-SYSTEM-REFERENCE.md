# Entity System — Condensed Working Reference

**For**: Developers, AI agents, and operators working with an Entity Core Protocol V7 implementation.
**Not for**: First-time implementation — see ENTITY-CORE-PROTOCOL.md for the full normative spec.

This document is the minimum context needed to work with the entity system: make requests, build handlers, understand responses, and reason about security. Everything here is grounded in V7; section references (§) point to the normative spec.

---

> **Path notation.** Paths in this document use peer-relative notation (without leading `/{peer_id}/`). All peer-relative paths resolve to the local peer's namespace: `system/tree` means `/{local_peer_id}/system/tree`. Every path in the entity tree is absolute at rest — rooted at a peer identity. See ENTITY-CORE-PROTOCOL.md §1.4 for the path model. Cross-peer examples use absolute paths with explicit peer identities.

## 1. What Is the Entity System

A peer-to-peer protocol where **typed, content-addressed entities** are stored in a **tree** (path-to-hash index) and exchanged via **authenticated requests**. Every peer has an identity (Ed25519 keypair), a tree, and handlers that process requests. Capabilities control what each peer can do.

Two wire message types: **EXECUTE** (request) and **EXECUTE_RESPONSE** (response). That's it.

---

## 2. Core Concepts

### Entity

The fundamental data unit. A typed payload with content-addressed identity.

```
# Logical entity
Entity = {
  type:         "system/tree/get-request"        # semantic type path
  data:         {path: "peer_id/local/files/..."}  # typed payload
}

# Entity with identity hash
Entity = {
  type:         "system/tree/get-request"        # semantic type path
  data:         {path: "peer_id/local/files/..."}  # typed payload
  content_hash: <33 bytes>                         # format_code(1) + SHA-256(32)
}
```

`content_hash = SHA-256(ECF_encode({type, data}))` — only type and data are hashed. The hash is prefixed with format code `0x00` (ECFv1-SHA-256), totaling 33 bytes. On the wire, hashes are CBOR byte strings.

### Envelope

Bundles a root entity with all referenced entities in a flat map:

```
envelope = {
  root: <the primary entity — an EXECUTE or EXECUTE_RESPONSE>
  included: {
    <content_hash>: <entity>,   # identity, signature, capability, etc.
    <content_hash>: <entity>,
    ...
  }
}
```

The `included` map carries everything needed to verify the request: the author's identity, the signature, the capability token, and the full delegation chain.

### Tree (Entity Tree)

Two-layer storage:

```
Location Index:  path → content_hash    (mutable, the "tree")
Content Store:   content_hash → entity  (immutable, deduplicated)
```

Paths are UTF-8 strings typed as `system/tree/path` (extends `primitive/string`). Absolute paths start with `/`: `/{peer_id}/local/files/readme.md`. Peer-relative paths (no leading `/`) resolve to `/{local_peer_id}/...`. The tree is flat (a map), not hierarchical — "directory" structure is implied by `/` separators. Listing uses prefix matching. The entity system has two typed address spaces: `system/hash` for content-space (what it is) and `system/tree/path` for naming-space (where it is).

### Peer Identity

```
KeyPair = Ed25519 (32-byte seed, 32-byte public key)
PeerID  = Base58(0x01 || 0x01 || SHA-256(public_key))    # 46 characters, typed as system/identity/peer-id
```

---

## 3. Making a Request

Every interaction is an EXECUTE sent inside an envelope. Here's the complete structure:

```
envelope = {
  root: {
    type: "system/protocol/execute",
    data: {
      request_id: "unique-id",                     # caller generates, UUID recommended
      uri:        "entity://peer_id/system/tree",   # target handler (system/tree/path)
      operation:  "get",                            # what to do
      resource:   {targets: ["peer_id/local/files/readme.md"]},  # resource target (targets/exclude: system/tree/path)
      params: {                                     # entity — has type, data, content_hash
        type: "system/tree/get-request",
        data: {},
        content_hash: <33 bytes>
      },
      author:     <hash of author's identity entity>,
      capability: <hash of capability token>,
      bounds:     {ttl: 64, budget: 100000}          # optional
    },
    content_hash: <33 bytes>
  },
  included: {
    <author_hash>:     {type: "system/peer", data: {peer_id, public_key, key_type}},
    <sig_hash>:        {type: "system/signature", data: {target: <execute_hash>, signer: <author_hash>, ...}},
    <cap_hash>:        {type: "system/capability/token", data: {grants: [...], granter: ..., grantee: ...}},
    <cap_sig_hash>:    {type: "system/signature", data: {target: <cap_hash>, signer: <granter_hash>, ...}},
    <granter_hash>:    {type: "system/peer", data: {...}},
    ...  # full delegation chain if capability was delegated
  }
}
```

**Key rules:**
- `params` and `result` are always entities (have type, data, content_hash) — not raw values
- Signatures are found by scanning `included` for `system/signature` where `target` matches
- The EXECUTE itself is signed by the author — sign its `content_hash`
- `author` and `capability` are required for all requests except connection setup

### Response

```
{
  type: "system/protocol/execute/response",
  data: {
    request_id: "unique-id",        # matches the request
    status:     200,                 # HTTP-like status code
    result: {                        # entity
      type: "some/type",
      data: {...},
      content_hash: <33 bytes>
    }
  }
}
```

**Status codes:** 200 success, 400 bad request, 401 auth failed, 403 forbidden, 404 not found, 409 conflict, 500 internal error, 501 not supported.

Error results use type `system/protocol/error` with `{code: "error_code", message: "..."}`.

---

## 4. Tree Operations — Reading and Writing Data

The tree handler (`system/tree`) is the primary data interface. Two operations:

### Reading an Entity

```
EXECUTE system/tree  operation: "get"
  resource: {targets: ["peer_id/local/files/readme.md"]}
  params: {type: "system/tree/get-request", data: {}}

→ Returns: the entity at that path
```

### Listing a Path (Trailing Slash)

```
EXECUTE system/tree  operation: "get"
  resource: {targets: ["peer_id/local/files/"]}
  params: {type: "system/tree/get-request", data: {}}

→ Returns: {type: "system/tree/listing", data: {path, entries: {name → {hash, has_children}}, count, offset}}
```

The trailing `/` on the target path means "list entries under this prefix." Empty string `""` lists the root.

### Writing an Entity

```
EXECUTE system/tree  operation: "put"
  resource: {targets: ["peer_id/data/my-key"]}
  params: {type: "system/tree/put-request", data: {entity: {type: "...", data: {...}}}}
```

### Removing a Binding

```
EXECUTE system/tree  operation: "put"
  resource: {targets: ["peer_id/data/my-key"]}
  params: {type: "system/tree/put-request", data: {}}
  # entity absent → removes the path binding
```

### Hash-Only Read

```
EXECUTE system/tree  operation: "get"
  resource: {targets: ["peer_id/local/files/readme.md"]}
  params: {type: "system/tree/get-request", data: {mode: "hash"}}
→ Returns just the content hash, not the full entity
```

### Pagination

Listings support `limit` and `offset` in the get-request. Entries are ordered lexicographically.

---

## 5. Handlers

A handler processes requests for a URI pattern. Handler manifests are `system/handler` entities stored at their pattern path (e.g., `system/tree`, `local/files`). The `system/handler` handler manages handler lifecycle — registration stores the manifest at the pattern path, creates the handler's grant at `system/capability/grants/{pattern}`, and derives a `system/handler/interface` index entry at `system/handler/{pattern}`.

### Handler Manifest

```
{
  type: "system/handler",
  data: {
    pattern:    "local/files",           # URI prefix this handler owns (system/tree/path)
    name:       "files",
    operations: {                          # operation-spec values:
      "read-metadata": {input_type: "...", output_type: "..."},  # input_type, output_type: system/type/name
      "watch":         {input_type: "...", output_type: "..."}
    }
  }
}
```

### Dispatch: How Requests Reach Handlers

1. Receive envelope, validate hashes
2. If EXECUTE → extract handler path from URI (strip `entity://peer_id/`)
3. Walk backward through path segments, find `system/handler` entity at longest matching prefix
4. Verify capability grants this operation on this handler's scope
5. Resolve handler grant from `system/capability/grants/{pattern}`
6. Handler processes the request with handler_grant + caller capability

Example: `entity://alice_id/local/files/readme.md` → handler path `local/files/readme.md` → matches handler pattern `local/files` → files handler processes it.

### System vs Domain Handlers

**System handlers** (six have registered patterns):
- `system/tree` — entity tree reads/writes/listing
- `system/handler` — handler lifecycle (register, unregister)
- `system/type` — type validation (validate operation)
- `system/capability` — runtime capability management
- `system/protocol/connect` — connection setup (hello, authenticate)
- `system/inbox/*` — inbox delivery (EXTENSION-INBOX.md)
- `system/continuation` — continuation advancement, resume, abandon (EXTENSION-CONTINUATION.md)

**Bootstrap handlers** (pre-loaded at system initialization):
- `system/tree`, `system/handler`, `system/type`, `system/protocol/connect`

All other handlers register through `system/handler`.

**System data paths** (data accessed via tree handler):
- `system/type/*` — type definitions (tree handler for data access; types handler for `validate`)
- `system/handler/*` — handler index (`system/handler/interface` entities)
- `system/capability/grants/*` — handler capability grants
- `system/subscription/*` — active subscriptions (tree handler for data access; subscription handler for `subscribe`, `unsubscribe`)
- `system/tree/instances/*` — non-default tree configs

Handler manifests at pattern paths (e.g., `system/tree`, `system/capability`) are also data entities accessible through the tree handler.

To read a type: `EXECUTE system/tree operation: "get" resource: {targets: ["system/type/primitive/string"]}`. To validate an entity: `EXECUTE system/type operation: "validate" resource: {targets: ["system/type/app/user"]} params: {entity: ..., type_name: "app/user"}`.

**Domain handlers** (user-installed, any non-system path):
- `local/files` — file operations
- `local/processes` — process management
- Any custom path you register

All handlers — system and domain — operate under the capability model (ENTITY-CORE-PROTOCOL.md §6.8). Handler grants are created by `system/handler` during registration and stored at `system/capability/grants/{pattern}`. Domain handlers receive capability-scoped access (view trees when implemented).

### Handler Context

Handlers receive a minimal context:

| Field | Purpose |
|-------|---------|
| `local_peer_id` | This peer's identity |
| `remote_peer_id` | Who made the request |
| `capability` | What they're authorized to do |
| `storage` | emit/get/list/remove entities |
| `bounds` | Resource limits for this request |
| `request_id` | Request correlation |

### Building a Handler

1. Choose a pattern (URI prefix your handler owns)
2. Define operations with input/output types
3. Register via `system/handler` handler (stores manifest at pattern path + interface entity at `system/handler/{pattern}`)
4. Implement each operation — receive params entity, return result entity
5. Check path-level capability grants for any path-specific operations

---

## 6. Security Model

### Capabilities

A capability token grants specific operations on specific handlers for specific resources. Grant dimension fields use `system/capability/path-scope` for path-valued dimensions (`handlers`, `resources`) and `system/capability/id-scope` for identifier-valued dimensions (`operations`, `peers`), both with `{include, exclude}`:

```
{
  type: "system/capability/token",
  data: {
    grants: [
      {handlers:   {include: ["system/tree"]},
       resources:  {include: ["system/type/*", "system/handler/*"]},
       operations: {include: ["get"]}},                       # tree handler get on system paths
      {handlers:   {include: ["system/tree"]},
       resources:  {include: ["local/files/home/*"]},
       operations: {include: ["get", "put"]}}                 # tree handler read+write on files
    ],
    granter: <granter identity hash>,
    grantee: <grantee identity hash>,
    created_at: 1737900000000,
    expires_at: 1737986400000         # optional
  }
}
```

Signed by the granter. Can be delegated (parent field) — child can only restrict, never amplify.

### Two-Level Authorization (Critical)

Every request requires both dispatch scope and handler-level path scope — **both must pass**:

1. **Dispatch scope** (`check_permission`, V7 §5.2): Can this peer call this handler with this operation on this resource target scope?
   - The `handlers` field in each grant names which handlers are authorized (pattern matching)
   - The `operations` field names which operations are authorized (exact match)
   - When `resource` is present on the EXECUTE, `resources` scope is checked at dispatch — full scope checking: effective target scope (targets minus caller excludes) must fit within effective grant scope (includes minus grant excludes)
   - All dimensions must match from a single grant entry

2. **Path scope** (`check_path_permission`, V7 §6.3, defense-in-depth): Can this peer access this specific path?
   - The handler uses `matches_scope` to filter grants by its own handler pattern, then checks the requested path against the `resources` scope
   - If the grant's `resources.exclude` patterns match, those paths are denied
   - When `resource` is present, this is a secondary check; when absent, this is the sole resource enforcement

**Example: Reading a type definition**
```
Capability grant:
  {handlers: {include: ["system/tree"]},
   resources: {include: ["system/type/*"]},
   operations: {include: ["get"]}}

Request: EXECUTE system/tree operation: "get"
  resource: {targets: ["system/type/primitive/string"]}

Step 1: check_permission → matches_scope("system/tree", grant.handlers) PASS,
        matches_scope("get", grant.operations) PASS,
        check_resource_scope({targets: ["system/type/primitive/string"]}, grant.resources) PASS → ALLOW
Step 2: check_path_permission (defense-in-depth) →
        matches_scope("system/type/primitive/string", grant.resources) PASS → ALLOW
→ Request proceeds
```

**Without the grant**: Request denied at dispatch — no grant authorizes the tree handler for get on that resource target.

### Grant Pattern Types

Patterns are used in the `include` and `exclude` arrays within scope fields of grants. All use the same `matches_pattern` algorithm (V7 §5.4), wrapped by `matches_scope` (V7 §5.2):

| Pattern | Example Field | Meaning |
|---------|---------------|---------|
| `system/tree` | `handlers` | Exact handler match |
| `system/*` | `handlers` | All system handlers |
| `*` | `handlers` | Any handler |
| `system/type/*` | `resources` | Subtree — all paths under `system/type/` |
| `*/system/type/*` | `resources` | Peer wildcard — any peer's type definitions |
| `peer_id/local/files/doc.txt` | `resources` | Exact path |
| `*` | `resources` | All paths |

### Excludes

The `resources` scope (and any other scope) can have `exclude` patterns to carve out exceptions:
```
{handlers:   {include: ["system/tree"]},
 resources:  {include: ["local/files/*"], exclude: ["local/files/private/*"]},
 operations: {include: ["get"]}}
```

Excludes within `resources` apply to data paths (checked in `check_path_permission` via `matches_scope`). Excludes are per-scope, not per-grant. To exclude a handler, omit it from `handlers.include`.

### Delegation

Capabilities form chains. Each child is signed by its parent's grantee (who becomes the child's granter). The entire chain must be in the envelope's `included` map. Rules:
- Child `handlers` scope must be subset of parent's handlers scope (`scope_subset`)
- Child `operations` scope must be subset of parent's operations scope
- Child `resources` scope must be subset of parent's resources scope (includes AND inherits excludes)
- Child `peers` scope must be subset of parent's peers scope
- Child expiration must not exceed parent's

### Verification Order (Full Chain)

1. Validate EXECUTE content hash
2. Find signature → verify signer matches author
3. Verify Ed25519 signature with author's public key
4. Verify capability grantee matches execute author
5. Walk capability chain: verify each link's signature, expiration, attenuation
6. Root capability must be granted by the local peer (`granter.peer_id == local_peer_id`)
7. Check handler scope permission
8. Handler checks path scope permission

---

## 7. Connection Establishment

New connections follow a 6-message handshake:

```
1. Initiator → EXECUTE hello            (peer_id, nonce, protocols)
2. Responder → EXECUTE_RESPONSE hello   (responder's hello data)
3. Initiator → EXECUTE authenticate     (peer_id, public_key, nonce, signature)
4. Responder → EXECUTE_RESPONSE authenticate (capability token for initiator)
5. Responder → EXECUTE authenticate     (peer_id, public_key, nonce, signature)
6. Initiator → EXECUTE_RESPONSE authenticate (capability token for responder)
```

- Connection path `system/protocol/connect` requires no auth
- After connection is established, all requests require `author` + `capability`
- Connection delivers the initial capability token — typically read-only system access
- Protocol version: `"entity-core/1.0"`

### Default Connection Capability

```
grants: [
  {handlers:   {include: ["system/tree"]},
   resources:  {include: ["system/type/*", "system/handler/*"]},
   operations: {include: ["get"]}},                       # tree handler get on system data
  {handlers:   {include: ["system/capability"]},
   resources:  {include: []},
   operations: {include: ["request"]}}                    # request capabilities (no tree access needed)
]
```

---

## 8. Tree Extension Operations

Beyond core get/put, the tree handler supports bulk operations (EXTENSION-TREE.md):

| Operation | Purpose | Params Type | Returns |
|-----------|---------|-------------|---------|
| `snapshot` | Capture tree state as content-addressed entity | `system/tree/snapshot-request` | `system/tree/snapshot` |
| `diff` | Compare two snapshots | `system/tree/diff-request` | `system/tree/diff` |
| `merge` | Apply snapshot bindings into a tree | `system/tree/merge-request` | `system/tree/merge-result` |
| `extract` | Bundle subtree as transferable envelope | `system/tree/extract-request` | `system/protocol/envelope` |
| `create` | Create a non-default tree | `system/tree/config` | `system/tree/config` |
| `destroy` | Remove a non-default tree | `primitive/string` (tree_id) | `primitive/bool` |

All target `system/tree` via EXECUTE. All accept optional `tree_id` for non-default trees.

### Sync Workflow

```
1. snapshot(prefix: "peer_id/local/files/") → snapshot_A on peer A
2. snapshot(prefix: "peer_id/local/files/") → snapshot_B on peer B
3. diff(base: snapshot_B, target: snapshot_A) → diff showing what changed
4. extract(prefix: "peer_id/local/files/", paths: [from diff]) → envelope with entities
5. merge(source: snapshot_from_extract, target_tree: default) → apply changes
```

### Merge Strategies

| Strategy | Behavior |
|----------|----------|
| `no-overwrite` | Place new paths. Report conflicts for existing. **(Default)** |
| `source-wins` | Overwrite everything. |
| `target-wins` | Only place new paths. Keep existing on conflict. |

Merge is additive — does not remove target paths absent from source. Use diff + put(null) for full sync.

---

## 9. Extensions Summary

### Inbox (EXTENSION-INBOX.md)

Async result delivery. Include `deliver_to` + `deliver_token` in EXECUTE -> get 202 acknowledgment -> result delivered later as an EXECUTE to the inbox URI. When EXTENSION-CONTINUATION is installed, the inbox handler delegates to the continuation handler for paths with continuation entities. Otherwise stores in tree. Single `receive` operation — message type carries semantics.

### Continuation (EXTENSION-CONTINUATION.md)

Execution chaining. Continuation entities at paths describe what to do when a result arrives: dispatch to a target handler (`system/continuation`), accumulate for fan-in (`system/continuation/join`), or record exhaustion. The continuation handler provides `advance` (trigger advancement), `resume` (restart suspended), and `abandon` (delete suspended) operations. Independent of EXTENSION-INBOX — any mechanism that delivers a result to a path can trigger advancement.

### Subscription (EXTENSION-SUBSCRIPTION.md)

Watch tree paths for changes. Depends on inbox extension. Subscribe to a path pattern → receive notifications (`created`/`updated`/`deleted`) via inbox delivery. Notifications carry metadata (URI, hash), not entity data by default.

### Content (EXTENSION-CONTENT.md)

Large entity chunking and transfer. Blob manifests (`system/content/blob`) reference chunks (`system/content/chunk`). FastCDC recommended for content-defined chunking. Primarily for deduplication and resumable transfer between peers.

### Compute (EXTENSION-COMPUTE.md)

Entity-native expression language. Six core expression types (`compute/literal`, `compute/lookup`, `compute/apply`, `compute/if`, `compute/let`, `compute/lambda`) plus inline types for arithmetic, comparison, and logic. Programs are entities — content-addressed, storable, transferable. Reactive mode: expressions re-evaluate when tree dependencies change (spreadsheet semantics). Budget and depth limits prevent runaway computation.

### Type Extension (EXTENSION-TYPE.md, planned)

Value-level constraint validation. Core type system describes **structure only** (fields, shapes, composition). The type extension adds constraints (range checks, patterns, enumerations) via handler dispatch — constraint type determines handler, constraint data provides parameters. Twelve standard constraint fields: `min`, `max`, `pattern`, `values`, `min_length`, `max_length`, `min_items`, `max_items`, `min_entries`, `max_entries`, `type_pattern`, `one_of`. Constraints live in the `system/type/constraint/*` namespace.

---

## 10. Type System Quick Reference

### Structure

The core type system is pure structure — it describes data shapes, not value predicates. Type definitions are entities of type `system/type` stored at `system/type/{type_path}`.

14 bootstrap types seed the system: 8 primitives (`primitive/string`, `primitive/bytes`, `primitive/uint`, `primitive/int`, `primitive/float`, `primitive/bool`, `primitive/null`, `primitive/any`) + 6 meta-types (`system/hash`, `system/tree/path`, `system/type/name`, `system/identity/peer-id`, `system/type`, `system/type/field-spec`).

### `system/type` Fields

| Field | Purpose |
|-------|---------|
| `name` | Type name (`system/type/name`, e.g., `system/protocol/execute`) |
| `extends` | Parent type for composition (`system/type/name`) |
| `fields` | Map of field name → field-spec |
| `layout` | Byte decomposition order (only for `primitive/bytes` subtypes) |
| `type_params` | Generic type parameter names |
| `type_args` | Concrete type bindings when extending a generic type |

### Field-Spec (Five-Way Invariant)

A field-spec MUST contain exactly one of: `type_ref`, `array_of`, `map_of`, `union_of`, or `type_param`. Plus optional modifiers: `optional`, `default`, `key_type`, `type_args`, `byte_size`.

### Conformance

- **Level 0** — Type-unaware. Full protocol participant. No validation.
- **Level 1** — Type-aware. Populates `system/type/*`. Resolves types.
- **Level 2** — Validating. Structural field validation (presence, shape, nesting).
- **Level 2+** — Constrained. Installs type extension, constraint validation via handler dispatch.

### Constraints (Extension)

Value constraints (`constraints` field on type definitions) are an open-type extension field — core preserves but does not interpret. The type extension (EXTENSION-TYPE.md) defines `system/type/constraint` and the constraint handler dispatch model.

---

## 11. Key Type Paths

### Protocol Types
| Type | Purpose |
|------|---------|
| `system/protocol/execute` | Request message |
| `system/protocol/execute/response` | Response message |
| `system/protocol/error` | Error result (code + message) |
| `system/protocol/envelope` | Wire envelope |
| `system/protocol/connect/hello` | Connection hello (peer_id: `system/identity/peer-id`, nonce, protocols) |
| `system/protocol/connect/authenticate` | Connection authenticate (peer_id: `system/identity/peer-id`, public_key, nonce, signature) |

### Type System Meta-Types
| Type | Purpose |
|------|---------|
| `system/type` | Type definition meta-type (name: `system/type/name`, extends: `system/type/name`, fields, layout, type_params, type_args) |
| `system/type/field-spec` | Field shape specification (type_ref: `system/type/name`, array_of, map_of, union_of, type_param + modifiers) |
| `system/tree/path` | Tree path — naming-space address (`primitive/string`). Handler patterns, URIs, resource targets. |
| `system/type/name` | Type name — type-space address (`primitive/string`). Type references, extends, field type_ref. |
| `system/identity/peer-id` | Peer identifier — identity-space address (`primitive/string`). Base58, 46 chars, self-describing. |
| `system/capability/path-scope` | Scope for path-valued grant dimensions: `handlers`, `resources` ({include, exclude}) |
| `system/capability/id-scope` | Scope for identifier-valued grant dimensions: `operations`, `peers` ({include, exclude}) |

### Identity & Security Types
| Type | Purpose |
|------|---------|
| `system/peer` | Peer identity (peer_id: `system/peer-id`, public_key, key_type). Renamed from `system/identity` in V7 v7.40. |
| `system/signature` | Signature entity (target, signer, algorithm, signature) |
| `system/capability/token` | Capability with grants, granter, grantee |
| `system/capability/grant` | Result type when delivering capabilities |
| `system/capability/grant-entry` | Single grant: handlers, resources, operations, peers, constraints (path dimensions use `system/capability/path-scope`, identifier dimensions use `system/capability/id-scope`, both with include/exclude) |
| `system/hash` | Content hash — content-space address (`primitive/bytes`, 33 bytes for SHA-256) |

### Tree Types
| Type | Purpose |
|------|---------|
| `system/tree/get-request` | Read/list request params |
| `system/tree/put-request` | Write/remove request params |
| `system/tree/listing` | Listing result (path: `system/tree/path`, entries, count, offset, optional `next_page`) |
| `system/tree/listing-entry` | Entry in listing (hash, has_children) |
| `system/tree/snapshot` | Captured tree state (prefix: `system/tree/path`, bindings) |
| `system/tree/diff` | Comparison result (added, removed, changed) |
| `system/tree/merge-request` | Merge params (source, strategy, prefixes) |
| `system/tree/merge-result` | Merge outcome (applied, skipped, conflicts) |
| `system/tree/config` | Non-default tree configuration |
| `system/handler` | Handler manifest (pattern: `system/tree/path`, operations, max_scope, internal_scope) |
| `system/handler/interface` | Handler discovery entry (pattern: `system/tree/path`, name, operations — no security config) |

### System Tree Paths
| Path | Content |
|------|---------|
| `system/handler/{name}` | Handler interface entities (discovery) |
| `system/type/{type_path}` | Type definitions |
| `system/capability/grants/{name}` | Handler capability grants |
| `system/tree/instances/{tree_id}` | Non-default tree configs |
| `system/subscription/{id}` | Active subscriptions |
| `system/inbox/{path}/{id}` | Inbox deliveries |

---

## 12. Wire Format Quick Reference

- **Framing**: 4-byte big-endian length prefix + CBOR payload
- **Encoding**: ECF (Entity Canonical Form) — deterministic CBOR per RFC 8949 s4.2
- **ECF rules**: sorted map keys (by encoded length then lexicographic), minimal integers, definite lengths, shortest floats, no duplicate keys
- **Hash**: `0x00` + SHA-256 of ECF-encoded `{type, data}` = 33 bytes
- **Signature**: Ed25519 over full hash bytes (33 bytes including format code)
- **PeerID**: Base58(0x01 || 0x01 || SHA-256(public_key)) = 46 characters

---

## 13. Common Patterns

### Read an entity from the tree
```
EXECUTE  uri: "system/tree"  operation: "get"
  resource: {targets: ["peer_id/system/type/system/peer"]}
  params: {type: "system/tree/get-request", data: {}}
```

### List all handlers
```
EXECUTE  uri: "system/tree"  operation: "get"
  resource: {targets: ["system/handler/"]}
  params: {type: "system/tree/get-request", data: {}}
```

### Write an entity
```
EXECUTE  uri: "system/tree"  operation: "put"
  resource: {targets: ["peer_id/data/my-key"]}
  params: {type: "system/tree/put-request", data: {
    entity: {type: "my/type", data: {key: "value"}, content_hash: <33 bytes>}
  }}
```

### Delete a path binding
```
EXECUTE  uri: "system/tree"  operation: "put"
  resource: {targets: ["peer_id/data/my-key"]}
  params: {type: "system/tree/put-request", data: {}}
  # entity absent → removes binding
```

### Call a domain handler
```
EXECUTE  uri: "entity://peer_id/local/files/readme.md"  operation: "read-metadata"
  resource: {targets: ["local/files/readme.md"]}
  params: {type: "local/files/read-request", data: {...}, content_hash: <33 bytes>}
```

### Snapshot for sync (with exclusion)
```
EXECUTE  uri: "system/tree"  operation: "snapshot"
  resource: {targets: ["peer_id/local/files/*"], exclude: ["peer_id/local/files/.git/*"]}
  params: {type: "system/tree/snapshot-request", data: {}}
```

---

## 14. Gotchas and Critical Rules

1. **Absolute paths start with `/`** — after URI normalization, paths are `/{peer_id}/rest` (absolute). Peer-relative paths (`rest`, no leading `/`) are resolved to absolute by canonicalization. Storage and dispatch always use absolute paths.
2. **Params and results are entities** — they have type, data, and content_hash, not raw values
3. **Signatures point TO content** — scan `included` for `system/signature` where `target == entity.content_hash`
4. **Grants have three required fields** — `handlers`, `resources`, `operations`. All three must be present. Each has one purpose.
5. **Two checks always** — dispatch scope (`check_permission` with resource target) AND handler path scope (`check_path_permission`, defense-in-depth) must both pass
6. **Entity fidelity** — validate hash on receipt, then trust it. Store original bytes. Forward original. Never re-serialize.
7. **Unknown fields preserved** — entities may contain fields not in their type definition. Don't strip them.
8. **Connection setup is the only special case** — no auth required. Everything else follows the same dispatch chain.
9. **Canonicalize both sides** — when pattern matching, canonicalize both the path AND the pattern with the local peer_id
10. **Root capabilities must be local** — `granter.data.peer_id == local_peer_id` (string comparison)
11. **System data paths use tree handler for access** — `system/type/*`, `system/handler/*` are tree data, accessed via tree handler with `{handlers: {include: ["system/tree"]}, resources: {include: ["system/type/*"]}, operations: {include: ["get"]}}`. Dedicated handlers (e.g., `system/type` for `validate`) provide domain operations beyond data access.
