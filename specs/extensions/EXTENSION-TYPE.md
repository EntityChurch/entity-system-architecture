# Type Extension — Normative Specification

**Version**: 1.2
**v1.2 naming normalization:** the 7 `system/type/constraint/*` type-path leaves are renamed snake → kebab per `STYLE-NAMING-CONVENTIONS.md` §3.2; **parameter keys stay snake** — the path is now `system/type/constraint/min-length`, the field key remains `min_length`. This is **breaking** for these extension type-entity content-hashes; impls land in lockstep at this fold.
**Status**: Active
**Conformance grade:** Draft (per `GUIDE-EXTENSION-DEVELOPMENT.md` §9). No cross-impl validation pass yet. Reference impls pending.

**Depends on:**
- `ENTITY-CORE-PROTOCOL.md` (v7.18+) — handler dispatch (§6.5), handler authority (§6.8), capability model (§5).
- `ENTITY-NATIVE-TYPE-SYSTEM.md` (v4.0+) — core type system (structural validation, type resolution, field-spec structure). This extension adds constraint semantics to the open-type `constraints` field.
- `ENTITY-CBOR-ENCODING.md` (v1.3+) — canonical ECF for `one_of` / `not_one_of` byte-equality (§5.5 normative).

**Used by (informative):**
- Any extension or application that attaches value-level constraints to type definitions registered at `system/type/{name}`. No current standard-extension consumer; cross-extension usage is via the open `constraints` field on `system/type/field-spec` — peers without this extension preserve it without interpretation.

**Owned namespaces (per GUIDE-EXTENSION-DEVELOPMENT §4.3 — closed):**
- `system/type/constraint/` — built-in constraint entity types and the standard constraint handler dispatch pattern.
- The type handler at `system/type` itself; type definitions at `system/type/{name}` are the storage convention seeded by `ENTITY-NATIVE-TYPE-SYSTEM.md` (this extension does not own that prefix — it composes with it).

**Owned handler ops:**
- `system/type:validate` (§2.3, §8.3–§8.5).
- `system/type:compare` (§7.2).
- `system/type:converge` (§7.4).
- `system/type:compatible` (§7.3).
- `system/type:adopt` (§7.5).
- `system/type:reconcile` (§7.6).
- `system/type/constraint/*:validate` (§5.1) — dispatched per-constraint by `system/type:validate`.

**Owned entity types:**
- All 11 standard constraint types at `system/type/constraint/{kind}` (§4) and their request/result types.
- `system/type/violation`, `system/type/field-comparison`, `system/type/field-incompatibility`, `system/type/compatibility-report`, `system/type/reconcile-result` (§8).

**Extension points exposed:**
- The constraint handler dispatch surface itself — any handler matching a constraint type path with a `validate` op satisfying §5.2 / §5.3 may serve as a constraint handler (§2.2).
- The named-format vocabulary in the standard `format` constraint (§4.5) — implementations MAY recognize additional format names beyond the well-known set; unknown names fail closed per §1.2.

**Extension points consumed:** None.

---

> **Path notation.** Paths in this document use peer-relative notation (without leading `/{peer_id}/`). All peer-relative paths resolve to the local peer's namespace: `system/tree` means `/{local_peer_id}/system/tree`. Every path in the entity tree is absolute at rest — rooted at a peer identity. See ENTITY-CORE-PROTOCOL.md §1.4 for the path model. Cross-peer examples use absolute paths with explicit peer identities.

## 1. Overview

The type extension adds value-level constraint validation and type analysis operations to the entity type system. The core type system (ENTITY-NATIVE-TYPE-SYSTEM.md) provides structural validation — field presence, field types, union matching, generic resolution. This extension adds:

- **Constraint validation** — value predicates dispatched to constraint handlers
- **Standard constraint types** — 11 constraint types covering range, pattern, enumeration, length, and typed references
- **Constraint narrowing** — inheritance rules guaranteeing Liskov substitution
- **Type analysis operations** — compare, converge, compatible, adopt, reconcile

### 1.1 Scope

This specification covers:
- Constraint dispatch model: how constraints are evaluated via handler calls
- Standard constraint handler: the built-in fixed evaluator for 11 constraint types
- Standard constraint type definitions and their semantics
- Constraint attachment: where constraints live on type definitions
- Constraint narrowing rules for the `extends` mechanism
- Type analysis operations: structural diff, intersection, compatibility, adoption, reconciliation

Out of scope:
- Structural validation algorithms (ENTITY-NATIVE-TYPE-SYSTEM.md §7)
- Generic type resolution (ENTITY-NATIVE-TYPE-SYSTEM.md §6.6)
- Schema generation (ENTITY-NATIVE-TYPE-SYSTEM.md §13)
- Compute-backed custom constraints **are admissible by construction** under §2.2 — any handler matching a constraint type path with a `validate` op satisfying §5.2 / §5.3 may serve as a constraint handler, and `EXTENSION-COMPUTE.md` (v3.19+) provides one such handler shape. The convergence semantics for compute as a constraint evaluator — specifically, the boundedness / determinism guarantees required to upgrade compute-backed constraints from Class 4 (handler opaque) to Class 1 or 2 (deterministic) — are deferred to a follow-on TYPE amendment. Until then, compute-backed constraints are treated as Class 4 by the type system: the type handler SHOULD apply budget / timeout limits per §2.4 and treat the result as authoritative within those bounds.

### 1.2 Design Principles

**Constraints are handler calls.** The type system's validate operation is structurally aware of dynamic constraint dispatch. After structural validation (built-in, core), the type handler encounters constraints on field-specs and dispatches to constraint handlers. The dispatch is uniform — standard or custom, the interface is the same.

**The standard constraint handler is a fixed evaluator.** It covers ~11 constraint types with built-in logic. It is deterministic, bounded, and always terminates (Class 1 convergence). It comes with this extension.

**Custom constraints extend the same interface.** Any handler that implements the `validate` operation with the standard request/response types can serve as a constraint handler. The type system doesn't distinguish standard from custom — it's always a handler call.

**"Pass" means validated; everything else is informational.** The validation result is a report, not a verdict. `valid: true` means the entity passed all evaluated checks — structural and constraint. `valid: false` means one or more checks did not pass. Each violation carries a `kind` that tells the caller what happened:

- `structural` — the entity's shape doesn't match the type definition
- `constraint` — a value failed a known constraint check
- `unknown_constraint` — the constraint type or format name was not recognized; the validator could not evaluate it

The validator never reports `valid: true` when it encountered constraints it couldn't evaluate. That would be a false positive. Instead, it reports `valid: false` with `kind: "unknown_constraint"` — honest about what it could and couldn't check.

**What the caller does with this is policy.** A strict caller rejects anything that isn't `valid: true`. A lenient caller may accept entities with only `unknown_constraint` violations — "structurally sound, known constraints pass, some constraints I can't evaluate." A migration tool may accept `structural` violations for forward-compatible fields. The validation system provides the information; the caller makes the decision.

**Constraint awareness is opt-in.** Peers without the type extension do not evaluate constraints — they operate at a lower conformance level where constraints are preserved but not interpreted. This is not a failure; it's Level 2 conformance. The type extension adds constraint evaluation at Level 2+. This is consistent with the capability system's "unknown caveats = reject" principle (ENTITY-CORE-PROTOCOL.md §5.6) — the system is honest about what it checked, and the policy layer decides what to accept.

### 1.3 Relationship to Other Specifications

| Document | Relationship |
|----------|-------------|
| ENTITY-NATIVE-TYPE-SYSTEM.md | Core type system — structural validation, type resolution, field-spec structure. This extension adds constraint semantics to the `constraints` open-type field. |
| ENTITY-CORE-PROTOCOL.md | Protocol — handler dispatch (§6.5), handler authority (§6.8), capability model (§5). Constraint dispatch uses the standard handler dispatch mechanism. |
| EXTENSION-COMPUTE.md | Compute expressions as constraint handlers (future). The dispatch interface supports this without changes. |
| EXTENSION-TREE.md | Tree handler — type definitions and constraint entities are tree data accessed via get/put. |

### 1.4 The Type Graph

**The type system is a graph of type definitions.** Each type definition is an entity (`system/type` per ENTITY-NATIVE-TYPE-SYSTEM.md §4.1). Each definition's field-specs carry `type_ref` references to other types; each `extends` declares a parent type; `array_of` / `map_of` carry element-type references; (with this extension) each `constraints` entry carries a constraint-type reference. Together the type entities form a directed graph whose nodes are type definitions and whose edges are the references between them.

**The graph is the coherency surface.** Structural validation walks the graph. Compatibility (§7.3) and reconciliation (§7.6) compare graph shapes. The `extends` chain is a path through the graph that the constraint-narrowing verification (§6) walks. Two types are equivalent when their graph projections are equivalent.

**Paths are a storage convention.** A type definition is bound at a tree path — by default, at `/{peer_id}/system/type/{name}` per the NATIVE-TYPE-SYSTEM convention — so it can be looked up by name. The path is **not** the type's identity (the type's content hash is); the path is **not** the only place a type may live (deployments can use alternate paths; the type system is path-agnostic at the graph level); the path is **not** the primary mechanism for understanding the type system (the graph is).

**The type name is the interop contract** (ENTITY-NATIVE-TYPE-SYSTEM.md §4.7). Two peers exchanging an entity with `type: "app/user"` agree on the string `"app/user"`. Where each peer stores the type definition is a local concern; what matters is that each peer can resolve the name to the right type entity, by whatever resolution strategy the peer uses (§1.5).

**Constraint dispatch.** This extension dispatches to constraint handlers based on the constraint type path (e.g., `system/type/constraint/min`); the handler pattern is a dispatch path, not a tree lookup. Standard constraint type *definitions* live at `system/type/system/type/constraint/*` by the default storage convention; nothing in the dispatch model requires them to be stored at any particular path.

See `GUIDE-EXTENSION-DEVELOPMENT.md` §3.5 for the general "paths are convention; the entity graph is coherence" pattern this section exemplifies.

### 1.5 Type Resolution Strategies

A type system implementation chooses one or more strategies for resolving a name to a type definition. The strategies below are normatively-described; specific implementations may add others as long as the resolution is deterministic and the graph integrity invariants below hold.

**Strategy 1: Path-convention lookup (default).** Lookup `lookup("system/type/" + name)`. Simple, fast, the recommended default. The convention's hardcoded prefix is a property of *this strategy*, not of the type system. Implementations adopting this as the sole strategy MUST surface the limitation to deployments that want to store types at alternate paths (use an alternate strategy below, or a follow-on amendment).

**Strategy 1 is the only strategy a conforming implementation of EXTENSION-TYPE v1.1 MUST support today.** Strategies 2–4 are normatively described — i.e., when an implementation chooses to support one, it MUST conform to the strategy's contract — but no conforming impl is *required* to ship Strategies 2–4. As `EXTENSION-QUERY.md` matures into broad cross-impl support, **Strategy 3 SHOULD become broadly supported.**

**Strategy 2: Configurable base path.** Lookup `lookup(base_path + "/" + name)` where `base_path` is configurable, defaulting to `"system/type"`. Handles the "all my types share a non-standard prefix" deployment case. Mixed-prefix `extends` chains are not handled — each chain link uses the same base path. Cross-base-path chains require Strategy 3 or 4.

**Strategy 3: Type-class iteration + filter (via EXTENSION-QUERY).** Use `system/query` to find entities of type `system/type` matching a predicate (typically `data.name == requested_name`). Supports types at arbitrary paths; supports finding all types matching a class predicate; works regardless of where the type is bound in the tree. Cost: one query per resolution unless cached.

**Strategy 4: Reverse-hash-index walking.** When the type's content hash is known, resolve directly via the content store. Useful for `type_ref: hash` references (where the hash is in-spec); also useful for chained resolution after a Strategy-1-or-2 first hop.

**Graph integrity invariants — independent of strategy.**

1. **Name → definition** resolution within a single peer MUST be deterministic. Two `extends` references to the same name MUST resolve to the same type definition entity.
2. **Definition → name** is not necessarily 1:1: a type entity can be referenced by multiple names (publishing the same type under multiple paths is permitted; the names are aliases of the same graph node).
3. **Cycle detection** in the `extends` chain MUST be implemented at the graph level (not the path level). A cycle in the `extends` graph fails type resolution closed.
4. **Graph reachability** is the soundness property: a type T is sound iff every type reachable from T via `type_ref` / `extends` / `array_of` / `map_of` / `constraints` resolves successfully under the chosen strategy.
5. **Cross-peer determinism is not an invariant.** Peer A and peer B MAY resolve the same type name to different definitions — different versions of `app/user`, definitions stored at different paths, definitions adopted from different sources. The type system makes these divergences **observable** via the type analysis operations: `compare` (§7.2) surfaces structural differences; `compatible` (§7.3) reports directional compatibility; `reconcile` (§7.6) merges with an explicit strategy. Cross-peer convergence is a deployment outcome of running these ops, not a protocol invariant. Reverse-hash references (`type_ref: hash`) bypass the name layer and resolve identically across peers when the hash is in the content store; name-based references (`type_ref: name`) are subject to per-peer resolution.

### 1.6 Terminology

| Term | Definition |
|------|------------|
| Constraint | A value predicate attached to a field-spec. An entity `{type, data}` where the type determines the handler and the data provides parameters. |
| Constraint handler | A handler that implements the `validate` operation for constraint types. Receives a value and constraint data, returns valid/invalid. |
| Standard constraint | One of the 11 constraint types provided by this extension, evaluated by the standard constraint handler. |
| Custom constraint | A constraint type evaluated by a user-registered handler, not the standard constraint handler. |
| Narrowing | The requirement that a child type's constraints are equal to or more restrictive than its parent's. |

---

## 2. Constraint Dispatch Model

### 2.1 Three Layers of Type Validation

```
Layer 1: Structural validation (core, built-in, no dispatch)
  Field presence, field types, union matching, generic resolution.
  Every peer gets this (conformance Level 2+).

Layer 2: Standard constraint validation (this extension, handler dispatch)
  The standard constraint handler evaluates ~11 constraint types.
  Dispatched like any handler — same dispatch chain, same capability model.
  Deterministic, bounded, always terminates.

Layer 3: Custom constraint validation (user-registered handlers)
  Same dispatch interface as Layer 2.
  Handler behavior is opaque to the type system (Class 4).
  Budget and timeout protection apply.
```

### 2.2 Constraint Dispatch

When the type handler's `validate` operation encounters constraints on a field-spec, it dispatches to the appropriate constraint handler based on the constraint entity's `type` field:

```
constraint_dispatch(value, constraint, ctx):
  ; The constraint type path determines the handler via standard dispatch
  ; e.g., "system/type/constraint/min" → handler at system/type/constraint/*
  ; e.g., "app/constraint/email"       → handler at app/constraint/* (if registered)
  result = ctx.dispatch_execute(
    constraint.type,             ; dispatch path
    "validate",                  ; operation
    {
      value:           value,
      constraint_type: constraint.type,
      constraint_data: constraint.data
    }
  )

  if result is error:
    ; Dispatch failed — no matching handler, handler error, timeout
    return {valid: false, reason: "constraint_dispatch_failed: " + result.message}

  return result  ; {valid: bool, reason?: string}
```

The constraint type path (e.g., `system/type/constraint/min`) is used as the dispatch target. The handler system resolves this by pattern matching against registered handlers — `system/type/constraint/*` matches. This is the same dispatch mechanism used for all handler dispatch (ENTITY-CORE-PROTOCOL.md §6.5). No special constraint infrastructure.

### 2.3 Full Validation Flow

```
type_validate(ctx, params):
  entity = params.entity
  type_def = resolve_type(params.type_path or entity.type)

  ; Phase 1: Structural validation (core, built-in, no dispatch)
  errors = validate_structural(entity, type_def)
  if errors: return {valid: false, violations: errors}   ; kind: "structural"

  ; Phase 2: Constraint validation (handler dispatch per constraint)
  for field_name, field_spec in type_def.effective_fields():
    if field_spec.constraints is absent: continue
    value = entity.data[field_name]
    if value is absent: continue  ; absent optional fields skip constraints

    for constraint in field_spec.constraints:
      result = constraint_dispatch(value, constraint, ctx)
      if not result.valid:
        errors.add({
          field:      field_name,
          kind:       classify(result),         ; see "Violation kind mapping" below
          constraint: constraint.type,
          reason:     result.reason
        })

  return {valid: errors.empty(), violations: errors}
```

**Absent vs null:** When a field is absent (key not present), constraints are skipped — the field doesn't exist. When a field is present with a null value, constraints apply to null. This is consistent with ENTITY-NATIVE-TYPE-SYSTEM.md §2.4.

**Early termination:** Implementations MAY stop after the first constraint failure per field (fail-fast) or collect all failures (comprehensive). The spec does not prescribe a strategy.

**Violation kind mapping (normative).** Every violation emitted by `type_validate` carries a `kind` field per §8.5. The mapping rule:

- **`kind: "structural"`** — emitted by Phase 1 structural validation when an entity's shape does not match the type definition (missing required field, wrong type, etc.).
- **`kind: "constraint"`** — emitted by Phase 2 when constraint dispatch **runs successfully** and the handler returns `valid: false`. The check ran; the value did not pass.
- **`kind: "unknown_constraint"`** — emitted by Phase 2 when constraint dispatch **could not evaluate** the check. This covers three substantively-equivalent cases: (a) no handler matches the constraint type path; (b) the dispatch returns a non-success status (timeout, error, capability denial, transport failure); (c) the standard constraint handler reaches its `default` arm because the constraint type is not in its known set, or the format name in a `format` constraint is not recognized.

The classifier mechanism — string matching on `result.reason` for the standard handler's `"unknown constraint type"` / `"unknown format"` markers, status-code inspection on the dispatch envelope, structured error payload — is **implementation-defined**; only the mapping outcome is normative. Implementations MAY choose any classifier mechanism that correctly distinguishes "the validator could not evaluate this check" (→ `unknown_constraint`) from "the check ran and the value did not pass" (→ `constraint`).

This mapping is what makes the `valid: true` posture honest (§1.2): a validator returning `valid: true` has run every check it knows how to run; any check it could not run is reported as `unknown_constraint`, never silently elided as a pass. This mapping was pinned after EXTENSION-TYPE v1.1 cross-impl Phase 1 surfaced Rust's first-round mis-classification (silent-pass as `kind: "constraint"`); Go and Python had implemented the mapping correctly from the start.

### 2.4 Budget and Timeout

Standard constraint dispatch (to the built-in handler) is bounded — the standard constraint handler always terminates in O(input size). No budget concern.

Custom constraint dispatch (to user-registered handlers) is Class 4 from the type system's perspective. The type handler SHOULD apply a timeout or budget limit to custom constraint dispatch. The handler dispatch mechanism (ENTITY-CORE-PROTOCOL.md §6.8) provides bounds via TTL and budget on the sub-request.

Implementations MAY configure different timeout thresholds for standard vs custom constraint dispatch. The recommended default for custom constraint dispatch is the same as any handler sub-request — inherit from the caller's bounds context.

---

## 3. Constraint Attachment

### 3.1 Constraints on Field-Specs

Constraints attach to individual fields via the `constraints` field on `system/type/field-spec`. This is an open-type extension field (ENTITY-NATIVE-TYPE-SYSTEM.md §2.4) — peers without the type extension preserve it without interpretation.

```cbor
; Example: user type with constraints
{
  "type": "system/type",
  "data": {
    "name": "app/user",
    "fields": {
      "name": {
        "type_ref": "primitive/string",
        "constraints": [
          {"type": "system/type/constraint/min-length", "data": {"min_length": 1}},
          {"type": "system/type/constraint/max-length", "data": {"max_length": 255}}
        ]
      },
      "age": {
        "type_ref": "primitive/uint",
        "constraints": [
          {"type": "system/type/constraint/min", "data": {"min": 0}},
          {"type": "system/type/constraint/max", "data": {"max": 200}}
        ]
      },
      "email": {
        "type_ref": "primitive/string",
        "constraints": [
          {"type": "system/type/constraint/pattern",
           "data": {"pattern": "^[^@]+@[^@]+\\.[^@]+$"}}
        ]
      }
    }
  }
}
```

### 3.2 Constraint Structure

Each constraint is an entity `{type, data}`:

- **type**: The constraint type path. Determines which handler evaluates the constraint. Standard constraints use `system/type/constraint/{kind}`. Custom constraints use any path with a registered handler.
- **data**: Constraint parameters. Structure depends on the constraint type. Each standard constraint type definition (§4) specifies its data fields.

Constraints on a field are conjunctive — ALL constraints must pass for the field to be valid.

### 3.3 Field-Spec Extension

The `constraints` field on `system/type/field-spec`:

```
constraints: {array_of: {type_ref: "core/entity"}, optional: true}
             ; Array of constraint entities, each {type, data, content_hash}
             ; Evaluated conjunctively — all must pass
             ; Absent = no constraints (unconstrained)
```

Peers at conformance Level 0-1 preserve `constraints` as an unknown open-type field. Peers at Level 2+ with this extension evaluate constraints during validation.

---

## 4. Standard Constraint Types

The standard constraint handler evaluates these 11 constraint types. Each is an individual entity type at `system/type/constraint/{kind}`.

### 4.1 Numeric Bounds

#### system/type/constraint/min

Minimum numeric value (inclusive).

```cbor
{
  "type": "system/type",
  "data": {
    "name": "system/type/constraint/min",
    "fields": {
      "min": {"type_ref": "primitive/float"}
    }
  }
}
```

**Semantics:** `value >= constraint.data.min`. Applies to `primitive/uint`, `primitive/int`, `primitive/float`. NaN comparisons return false (NaN is not >= anything).

#### system/type/constraint/max

Maximum numeric value (inclusive).

```cbor
{
  "type": "system/type",
  "data": {
    "name": "system/type/constraint/max",
    "fields": {
      "max": {"type_ref": "primitive/float"}
    }
  }
}
```

**Semantics:** `value <= constraint.data.max`. Same type applicability and NaN handling as `min`.

### 4.2 Length and Count Bounds

#### system/type/constraint/min-length

Minimum length for strings (codepoints) or bytes (byte count).

```cbor
{
  "type": "system/type",
  "data": {
    "name": "system/type/constraint/min-length",
    "fields": {
      "min_length": {"type_ref": "primitive/uint"}
    }
  }
}
```

**Semantics:** For `primitive/string`: `codepoint_count(value) >= min_length`. For `primitive/bytes`: `byte_length(value) >= min_length`.

#### system/type/constraint/max-length

Maximum length for strings (codepoints) or bytes (byte count).

```cbor
{
  "type": "system/type",
  "data": {
    "name": "system/type/constraint/max-length",
    "fields": {
      "max_length": {"type_ref": "primitive/uint"}
    }
  }
}
```

**Semantics:** For `primitive/string`: `codepoint_count(value) <= max_length`. For `primitive/bytes`: `byte_length(value) <= max_length`.

#### system/type/constraint/min-count

Minimum element count for arrays or map entries.

```cbor
{
  "type": "system/type",
  "data": {
    "name": "system/type/constraint/min-count",
    "fields": {
      "min_count": {"type_ref": "primitive/uint"}
    }
  }
}
```

**Semantics:** For `array_of` fields: `array_length(value) >= min_count`. For `map_of` fields: `map_size(value) >= min_count`.

#### system/type/constraint/max-count

Maximum element count for arrays or map entries.

```cbor
{
  "type": "system/type",
  "data": {
    "name": "system/type/constraint/max-count",
    "fields": {
      "max_count": {"type_ref": "primitive/uint"}
    }
  }
}
```

**Semantics:** For `array_of` fields: `array_length(value) <= max_count`. For `map_of` fields: `map_size(value) <= max_count`.

### 4.3 Pattern Matching

#### system/type/constraint/pattern

Regular expression match on string values.

```cbor
{
  "type": "system/type",
  "data": {
    "name": "system/type/constraint/pattern",
    "fields": {
      "pattern": {"type_ref": "primitive/string"}
    }
  }
}
```

**Semantics:** RE2 syntax. Full-match semantics — the entire string must match (as if anchored with `^...$`). Applies to `primitive/string` and types that extend `primitive/string` (including `system/tree/path`, `system/type/name`, `system/peer-id`).

RE2 guarantees linear-time evaluation. Implementations MUST use RE2 or an equivalent linear-time engine. PCRE and other backtracking engines are NOT conformant.

### 4.4 Enumeration

#### system/type/constraint/one-of

Value must be one of the listed values.

```cbor
{
  "type": "system/type",
  "data": {
    "name": "system/type/constraint/one-of",
    "fields": {
      "values": {"array_of": {"type_ref": "primitive/any"}}
    }
  }
}
```

**Semantics:** `value` must be CBOR-byte-equal to at least one element in `values`. Comparison uses canonical CBOR encoding (ECF) — two values are equal iff their ECF byte representations are identical.

#### system/type/constraint/not-one-of

Value must NOT be one of the listed values.

```cbor
{
  "type": "system/type",
  "data": {
    "name": "system/type/constraint/not-one-of",
    "fields": {
      "values": {"array_of": {"type_ref": "primitive/any"}}
    }
  }
}
```

**Semantics:** `value` must NOT be CBOR-byte-equal to any element in `values`.

### 4.5 Format Validation

#### system/type/constraint/format

Named format validation on string values.

```cbor
{
  "type": "system/type",
  "data": {
    "name": "system/type/constraint/format",
    "fields": {
      "format": {"type_ref": "primitive/string"}
    }
  }
}
```

**Semantics:** The `format` field names a validation format. The format vocabulary is open — implementations define which format names they recognize. This constraint type is an extension point within the standard constraint handler: adding a new format requires no new constraint type or handler registration, just recognition of a new format name string.

Implementations **MUST** recognize the following well-known format names with the semantics referenced in the spec column (v1.1 strengthens SHOULD → MUST):

| Format | Validation | Reference |
|--------|-----------|-----------|
| `"uri"` | Valid URI | RFC 3986 |
| `"date-time"` | ISO 8601 date-time | RFC 3339 |
| `"date"` | ISO 8601 date | RFC 3339 §5.6 full-date |
| `"uuid"` | UUID | RFC 4122 |
| `"base58"` | Valid Base58 encoding | `ENTITY-CORE-PROTOCOL.md` §7 |
| `"re2"` | Valid RE2 regex pattern | |

Implementations **MAY** recognize additional format names (e.g., `"email"`, `"ipv4"`, `"phone"`).

**Unknown format names fail closed** — consistent with the unknown constraint type principle (§1.2). If the standard constraint handler does not recognize a format name, it MUST return `{valid: false, reason: "unknown format: {name}"}`. The caller receives a violation with `kind: "unknown_constraint"`. The handler is honest: it cannot validate what it doesn't understand. Passing silently would create a false sense of validation.

This means implementations SHOULD support the well-known formats listed above before deploying type definitions that use them. A type definition using `format: "email"` on a peer whose standard constraint handler doesn't recognize `"email"` will fail validation — the same behavior as using a custom constraint type with no registered handler.

### 4.6 Typed References

#### system/type/constraint/type-pattern

Validate that a referenced entity (by hash or path) has a type matching a glob pattern.

```cbor
{
  "type": "system/type",
  "data": {
    "name": "system/type/constraint/type-pattern",
    "fields": {
      "pattern": {"type_ref": "primitive/string"}
    }
  }
}
```

**Semantics:** Applies to fields of type `system/hash` or `system/tree/path`.

For `system/hash` fields: resolve the hash to an entity (content store or tree), check that the entity's `type` field matches the glob pattern.

For `system/tree/path` fields: resolve the path to an entity (tree lookup), check that the entity's `type` field matches the glob pattern. Path validation is temporal — the entity at the path can change. Validated at write time, not continuously.

**Glob matching:** `*` matches any single path segment. `**` matches zero or more segments. Pattern `system/capability/*` matches `system/capability/grant-entry` but not `system/capability/path-scope/foo`.

**Resolution failure:** If the referenced entity cannot be resolved (hash not in content store, path not bound), validation SHOULD pass with a warning. The `type_pattern` constraint validates the type of reachable entities, not their existence. Existence checking is a separate concern (referential integrity Level 3, implementation-defined).

**Content store access:** Resolving a hash for `type_pattern` validation may require content store access. The default model is tree-scoped — the hash must be resolvable through a tree-bound entity. The layered access model (tree-scoped default, explicit allowance for full content store access, content extension namespaces for granular access) parallels EXTENSION-QUERY.md §5.1 and the landed compute content-store-scoping model (EXTENSION-COMPUTE.md v3.7+, originally proposed as `proposals/implemented/PROPOSAL-COMPUTE-CONTENT-STORE-SCOPING.md` D3–D6). The same model applies to type validation: without an explicit content store access allowance, `type_pattern` can only validate entities reachable through tree paths covered by the caller's capability.

---

## 5. Standard Constraint Handler

### 5.1 Handler Manifest

```cbor
{
  "type": "system/handler",
  "data": {
    "pattern": "system/type/constraint/*",
    "name": "standard-constraints",
    "operations": {
      "validate": {
        "input_type": "system/type/constraint/validate-request",
        "output_type": "system/type/constraint/validate-result"
      }
    }
  }
}
```

The handler pattern `system/type/constraint/*` matches all standard constraint type paths (`system/type/constraint/min`, `system/type/constraint/pattern`, etc.).

**Registration path.** Per `GUIDE-EXTENSION-DEVELOPMENT.md` §4.9 and `ENTITY-CORE-PROTOCOL.md` §6.6, the manifest's `pattern` field is advertisement-only; the dispatcher does not interpret glob notation. The `system/handler` entity itself is registered at the **literal prefix** `system/type/constraint` (no trailing `/*`), and the manifest is discoverable at `system/handler/system/type/constraint`. The handler then catches every standard constraint dispatch (`system/type/constraint/min`, `…/max`, etc.) via the ENTITY-CORE-PROTOCOL.md §6.6 longest-prefix walk. This is the universal "catch the subtree" convention — same shape as `system/clock`, `system/query`, `system/tree`, `system/content`, etc.

### 5.2 Validate Request Type

```cbor
{
  "type": "system/type",
  "data": {
    "name": "system/type/constraint/validate-request",
    "fields": {
      "value":           {"type_ref": "primitive/any"},
      "constraint_type": {"type_ref": "system/type/name"},
      "constraint_data": {"type_ref": "primitive/any"}
    }
  }
}
```

| Field | Description |
|-------|-------------|
| `value` | The field value to validate |
| `constraint_type` | The constraint type path (e.g., `"system/type/constraint/min"`) |
| `constraint_data` | The constraint's data (e.g., `{"min": 0}`) |

### 5.3 Validate Result Type

```cbor
{
  "type": "system/type",
  "data": {
    "name": "system/type/constraint/validate-result",
    "fields": {
      "valid":  {"type_ref": "primitive/bool"},
      "reason": {"type_ref": "primitive/string", "optional": true}
    }
  }
}
```

| Field | Description |
|-------|-------------|
| `valid` | Whether the value satisfies the constraint |
| `reason` | Human-readable explanation when invalid (absent when valid) |

### 5.4 Standard Handler Algorithm

```
standard_constraint_validate(ctx, params):
  value = params.value
  constraint_type = params.constraint_type
  data = params.constraint_data

  match constraint_type:
    "system/type/constraint/min":
      if not is_numeric(value): return {valid: false, reason: "min: not numeric"}
      return {valid: value >= data.min, reason: "must be >= " + data.min}

    "system/type/constraint/max":
      if not is_numeric(value): return {valid: false, reason: "max: not numeric"}
      return {valid: value <= data.max, reason: "must be <= " + data.max}

    "system/type/constraint/min-length":
      len = measure_length(value)  ; codepoints for string, bytes for bytes
      return {valid: len >= data.min_length, reason: "length must be >= " + data.min_length}

    "system/type/constraint/max-length":
      len = measure_length(value)
      return {valid: len <= data.max_length, reason: "length must be <= " + data.max_length}

    "system/type/constraint/min-count":
      count = collection_size(value)
      return {valid: count >= data.min_count, reason: "count must be >= " + data.min_count}

    "system/type/constraint/max-count":
      count = collection_size(value)
      return {valid: count <= data.max_count, reason: "count must be <= " + data.max_count}

    "system/type/constraint/pattern":
      if not is_string(value): return {valid: false, reason: "pattern: not a string"}
      return {valid: re2_full_match(data.pattern, value),
              reason: "must match pattern: " + data.pattern}

    "system/type/constraint/one-of":
      return {valid: ecf_byte_equal_any(value, data.values),
              reason: "must be one of: " + data.values}

    "system/type/constraint/not-one-of":
      return {valid: not ecf_byte_equal_any(value, data.values),
              reason: "must not be one of: " + data.values}

    "system/type/constraint/format":
      return validate_format(value, data.format)

    "system/type/constraint/type-pattern":
      return validate_type_pattern(value, data.pattern, ctx)

    default:
      ; Standard handler does not recognize this constraint type
      return {valid: false, reason: "unknown constraint type: " + constraint_type}
```

### 5.5 Algorithm Classification

Per ENTITY-CORE-PROTOCOL.md §7, algorithms in this extension are classified:

| Algorithm | Classification | Requirement |
|-----------|---------------|-------------|
| Standard constraint evaluation (§5.4) | **Conformance** | Implementations MUST produce the same `valid: true/false` result for identical `(value, constraint_type, constraint_data)` inputs. Internal evaluation logic may vary. |
| Constraint dispatch (§2.2) | **Reference** | Shows the expected dispatch flow. Implementations MAY use batch dispatch, caching, or pre-built dispatch tables. The result MUST be equivalent. |
| Full validation flow (§2.3) | **Reference** | Shows the phased validation approach. Implementations MAY reorder, parallelize, or short-circuit. The validate-result MUST contain the same violations. |
| Narrowing verification (§6.4) | **Conformance** | Implementations MUST accept/reject the same `extends` relationships. The narrowing rules (§6.2) define the outcome; the verification algorithm is illustrative. |
| Type analysis operations (§7) | **Reference** | Compare, compatible, converge, adopt, reconcile describe expected behavior. Implementations may use different algorithms as long as the results are structurally equivalent. |

**What this means for implementers:** The pseudocode in this spec shows expected behavior and serves as a reference implementation. Implementations are free to optimize (batch constraint dispatch, cache resolved types, parallelize validation) as long as they produce equivalent results. The critical requirement is that two implementations validating the same entity against the same type definition report the same violations.

**Exception — `one_of` and `not_one_of` equality:** These constraints use ECF byte equality (canonical CBOR encoding). This is a **normative** requirement — two implementations MUST agree on whether a value matches a `one_of` list. ECF deterministic encoding (ENTITY-CBOR-ENCODING.md) ensures this.

### 5.6 Convergence Properties

The standard constraint handler is a **fixed evaluator** — deterministic, bounded, always terminates:

- All 11 constraint evaluations are O(input size) or better
- No tree writes (read-only validation)
- No unbounded loops
- RE2 guarantees linear-time regex evaluation
- `type_pattern` resolution is bounded by tree depth

This makes standard constraints **Class 1** in the convergence classification — they always converge, require no coordination, and produce deterministic results.

Custom constraint handlers are **Class 4** from the type system's perspective — the handler is a black box. The type handler SHOULD apply budget limits to custom constraint dispatch via the sub-request bounds.

---

## 6. Constraint Narrowing

### 6.1 Principle

When a child type extends a parent type, the child's constraints MUST be equal to or more restrictive than the parent's. This guarantees Liskov substitution: any entity valid for the child type is also valid for the parent type.

### 6.2 Narrowing Rules

| Constraint | Child is narrower when |
|-----------|----------------------|
| `min` | child.min >= parent.min |
| `max` | child.max <= parent.max |
| `min_length` | child.min_length >= parent.min_length |
| `max_length` | child.max_length <= parent.max_length |
| `min_count` | child.min_count >= parent.min_count |
| `max_count` | child.max_count <= parent.max_count |
| `pattern` | Equal patterns (byte-identical pattern strings under ECF) narrow trivially. Non-equal patterns **default to incomparable** — implementations MUST NOT consider one pattern narrower than another unless the patterns are byte-equal. Regex subset relationships are undecidable in general; the spec does not pin a sub-pattern recognizer. Deployments that want richer narrowing layer it explicitly via a custom constraint type. (v1.1) |
| `one_of` | child.values ⊆ parent.values |
| `not_one_of` | child.values ⊇ parent.values |
| `format` | Equal format names narrow trivially. Sub-format relationships are **implementation-defined and default to "incomparable"** — implementations SHOULD NOT recognize a sub-format relationship unless the relationship is normatively pinned (no current normative sub-format pins exist). This is the conservative interop default; deployments that want richer narrowing layer it explicitly. (v1.1) |
| `type_pattern` | Child pattern is more specific (longer prefix or exact match) |

### 6.3 Absent Constraints

A parent field with no constraint is unconstrained — the child MAY add any constraint (adding a constraint always narrows). A child MUST NOT remove a constraint present on the parent — the effective constraint set only grows through the `extends` chain.

### 6.4 Narrowing Verification

Narrowing is verified when a type definition with `extends` is registered or validated:

```
verify_narrowing(child_def, parent_def):
  for field_name in parent_def.effective_fields():
    parent_constraints = parent_def.field_constraints(field_name)
    child_constraints = child_def.field_constraints(field_name)

    for parent_c in parent_constraints:
      ; Find matching constraint type in child
      child_c = find_constraint(child_constraints, parent_c.type)
      if child_c is null:
        ; Child removed parent constraint — violation
        return error("narrowing violation: child removed " + parent_c.type + " on " + field_name)
      if not is_narrower(child_c, parent_c):
        return error("narrowing violation: child widened " + parent_c.type + " on " + field_name)

  return ok
```

---

## 7. Type Analysis Operations

The type handler at `system/type` provides analysis operations beyond core's `validate`. These operations are read-only — they return results inline without writing to the tree.

### 7.1 Handler Manifest

```cbor
{
  "type": "system/handler",
  "data": {
    "pattern": "system/type",
    "name": "types",
    "operations": {
      "validate":   {"input_type": "system/type/validate-request",
                     "output_type": "system/type/validate-result"},
      "compare":    {"input_type": "system/type/compare-request",
                     "output_type": "system/type/compare-result"},
      "converge":   {"input_type": "system/type/converge-request",
                     "output_type": "system/type"},
      "compatible": {"input_type": "system/type/compatible-request",
                     "output_type": "system/type/compatibility-report"},
      "adopt":      {"input_type": "system/type/adopt-request",
                     "output_type": "system/type"},
      "reconcile":  {"input_type": "system/type/reconcile-request",
                     "output_type": "system/type/reconcile-result"}
    }
  }
}
```

### 7.2 compare

Structural diff of two type definitions.

**Request:** `system/type/compare-request`
```cbor
{
  "type": "system/type",
  "data": {
    "name": "system/type/compare-request",
    "fields": {
      "type_a": {"type_ref": "system/tree/path"},
      "type_b": {"type_ref": "system/tree/path"}
    }
  }
}
```

**Result:** `system/type/compare-result`
```cbor
{
  "type": "system/type",
  "data": {
    "name": "system/type/compare-result",
    "fields": {
      "type_a_path":    {"type_ref": "system/tree/path"},
      "type_b_path":    {"type_ref": "system/tree/path"},
      "shared":         {"map_of": {"type_ref": "system/type/field-comparison"}},
      "only_a":         {"array_of": {"type_ref": "primitive/string"}},
      "only_b":         {"array_of": {"type_ref": "primitive/string"}},
      "incompatible":   {"array_of": {"type_ref": "system/type/field-incompatibility"}, "optional": true}
    }
  }
}
```

**Algorithm:** Resolve both type definitions to their effective fields. Compare field-by-field: shared fields (same name, compatible types), fields only in A, fields only in B, incompatible fields (same name, different types). Constraints are compared separately — constraint differences do not affect structural compatibility (ENTITY-NATIVE-TYPE-SYSTEM.md §12.2).

### 7.3 compatible

Directional compatibility check — can entities of type A satisfy type B?

**Request:** `system/type/compatible-request`
```cbor
{
  "type": "system/type",
  "data": {
    "name": "system/type/compatible-request",
    "fields": {
      "type_a":    {"type_ref": "system/tree/path"},
      "type_b":    {"type_ref": "system/tree/path"},
      "direction": {"type_ref": "primitive/string",
                    "constraints": [
                      {"type": "system/type/constraint/one-of",
                       "data": {"values": ["forward", "backward", "bidirectional"]}}
                    ]}
    }
  }
}
```

**Result:** `system/type/compatibility-report`
```cbor
{
  "type": "system/type",
  "data": {
    "name": "system/type/compatibility-report",
    "fields": {
      "type_a_path":          {"type_ref": "system/tree/path"},
      "type_b_path":          {"type_ref": "system/tree/path"},
      "direction":            {"type_ref": "primitive/string"},
      "level":                {"type_ref": "primitive/string",
                               "constraints": [
                                 {"type": "system/type/constraint/one-of",
                                  "data": {"values": ["fully_compatible", "forward_only", "backward_only", "partially_compatible", "incompatible"]}}
                               ]},
      "shared_fields":        {"array_of": {"type_ref": "primitive/string"}},
      "incompatible_fields":  {"array_of": {"type_ref": "system/type/field-incompatibility"}, "optional": true},
      "missing_required_a":   {"array_of": {"type_ref": "primitive/string"}, "optional": true},
      "missing_required_b":   {"array_of": {"type_ref": "primitive/string"}, "optional": true}
    }
  }
}
```

### 7.4 converge

Compute the intersection type across multiple definitions — the minimal structure all definitions can handle.

**Request:** `system/type/converge-request`
```cbor
{
  "type": "system/type",
  "data": {
    "name": "system/type/converge-request",
    "fields": {
      "type_paths": {"array_of": {"type_ref": "system/tree/path"},
                     "constraints": [
                       {"type": "system/type/constraint/min-count", "data": {"min_count": 2}}
                     ]}
    }
  }
}
```

**Result:** A `system/type` entity representing the converged type. The caller may store it via tree put or use it for analysis only.

### 7.5 adopt

Install a remote peer's type definition locally. Reads the type from the remote peer's tree, rewrites the name and any `extends` references that point to the same peer, and returns the result for the caller to store.

**Request:** `system/type/adopt-request`
```cbor
{
  "type": "system/type",
  "data": {
    "name": "system/type/adopt-request",
    "fields": {
      "source_path": {"type_ref": "system/tree/path"},
      "local_name":  {"type_ref": "system/type/name", "optional": true}
    }
  }
}
```

| Field | Description |
|-------|-------------|
| `source_path` | Full tree path to the source type definition (e.g., `/{peer_id}/system/type/sensor/temperature`) |
| `local_name` | Local type name to use (e.g., `sensor/temperature`). If absent, derived from source_path by stripping peer prefix and `system/type/`. |

**What adopt does beyond tree `get`:**

1. **Name rewriting.** Sets `data.name` to `local_name`.
2. **Extends chain resolution.** If the type's `extends` field references a type from the same remote peer (e.g., `sensor/base` which resolves to `/{peer_id}/system/type/sensor/base`), the handler checks whether a local equivalent exists. If so, it updates the reference. If not, it flags the dependency — the caller needs to adopt the parent type first.
3. **Collision detection.** Checks whether `system/type/{local_name}` already exists locally. Returns a warning field if so (open-type extension on the result).

**Result:** A `system/type` entity with rewritten name and resolved references. The caller decides whether to `put` it into the local tree. The handler does NOT write to the tree directly.

### 7.6 reconcile

Merge diverged type definitions into a single definition using a strategy. This is the type-level equivalent of revision merge — it understands type structure (fields, constraints, optionality) rather than treating definitions as opaque entities.

**When reconcile is needed:** Two peers independently evolve the same logical type. PeerA adds `phone` to `app/contact`. PeerB adds `address`. A generic tree merge would conflict on the type definition entity (different hashes). Reconcile understands that both additions are compatible — the merged type has both fields.

**Request:** `system/type/reconcile-request`
```cbor
{
  "type": "system/type",
  "data": {
    "name": "system/type/reconcile-request",
    "fields": {
      "type_paths": {"array_of": {"type_ref": "system/tree/path"},
                     "constraints": [
                       {"type": "system/type/constraint/min-count", "data": {"min_count": 2}}
                     ]},
      "strategy":   {"type_ref": "primitive/string",
                     "constraints": [
                       {"type": "system/type/constraint/one-of",
                        "data": {"values": ["intersect", "union", "prefer"]}}
                     ]}
    }
  }
}
```

**Strategies:**

- **intersect** — Keep only fields present in ALL definitions with compatible types. Fields unique to one definition are dropped. Produces the minimal common structure. Most conservative.
- **union** — Keep ALL fields from all definitions. Fields not present in all definitions become optional. Incompatible fields (same name, different types) are reported and excluded. Most inclusive.
- **prefer** — First type path is the preferred definition. Other definitions supply additional fields (as optional). Incompatible fields use the preferred definition's version. Most opinionated.

**Constraint reconciliation:** When the same field has different constraints across definitions:
- `intersect`: use the most restrictive constraint from any definition
- `union`: use the least restrictive constraint from any definition
- `prefer`: use the preferred definition's constraints

**Result:** `system/type/reconcile-result`
```cbor
{
  "type": "system/type",
  "data": {
    "name": "system/type/reconcile-result",
    "fields": {
      "reconciled_type":      {"type_ref": "core/entity"},
      "strategy_used":        {"type_ref": "primitive/string"},
      "sources":              {"array_of": {"type_ref": "system/tree/path"}},
      "fields_dropped":       {"array_of": {"type_ref": "primitive/string"}, "optional": true},
      "fields_made_optional": {"array_of": {"type_ref": "primitive/string"}, "optional": true},
      "incompatibilities":    {"array_of": {"type_ref": "system/type/field-incompatibility"}, "optional": true}
    }
  }
}
```

| Field | Description |
|-------|-------------|
| `reconciled_type` | The merged `system/type` entity |
| `strategy_used` | Strategy that was applied |
| `sources` | Source type paths that were reconciled |
| `fields_dropped` | Fields excluded (intersect strategy) |
| `fields_made_optional` | Fields made optional because not in all sources (union strategy) |
| `incompatibilities` | Fields that could not be reconciled (different types across sources) |

The result is returned to the caller. The caller decides whether to `put` it into the tree.

---

## 8. Supporting Type Definitions

### 8.1 system/type/field-comparison

```cbor
{
  "type": "system/type",
  "data": {
    "name": "system/type/field-comparison",
    "fields": {
      "type_match":       {"type_ref": "primitive/bool"},
      "constraint_match": {"type_ref": "primitive/bool"},
      "a_optional":       {"type_ref": "primitive/bool"},
      "b_optional":       {"type_ref": "primitive/bool"},
      "detail":           {"type_ref": "primitive/string", "optional": true}
    }
  }
}
```

### 8.2 system/type/field-incompatibility

```cbor
{
  "type": "system/type",
  "data": {
    "name": "system/type/field-incompatibility",
    "fields": {
      "field_name": {"type_ref": "primitive/string"},
      "a_type":     {"type_ref": "system/type/name"},
      "b_type":     {"type_ref": "system/type/name"},
      "reason":     {"type_ref": "primitive/string"}
    }
  }
}
```

### 8.3 system/type/validate-request

```cbor
{
  "type": "system/type",
  "data": {
    "name": "system/type/validate-request",
    "fields": {
      "entity":    {"type_ref": "core/entity"},
      "type_path": {"type_ref": "system/type/name", "optional": true}
    }
  }
}
```

| Field | Description |
|-------|-------------|
| `entity` | The entity to validate |
| `type_path` | Type to validate against. If absent, uses `entity.type`. |

### 8.4 system/type/validate-result

```cbor
{
  "type": "system/type",
  "data": {
    "name": "system/type/validate-result",
    "fields": {
      "valid":                {"type_ref": "primitive/bool"},
      "violations":           {"array_of": {"type_ref": "system/type/violation"}, "optional": true},
      "unevaluated_fields":   {"array_of": {"type_ref": "primitive/string"}, "optional": true}
    }
  }
}
```

| Field | Description |
|-------|-------------|
| `valid` | Whether all evaluated checks passed. `true` = every check that ran succeeded. Does NOT mean every possible check was run — see `unevaluated_fields`. |
| `violations` | List of checks that did not pass (absent when `valid: true` and no `unknown_constraint` violations). |
| `unevaluated_fields` | Open-type extension fields on the type definition or field-specs that the validator detected but could not interpret. For example, a Level 2 peer sees `constraints` on a field-spec but has no type extension — it reports `"constraints"` here. A Level 2+ peer that encounters a future extension field (e.g., `"triggers"`) it doesn't understand reports it here. Absent when the validator interpreted all fields. |

This gives the caller the complete picture:
- `valid: true`, no `unevaluated_fields` — clean pass, everything checked
- `valid: true`, `unevaluated_fields: ["constraints"]` — structural pass, but constraints were not evaluated (no type extension). Caller decides if that's acceptable.
- `valid: false`, violations with `kind: "unknown_constraint"` — type extension ran but couldn't evaluate specific constraints. The constraints WERE detected and reported, just not validated.
- `valid: false`, violations with `kind: "structural"` — shape mismatch, unambiguous failure

### 8.5 system/type/violation

```cbor
{
  "type": "system/type",
  "data": {
    "name": "system/type/violation",
    "fields": {
      "field":      {"type_ref": "primitive/string"},
      "kind":       {"type_ref": "primitive/string",
                     "constraints": [
                       {"type": "system/type/constraint/one-of",
                        "data": {"values": ["structural", "constraint", "unknown_constraint"]}}
                     ]},
      "constraint": {"type_ref": "system/type/name", "optional": true},
      "reason":     {"type_ref": "primitive/string"}
    }
  }
}
```

| Field | Description |
|-------|-------------|
| `field` | Field name where the violation occurred |
| `kind` | Violation category: `"structural"` (field missing, wrong type), `"constraint"` (value failed a constraint check), `"unknown_constraint"` (constraint dispatch could not evaluate — fail closed). The mapping rule from constraint dispatch outcome to kind is normatively pinned in §2.3 "Violation kind mapping." |
| `constraint` | Constraint type path that failed (absent for structural violations) |
| `reason` | Human-readable explanation. Implementations **SHOULD** populate `reason` with a concise explanation that names the constraint and (where applicable) the constraint's parameters (e.g., `"length must be >= 1"`, `"unknown constraint type: app/constraint/foo"`). Empty `reason` strings are conformant per the type definition above but are unhelpful for debugging and observability; consumers reading violation reports rely on `reason` to understand what failed. |

The `kind` field tells the caller what happened, not what to do about it:
- `structural` — the entity's shape doesn't match the type definition. Peers at any conformance level agree on this.
- `constraint` — the entity's values don't satisfy a known constraint. The check ran and the value didn't pass.
- `unknown_constraint` — the validator could not evaluate the check (no handler matches the constraint type, dispatch returned non-success, or the standard handler reached its `default` arm for an unrecognized constraint type or unknown format name). The validator is reporting what it couldn't evaluate — the caller decides whether unevaluated constraints are acceptable for their use case. See §2.3 "Violation kind mapping" for the normative mapping rule and §1.2 for the "valid: true means validated" posture this enables.

---

## 9. Write Authorization

The type handler and standard constraint handler are read-only — all operations return results inline without writing to the tree.

| Handler | Operation | Write target | Authorization |
|---------|-----------|-------------|---------------|
| system/type | validate | No writes | — |
| system/type | compare | No writes | — |
| system/type | converge | No writes | — |
| system/type | compatible | No writes | — |
| system/type | adopt | No writes (returns entity for caller to put) | — |
| system/type | reconcile | No writes | — |
| system/type/constraint/* | validate | No writes | — |

Cross-peer operations (`compare`, `compatible`, `adopt`) read from remote peer type definitions. The type handler's grant must cover `*/system/type/*` with read access.

See ENTITY-CORE-PROTOCOL.md §6.8 for the general write authorization model.

**Tree-write-time validation (informative; v1.1).** Some deployments configure tree-write-time validation: on `system/tree:put`, the tree handler dispatches `system/type:validate` over the entity before committing. The cap surface for this pattern lives on `system/tree`, not on `system/type` — the writer needs the `put` grant on the tree path; the tree handler internally calls the type handler with its own dispatch cap. This is consistent with the ENTITY-CORE-PROTOCOL.md §6.8 caller-specified-paths rule (the tree handler invokes the type handler under its own authority for an internal validation pass). EXTENSION-TREE describes the wiring; this extension's `system/type:validate` op is the inline contract.

---

## 10. Capability Requirements

### 10.1 Caller Grant

Callers need a grant to invoke type operations:

```cbor
{
  "handlers": ["system/type"],
  "resources": {"include": ["system/type/*"]},
  "operations": {"include": ["validate", "compare", "converge", "compatible", "adopt", "reconcile"]}
}
```

Callers may be granted a subset of operations.

### 10.2 Type Handler Grant

The type handler needs grants for:

```cbor
; Read local type definitions
{
  "handlers": ["system/tree"],
  "resources": {"include": ["system/type/*"]},
  "operations": {"include": ["get", "list"]}
}

; Read remote peer type definitions (for cross-peer compare, adopt)
{
  "handlers": ["system/tree"],
  "resources": {"include": ["/*/system/type/*"]},
  "operations": {"include": ["get", "list"]}
}

; Dispatch to constraint handlers (for validate)
{
  "handlers": ["system/type/constraint/*"],
  "resources": {"include": ["system/type/constraint/*"]},
  "operations": {"include": ["validate"]}
}
```

---

## 11. Constants

### 11.1 Standard Constraint Types

| Type path | Applies to | Description |
|-----------|-----------|-------------|
| `system/type/constraint/min` | numeric | Minimum value (inclusive) |
| `system/type/constraint/max` | numeric | Maximum value (inclusive) |
| `system/type/constraint/min-length` | string, bytes | Minimum length |
| `system/type/constraint/max-length` | string, bytes | Maximum length |
| `system/type/constraint/min-count` | array, map | Minimum element count |
| `system/type/constraint/max-count` | array, map | Maximum element count |
| `system/type/constraint/pattern` | string | RE2 full match |
| `system/type/constraint/one-of` | any | Enumerated allowed values |
| `system/type/constraint/not-one-of` | any | Enumerated disallowed values |
| `system/type/constraint/format` | string | Named format validation |
| `system/type/constraint/type-pattern` | hash, path | Referenced entity type glob |

---

## 12. Conformance

### 12.1 MUST Implement

- Constraint dispatch model (§2) — constraints on field-specs dispatched to handlers
- Standard constraint handler with all 11 constraint types (§4, §5)
- Validate operation on the type handler — structural + constraint (§2.3)
- Constraint narrowing verification for `extends` (§6)
- Fail closed on unknown constraint types (§1.2)
- RE2 or equivalent linear-time regex engine for `pattern` constraints (§4.3)
- Validate request/result types (§5.2, §5.3)

### 12.2 SHOULD Implement

- Compare operation (§7.2)
- Compatible operation (§7.3)
- Constraint validation at system boundaries — entity receipt, handler output (ENTITY-NATIVE-TYPE-SYSTEM.md §1.6)
- Budget/timeout on custom constraint dispatch (§2.4)
- Narrowing verification at type registration time (§6.4)

### 12.3 MAY Implement

- Converge operation (§7.4)
- Adopt operation (§7.5)
- Reconcile operation (§7.6)
- Custom format validators beyond the standard set (§4.5)
- Batching constraint dispatch (multiple constraints per handler call)
- Constraint validation timing beyond system boundaries (on tree write, on explicit request)

---

## 13. Document history

| Version | Date | Change |
|---|---|---|
| 1.1 + Amendment 1 | — | **Cross-impl Phase 1 closeout** per `proposals/implemented/PROPOSAL-TYPE-V1.1-CROSS-IMPL-CLOSEOUT.md`. Four asks absorbed from Go's cross-impl review after all three impls (Go / Rust / Python) closed at PASS=30/30 (Go, Python) and PASS=27 + WARN=3 MAY-ops-absent (Rust). **Ask 1**: `array_of` / `map_of` wire-shape rule pinned in `ENTITY-NATIVE-TYPE-SYSTEM.md` §2.8 (new) — `core/entity` inner type_ref → entity envelopes; named-type inner type_ref → flat records. **Ask 2**: handler registration convention pinned in `GUIDE-EXTENSION-DEVELOPMENT.md` §4.9 (new) + cross-ref in `ENTITY-CORE-PROTOCOL.md` §6.6; companion callout added here at §5.1 — manifest's `pattern` field is advertisement-only; `system/handler` entity registers at the bare prefix (`system/type/constraint`, not `…/*`). **Ask 3**: violation-kind mapping pinned in §2.3 (new "Violation kind mapping" subsection) — dispatch-could-not-evaluate → `unknown_constraint`; dispatch-ran-and-failed → `constraint`; classifier mechanism is implementation-defined; §8.5 violation-type description updated with cross-ref. **Ask 4**: SHOULD-non-empty `reason` nudge added to §8.5. All three impls already conform to the pinned rules; no behavior change. Cross-impl gate (§5.5 ECF byte-equality `one_of` / `not_one_of`) remains the load-bearing surface and passes on all three. |
| 1.1 | — | **Six amendments landed** per `proposals/implemented/PROPOSAL-EXTENSION-TYPE-V1.1.md`. **B1**: §1.4 rewritten as "The Type Graph" — types form a graph; paths are storage convention; the graph is the coherency surface. New §1.5 "Type Resolution Strategies" — four normatively-described strategies (path-convention default; configurable base path; type-class iteration via EXTENSION-QUERY; reverse-hash-index walking) plus five graph-integrity invariants. Strategy 1 (path-convention) is the only strategy a v1.1-conforming impl MUST support today; Strategy 3 SHOULD become broadly supported as EXTENSION-QUERY matures; cross-peer determinism is explicitly NOT an invariant — divergences are observable via `compare` / `compatible` / `reconcile`. Existing §1.5 Terminology renumbered to §1.6. The §1.4 "known limitation" framing of the hardcoded prefix is replaced — the limitation is now correctly attributed to the default strategy, not the type system. **B2**: §4.5 well-known format vocabulary promoted from SHOULD-recognize to MUST-recognize-at-listed-semantics; §6.2 format-narrowing row updated — sub-format relationships default to "incomparable" (interop-safe). **B3**: §6.2 pattern-narrowing row updated — equal patterns narrow; non-equal patterns default to incomparable (no speculative regex-subset recognition). **B4**: §9 tree-write-time validation cap surface clarification — the cap surface lives on `system/tree`, not on `system/type`. **B5**: §1.1 compute-backed-constraints scope confirmed — admissible by construction as Class 4 under §2.2; convergence-semantics upgrade to Class 1/2 deferred to a follow-on amendment. **B6** (companion): `GUIDE-EXTENSION-DEVELOPMENT.md` new §3.5 "Paths are convention; the entity graph is coherence" — landed in the same round; TYPE v1.1 §1.4/§1.5 and CONTENT v3.5 §5.3 are the worked examples. Go's `RESPONSE-EXTENSION-CONTENT-TYPE-PROPOSALS` review absorbed (O7/O8/O9 / Q4/Q6). |
| 1.0 | — | Initial draft. Constraint dispatch model (§2), 11 standard constraint types (§4), standard constraint handler (§5), narrowing rules (§6), type analysis operations — compare / converge / compatible / adopt / reconcile (§7), conformance MUST / SHOULD / MAY (§12). |

**Hygiene pass:** Header reformatted to `GUIDE-EXTENSION-DEVELOPMENT.md` §3.3 shape (added Used-by, Owned namespaces, Owned ops, Owned entity types, Extension points exposed/consumed). Conformance grade added per GUIDE §9 (Draft). §1.1 future-cross-ref to `EXTENSION-COMPUTE.md §10.3` updated to current compute version (v3.19+) and reframed — the dispatch surface already admits a compute-backed constraint handler; convergence semantics defer to a follow-on TYPE amendment. §4.6 stale reference to `PROPOSAL-COMPUTE-AMENDMENTS C4` updated to the landed `PROPOSAL-COMPUTE-CONTENT-STORE-SCOPING.md` (D3–D6 → EXTENSION-COMPUTE v3.7). No normative change. Source: review against the GUIDE-EXTENSION-DEVELOPMENT initial draft.
