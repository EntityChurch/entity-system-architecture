# Query Extension — Normative Specification

**Version**: 1.7
**Status**: Active
**Depends**: ENTITY-CORE-PROTOCOL.md (v7.9+), ENTITY-NATIVE-TYPE-SYSTEM.md (v4.0+), EXTENSION-SUBSCRIPTION.md (v3.5+, for tree change event semantics)

---

> **Path notation.** Paths in this document use peer-relative notation (without leading `/{peer_id}/`). All peer-relative paths resolve to the local peer's namespace: `system/tree` means `/{local_peer_id}/system/tree`. Every path in the entity tree is absolute at rest — rooted at a peer identity. See ENTITY-CORE-PROTOCOL.md §1.4 for the path model. Cross-peer examples use absolute paths with explicit peer identities.

## 1. Introduction

### 1.1 Purpose

The query extension adds secondary indexes and a compositional query mechanism to the entity system. Without it, the only data access patterns are point lookup by hash (content store), point lookup by path (tree), and prefix listing (tree). Finding entities by type, by field value, or by reference requires scanning the entire tree.

The query extension provides:
- **Type index** — find entities by type name
- **Reverse hash index** — find entities that reference a given hash
- **Path link index** — find entities that reference a given tree path
- **Field index** — find entities by field values (selective, per-type configuration)
- **Query handler** — compositional query operations over these indexes

### 1.2 Scope

This extension specifies:
- Index semantics (what each index tracks, maintenance requirements)
- Query expression and result types
- Query handler operations (`find`, `count`)
- Capability integration (tree scope, content store scope, constraints, allowances)
- Conformance levels

This extension does NOT specify:
- Index data structures (implementation choice)
- Graph traversal operations (Level 3, future)
- Cross-peer query handler (Level 3, future — types defined here are reusable)
- Cascade operations (handler domain logic using reverse index infrastructure)
- Aggregation operations (Level 3, future)

### 1.3 Conformance Levels

**Level 1 (Core Query):** Type index, reverse hash index, query handler with `find` and `count`, capability-filtered results, cursor-based pagination. MUST for conformant implementations.

**Level 2 (Field Query):** All of Level 1, plus field index, field predicates, compound queries, index configuration, path-prefix zone summaries (§2.5). SHOULD for implementations supporting domain-specific queries.

**Level 3 (Advanced, future):** Graph traversal, aggregation, cross-peer queries, type hierarchy matching. Reserved for future specification.

---

## 2. Secondary Indexes

### 2.1 Type Index

The type index maps entity type names to the set of (path, hash) pairs where entities of that type are bound in the tree.

```
type_name → set of {path, hash}
```

**Semantics:**

- Every entity bound in the tree has exactly one entry in the type index, keyed by its `type` field.
- An entity bound at multiple tree paths has one entry per path.
- The type index has two entry forms: tree-bound entries `(type, path, hash)` maintained on tree-change events, and content-side entries `(type, null, hash)` maintained on content-store events (§3.1). Implementations supporting `allowances.scope: "content_store"` MUST maintain content-side entries; implementations not supporting that scope MAY maintain only tree-bound entries.

**Maintenance:**

On tree change event `(event_type, uri, new_hash, previous_hash)`:

- `created` or `updated`: Resolve entity at `new_hash` from content store. Add entry `(uri, new_hash)` under `entity.type`.
- `updated` or `deleted`: Resolve entity at `previous_hash`. Remove entry `(uri, previous_hash)` from its type.

Cost: O(1) per tree change.

**Conformance:** MUST (Level 1).

### 2.2 Reverse Hash Index

The reverse hash index maps content hashes to the set of entities that reference them. It tracks all `system/hash` values found in entity data, at any nesting depth.

```
referenced_hash → set of {source_path, source_type, field_name}
```

**Semantics:**

- For each entity bound in the tree, every `system/hash` value in its `data` fields creates a reverse index entry pointing back to that entity.
- The index builder MUST walk entity data recursively: hash values inside arrays (`array_of`), maps (`map_of`), and nested type references (`type_ref`) MUST be indexed. The entity's type definition provides the structure to traverse.
- `field_name` records the top-level field containing the reference. For nested references, this is the root field name (not the full path within nested structures).

**Maintenance:**

On tree change:

- `created` or `updated`: Resolve entity at `new_hash`. Walk data fields recursively, collecting all `system/hash` values. For each, add reverse entry `(uri, entity.type, field_name)`.
- `updated` or `deleted`: Resolve entity at `previous_hash`. Remove its reverse entries.

Cost: O(hash_references_per_entity) per tree change.

**Use cases:**

- **GC safety:** `reverse_index.get(hash)` returns empty → safe to delete from content store.
- **Impact analysis:** "What depends on this type definition?" → find all entities referencing the type's hash.
- **Cascade:** "What needs updating when this entity changes?" → find referrers, rebuild with new hash.

**Conformance:** MUST (Level 1).

### 2.3 Path Link Index

The path link index maps tree paths to the set of entities that reference them. It tracks `system/tree/path` values found in entity data, identified by the field's type declaration.

```
referenced_path → set of {source_path, source_type, field_name}
```

**Semantics:**

- For each entity bound in the tree, every `system/tree/path` value in its `data` fields creates a path link entry.
- Fields are identified as path references by their type definition: `type_ref: "system/tree/path"`. Arbitrary `primitive/string` values that happen to contain paths are NOT indexed.
- The index builder MUST walk arrays and nested structures, same as the reverse hash index.

**Maintenance:** Same pattern as reverse hash index, for `system/tree/path` values. Cost: O(path_references_per_entity) per tree change.

**Use cases:**

- Backlinks: "What entities link to this path?" (wiki, knowledge base)
- Impact analysis for path reorganization: "What breaks if I move this path?"
- Broken link detection: path in index has no tree binding

**Conformance:** SHOULD (Level 1). Implementations without the path link index can answer path reference queries via field index lookup or type scan fallback.

### 2.4 Field Index

The field index maps (type, field, value) tuples to the set of entities matching that combination. Field indexes are selective — only configured (type, field) pairs are indexed.

```
(type_name, field_name, field_value) → set of {path, hash}
```

**Semantics:**

- The field index key ALWAYS includes the type name. Fields are scoped to their declaring type. Two types with a field of the same name have independent index partitions.
- Only configured (type, field) pairs are indexed. Unconfigured fields are not indexed. This is the right default — field indexes are costly for large datasets.
- Field values are stored in their logical form (the decoded value), not their wire encoding. Comparison semantics are defined per primitive type (§4.4).

**Configuration:**

Which (type, field) pairs to index is an operational decision. The spec defines the semantics; the configuration mechanism is implementation-defined. Configuration MAY be entity-native (entities stored at well-known tree paths) or implementation-level (startup configuration, handler parameters).

```
; Example configuration (informative, not normative):
{type: "system/query/index-config", data: {
  type_name: "app/user",
  fields: ["city", "role", "created_at"]
}}
```

**Maintenance:**

On tree change for an entity whose type has configured field indexes:

- Resolve entity, decode per type definition, extract indexed field values.
- Add/remove field index entries for each configured field.

Cost: O(indexed_fields) per tree change. Requires type-aware decoding (shared infrastructure with type extension validation).

**Fallback:** A field query against an un-indexed field SHOULD succeed by falling back to a type-index scan: enumerate all entities of the queried type, decode each, apply the field predicate. This is O(entities_of_type) instead of O(matching_entities). Implementations MAY return an informational note in the response indicating a scan was used.

**Conformance:** SHOULD (Level 2).

### 2.4.1 Adaptive Index Promotion (Informative)

The configured field index model in §2.4 requires operators to declare which `(type, field)` pairs to index before query patterns are observed. Implementations MAY supplement this with adaptive promotion: observe fallback-scan patterns and promote heavily-queried `(type, field)` pairs to maintained indexes automatically.

**Pattern (informative, not normative):**

1. **Observation.** Record `(type, field, predicate_shape, scan_cost)` for each fallback scan triggered by §2.4.
2. **Partial retention.** On scan completion, retain a partial mapping for values the query actually probed. Subsequent queries refine this partial structure (cracking-inspired).
3. **Promotion threshold.** Cross a threshold (count of scans, cumulative scan cost, or a cost-based projection) → promote `(type, field)` to a fully maintained field index per §2.4.
4. **Capability gating.** Auto-promotion adds permanent write amplification; implementations SHOULD gate it under a capability allowance so operators can opt in per peer or per scope. The exact allowance shape (boolean, scoped, etc.) is left to the capability allowance system; a future amendment should align this with the broader allowance design.
5. **Demotion.** A promoted index that sees no queries for a configurable window MAY be dropped. Subsequent queries pay scan cost again. Symmetric with promotion.
6. **Observability.** Promotions and demotions SHOULD emit entities (e.g., `system/query/index-promoted`) so operators can inspect and override the system's choices.

This pattern follows the database cracking line of research (Idreos & Kersten, CWI). It is informative because the precise threshold policy, retention strategy, and demotion criteria are implementation-defined. The principle is: pay write-amplification cost for indexes only on `(type, field)` pairs that the workload actually queries.

### 2.5 Path-Prefix Zone Summaries

Path-prefix zone summaries provide automatic min/max statistics for primitive-ordered fields per (path-prefix, type, field) triple. They enable data-skipping for range predicates without requiring per-field index configuration.

```
(prefix, type_name, field_name) → { min, max, count, null_count }
```

**Semantics:**

- For each tree-bound entity of a type with primitive-ordered fields (`primitive/string`, `primitive/int`, `primitive/uint`, `primitive/float`, etc.), every ancestor prefix of the entity's path receives a summary entry for each indexable field.
- "Ancestor prefix" includes the path's own components and the root. A summary at prefix `app/users/` covers all entities under `app/users/*`.
- Primitive-ordered fields are determined by the type definition: any field whose `type_ref` points to a primitive type with a defined ordering.
- The summary tracks min, max, count of present values, and count of null/absent values.

**Maintenance:**

On tree change `(event_type, uri, new_hash, previous_hash)`:

- `created` or `updated`: For each ancestor prefix, update min/max with the entity's primitive field values; increment count or null_count.
- `deleted` or `updated` (previous_hash version): For each ancestor prefix, decrement counts and (if needed) recompute min/max from a scan of the prefix.

Cost: O(tree_depth × indexed_fields) per write — effectively O(1) for realistic trees.

**Use:**

For a query with `type_filter=T, field_filter=F op V, path_prefix=P`:

1. Look up summary at `(P, T, F)`.
2. If summary's range doesn't overlap the predicate (`V` outside `[min, max]` for range predicates), skip the entire subtree under P.
3. Otherwise, descend into the subtree and apply the predicate to each entity (or use Tier B/C indexes if available).

**Conformance:** SHOULD (Level 2). Implementations supporting Level 2 SHOULD maintain zone summaries for all primitive-ordered fields by default. The maintenance cost is bounded; the storage cost is two values per (prefix, type, field) triple. Implementations MAY make zone summaries configurable per type if memory pressure demands it.

---

## 3. Index Maintenance

### 3.1 Index Maintenance Events

Secondary indexes are maintained in response to two event types from the emit pipeline (SYSTEM-COMPOSITION.md §1.1):

**Content-store event** — fires on every `content_store.put(entity)` that adds new content:

```
on_content_store_event(hash, entity)
```

Updates:
- **Type index** (content-side entry): `(entity.type, hash)` with null path.
- **Reverse hash index**: walk entity data; for each `system/hash` ref, record `(referenced_hash) → (source_hash, source_type, field_name)`.
- **Field index**: extract configured field values; record `(type, field, value) → {hash}`.
- **Zone summaries** (§2.5): update min/max for each indexable primitive field.

This is where the bulk of index maintenance work happens. Every entity entering the content store — tree-bound or not — is indexed at this phase.

**Tree-change event** — fires on every `location_index.set/delete` that changes a binding:

```
on_tree_change(event_type, uri, new_hash, previous_hash)
```

Where `event_type` is `created`, `updated`, or `deleted`. This is the same event defined by EXTENSION-SUBSCRIPTION.md §4.1. Updates:
- **Path link index**: walk entity data; for each `system/tree/path` ref, record `(referenced_path) → (source_path, source_type, field_name)`.
- **Type index (path annotation)**: annotate the existing type-index entry for `new_hash` with `uri`. On `deleted` or `updated`, remove/replace the path annotation on the entry for `previous_hash`.

Work at this phase is minimal; most of the index data was populated at content-event time. The path link index is the only path-scoped structure requiring full walk work; the type index update is a single-column annotation on an existing row.

For tree-bound entities, the content-store event fires first (populating content-side index data), then the tree-change event fires (adding the path). For content-only entities, only the content-store event fires; the entry has a null path.

**Cost.** Content-side: O(indexed_fields + hash_refs + ancestor_prefixes_for_zone) per content-store event. Tree-side: O(path_refs) per tree-change event, plus O(1) for the path annotation.

**Conformance.** Maintenance on content-store events is MUST when the implementation supports `allowances.scope: "content_store"` queries (§5.5.2); otherwise MAY. Maintenance on tree-change events is MUST at Level 1.

Query index maintenance runs as the first emit pathway consumer for tree-change events (transparent, no emission), and at position 1 on content-store events after persistence (transparent, no emission), as specified in SYSTEM-COMPOSITION.md §2.2. This ensures indexes are current before any subsequent consumer queries them. The query extension exposes both a handler (for explicit EXECUTE dispatch to `system/query`) and a synchronous emit consumer (for index maintenance) — see SYSTEM-COMPOSITION.md §2.6.

### 3.2 Maintenance Mechanism

Index maintenance is typically an **implementation-level hook** in the emit pathway — synchronous with the tree write that triggers it. This ensures queries see consistent state immediately after a write completes.

Implementations MAY also use protocol-level subscriptions for index maintenance, with the caveat that subscription entities themselves are tree-bound and require indexing — introducing a bootstrapping dependency. The spec does not mandate the mechanism.

### 3.3 Synchronous Consistency

Index updates MUST be consistent with tree state: after a tree write returns successfully, subsequent queries MUST reflect the write. This means index updates are either synchronous with the write or appear so from the query handler's perspective.

### 3.4 Rebuild

All secondary indexes can be rebuilt from a full scan of the tree and content store:

```
rebuild_indexes():
  clear all secondary indexes
  ; Phase 1: content-side indexes
  for hash in content_store.all_hashes():
    entity = content_store.get(hash)
    update_type_index_content(entity.type, hash)    ; null path
    update_reverse_index(entity, hash)
    update_field_indexes(entity, hash)
    update_zone_summaries(entity, hash)
  ; Phase 2: path annotations + path link index
  for (uri, hash) in tree.all_bindings():
    annotate_type_index_with_path(uri, hash)
    entity = content_store.get(hash)
    update_path_link_index(entity, uri)
```

Rebuild cost: O(all content-store entities) for Phase 1 + O(all tree bindings) for Phase 2. This is the recovery mechanism for corrupted or lost indexes. Acceptable for startup; not for normal operation.

Implementations that do not support `allowances.scope: "content_store"` MAY skip Phase 1's work for non-tree-bound entities: iterate `tree.all_bindings()`, resolve each entity, and call both content-side and tree-side index update functions in a single pass. The choice is implementation-defined based on which query scopes are supported.

---

## 4. Query Types

### 4.1 system/query/expression

The query expression type describes a declarative query over the indexed entity set.

```
system/query/expression := {
  fields: {
    type_filter:       {type_ref: "primitive/string", optional: true}
                       ; Entity type name or glob pattern (e.g., "app/user", "system/*").
                       ; Uses type index. Supports glob matching: "*" matches any
                       ; suffix. REQUIRED when field_filters is non-empty.

    field_filters:     {array_of: {type_ref: "system/query/field-predicate"}, optional: true}
                       ; Filter by field values. Uses field index (or type-scan fallback).
                       ; type_filter MUST be present when this field is non-empty.

    ref_filter:        {type_ref: "system/hash", optional: true}
                       ; Find entities referencing this hash. Uses reverse hash index.

    path_filter:       {type_ref: "system/tree/path", optional: true}
                       ; Find entities referencing this path. Uses path link index
                       ; (or field scan fallback).

    path_prefix:       {type_ref: "system/tree/path", optional: true}
                       ; Restrict results to entities under this tree prefix.

    limit:             {type_ref: "primitive/uint", optional: true}
                       ; Maximum results to return. Default: implementation-defined.

    cursor:            {type_ref: "primitive/string", optional: true}
                       ; Opaque pagination cursor from a previous result.
                       ; Tied to this query's expression and ordering.

    order_by:          {type_ref: "primitive/string", optional: true}
                       ; Field name to sort results by. Default: path (lexicographic).

    descending:        {type_ref: "primitive/bool", optional: true}
                       ; Sort direction. Default: false (ascending).

    include_entities:  {type_ref: "primitive/bool", optional: true}
                       ; If true, result is returned as a system/envelope with matched
                       ; entities in its included map. Default: false.
  }
}
```

**Composition:** All present filters are conjunctive (AND). An entity matches the query if:
- Its type matches `type_filter` (if present) — exact or glob match
- Its fields satisfy ALL `field_filters` (if present)
- It references `ref_filter` hash in any data field (if present)
- It references `path_filter` path in any `system/tree/path` field (if present)
- Its tree path starts with `path_prefix` (if present)

**`path_prefix` canonicalization (normative).** When present, `path_prefix` is canonicalized per ENTITY-CORE-PROTOCOL.md §5.4 before use:

| Value | After canonicalization | Effect |
|---|---|---|
| absent | — | No path scoping applied (field is optional) |
| `""` (empty string) | `/{local_peer_id}/` | Scan local peer's entire tree |
| `"/"` | `"/"` | All peers, all paths (cross-peer wildcard scope) |
| `"some/prefix"` | `/{local_peer_id}/some/prefix` | Scoped scan under local peer's prefix |
| `"/{peer_id}/some/prefix"` | (already absolute) | Scoped scan under named peer's prefix |

Field-present-with-empty-value (`path_prefix: ""`) is a legitimate filter and is NOT subject to the `empty_query` gate (§5.4). The empty_query gate triggers only when **no filter fields are present at all**.

**Per-result capability filtering applies regardless of `path_prefix` breadth.** The `path_prefix` scopes the *scan*; the per-result capability check (ENTITY-CORE-PROTOCOL.md §6.6) scopes the *visibility*. Results blocked by the caller's capability scope MUST NOT appear in the result set, even under the `"/"` cross-peer wildcard. This mirrors the membership principle stated for `discover_handlers` conformance (SDK-OPERATIONS §9.3) and applies uniformly to `find` and `count`. Cross-peer wildcard scans are bounded by the dispatcher's per-entity capability gate, not by `path_prefix` itself.

**Validation rules:**
- `type_filter` MUST be present when `field_filters` is non-empty (status 400 if absent). Fields are type-scoped; a field predicate without a type is semantically undefined.
- `cursor` is opaque and tied to the query's expression and `order_by` setting. A cursor from one query MUST NOT be used with a different query expression or ordering.
- `order_by` names a field in the entity's data. When absent, results are ordered by path (lexicographic ascending). When present, results are ordered by the named field's value. `descending` reverses the order.

### 4.2 system/query/field-predicate

```
system/query/field-predicate := {
  fields: {
    field:    {type_ref: "primitive/string"}
              ; Field name in entity data.
    operator: {type_ref: "primitive/string"}
              ; Comparison operator (see §4.4).
    value:    {type_ref: "primitive/any", optional: true}
              ; Comparison value. Absent for unary operators (exists).
  }
}
```

### 4.3 system/query/result

```
system/query/result := {
  fields: {
    matches:  {array_of: {type_ref: "system/query/match"}}
    total:    {type_ref: "primitive/uint"}
              ; Total matching count (capability-filtered). Reflects the full
              ; result set size, not just this page.
    has_more: {type_ref: "primitive/bool"}
              ; True if more results exist beyond this page.
    cursor:   {type_ref: "primitive/string", optional: true}
              ; Cursor for next page. Absent if no more results.
  }
}
```

```
system/query/match := {
  fields: {
    path:  {type_ref: "system/tree/path", optional: true}
           ; Tree path where entity is bound (peer_id/rest/of/path).
           ; Always peer-namespaced per ENTITY-CORE-PROTOCOL §1.4.
           ; Absent for content-store-only entities (content_store scope, §5.4).
    hash:  {type_ref: "system/hash"}
           ; Content hash of the matched entity.
    type:  {type_ref: "system/type/name"}
           ; Entity type.
  }
}
```

**Result semantics:**

- `total` is the exact count of all matching, capability-filtered entities — not just the current page. This value MUST reflect the post-filter count, not the raw index count. Computing `total` requires evaluating the full result set.
- When `include_entities` is true, the handler SHOULD return a `system/envelope` result (root = `system/query/result`, included = matched entities keyed by content hash).
- Results are ordered per `order_by` / `descending` (default: path ascending).
- Pagination is live, not snapshot. Each page reflects current tree state at the time of the request. Entities added or removed between pages affect subsequent pages.
- Cursors are opaque, implementation-defined, and MAY expire. An expired or invalid cursor returns status 400.

### 4.4 Operators

Operators are organized by conformance level:

**Level 1 operators** (MUST support):

| Operator | Arity | Semantics |
|----------|-------|-----------|
| `eq` | binary | Field value equals `value`. |
| `not_eq` | binary | Field value does not equal `value`. |
| `in` | binary | Field value is one of the values in `value` (which MUST be an array). |
| `exists` | unary | Field is present and non-null. `value` is absent. |

**Level 2 operators** (SHOULD support):

| Operator | Arity | Semantics |
|----------|-------|-----------|
| `gt` | binary | Field value is greater than `value`. |
| `lt` | binary | Field value is less than `value`. |
| `gte` | binary | Field value is greater than or equal to `value`. |
| `lte` | binary | Field value is less than or equal to `value`. |
| `prefix` | binary | Field value (string) starts with `value` (string). |
| `substring` | binary | Field value (string) contains `value` (string) as a substring. |
| `contains` | binary | Field value (array) contains `value` as an element. |

**Comparison semantics by type:**

| Field type | `eq` / `not_eq` | `gt` / `lt` / `gte` / `lte` | `prefix` / `substring` | `contains` |
|-----------|-----------------|---------------------------|----------------------|-----------|
| `primitive/string` | Byte equality (CBOR canonical) | Lexicographic byte order | Byte prefix / substring | N/A (not array) |
| `primitive/uint`, `primitive/int`, `primitive/float` | Numeric equality | Numeric order | N/A (not string) | N/A |
| `primitive/bool` | Boolean equality | N/A | N/A | N/A |
| `system/hash` | Byte equality | N/A | N/A | N/A |
| `system/tree/path` | Byte equality | Lexicographic | Byte prefix | N/A |
| Array types | Element-wise equality | N/A | N/A | Element membership |

Applying an operator to an incompatible field type (e.g., `gt` on a boolean) returns status 400.

Cross-type comparison (e.g., `gt` comparing a string value to a numeric `value`) returns status 400.

---

## 5. Query Handler

### 5.1 Handler Registration

```
Handler:
  pattern:    "system/query"
  operations: {
    find:  {input_type: "system/query/expression", output_type: "system/query/result"}
    count: {input_type: "system/query/expression", output_type: "primitive/uint"}
  }
```

The query handler is a system handler registered at `system/query`. It operates under the standard capability model (§5.3).

### 5.2 find Operation

Evaluates a query expression against indexes and returns matching entities, filtered by the caller's capability scope.

```
EXECUTE system/query  operation: "find"
  params: {type: "system/query/expression", data: {
    type_filter: "app/user",
    field_filters: [{field: "city", operator: "eq", value: "Seattle"}],
    path_prefix: "app/users/",
    limit: 50,
    include_entities: true
  }}
```

**Execution algorithm:**

```
handle_find(ctx, expression):
  ; 1. Resolve query constraints and allowances.
  ;    The dispatch layer already ran check_permission (core §5.2) to authorize
  ;    this EXECUTE. The query handler needs the constraints and allowances from
  ;    the matching grant entry. Implementations MUST expose the matching grant's
  ;    constraints and allowances to the handler, either through the execution
  ;    context or by re-evaluating grant matching within the handler.
  ;    If the matching grant has no constraints/allowances fields, defaults apply.
  constraints = ctx.matching_grant.constraints or {}
  allowances  = ctx.matching_grant.allowances or {}
  scope = allowances.scope or "tree"

  ; 2. Content store scope REQUIRES type_scope on the grant constraints
  if scope == "content_store" and constraints.type_scope is null:
    return error(403, "content_store_requires_type_scope")

  ; 3. Validate type_filter against type_scope (fail fast)
  ;    Uses matches_scope from core protocol §5.2 — same function used for
  ;    id-scope matching on operations and peers dimensions.
  if constraints.type_scope is not null and expression.type_filter is not null:
    if not matches_scope(expression.type_filter, constraints.type_scope, ctx.local_peer_id):
      return error(403, "type_not_authorized")

  ; 4. Validate expression structure
  if expression.field_filters is not null and expression.type_filter is null:
    return error(400, "type_filter_required",
      "type_filter is required when field_filters is present")

  ; 5. Execute against indexes.
  ;    Index lookup strategy is implementation-defined. When multiple filters
  ;    are present, the implementation intersects result sets (all filters
  ;    are conjunctive). Typical strategy: start with the most selective
  ;    index, then filter remaining candidates against other criteria.
  candidates = execute_index_lookups(expression, scope)

  ; 6. Filter by capability — two checks per candidate
  filtered = []
  for candidate in candidates:

    ; 6a. Type check (when type_scope is set on constraints)
    if constraints.type_scope is not null:
      if not matches_scope(candidate.type, constraints.type_scope, ctx.local_peer_id):
        continue  ; type not authorized — skip silently

    ; 6b. Path check
    if scope == "tree":
      ; Tree scope: every result MUST have a path, MUST pass path check
      if candidate.path is null: continue
      if not check_path_permission("get", candidate.path, ctx.capability,
                                    "system/query", ctx.local_peer_id): continue
    else:  ; content_store
      ; Content store scope: path check for entities that HAVE paths,
      ; pathless entities authorized by type_scope (checked in 6a)
      if candidate.path is not null:
        if not check_path_permission("get", candidate.path, ctx.capability,
                                      "system/query", ctx.local_peer_id): continue
      ; Pathless entities: if we reach here, type_scope passed (6a) — authorized

    filtered.add(candidate)

  ; 7. Sort
  ;    Default: by path, ascending. Null paths (content_store scope) sort
  ;    AFTER all path-bearing results.
  ;    When order_by names a field: sort by that field's value. Entities
  ;    where the field is absent sort AFTER entities where it is present.
  sort_key = expression.order_by or "path"
  sort(filtered, sort_key, expression.descending or false, nulls_last=true)

  ; 8. Apply limits — grant constraint caps the query limit
  effective_limit = min(
    expression.limit or DEFAULT_QUERY_LIMIT,
    constraints.max_results or MAX_QUERY_LIMIT
  )

  ; 9. Paginate
  ;    Cursor encodes position in the sorted, filtered result set.
  ;    Cursor is opaque, implementation-defined, and tied to the query's
  ;    sort_key + descending setting. A cursor from a different query
  ;    or different ordering MUST be rejected (status 400).
  start = resolve_cursor(expression.cursor, sort_key)
  page = filtered[start..start + effective_limit]

  ; 10. Build response
  result = {
    matches: page,
    total: len(filtered),
    has_more: start + effective_limit < len(filtered),
    cursor: encode_cursor(page[-1], sort_key) if has_more else null
  }

  ; 11. Include entities if requested — wrap result in system/envelope
  if expression.include_entities:
    result_envelope_included = {}
    for match in page:
      result_envelope_included[match.hash] = content_store.get(match.hash)
    return {type: "system/envelope", data: {root: result, included: result_envelope_included}}

  return result
```

**Implementation notes on the algorithm:**

**Constraint and allowance access (step 1).** The dispatch layer runs `check_permission` (core §5.2) which returns ALLOW/DENY. The query handler additionally needs the matching grant's `constraints` and `allowances` fields. Implementations SHOULD pass the matching grant entry through the execution context (e.g., `ctx.matching_grant`). Alternatively, the handler MAY re-evaluate grant matching internally — the result is the same since the same grant must match.

**Pattern matching (step 3, 6a).** Both steps use `matches_scope` from ENTITY-CORE-PROTOCOL.md (§5.2) — the same function that evaluates `id-scope` include/exclude patterns for operations and peers dimensions. `type_filter` in the expression supports the same pattern syntax as `matches_pattern` (core §5.4): exact match, trailing `/*` for prefix match, bare `*` for match-all. When a glob `type_filter` (e.g., `"app/*"`) is checked against a `type_scope` (e.g., `{include: ["app/user", "app/order"]}`), the check asks: "is every type matching `app/*` authorized?" This is conservative — if the type_scope doesn't include the wildcard `"app/*"` or broader, the check fails. Callers should use specific type names when type_scope is narrow.

**Index intersection (step 5).** When multiple filters are present (e.g., `type_filter` + `ref_filter` + `path_prefix`), the implementation intersects results from each index. Strategy is implementation-defined. Typical approach: query the most selective index first, then filter candidates against remaining criteria. The spec does not mandate an execution order.

**Null ordering (step 7).** Nulls sort last by default. Entities without a `path` (content store scope) appear after all path-bearing results when sorted by path. Entities where an `order_by` field is absent appear after entities where it is present.

**Algorithm presentation (steps 6-9).** The algorithm is presented as if `filtered` is fully materialized in memory. Real implementations SHOULD integrate cursor resolution with index scanning and filtering to avoid materializing the full result set. The `total` count MAY require a separate counting pass.

**Unsupported features.** When an implementation receives expression fields it does not support (`order_by` at Level 1, `include_entities` at Level 1, `field_filters` at Level 1), it SHOULD silently ignore them and return results in default order without included entities. It MUST NOT return an error for unrecognized optional fields — this preserves forward compatibility. Fields ignored by `count` (`limit`, `cursor`, `order_by`, `descending`, `include_entities`) are likewise silently ignored.

### 5.3 count Operation

Returns the total number of matching entities, subject to capability filtering. Same expression type as `find`, but returns only the count.

```
EXECUTE system/query  operation: "count"
  params: {type: "system/query/expression", data: {
    type_filter: "app/user",
    path_prefix: "app/users/"
  }}

→ Returns: {type: "primitive/uint", data: 42}
```

The count is exact and capability-filtered. Computing `count` requires evaluating the full result set and applying capability checks — same cost as `find` without materialization. The `limit`, `cursor`, `order_by`, `descending`, and `include_entities` fields are silently ignored for `count`.

### 5.4 Error Conditions

| Status | Code | Condition |
|--------|------|-----------|
| 400 | `type_filter_required` | `field_filters` present but `type_filter` absent |
| 400 | `invalid_operator` | Operator not recognized or incompatible with field type (e.g., `gt` on boolean) |
| 400 | `cross_type_comparison` | Operator value type doesn't match field's declared type |
| 400 | `invalid_cursor` | Cursor is expired, malformed, or from a different query/ordering |
| 400 | `empty_query` | No filter fields present (no `type_filter`, `path_prefix`, `field_filters`, `ref_filter`, or `path_filter`) — would match all entities. This is a DoS-prevention gate against unbounded scans; implementations SHOULD reject rather than performing a full scan. Field-present-with-empty-value (e.g., `path_prefix: ""`) is a legitimate filter per §4.1 and does NOT trigger this gate. |
| 403 | `type_not_authorized` | `type_filter` does not match `type_scope` on the grant's constraints |
| 403 | `content_store_requires_type_scope` | `allowances.scope: "content_store"` on grant but `constraints.type_scope` is absent |

### 5.5 Capability Integration

The query handler uses the standard four-dimension capability model. Query-specific behavior is controlled via the `constraints` and `allowances` fields on grant entries. Constraints narrow access (absent = unconstrained). Allowances expand access (absent = most restricted). See ENTITY-CORE-PROTOCOL.md §5.6 for the general model.

**Grant example (tree-scoped):**
```
{
  handlers:    {include: ["system/query"]},
  resources:   {include: ["app/*"]},
  operations:  {include: ["find", "count"]},
  constraints: {max_results: 500, type_scope: {include: ["app/user", "app/order/*"]}}
}
```

**Grant example (content-store-scoped):**
```
{
  handlers:    {include: ["system/query"]},
  resources:   {include: ["*"]},
  operations:  {include: ["find", "count"]},
  constraints: {type_scope: {include: ["*"]}},
  allowances:  {scope: "content_store"}
}
```

#### 5.5.1 Query Constraint and Allowance Types

```
system/query/constraints := {
  fields: {
    max_results: {type_ref: "primitive/uint", optional: true}
                 ; Maximum results per query. Null = implementation default.

    type_scope:  {type_ref: "system/capability/id-scope", optional: true}
                 ; Which entity types can be queried.
                 ; Uses id-scope {include, exclude} with glob matching.
                 ; REQUIRED when scope allowance is "content_store".
                 ; Null with tree scope = any type at authorized paths.
  }
}

system/query/allowances := {
  fields: {
    scope:       {type_ref: "primitive/string", optional: true}
                 ; "content_store" grants access to non-tree-bound entities.
                 ; Absent = tree scope only (default, most restricted).
  }
}
```

#### 5.5.2 Two Query Scopes

**Tree scope (default).** No `allowances`, or `allowances.scope` absent. Query results are restricted to entities bound in the tree at paths the caller's `resources` scope covers. Every result has a non-null `path`. Capability filtering uses `check_path_permission` per result — same pattern as tree listing.

**Content store scope.** `allowances.scope: "content_store"`. Query results can include entities from the content store regardless of tree binding. Results for entities not bound in the tree have `path: null`. `type_scope` is REQUIRED on the constraints — the granter must explicitly declare which types are accessible without path filtering.

| Aspect | Tree scope | Content store scope |
|--------|-----------|-------------------|
| Results | Tree-bound entities only | All entities (including unbound) |
| Path filtering | Per-result `check_path_permission` | Applied to entities with paths; pathless entities pass if type authorized |
| Type filtering | Optional (via `type_scope`) | Required (via `type_scope`) |
| Default | Yes (no allowances needed) | No (explicit `scope` allowance required) |
| `path_prefix` | Restricts to subtree | Restricts tree-bound results; unbound results unaffected |

#### 5.5.3 Result Invariants

- All `total` and `count` values MUST reflect the capability-filtered result set, never the raw index count.
- An entity outside the caller's scope MUST be indistinguishable from non-existent.
- Pagination applies after capability filtering: page N contains the Nth page of filtered results, not the Nth page of raw results with filtering applied.

#### 5.5.4 Constraint and Allowance Attenuation

The core attenuation algorithm (ENTITY-CORE-PROTOCOL.md §5.6) enforces structural invariants for query constraints and allowances:

- **Key retention**: `max_results` and `type_scope` cannot be dropped from constraints if parent has them.
- **Key containment**: `scope` cannot be added to allowances if parent doesn't have it.
- **Byte equality**: All constraint and allowance values must be byte-identical between parent and child.

With byte equality (the conformance default), constraint and allowance values are frozen during delegation — the delegatee gets exactly the parent's `max_results`, `type_scope`, and `scope` values. This is sufficient for most delegation scenarios.

If an implementation provides handler-mediated attenuation relaxation (ENTITY-CORE-PROTOCOL.md §5.6), the query handler would additionally verify:
- `max_results`: delegated value MUST be less than or equal to parent's value. Null (unlimited) can delegate to any value. A specific value cannot delegate to null.
- `type_scope`: delegated include set MUST be a subset of parent's include set. Delegated exclude set MAY add entries. Cannot widen type access.
- `scope`: "content_store" can delegate to "content_store" (same). Cannot widen "tree" to "content_store" (the core already prevents this — adding the key would fail the allowance containment check).

#### 5.5.5 Query Handler Internal Scope

The query handler requires broad read access to maintain indexes. Its handler registration SHOULD declare an `internal_scope` covering the paths it needs:

```
; Query handler internal scope (implementation-level):
internal_scope: {
  resources: {include: ["*"]},     ; reads all tree paths for indexing
  operations: {include: ["get"]}   ; read-only
}
```

This grants the query handler implementation access to read all tree-bound entities. This is an implementation-level grant (not exposed to callers). Callers access the query handler through their own grants, which are filtered through the capability model. The handler's broad access does not flow through to callers.

---

## 6. Security Considerations

### 6.1 Information Leakage via Existence

The `ref_filter` field lets a caller probe whether anything at their authorized paths references a given hash. An empty result means either: nothing references it, or references exist only at unauthorized paths. The caller cannot distinguish these cases (§5.4.3: unauthorized entities are indistinguishable from non-existent).

However, the ability to probe arbitrary hashes is itself an information channel. A caller could test whether specific content exists in the system by constructing entities that reference target hashes and checking whether the reverse index has entries. Implementations SHOULD consider whether `ref_filter` queries should be restricted to hashes the caller is already authorized to know about. The `constraints` type can be extended with a `ref_scope` field for this purpose if needed.

### 6.2 Timing Side-Channels

A query matching many entities (filtered down to few visible results) takes longer than a query matching few entities. An attacker could infer the approximate size of data outside their scope by measuring response time. This is a general property of filtered access (tree listing has the same issue) and is considered an implementation concern, not a protocol concern. Implementations MAY add constant-time padding or rate limiting to mitigate.

### 6.3 Content Store Scope Privileges

Content store scope (`allowances.scope: "content_store"`) is a significant privilege escalation — it grants access to entities not visible through the tree. Granters SHOULD:
- Only grant content store scope to trusted system components (GC, sync engine, admin tools)
- Always set `type_scope` to the narrowest necessary set
- Prefer tree scope for application-level queries

The reversed default (no allowances = tree scope) ensures that a grant for the query handler without explicit allowances does not accidentally expose content store access.

### 6.4 include_entities and Data Access

When `include_entities: true`, the query handler includes full entity data in the result envelope (`system/envelope`). For tree-scoped queries, the path check (step 6b) authorizes both the match metadata and the entity data — same authorization as a tree `get` on that path. For content-store-scoped queries, pathless entities are authorized by `type_scope` only. The `type_scope` grant is the authorization for both query visibility and entity data access in this mode.

---

## 7. Interaction with Other Extensions

### 7.1 Subscription Extension

The subscription extension provides the tree change event semantics used for index maintenance (§3.1). The query extension consumes these events but does not require implementations to use protocol-level subscriptions — implementation-level hooks in the emit pathway are sufficient and typical.

### 7.2 Type Extension

The type extension's decode algorithm (ENTITY-NATIVE-TYPE-SYSTEM.md §7) is shared infrastructure for field index maintenance. The field index builder uses the same type-aware decoding that the types handler uses for validation.

The type extension's `type_pattern` constraint (when specified) enables typed reference queries: the reverse hash index finds referencing entities, and the type constraint validates the reference type.

### 7.3 Content Extension

The content extension provides protocol-level access to the content store. The query handler resolves entities from the content store for index maintenance and for `include_entities` result delivery. This is typically implementation-level access, not handler-to-handler dispatch.

### 7.4 Version Extension

The version extension's merge operation can use query indexes for conflict detection. Type-aware merge strategies can query for entities of a specific type in the merge set, then apply type-specific resolution. The version extension's pagination convention (`limit` + `cursor` + `has_more`) is adopted by the query extension.

---

## 8. Constants and Limits

| Constant | Default | Description |
|----------|---------|-------------|
| `DEFAULT_QUERY_LIMIT` | 100 | Default `limit` when not specified in expression |
| `MAX_QUERY_LIMIT` | 10,000 | Maximum `limit` value (may be further constrained by `max_results` on grant) |
| `MAX_FIELD_FILTERS` | 16 | Maximum number of field predicates per query |
| `MAX_IN_VALUES` | 100 | Maximum number of values in an `in` operator |

Implementations MAY define different defaults. These values are recommendations for interoperability.

---

## 9. Implementation Guidance (Informative)

### 9.1 In-Memory Indexes

The simplest implementation uses in-memory data structures:

- **Type index:** `HashMap<String, Vec<(String, Hash)>>` — O(1) lookup by type.
- **Reverse hash index:** `HashMap<Hash, Vec<(String, String, String)>>` — O(1) lookup by hash.
- **Path link index:** `HashMap<String, Vec<(String, String, String)>>` — O(1) lookup by path.
- **Field index:** `BTreeMap` per (type, field) — O(log n) for range queries, O(1) for equality.

Memory cost: ~100 bytes per entity for type index, ~150 bytes per reference for reverse/path link indexes. For 10,000 entities with ~3 references each: total ~6 MB.

Indexes are rebuilt on startup by scanning the tree + content store. For 10,000 entities this takes milliseconds; for 1,000,000 entities, seconds.

### 9.2 SQL Backend

A SQL database (SQLite, Postgres) provides persistence, range queries, and concurrent access. The query expression type translates mechanically to SQL WHERE clauses. See EXPLORATION-QUERY-SQL-COMPARATIVE-ANALYSIS.md for the detailed mapping.

Key consideration: field index values should be stored in SQL-native types (INTEGER, REAL, TEXT), not raw CBOR encoding, because CBOR byte order does not match numeric order for range comparisons.

### 9.3 Hybrid Model

Hot path (type index, reverse index) in memory for fast lookups. Field indexes in SQL for persistence and range queries. Content store as source of truth, indexes as derived data.

### 9.4 Index Build Strategies

- **Eager (on startup):** Scan tree, build all indexes before accepting queries.
- **Lazy (on first query):** Build on demand. Fast startup, slow first query.
- **Background:** Accept queries immediately, build in background. Eventually consistent until complete.
- **Snapshot-based:** Walk Merkle trie for ordered enumeration. Checkpointable.

---

## 10. Conformance

### 10.1 Level 1 Conformance

An implementation claiming Level 1 conformance MUST:
- Maintain a type index covering all tree-bound entities
- Maintain a reverse hash index covering all tree-bound entities
- Register a query handler at `system/query` with `find` and `count` operations
- Support `type_filter`, `ref_filter`, and `path_prefix` in query expressions
- Support `eq`, `not_eq`, `in`, and `exists` operators
- Apply capability filtering to all query results
- Support cursor-based pagination
- Return capability-filtered `total` counts

An implementation claiming Level 1 conformance SHOULD:
- Maintain a path link index
- Support `path_filter` in query expressions
- Support content store scope via `system/query/allowances`

### 10.2 Level 2 Conformance

An implementation claiming Level 2 conformance MUST meet all Level 1 requirements and additionally:
- Support field indexes for configured (type, field) pairs
- Support `field_filters` in query expressions
- Support `gt`, `lt`, `gte`, `lte` operators on numeric fields
- Support `prefix` operator on string fields
- Fall back to type-scan for field queries on un-indexed fields

An implementation claiming Level 2 conformance SHOULD:
- Support `substring` operator on string fields
- Support `contains` operator on array fields
- Support `order_by` and `descending` in query expressions
- Support `include_entities` in query expressions

---

## 11. Design Notes

### 11.1 Content Store Projections

The tree is one projection of the content store — organized by path hierarchy. The query indexes are additional projections: by type, by reference, by field value. Tags, arbitrary entity sets, and other grouping mechanisms are further possible projections.

Each projection imposes different structure on the same underlying content:

| Projection | Structure | Key |
|-----------|-----------|-----|
| Tree | Hierarchical paths | Path string |
| Type index | Flat, partitioned | Type name |
| Reverse hash index | Graph, by reference | Content hash |
| Path link index | Graph, by path reference | Path string |
| Field index | Sorted, by value | (Type, field, value) |
| Tag set (future) | Flat, many-to-many | Tag name |
| Stored query (future) | Dynamic, criteria-based | Query expression |

The query extension provides indexes 2-5. Tags and dynamic sets are implementable as conventions over existing primitives (tree path conventions + query indexes). A more general "content store projection" concept — where projections are first-class, transferable, and composable — is a potential future exploration area.

The connection to Entity Component Systems (ECS) is structural: ECS archetypes are type index partitions, ECS systems are query-driven handlers, ECS component queries are conjunctive field filters. The query extension provides the infrastructure that makes ECS-style patterns expressible in the entity system.

### 11.2 Reference Strategy Guidance

Domain handlers designing entity relationships should choose reference types based on the relationship semantics:

**Hash reference (`system/hash`)** — for snapshot/contractual relationships. The referent is fixed at creation time. Use for: capabilities → grants, versions → snapshots, signatures → targets, audit records.

**Path reference (`system/tree/path`)** — for live/evolving relationships. The referent tracks current state. Use for: handler → interface, type resolution, navigational links. No cascade needed.

**Dual reference (hash + path together)** — for cross-peer and verified links. Path gives navigation, hash gives verification. Use for: cross-peer document links, bookmarks with freshness detection.

The reverse hash index enables cascade for hash references (find referrers, rebuild with new hash). The path link index enables backlink discovery for path references. Both are query extension infrastructure that makes reference management practical at scale.

---

*EXTENSION-QUERY.md — Draft specification for the entity system query extension. v1.6: split index maintenance across content-store events (bulk work — type/reverse/field/zone) and tree-change events (path annotation + path link index); §3.4 rebuild walks both content store and tree; §2.1 type index now has two normative entry forms (tree-bound, content-side). v1.5: added §2.5 path-prefix zone summaries (Tier A, SHOULD at Level 2) and §2.4.1 adaptive index promotion (Tier C, informative).*
