# Continuation Extension — Normative Specification

**Version**: 1.20

**Status**: Active
**Depends**: ENTITY-CORE-PROTOCOL.md (v7.33+)
**Proposal**: PROPOSAL-CONTINUATION-MODEL.md, PROPOSAL-CONTINUATION-SPEC-AMENDMENT.md (A1-A6), PROPOSAL-DELIVERY-AND-INBOX-RENAME.md (D5, D6), PROPOSAL-CONTINUATION-TRANSFORM-AND-ENVELOPE-AMENDMENTS.md (S1), PROPOSAL-COHERENT-CAPABILITY-AUTHORITY.md (CT1, CT2, CT3)

---

> **Path notation.** Paths in this document use peer-relative notation (without leading `/{peer_id}/`). All peer-relative paths resolve to the local peer's namespace: `system/tree` means `/{local_peer_id}/system/tree`. Every path in the entity tree is absolute at rest — rooted at a peer identity. See ENTITY-CORE-PROTOCOL.md §1.4 for the path model. Cross-peer examples use absolute paths with explicit peer identities.

## 1. Overview

The continuation extension enables execution chaining — determining what happens when a result arrives at a path. A **continuation entity** at a path describes what to do with an incoming result: dispatch to a target handler, accumulate for fan-in, or record exhaustion.

The `system/delivery-spec` type referenced by this extension (in `on_error` and `deliver_to` fields) is defined in ENTITY-CORE-PROTOCOL.md §3.11.

This extension defines execution logic only:

- Continuation types (forward, join, suspended, transform)
- Continuation handler (advance, resume, abandon)
- Advancement algorithm
- Dispatch capability model
- Suspension lifecycle
- Structural transforms (extract, select)

This extension is independent of EXTENSION-INBOX.md. The advancement algorithm takes a path and a result — it does not depend on how the result arrived. EXTENSION-INBOX.md provides one delivery mechanism (async inbox transport). Direct `advance` operations, future mechanisms, or any handler that dispatches to `system/continuation` with an `advance` operation are equally valid sources.

**Installation combinations**:

| CONTINUATION | INBOX | Result |
|-------------|----------|--------|
| yes | yes | Full flow: async delivery triggers continuation advancement |
| yes | no | Suspended continuations, direct advancement via `advance` operation |
| no | yes | Async delivery with tree storage (EXTENSION-INBOX tree storage behavior) |
| no | no | Sync-only peer |

### 1.1 Coherent Capability

`system/continuation` and `system/continuation/join` entities are load-bearing: their `dispatch_capability` field authorizes a future dispatch by the continuation handler when `advance` fires. Use `system/continuation:install` to create them. The install operation validates the embedded `dispatch_capability` against the writer's authority chain (§3.2) and persists the entity under the handler's own grant.

Direct `tree:put` to `system/continuation/suspended/*` is permitted but bypasses the install operation's validation; continuations created that way are not guaranteed to satisfy the `dispatch_capability` authority invariant. Application-level capability grants should cover `system/continuation:install` rather than direct `tree:put` to the namespace. Direct `tree:put` is appropriate for system-extension code (handlers writing under their own grant) and administrative/bootstrap contexts. See ENTITY-CORE-PROTOCOL.md §6.3 for the kernel-vs-handler principle.

**Sub-identity behavior (informative).** Delegated sub-identities (non-peer actors with delegated grants from a peer's root) calling `install` directly are naturally rejected by the chain-root check — the dispatch_capability must chain to a peer identity that the continuation handler can wield, and the sub-identity isn't in that chain. This is by design: sub-identities don't set up deferred dispatches directly. They invoke handler operations that may internally create continuations under the handler's own authority.

---

## 2. Type Definitions

### 2.1 Forward Continuation

Stored at a path. Describes what to do when a result arrives. The continuation is a closure: `params` is the captured environment, `result_field` is the argument slot, `{target, operation}` is the function to call.

```
system/continuation := {
  fields: {
    target:               {type_ref: "system/tree/path"}
    operation:            {type_ref: "primitive/string"}
    resource:             {type_ref: "system/protocol/resource-target", optional: true}
    params:               {type_ref: "primitive/any", optional: true}
    result_transform:     {type_ref: "system/continuation/transform", optional: true}
    result_field:         {type_ref: "primitive/string", optional: true}
    result_merge:         {type_ref: "primitive/bool", optional: true}
                           ; When true, the (transformed) result — which MUST be
                           ; a map, typically a `select` output — is shallow-merged
                           ; into static `params` at top level (Merge mode, below).
                           ; Mutually exclusive with `result_field`.
    on_error:             {type_ref: "system/delivery-spec", optional: true}
    deliver_to:           {type_ref: "system/delivery-spec", optional: true}
    remaining_executions: {type_ref: "primitive/uint", optional: true}
    dispatch_capability:  {type_ref: "system/hash", optional: true}
                           ; MUST be present on continuations that dispatch
                           ; (advance operation). The creating handler — caller
                           ; or system — provides this at creation time. Optional
                           ; only for continuations that do not dispatch (e.g.,
                           ; storage-only inbox entries without continuation
                           ; semantics).
  }
}
```

| Field | Purpose |
|-------|---------|
| `target` | Handler path to dispatch to |
| `operation` | Operation to invoke on target handler |
| `resource` | Resource scope for the dispatched EXECUTE |
| `params` | Pre-built params — the captured environment |
| `result_transform` | Structural navigation on result before injection |
| `result_field` | Field name in `params` to inject the (transformed) result into |
| `result_merge` | When true, shallow-merge the (transformed) map result into static `params` at top level (Merge mode). Mutually exclusive with `result_field`. |
| `on_error` | Delivery spec for error delivery (compensation) — symmetric with `deliver_to` |
| `deliver_to` | Chain link — delivery spec for the next step in the chain |
| `remaining_executions` | null = unlimited (standing). N = N executions remaining. 0 = exhausted. Decremented on successful dispatch acceptance, not on dispatch completion — for async chains the dispatch is fire-and-forget at this point. |
| `dispatch_capability` | Capability token hash authorizing dispatch to target |

**Dispatch modes** (determined by field presence):

| `params` | `result_field` | `result_merge` | Mode | Behavior |
|----------|---------------|---------------|------|----------|
| absent | absent | false/absent | Pass-through | `final_params = result` (result IS the params) |
| present | present | false/absent | Inject | `final_params = params; params[result_field] = result` |
| present | absent | false/absent | Trigger | `final_params = params` (result ignored) |
| present | absent | **true** | **Merge** | `final_params = shallow_merge(params, result)` — top-level key union; result keys win on collision; result MUST be a map (non-map → static-only + `merge_value_not_map` marker, §3.4) |
| absent | absent | **true** | Merge (empty scaffold) | `final_params = result` when result is a map (merge into `{}`) |
| absent | present | — | Invalid | Return error 400 (`invalid_continuation`) |
| — | present | **true** | Invalid | `result_merge` + `result_field` are mutually exclusive — return error 400 (`invalid_continuation`) at install |

**Merge mode (v1.16).** `result_merge: true` shallow-merges the post-transform value (which MUST be a map — typically a `result_transform.select` output) into the static `params` at top level, producing a flat param map. This is the assembly path for dispatching an op that takes a static scaffold plus several dynamic fields pulled from the result (e.g. `op(prefix, base, target)` where `prefix` is static and `base`/`target` come from a notification) — which Inject mode cannot express (it nests the whole value under one `result_field` key) and Pass-through cannot express (it drops the static scaffold). Semantics: shallow only (top-level key union, no recursion); result keys win on collision (dynamic data overrides static defaults); `result_merge: false` is identical to absent. Per `proposals/implemented/PROPOSAL-CONTINUATION-MERGE-ASSEMBLY.md`.

### 2.2 Structural Transform

Specifies structural navigation on the result. Applied before dispatch mode logic.

```
system/continuation/transform := {
  fields: {
    extract:           {type_ref: "primitive/string", optional: true}
    select:            {map_of: {type_ref: "primitive/string"}, optional: true}
    transform_ops:     {array_of: {type_ref: "system/continuation/transform-op"}, optional: true}
                       ; Ordered list of closed, total, pure, bounded field
                       ; operations. Applied after extract/select, before the
                       ; *_extract fields. See "Bounded field operations" below.
    resource_extract:  {type_ref: "primitive/string", optional: true}
                       ; Dotted path into the post-extract/post-select value.
                       ; Produces the resource targets for the dispatched EXECUTE.
                       ; Overrides static resource on the continuation entity when
                       ; present and resolves.
    target_extract:    {type_ref: "primitive/string", optional: true}
                       ; Dotted path producing target URI for the EXECUTE.
    operation_extract: {type_ref: "primitive/string", optional: true}
                       ; Dotted path producing operation name for the EXECUTE.
  }
}
```

**`extract`**: Dotted path into the result. `"data.metrics.score"` navigates to `result.data.metrics.score`. If any segment is missing, the extracted value is null.

**`select`**: Map of `{destination_field: source_path}`. Each source_path is a dotted path into the extracted value. Produces a new map with the specified field names and extracted values.

**Processing pipeline**: `extract -> select -> transform_ops -> *_extract`. All are optional. When both extract and select are present, extract runs first and select operates on the extracted value; `transform_ops` then runs on the post-extract/post-select value; the `*_extract` fields run last.

**EXECUTE field extraction.** Three additional optional fields allow the transform to populate EXECUTE fields from the navigated value:

**`resource_extract`**: Dotted path into the post-extract/post-select value. The resolved value produces the resource targets for the dispatched EXECUTE. If the resolved value is a string, it is wrapped as `{targets: [value]}`. If the resolved value is an array, it is wrapped as `{targets: value}`. If the resolved value is already a well-formed resource target object (has a `targets` field), it is used as-is. Overrides the static `resource` on the continuation entity when present and when the path resolves. If navigation fails, the static `resource` is used.

**`target_extract`**: Dotted path into the post-extract/post-select value. The resolved value (a string) becomes the target URI for the dispatched EXECUTE. Overrides the static `target` on the continuation entity when present and when the path resolves. If navigation fails, the static `target` is used.

**`operation_extract`**: Dotted path into the post-extract/post-select value. The resolved value (a string) becomes the operation for the dispatched EXECUTE. Overrides the static `operation` on the continuation entity when present and when the path resolves. If navigation fails, the static `operation` is used.

The `*_extract` fields are applied after `extract`, `select`, and `transform_ops`. They navigate the same value that dispatch mode logic receives. The EXECUTE field extractions and dispatch mode assembly are independent — both operate on the post-extract/post-select value.

**Capability interaction.** The `dispatch_capability` on the continuation is checked at dispatch time (step 5 in §3.5) against the **final** EXECUTE — including any dynamically-resolved fields. A capability scoped to `{resources: {include: ["peer_b/data/shared/*"]}}` authorizes any dynamically-extracted path matching that pattern. No capability model change is needed.

**Bounded field operations (`transform_ops`).** An optional ordered list of typed field operations applied **after `extract`/`select`, before the `*_extract` fields**. Each op reads and/or writes named fields of the post-navigation value. They exist so a common, total, declarative need — chiefly the cross-peer-notification→local-path rewrite (strip one statically-known prefix, prepend one statically-known prefix) — stays inside the declarative, statically-analyzable transform instead of forcing an opaque handler step into an otherwise-declarative chain (which would puncture the declarative legibility the transform exists to provide).

```
system/continuation/transform-op := {
  fields: {
    op:      {type_ref: "primitive/string"}        ; operation name (table below)
    field:   {type_ref: "primitive/string", optional: true}
    into:    {type_ref: "primitive/string", optional: true}
    fields:  {array_of: {type_ref: "primitive/string"}, optional: true}
    prefix:  {type_ref: "primitive/string", optional: true}
    literal: {type_ref: "primitive/string", optional: true}
    from:    {type_ref: "primitive/string", optional: true}
    to:      {type_ref: "primitive/string", optional: true}
    sep:     {type_ref: "primitive/string", optional: true}
    range:   {type_ref: "primitive/string", optional: true}
  }
}
```

| `op` | Shape | Effect |
|---|---|---|
| `strip_prefix` | `{field, prefix}` | `field` with the literal `prefix` removed if present (no-op if absent) |
| `prepend` | `{field, literal}` | `literal` prepended to `field` |
| `append` | `{field, literal}` | `literal` appended to `field` |
| `join` | `{fields:[...], sep, into}` | the listed fields joined by `sep` into `into` |
| `replace_literal` | `{field, from, to}` | every literal (non-regex) `from` in `field` replaced with `to` |
| `split` | `{field, sep, into}` | `field` split on `sep` into an array at `into` |
| `slice` | `{field, range, into}` | bounded segment of `field` (by `range`) into `into` |
| `collect_keys` | `{field, into}` OR `{fields:[...], into}` | array of keys from the map(s) projected, written to `into`. Singular form (`field`) projects one map's keys. Plural form (`fields: [...]`) projects keys from each listed map and concatenates the result in list order. Empty map → empty array; concatenating empties → empty array. Missing/non-map at any listed field → no-op (best-effort rule applies). `field` and each entry in `fields` follow the dotted-path navigation rules from `extract` (§2.2). A single op MUST NOT set both `field` and `fields`; impls MUST reject install with `400 invalid_transform_args` when both are present. Empty `into` is a silent no-op per the general best-effort rule. |
| `deref_included` | `{field}` | reads `field` as a `system/hash` reference and replaces it with the entity bound to that hash in the **envelope's `included` map**. Missing field, non-hash value, or hash absent from `included` → no-op (best-effort rule). **In-flight envelope navigation** — the entity was bundled by the sender per ENTITY-CORE-PROTOCOL.md §3.1 and is structurally present at advance time — **not** a tree/store read (see Boundary). Does not assume any fixed hash length (`system/hash` is variable-length, ENTITY-CORE-PROTOCOL.md §1.2). Lets a continuation chain consume an entity delivered in the envelope — e.g. an `include_payload` notification (EXTENSION-SUBSCRIPTION.md §2.2) — and feed it into a `tree:put` without an opaque handler step. Requires request-side `included` preservation (ENTITY-CORE-PROTOCOL.md §3.3) so the map reaches the continuation. |

**Admissibility (the contract — this, not the table, is what conformance binds).** An op is admissible iff it is **total** (defined for every input; a missing field is a documented no-op/null, consistent with the best-effort rule below), **pure** (output a function of inputs only — no clock, RNG, I/O, tree, or capability access), **bounded** (no unbounded iteration; `split`/`join`/`slice` bounded by input length), and **statically analyzable** (the analyzer can determine the op's effect on field shapes). Conditionals, loops, regular expressions, and arbitrary expressions are **not** admissible — those remain a handler step (the legitimate off-ramp). `transform_ops` is field plumbing, not a computation surface. Implementations supporting transforms MUST implement the agreed op set with these semantics; an **unrecognized `op` MUST be rejected at install (fail-closed), never silently skipped** — with status **`400 unknown_transform_op`** (the error-code string is pinned for cross-impl conformance assertions; reject-with-this-code is the binding behavior, the string is the pinned spelling). The op table is the initial set; it grows on demonstrated need under the same admissibility contract — not all ops need ship in the first implementation, but the contract is fixed.

**Boundary**: Extract, select, and EXECUTE field extraction are pure structural navigation. `transform_ops` additionally provides a closed, total, pure, bounded set of field operations. Together they remain free of conditionals, iteration, regular expressions, tree references, and general computation — any of which is a handler step, not a transform.

**Envelope navigation is in-scope; tree navigation is not.** `deref_included` resolves a hash field against the *envelope's* `included` map — the in-flight message payload, structurally available at advance time and pure (a function of the input envelope, exactly as `extract` is a function of the prior-step result). This is categorically distinct from a **tree reference** (a read of mutable tree/store state), which remains excluded. Justification: the receiver *already* resolves entities referenced by hash from `EXECUTE.data` against `included` per ENTITY-CORE-PROTOCOL.md §3.1 — `deref_included` exposes that same structural navigation to the transform layer, adding no new capability. It is total (no-op on miss), pure (`included` is part of the request, not the tree — no I/O), bounded (one lookup), and statically analyzable (hash → entity-shape).

**Transform error handling**: If CBOR decoding of the result fails during transform processing, or if `extract` navigates to a missing path, the implementation MUST pass the original unmodified result to the dispatch mode logic. Transforms are best-effort structural navigation — they do not produce errors.

If `select` produces a map where all values are null (every source path missed), the implementation SHOULD pass the null map to dispatch mode logic (not fall back to the original result). The select operation succeeded — it produced the specified shape with null values.

### 2.3 Join Continuation

Fan-in continuation — accumulates results from parallel operations.

```
system/continuation/join := {
  fields: {
    expected:             {array_of: {type_ref: "primitive/string"}}
    received:             {map_of: {type_ref: "primitive/any"}, optional: true}
    target:               {type_ref: "system/tree/path"}
    operation:            {type_ref: "primitive/string"}
    resource:             {type_ref: "system/protocol/resource-target", optional: true}
    params:               {type_ref: "primitive/any", optional: true}
    result_field:         {type_ref: "primitive/string", optional: true}
    on_error:             {type_ref: "system/delivery-spec", optional: true}
    deliver_to:           {type_ref: "system/delivery-spec", optional: true}
    remaining_executions: {type_ref: "primitive/uint", optional: true}
    dispatch_capability:  {type_ref: "system/hash", optional: true}
                           ; MUST be present on continuations that dispatch
                           ; (advance operation). The creating handler — caller
                           ; or system — provides this at creation time. Optional
                           ; only for continuations that do not dispatch (e.g.,
                           ; storage-only inbox entries without continuation
                           ; semantics).
  }
}
```

**`expected`**: Array of slot names. Results arrive at `{join_path}/{slot_name}`.

**`received`**: Accumulated results. Initially absent or empty. Updated on each slot delivery. When all expected slots are present in `received`, the join dispatches.

**Slot delivery**: When a result arrives at a child path of a join entity, the handler:

1. Reads join entity at the parent path
2. Validates `slot_name` is in `expected`
3. Rejects if `slot_name` is already in `received` (exactly-once per round)
4. Copies `received`, sets `received[slot_name] = result`
5. If all expected slots are filled -> dispatch (the completed `received` map is the result, injected via `result_field` or passed through)
6. If not complete -> update join entity at parent path using CAS (`expected_hash`)

**Concurrency**: Simultaneous slot arrivals are serialized by the tree's emit pathway. CAS via `expected_hash` on the join entity update prevents lost slot results. Duplicate delivery to an already-filled slot is rejected with 409 `slot_already_filled` (§3.4). The CAS retry loop re-reads the join entity, so the "slot already filled" check is always against current state — this gives exactly-once slot semantics within each round. After a standing join resets `received` on dispatch, slots can be filled again for the next round.

**Implementation note**: Implementations MAY use process-local locking (e.g., a mutex per join path) as an optimization when CAS is not required (single-process deployment). The CAS mechanism via `expected_hash` remains normative for multi-process and distributed deployments. Implementations using process-local locking SHOULD clean up lock state when join entities are deleted.

### 2.4 Suspended Continuation

System-created when an operation cannot complete.

```
system/continuation/suspended := {
  fields: {
    target:          {type_ref: "system/tree/path"}
    operation:       {type_ref: "primitive/string"}
    resource:        {type_ref: "system/protocol/resource-target", optional: true}
    params:          {type_ref: "primitive/any", optional: true}
    reason:          {type_ref: "primitive/string"}
    chain_id:        {type_ref: "primitive/string"}
    original_author: {type_ref: "system/hash"}
    suspended_at:    {type_ref: "primitive/uint"}
  }
}
```

Stored at `system/continuation/suspended/{id}`. Created by:

- The dispatch layer, via registered suspension handler (§3.8)
- The continuation handler, on dispatch failure during advancement (§3.3)
- Handlers, on unrecoverable errors they want to preserve for later resolution

The suspended entity captures enough state to reconstruct the original EXECUTE with fresh bounds. `reason` identifies why the operation was suspended. `original_author` identifies who initiated the chain. `suspended_at` records when suspension occurred.

### 2.5 Advance Request

Input type for the `advance` operation on the continuation handler.

```
system/continuation/advance-request := {
  fields: {
    result: {type_ref: "primitive/any", optional: true}
    status: {type_ref: "primitive/uint", optional: true}
  }
}
```

| Field | Purpose |
|-------|---------|
| `result` | The result value to advance with. Absent = null result. |
| `status` | HTTP-style status of the result. Absent = success (200). |

The continuation handler owns this contract. Callers (inbox handler, direct users) construct an advance-request and dispatch to `system/continuation` with operation `advance`.

### 2.6 Resume and Abandon Requests

```
system/continuation/resume-request := {
  fields: {
    bounds:     {type_ref: "system/bounds", optional: true}
    resolution: {type_ref: "primitive/any", optional: true}
    deliver_to: {type_ref: "system/delivery-spec", optional: true}
  }
}

system/continuation/abandon-request := {
  fields: {}
}
```

### 2.7 Install Request and Result

The `install` operation is the proper create path for continuation entities — see §3.2 and §1.1 (Coherent Capability).

**No wrapper request type (v1.7).** Callers pass a `system/continuation` or `system/continuation/join` entity directly as `params`. The install path is carried in `EXECUTE.resource.targets[0]` per the path-as-resource convention (ENTITY-CORE-PROTOCOL.md §3.2). The install handler discriminates forward vs join on `params.type`. There is no `system/continuation/install-request` wrapper — every field that would have been on it is either data on the continuation entity itself (`target`, `operation`, `resource`, `params`, `result_field`, `dispatch_capability`, `on_error`, `deliver_to`, `remaining_executions`) or the install path (now resource).

```
system/continuation/install-result := {
  fields: {
    path: {type_ref: "system/tree/path"}            ; echoes the suspended path where the continuation was installed
  }
}
```

(`install-result` retained for forward-compat — confirms the install path and leaves room for additional install metadata in future revisions.)

---

## 3. Handlers

### 3.1 Continuation Handler

```
system/handler := {
  pattern:    "system/continuation"
  name:       "continuations"
  operations: {
    install: {
      input_type:  "system/continuation"          ; or system/continuation/join — dispatched on params.type
      output_type: "system/continuation/install-result"
    }
    advance: {input_type: "system/continuation/advance-request"}
    resume:  {input_type: "system/continuation/resume-request"}
    abandon: {input_type: "system/continuation/abandon-request"}
  }
}
```

Handler entity at pattern path `system/continuation`. Dispatch resolves any URI under `system/continuation/` to this handler via longest-prefix match (ENTITY-CORE-PROTOCOL.md §6.6).

The `install` operation is the proper create path for continuation entities — see §3.2.

The `advance` operation implements the continuation advancement algorithm (§3.3-§3.6). The continuation path is specified via `resource.targets[0]`. Any handler or extension can trigger advancement by dispatching to `advance`.

The `resume` operation reconstructs an EXECUTE from a suspended continuation and dispatches it (§3.7).

The `abandon` operation deletes a suspended continuation (§3.8).

### 3.1a Authority-chain check: in-chain, not chain-root (normative)

The install-time authorization check on an embedded `dispatch_capability` (§3.2 step 4, via `check_creator_authority`, ENTITY-CORE-PROTOCOL.md §5.5) is an **in-chain** check: it collects the full authority chain and verifies the writer's identity **appears as a granter anywhere in the chain** — *not* that the chain *roots at* the writer. The error path is `403 embedded_cap_unauthorized` ("Writer identity not in dispatch_capability authority chain").

This distinction is silent for local continuations and load-bearing for cross-peer ones:

- **Local (caller- or system-created).** The `dispatch_capability` is attenuated from the installer's own authority, so the chain *both* roots at the local peer identity *and* has the installer in it. In-chain and rooted-at-installer coincide; "the chain-root check passes by construction" is true only as a description of that coincidence, not as an independent requirement.
- **Cross-peer (a step whose `target` is a remote peer B).** The chain MUST root at the authority **B** conferred (so B's advance-time `VerifyChain`, ENTITY-CORE-PROTOCOL.md §5.2, succeeds) with the installer **in-chain as the re-attenuation leaf granter** (so the install-time in-chain check passes). A chain *rooted at the installer* is the local sufficient condition only and is **wrong** for a remote target — B cannot verify it. See §4.2 case 3.

Wherever this spec or a conformance test says "chain root against author" / "chain-root check" for the install-time authorization, read it as the in-chain check defined here. §3.2, §4.2, and §8.1 are written to this rule; the prior chain-root phrasing was pre-correction residue that, taken verbatim cross-peer, reproduces the dispatch-capability collapse this section prevents.

### 3.2 Install Operation

Creates a continuation entity (forward or join) at a suspended path under `system/continuation/suspended/*`. This is the proper create path — direct `tree:put` of `system/continuation` / `system/continuation/join` entities is reserved for system-extension and administrative use (see §1.1 and ENTITY-CORE-PROTOCOL.md §6.3).

The caller passes a continuation entity directly as `params`; the install path is carried in `EXECUTE.resource.targets[0]` per the path-as-resource convention (ENTITY-CORE-PROTOCOL.md §3.2). The handler dispatches forward vs join on `params.type` — one operation, two accepted entity types.

```
handle_install(ctx, params):
  ; Step 1: install path comes from EXECUTE.resource per V7 §3.2.
  if ctx.resource is null or len(ctx.resource.targets) != 1:
    return error(400, "ambiguous_resource",
      "install requires exactly one resource target (the suspended path)")
  install_path = ctx.resource.targets[0]

  ; Step 2: discriminate forward vs join on entity type. params IS the
  ; continuation entity to install.
  if params.type != "system/continuation" and params.type != "system/continuation/join":
    return error(400, "invalid_params",
      "install expects system/continuation or system/continuation/join in params")
  continuation = params

  ; Step 3: validate required fields on the entity.
  if continuation.data.dispatch_capability is null:
    return error(400, "missing_dispatch_capability",
      "continuation requires dispatch_capability for the deferred dispatch")
  if continuation.type == "system/continuation":
    if continuation.data.target is null or continuation.data.operation is null:
      return error(400, "invalid_continuation",
        "forward continuation requires target and operation")
    ; result_merge and result_field are mutually exclusive (Merge vs Inject mode) — v1.16
    if continuation.data.result_merge == true and continuation.data.result_field != null:
      return error(400, "invalid_continuation",
        "result_merge: true is mutually exclusive with result_field")

  ; Step 4: R1 creator-authorization check on the embedded dispatch_capability.
  ; Uses check_creator_authority (V7 §5.5) which collects the full authority
  ; chain via collect_authority_chain, verifies reachability, then checks
  ; whether the writer's identity appears as granter anywhere in the chain.
  cap_entity = resolve(continuation.data.dispatch_capability, ctx)
  if cap_entity is null:
    return error(404, "dispatch_capability_not_found",
      "Referenced capability entity not in content store or envelope")
  (found, chain, err) = check_creator_authority(cap_entity, ctx.execute.data.author, ctx)
  if err == ChainUnreachable:
    return error(404, "chain_unreachable",
      "Embedded capability's authority chain is not fully resolvable")
  if not found:
    return error(403, "embedded_cap_unauthorized",
      "Writer identity not in dispatch_capability authority chain")

  ; Step 5: Persist the continuation + authority chain to local content store.
  ; The chain was already collected by check_creator_authority — no re-walk.
  ; The continuation is bound at install_path under the handler's own grant
  ; (handler-managed namespace per V7 §6.8).
  ctx.entity_tree.put(install_path, continuation)
  for entity in chain:
    ctx.content_store.put(entity)

  return {status: 200, result: {path: install_path}}
```

**System-created continuations.** When a system handler creates a continuation on behalf of an external caller (e.g., the inbox handler installing a continuation for delivery routing), the handler invokes `install` under its own grant. The handler's identity is the `ctx.execute.data.author` for the install dispatch; the embedded `dispatch_capability` is the handler's own scoped grant attenuated from its authority. The chain roots at the local peer identity *and* the handler is in it as granter, so the §3.1a in-chain check passes by construction (the local case where in-chain ⇔ rooted-at-installer). No special exemption needed — the rule applies uniformly.

### 3.3 Advancement Algorithm

```
advance_at_path(path, result, status):
  ; status is optional — null means success (200)
  effective_status = status or 200

  ; 1. Check for continuation entity at the path
  continuation = entity_tree.get(path)

  ; 2. Forward continuation — advance immediately
  if continuation != null and continuation.type == "system/continuation":
    return advance_forward(continuation, result, effective_status, path)

  ; 3. Join slot — check if parent is a join entity
  if continuation is null or continuation.type not in ["system/continuation", "system/continuation/join"]:
    parent_path = parent_of(path)
    slot_name = last_segment(path)
    join = entity_tree.get(parent_path)
    if join != null and join.type == "system/continuation/join":
      return advance_join_slot(join, parent_path, slot_name, result)

  ; 4. Direct advance to join path (not a slot) — error
  if continuation != null and continuation.type == "system/continuation/join":
    return error(400, "join_requires_slot_path")

  ; 5. No continuation at path — no-op
  return {status: 200, result: {advanced: false}}
```

### 3.4 Forward Advancement

```
advance_forward(continuation, result, status, continuation_path):
  ; Check exhaustion
  if continuation.data.remaining_executions != null:
    if continuation.data.remaining_executions == 0:
      return {status: 200, result: {advanced: false, exhausted: true}}

  is_error = status >= 400

  ; Error path: status >= 400
  if is_error and continuation.data.on_error != null:
    ; on_error is a system/delivery-spec — deliver through inbox handler
    error_delivery_spec = continuation.data.on_error
    dispatch({
      uri:       error_delivery_spec.uri
      operation: error_delivery_spec.operation or "receive"
      resource:  {targets: [error_delivery_spec.uri]}
      params:    {
        type: "system/protocol/inbox/delivery"
        data: {original_request_id: context.request_id, status: status, result: result}
      }
    })
    ; Error delivery uses the same dispatch mechanics as normal continuation
    ; dispatch: same resource resolution, same capability check (via
    ; dispatch_capability), same authorization rules.
    ;
    ; Error delivery is best-effort — dispatch result is not checked.
    ; Transient failure during on_error delivery is silently lost (no suspension,
    ; no propagation). This is intentional: on_error is a compensation path,
    ; not a guaranteed delivery. Adding suspension to on_error would create
    ; recursive error-handling complexity with diminishing returns.
    ;
    ; If the continuation at the on_error path also has on_error, the error
    ; can chain through multiple error paths. Chain depth (§3.8) is the
    ; backstop against unbounded error routing.
    return {status: 200, result: {advanced: true, error_routed: true}}
  else:
    ; Forward path
    dispatch_result = execute_dispatch(continuation, result)

  ; Handle dispatch result
  if dispatch_result.error != null:
    if dispatch_result.error.transient:
      ; Transient failure — suspend for later retry
      ; Do NOT decrement remaining_executions
      suspend_execution(
        target:    continuation.data.target
        operation: continuation.data.operation
        resource:  continuation.data.resource
        params:    dispatch_result.assembled_params
        reason:    dispatch_result.error.reason
        chain_id:  dispatch_result.chain_id or generate_id()
      )
      ; Note: the write-ahead delivery entity stored by the inbox handler
      ; (EXTENSION-INBOX.md §3.2) remains in the tree since advancement
      ; did not succeed. Implementations SHOULD clean up orphaned write-ahead
      ; entities when the suspended continuation is resumed or abandoned.
      return {status: 200, result: {advanced: false, suspended: true}}

    if dispatch_result.error.permanent:
      ; Permanent failure — route to on_error if available
      if continuation.data.on_error != null and not is_error:
        ; Avoid infinite error loops: only route to on_error when the
        ; original delivery was not itself an error (prevents the forward
        ; path's dispatch failure from looping through on_error if the
        ; inbound delivery already carried error status).
        ; on_error dispatch is best-effort — result not checked (see above).
        error_delivery_spec = continuation.data.on_error
        dispatch({
          uri:       error_delivery_spec.uri
          operation: error_delivery_spec.operation or "receive"
          resource:  {targets: [error_delivery_spec.uri]}
          params:    {
            type: "system/protocol/inbox/delivery"
            data: {original_request_id: context.request_id, status: dispatch_result.error.status, result: dispatch_result.error}
          }
        })
      ; Do NOT decrement remaining_executions
      return error(dispatch_result.error.status, dispatch_result.error.code)

  ; Lifecycle — only on successful dispatch
  if continuation.data.remaining_executions != null:
    updated = copy(continuation)
    updated.data.remaining_executions = continuation.data.remaining_executions - 1
    current_hash = continuation.content_hash
    entity_tree.put(continuation_path, updated, expected_hash: current_hash)
    ; On 409 conflict: re-read continuation, re-check remaining_executions, retry
    ; At 0: entity records exhausted state. Implementations SHOULD clean up
    ; exhausted continuations. The entity MUST NOT advance when at 0.

  return {status: 200, result: {advanced: true}}
```

**Forward-dispatch outcome classification (normative — v1.10).** `dispatch_result.error` above denotes a **dispatch delivery/processing failure**: the EXECUTE could not be delivered or processed *as a dispatch* — transport failure, unresolvable target, malformed dispatch, or the dispatch's own capability check failing. It is **not** the dispatched handler's response status. A *delivered* EXECUTE that returns a handler-level non-2xx (e.g. 403/404/500) is a **completed forward dispatch**, not a `dispatch_result.error`: a forward continuation propagates the result onward — it is a closure invocation, not an RPC, and the dispatched response is not threaded back (in the success path either). On a delivered handler-level non-2xx the algorithm therefore proceeds past the `dispatch_result.error` branch: `remaining_executions` is decremented and `{advanced: true}` is returned. `remaining_executions` counts completed dispatch attempts, not successful downstream outcomes. Implementations MUST classify uniformly: a returned handler-level non-2xx MUST NOT be promoted to `dispatch_result.error.transient`/`.permanent`, and MUST NOT be silently retried or suspended on that basis. (This pins behavior that §3.4 previously left undefined — the cross-impl ambiguity surfaced by the entity-core-go v1.9 peers-status report; the reference impl already behaves this way.)

Observability of a forward dispatch's downstream verdict. Forward continuations are fire-and-forget by design (a closure invocation, not an RPC). Three cases produce observable failure markers (all informational, none reactive), all of type `system/runtime/chain-error-lost`, bound at a **per-occurrence** path `system/runtime/chain-errors/lost/{chain_id}/{step_index}/{reason}/{marker_hash}` (v1.20 path scheme — canonical home is §3.10.1):

**Path-scheme cross-reference (v1.20).** The path strings below show the v1.16-era per-reason form `.../{reason}` for historical continuity; the v1.20 normative form appends `/{marker_hash}` as a terminal segment per §3.10.1 (each distinct observation lands at its own path; tree IS the event log; redelivery dedupes when impls capture `timestamp` at failure-origination per §3.10.6). Treat any v1.16/v1.13/v1.9 path string in this section as superseded by §3.10.1; the historical idempotency claims (re-binding-overwrites / flap-doesn't-multiply) were structurally false under v1.16/v1.19 (latent since v1.9 A.1 when `timestamp` body field was introduced) and are corrected by §3.10.1's v1.20 path scheme.

- **(A.1, v1.10) `on_error` dispatch itself fails** — `reason: "on_error_dispatch_failed"`. See "Lost-error marker (`on_error` delivery failure)" below.
- **(v1.13, **reason vocabulary AMENDED v1.19** — see §3.10.5) No-`on_error` forward continuation receives a handler-level non-2xx response** — current convention: `{reason}` = the response's `result.data.code` verbatim per §3.10.5 (e.g., `capability_denied` for 403, `not_found` for 404, `internal` for 500). The v1.13 reason value `"forward_dispatch_non2xx"` is deprecated as of v1.19; existing markers under the deprecated reason remain readable as historical state. See "Lost-error marker (no-`on_error` forward dispatch non-2xx)" below for the historical text + v1.19 migration note.
- **(v1.16) Merge-mode assembly meets a non-map value** — `reason: "merge_value_not_map"`. See "Lost-error marker (merge-mode non-map value)" below and §2.2 `result_merge`.

**Per-occurrence path (v1.16 per-reason subsegment + v1.20 terminal marker_hash).** Markers are bound at `.../{chain_id}/{step_index}/{reason}/{marker_hash}` (v1.20 path scheme; canonical home §3.10.1). The `{reason}` is a path subsegment, not just a body field, so distinct reasons occupy distinct sibling subtrees rather than overwriting each other. **Each distinct occurrence within a reason** lands at its own `{marker_hash}` terminal segment (v1.20) — a flapping target produces multiple markers, intentionally; the tree IS the event log. Same-content re-binding (e.g., subscription redelivery of the same logical event under §3.10.6's timestamp-capture discipline) is genuine content-addressed `tree:put` no-op. This replaces the v1.10–v1.14 shared `.../{chain_id}/{step_index}` path (where the latest writer of *any* reason clobbered the others) AND the v1.16/v1.19 `.../{chain_id}/{step_index}/{reason}` path (where the v1.16 "re-binding is idempotent" claim was structurally false given the body `timestamp` field — corrected by v1.20). The motivating case: `merge_value_not_map` is an assembly-phase observation that causally *precedes* the dispatch outcome — under the old shared path it was guaranteed to be overwritten by the downstream `forward_dispatch_non2xx` it usually triggers, hiding the root cause behind the symptom. The subsegment lets both survive and be listed under `.../{chain_id}/{step_index}/`. Behavioral-for-observability change only; markers carry no reactive behavior, so no control path depends on the path shape. (Per `proposals/implemented/PROPOSAL-CONTINUATION-MERGE-ASSEMBLY.md` §10a, option 2a.)

Neither marker triggers advancement, retry, or any reactive behavior; all preserve the fire-and-forget semantics while making the failure observable. `remaining_executions` is still decremented normally on every completed dispatch (success or non-2xx); the marker adds visibility, not behavior.

**Lost-error marker (`on_error` delivery failure).** The two `on_error` dispatches above are best-effort by design — a compensation path, not guaranteed delivery (adding suspension would create recursive error-handling complexity with diminishing returns). The hazard that leaves is a chain whose `on_error` delivery *itself* fails (mis-shaped error result, wrong delivery URI, downstream handler not ready) having **no observable surface at all** — from the developer's side: "I installed the chain, fed it input, nothing happened, no errors anywhere."

When an `on_error` dispatch fails (transient or permanent), implementations SHOULD bind a lost-error marker entity — entity **`type`: `system/runtime/chain-error-lost`** — at `system/runtime/chain-errors/lost/{chain_id}/{step_index}/on_error_dispatch_failed/{marker_hash}` (v1.20 path scheme; canonical home §3.10.1) capturing:

- the original error code and status,
- the `on_error` delivery URI that failed,
- the original request ID,
- a timestamp,
- `reason: "on_error_dispatch_failed"` (matches the path subsegment; distinguishes from the v1.13 / v1.16 cases).

**Path and type are pinned (cross-impl)**: the marker entity `type` MUST be `system/runtime/chain-error-lost` so the marker's content hash agrees across implementations. `{step_index}` is the **original request ID** of the dispatch whose `on_error` delivery failed — there is no numeric step counter. **Under v1.20** (per §3.10.1): each distinct failure observation lands at its own `{marker_hash}` terminal segment under `.../{step_index}/on_error_dispatch_failed/`. A retried-and-still-failing delivery produces a new observation (new `timestamp` per §3.10.6 timestamp-capture discipline → new `content_hash` → new path); the tree records the full retry history rather than overwriting. Bytes-identical redeliveries (same logical event redelivered) dedupe by content-addressing (same `content_hash` → same path → `tree:put` no-op). Distinct reasons at the same step occupy distinct sibling `{reason}` subtrees (unchanged). The v1.16 "re-binding overwrites" framing was structurally false under any version since v1.9 A.1 introduced the body `timestamp` field — corrected by v1.20.

The marker is informational. Consumers MAY aggregate it for diagnostics or operator surfacing. The marker MUST NOT trigger advancement, retry, or any other reactive behavior — it is an observation sink, not a control path; this preserves the best-effort semantics above (the marker adds visibility, not delivery). Implementations MAY garbage-collect markers after a configured retention window (suggested default: 24 hours).

This codifies `system/runtime/chain-errors/` as the conventional sub-purpose for chain-error sinks (per `GUIDE-PEER-CONCERNS-AND-NAMESPACES` §4.1 — "specs describing particular purposes enumerate sub-purposes"); `.../lost/` is the lost-error sink (covering both the A.1 `on_error`-delivery-failure case above and the v1.13 no-`on_error` forward-dispatch non-2xx case below).

**Lost-error marker (no-`on_error` forward dispatch non-2xx — v1.13; reason vocabulary AMENDED v1.19).** When a forward continuation with no `on_error` receives a handler-level non-2xx response, implementations SHOULD bind a lost-error marker entity — entity **`type`: `system/runtime/chain-error-lost`** (same type as the A.1 case above) — at:

```
system/runtime/chain-errors/lost/{chain_id}/{step_index}/{result.data.code}/{marker_hash}
```

where `{result.data.code}` is the response's `code` value verbatim per §3.10.5 (e.g., `capability_denied` for 403, `not_found` for 404, `internal` for 500, etc.). The marker body captures:

- the handler-level status code returned by the dispatched target (`status` field),
- the dispatched target URI (`target_uri`),
- the original request ID (`step_index`, denormalized),
- a timestamp,
- `code`: the value of `result.data.code` (matches the path subsegment, denormalized for body-side inspection per §3.10.6 — see also `reason` field which holds the same value for backward compatibility with v1.18 readers).

**Trigger range:** all status codes ≥ 400 (consistent with the `is_error` definition above). **Trigger timing:** bound on every non-2xx occurrence (not only on `remaining_executions` exhaustion). **Under v1.20** (per §3.10.1): each occurrence lands at its own `{marker_hash}` terminal segment — a flapping target that fails 10 times at 10 wall-clock moments produces 10 distinct marker entities at 10 distinct paths under `.../{step_index}/{result.data.code}/` (the tree IS the event log). Same-content re-binding (e.g., subscription redelivery of the same logical event under the §3.10.6 timestamp-capture discipline) is genuine content-addressed `tree:put` no-op. **Distinct error codes at the same step coexist as sibling `{reason}` subtrees** — a 500 then a 503 at the same step produces marker entities under `.../{step_index}/internal/{hash_a}` and `.../{step_index}/unavailable/{hash_b}` respectively. The v1.13 "re-binding overwrites within an error code" framing was structurally false under any version since v1.9 A.1 introduced the body `timestamp` field — corrected by v1.20.

**DEPRECATION (v1.19):** the v1.13 reason `"forward_dispatch_non2xx"` is deprecated. It conflated distinct error codes under a single catch-all path — silently broke the v1.16 sibling-paths property by clobbering the actual error vocabulary the response already provides. New binds use `result.data.code` as `{reason}`; existing markers under the deprecated reason value are still readable as historical state. Migration ~5-10 LoC per impl; single-commit per impl (no long-lived transition window).

**`{step_index}` for v1.13 markers (v1.14 pin).** `{step_index}` MUST be the **original request ID** of the forward dispatch that returned non-2xx — identical to the A.1 convention pinned above. Not the cascade depth, not an internal step counter; the request ID is the only stable, content-addressable key for the logical step. **Under v1.20** (per §3.10.1): step-identity stability is still required (so redeliveries of the same logical step's same observation land at the same `(chain_id, step_index, reason, marker_hash)` coordinate and dedupe via content-addressing). The retry-idempotency framing of v1.14 is now interpreted as same-content re-bind idempotency (per §3.10.1 + §3.10.6 timestamp-capture discipline); distinct observations within the same logical step occupy distinct `{marker_hash}` terminal segments. Cross-impl absorption found Rust picked `cascade_depth` (not stable across the impl's internal book-keeping) while Go picked `RequestID`; this pin matches Go and the A.1 convention.

The marker is informational. Consumers MAY aggregate it for diagnostics or operator surfacing. The marker MUST NOT trigger advancement, retry, or any other reactive behavior — `remaining_executions` is still decremented normally; the dispatch is still classified as a completed forward dispatch (per the v1.10 classification above); the chain still advances. The marker adds visibility, not behavior. Implementations MAY garbage-collect markers after a configured retention window (suggested default: 24 hours; same as the A.1 case).

This closes the silent-burn observability gap that v1.10's "Known limitation (flagged, not closed here)" note deliberately deferred. Per `proposals/PROPOSAL-CONTINUATION-NO-ON-ERROR-SINK.md` (I-8). The fix is purely additive — no behavioral change to dispatch, no new entity types beyond the existing A.1 marker, no wire-format change.

**Lost-error marker (merge-mode non-map value — v1.16).** When a continuation with `result_merge: true` (§2.2 dispatch-mode table) reaches the assembly step with a post-transform value that is **not a map**, the merge degrades to static-`params`-only and the intended dynamic fields are silently dropped — the dispatch proceeds, but with incomplete params (and so usually returns non-2xx downstream). Because this is an assembly-phase misconfiguration that causally precedes — and is masked by — the downstream outcome, implementations SHOULD bind a lost-error marker entity — entity **`type`: `system/runtime/chain-error-lost`** — at `system/runtime/chain-errors/lost/{chain_id}/{step_index}/merge_value_not_map/{marker_hash}` (v1.20 path scheme; canonical home §3.10.1) capturing:

- the post-transform value's type (what was produced instead of a map),
- the dispatched target URI,
- the original request ID,
- a timestamp,
- `reason: "merge_value_not_map"`.

`{step_index}` is the original request ID (same convention as above). The marker is informational — MUST NOT trigger advancement, retry, or any reactive behavior; the dispatch proceeds with static-only params regardless; the chain advances. Same retention as the other lost-error markers. The per-reason subsegment is load-bearing here: this marker is the **root cause** and the dispatch it precedes usually emits a `forward_dispatch_non2xx` marker (the **symptom**) for the same step — under the old shared path the symptom would overwrite the cause within milliseconds and post-hoc poll/list inspection would never recover it. Per `proposals/implemented/PROPOSAL-CONTINUATION-MERGE-ASSEMBLY.md`.

**Chain-trace anomaly detection.** A forward dispatch that completes with NEITHER (i) a `remaining_executions` decrement visible on the continuation entity at `<install_path>` NOR (ii) a chain-error marker under `system/runtime/chain-errors/{lost,rejected}/{chain_id}/{step_index}/` is **anomalous** — it indicates a dispatcher-level silent drop. Inspect consumers MAY treat this case as a §9 #8 anomaly signal (per `GUIDE-INSPECTABILITY.md` v1.1 §9 #8); `validate-peer` MAY treat it as a `CAT-CHAIN-COMPLETION` conformance failure for forward-dispatch chains. The substrate provides no behavior on detecting the anomaly — it's an observability invariant that lets tooling distinguish "the chain ran and completed (success or marked failure)" from "the chain never ran or silently dropped." Per the chain-participation-invariants proposal §2.1.

### 3.5 Join Slot Advancement

```
advance_join_slot(join, join_path, slot_name, result):
  ; Check exhaustion
  if join.data.remaining_executions != null:
    if join.data.remaining_executions == 0:
      return {status: 200, result: {advanced: false, exhausted: true}}

  ; Validate slot
  if slot_name not in join.data.expected:
    return error(400, "unexpected_slot")

  ; Exactly-once: reject if slot already filled in this round
  received = copy(join.data.received or {})
  if slot_name in received:
    return error(409, "slot_already_filled")

  ; Accumulate
  received[slot_name] = result

  ; Check completeness
  if set(keys(received)) == set(join.data.expected):
    ; All slots filled — dispatch
    ; The completed received map is the "result" for dispatch mode logic
    dispatch_result = execute_dispatch(join, received)

    ; Handle dispatch result
    if dispatch_result.error != null:
      ; Do NOT decrement remaining_executions
      ; Do NOT reset received (preserve accumulated state)
      if dispatch_result.error.transient:
        suspend_execution(...)
        return {status: 200, result: {advanced: false, suspended: true}}
      return error(dispatch_result.error.status, dispatch_result.error.code)

    ; Lifecycle — only on successful dispatch
    if join.data.remaining_executions != null:
      updated = copy(join)
      updated.data.remaining_executions = join.data.remaining_executions - 1
      updated.data.received = {}  ; Reset for next round
      entity_tree.put(join_path, updated)
      ; At 0: entity records exhausted state. SHOULD clean up.
    else:
      ; Standing join (remaining_executions: null) — reset received
      updated = copy(join)
      updated.data.received = {}
      entity_tree.put(join_path, updated)
    return {status: 200, result: {advanced: true}}
  else:
    ; Not complete — update join entity
    updated = copy(join)
    updated.data.received = received
    current_hash = join.content_hash
    entity_tree.put(join_path, updated, expected_hash: current_hash)
    ; On 409 conflict: re-read join, merge received, retry
    return {status: 200, result: {advanced: true}}
```

### 3.6 Execute Dispatch

```
execute_dispatch(continuation, raw_result):
  ; Step 1: Transform
  value = raw_result
  if continuation.data.result_transform != null:
    transform = continuation.data.result_transform
    if transform.extract != null:
      extracted = navigate(value, transform.extract)
      if extracted != NAVIGATION_FAILED:
        value = extracted
      ; else: pass original value through (best-effort navigation)
    if transform.select != null:
      mapped = {}
      for dest_field, source_path in transform.select:
        mapped[dest_field] = navigate(value, source_path)
      value = mapped

  ; Step 2: Assemble params
  ; (result_merge + result_field is rejected at install — §2.1, mutually exclusive)
  if continuation.data.result_merge == true:                      ; merge (v1.16)
    if not is_map(value):
      bind_lost_error_marker(chain_id, request_id,                ; §3.4 — observable, no reactive behavior
                             reason: "merge_value_not_map",
                             value_type: type_of(value))
      final_params = continuation.data.params or {}               ; no-op merge: static-only; dispatch still proceeds
    else:
      final_params = shallow_merge(continuation.data.params or {}, value)   ; top-level union; value keys win
  else if continuation.data.params is null and continuation.data.result_field is null:
    final_params = value                                          ; pass-through
  else if continuation.data.params != null and continuation.data.result_field != null:
    final_params = copy(continuation.data.params)
    final_params[continuation.data.result_field] = value          ; inject
  else if continuation.data.params != null:
    final_params = continuation.data.params                       ; trigger
  else:
    return error(400, "invalid_continuation")                     ; result_field without params

  ; Step 3: Build EXECUTE
  ;
  ; Resolve dynamic EXECUTE fields from the transform, falling back to
  ; static values on the continuation entity.
  if continuation.data.result_transform != null:
    transform = continuation.data.result_transform
  else:
    transform = null

  execute = {
    uri:       resolve_or_default(value, transform, "target_extract",    continuation.data.target)
    operation: resolve_or_default(value, transform, "operation_extract", continuation.data.operation)
    resource:  resolve_or_default_resource(value, transform, "resource_extract", continuation.data.resource)
    params:    final_params
  }
  ; Note: the *_extract fields navigate the post-transform `value` (above),
  ; independently of how Step 2 assembled `final_params`. Merge mode does not
  ; change which value *_extract reads — it is still the post-transform value,
  ; not `final_params`. This holds identically for pass-through, inject, and
  ; merge assembly modes.

  ; Step 4: Chain link
  if continuation.data.deliver_to != null:
    execute.deliver_to = continuation.data.deliver_to
    execute.deliver_token = generate_internal_deliver_token(continuation.data.deliver_to)

  ; Step 5: Capability — dispatch_capability MUST be present for dispatching continuations.
  ; Revocation is handled automatically by dispatch-layer verify_request
  ; (ENTITY-CORE-PROTOCOL.md §5.2 step 4, per V3 C2): if the stored
  ; dispatch_capability has been revoked since the continuation was created,
  ; the dispatched EXECUTE fails with permission_denied, and the error flows
  ; back through the continuation's error handling path (§17 — error
  ; continuation). No continuation-specific revocation check is needed.
  if continuation.data.dispatch_capability == null:
    return error(400, "missing_dispatch_capability",
      "Continuation must have dispatch_capability to dispatch")
  execute.capability = continuation.data.dispatch_capability

  ; Step 6: Dispatch with fresh bounds
  dispatch(execute, bounds: {
    ttl:      peer_default_ttl
    budget:   peer_default_budget
    chain_id: context.chain_id or generate_id()
  })
```

**Helper functions** for dynamic EXECUTE field resolution:

```
resolve_or_default(value, transform, field_name, default):
  if transform is null: return default
  extract_path = transform[field_name]
  if extract_path is null: return default
  extracted = navigate(value, extract_path)
  if extracted is null or extracted == NAVIGATION_FAILED: return default
  return extracted

resolve_or_default_resource(value, transform, field_name, default):
  if transform is null: return default
  extract_path = transform[field_name]
  if extract_path is null: return default
  extracted = navigate(value, extract_path)
  if extracted is null or extracted == NAVIGATION_FAILED: return default
  ; Wrap extracted value into resource-target structure
  if is_string(extracted): return {targets: [extracted]}
  if is_array(extracted):  return {targets: extracted}
  if is_object(extracted) and extracted.targets is not null: return extracted
  return default
```

`resource_extract` has special handling because the `resource` field is `system/protocol/resource-target` (`{targets: [...]}`) but the extracted value is typically a URI string or array of URI strings. The resolve wraps a string into the targets array structure. If the extracted value is already a well-formed resource target object (has a `targets` field), it is used as-is.

### 3.7 Resume Operation

```
handle_resume(ctx, params):
  ; params is system/continuation/resume-request
  suspended_path = ctx.resource.targets[0]
  suspended = entity_tree.get(suspended_path)

  if suspended is null:
    return error(404, "not_found")
  if suspended.type != "system/continuation/suspended":
    return error(400, "not_suspended")

  ; Build EXECUTE from suspended state
  execute = {
    uri:       suspended.data.target
    operation: suspended.data.operation
    resource:  suspended.data.resource
    params:    suspended.data.params
    bounds:    params.bounds or peer_defaults
  }

  ; Merge resolution if provided
  if params.resolution != null:
    execute.params = merge(execute.params, params.resolution)

  ; Add delivery spec if provided
  if params.deliver_to != null:
    execute.deliver_to = params.deliver_to

  ; Delete suspended entity
  entity_tree.put(suspended_path, null)

  ; Dispatch
  return dispatch(execute)
```

### 3.8 Abandon Operation

```
handle_abandon(ctx, params):
  suspended_path = ctx.resource.targets[0]
  suspended = entity_tree.get(suspended_path)

  if suspended is null:
    return error(404, "not_found")
  if suspended.type != "system/continuation/suspended":
    return error(400, "not_suspended")

  entity_tree.put(suspended_path, null)
  return {status: 200, result: {abandoned: true, path: suspended_path}}
```

### 3.9 Suspension from Dispatch Layer

The dispatch layer MAY register a suspension handler. When registered, the dispatch layer creates a suspended continuation instead of returning an error response on:

- TTL exhaustion (bounds `ttl` reaches 0)
- Budget exhaustion (bounds `budget` reaches 0)
- Chain depth exceeded (implementation-defined maximum)

The suspension handler interface:

```
suspend(target, operation, resource, params, reason, chain_id, author) -> suspended_path
```

The dispatch layer calls `suspend()` with the EXECUTE context that could not complete. The suspension handler creates a `system/continuation/suspended` entity at `system/continuation/suspended/{generated_id}` and returns the path.

The dispatch layer returns the error response with an additional field:

```
{status: 429, result: {code: "bounds_exceeded", suspended_at: suspended_path}}
```

If no suspension handler is registered, the dispatch layer returns the error response without creating a suspended entity. This preserves backward compatibility with sync-only peers.

**Chain depth tracking**: The dispatch layer threads a `chain_depth` counter through the execution context. The counter is incremented on each continuation advancement dispatch. When it exceeds the implementation-defined maximum, the dispatch layer calls `suspend()` with `reason: "chain_depth_exceeded"`. Continuation advancement dispatches with fresh TTL/budget but inherits the chain depth counter.

**Cross-peer limitation**: `chain_depth` is a per-peer execution context counter — it is not carried in `system/bounds` on the wire. Cross-peer continuation chains reset chain depth at each peer boundary. This is acceptable: each peer independently bounds its own local execution depth. Global chain length is bounded by TTL on the wire and by deliver token expiry. If global chain depth tracking is needed, implementations MAY propagate a chain depth field in bounds as a non-normative extension.

---

### 3.10 Chain-Error Marker Convention (cross-cutting; normative)

This section is the canonical home for the chain-error marker convention. §3.4's lost-marker discussion (v1.16, A.1, v1.13) remains in place for the forward-advancement-specific context, but the convention details — kinds, paths, binding authority, body fields, reason registry — live here. New chain-error variants reference this section.

**Two kinds of chain-error markers** are bound under `system/runtime/chain-errors/`:

- **`lost`** — sender / originator side; the chain dispatch was attempted but its outcome was not delivered back to the chain step that originated it (timeout, downstream non-2xx, on-error-delivery failure, merge-mode assembly fault, cap-rejection mirror).
- **`rejected`** — receiver / dispatcher side; the inbound chain dispatch was refused on cap-check BEFORE reaching any handler. Bound by the core dispatcher (cap-rejection happens at the dispatch boundary, not inside any extension handler).

The kind distinction lives in the path; the entity type for both is `system/runtime/chain-error-lost` (unchanged from v1.16 §3.4 to avoid type proliferation; the kind segment carries the distinction).

#### 3.10.1 Path scheme (normative; v1.20)

```
system/runtime/chain-errors/{kind}/{chain_id}/{step_index}/{reason}/{marker_hash}
```

where:
- `{kind}` is `lost` or `rejected` per the two-kinds enumeration above
- `{chain_id}` is the chain identifier from the bounds carrier
- `{step_index}` is the chain-step identifier (per v1.16 §3.4: defined as the original request ID at the step boundary; same definition applies for rejected markers)
- `{reason}` is the error `code` per §3.10.5 (canonical source per failure category), or an impl-specific value (consumers MUST treat unknown reasons as informational). Path-safety per ENTITY-CORE-PROTOCOL.md §1.4.
- `{marker_hash}` (v1.20) is the marker entity's own `content_hash`, encoded in the **ENTITY-CORE-PROTOCOL.md §3.5 invariant-pointer hex form** — `hex.EncodeToString(marker.content_hash.Bytes())` — lowercase, format code byte included, 66 hex chars for ECFv1-SHA-256 (the `00` format-code prefix gives a visible discriminator vs other path segments). This is the **same encoding ENTITY-CORE-PROTOCOL.md §3.5 already pins for signature paths and chain-participating capability paths** (`{peer_id}/system/signature/{content_hash_hex}`, etc., since V7 v7.37); chain-error markers reuse it without introducing a new convention. The ENTITY-CORE-PROTOCOL.md §1.2 line 145 display form `ecfv1-sha256:<hex>` is UI-only and MUST NOT be used on paths. This terminal segment makes each distinct observation land at its own path.

**Per-occurrence addressing (normative; v1.20).** Each distinct marker observation lands at its own path because each distinct observation produces a distinct marker `content_hash` (the body's `timestamp` field at minimum varies per occurrence — see §3.10.6 timestamp-capture discipline). The tree is therefore the event log for chain errors: a flapping target that fails 10 times at 10 wall-clock moments produces 10 distinct marker entities at 10 distinct paths.

**Content-addressed idempotency (normative; v1.20 — corrected from v1.16/v1.19).** Re-binding bytes-identical body (same `content_hash`) at the same path is a `tree:put` no-op by construction. This covers the *redelivery* case (subscription redelivery of the same logical event under the §3.10.6 timestamp-capture discipline produces bytes-identical bodies, so does not multiply markers). It does NOT cover the *re-occurrence* case (10 actual flaps produce 10 markers, intentionally — observers want the full event history).

**Sibling-path coexistence.** Distinct `{reason}` codes at the same `(chain_id, step_index)` coexist as distinct sibling paths under that prefix; each `{reason}` then has zero-or-more `{marker_hash}` children (one per distinct observation).

**Migration note (v1.16/v1.19 → v1.20).** Prior versions terminated the path at `{reason}`, claiming "same-reason re-binding is idempotent (content-addressed)" — this was structurally false because `timestamp` body field (introduced v1.9 A.1) made each occurrence's `content_hash` differ. Three impls would have resolved the contradiction independently at the `tree:put` boundary (last-writer-wins / CAS-create / overwrite) → silent cross-impl divergence on first flap. v1.20 closes the contradiction by adding the terminal `{marker_hash}` segment; the §3.10.1 idempotency claim is now structurally true. Pre-v1.20 markers at the shorter path remain readable as historical state; new binds land at the v1.20 path. ~5-10 LoC per impl — single string concat at the bind site. Per the Stage-4 transport-and-observability proposal §1.2c.

#### 3.10.2 Lost variant (sender / originator side)

Bound by the dispatching component on the sender side when an outbound EXECUTE fails or downstream returns a non-2xx. The `{reason}` segment is the `code` field of the error description per §3.10.5 (one rule for all `{reason}` values; covers internal-engine, response-derived, and transport-level failures uniformly).

Examples by category:

- **Engine-internal failure** — `{reason}` is from EXTENSION-CONTINUATION Appendix A (e.g., `on_error_dispatch_failed`, `merge_value_not_map`)
- **Response-derived failure** (downstream returned non-2xx) — `{reason}` is the response's `result.data.code` verbatim (ENTITY-CORE-PROTOCOL.md §3.3 line 736; e.g., `capability_denied` for 403, `not_found` for 404, codes from the responding handler's appendix for handler-specific failures)
- **Transport-level failure** (no response received) — `{reason}` is from ENTITY-CORE-PROTOCOL.md §6.12 (`recv_timeout`, `connection_broken`, `protocol_error`)

Bound under the sender-side component's component-owned authority (per §3.10.7).

**Cap-rejection mirror.** When the sender's outbound EXECUTE comes back as a 403 carrying `ErrorData.rejected_marker` (i.e., the receiver bound a `rejected` marker per §3.10.3), the sender binds a corresponding `lost` marker with `{reason}` = `capability_denied` (the same code) and includes `rejected_marker_hash` in the body per §3.10.6. Both peers' markers share the same `{reason}` segment; the kind segment (`lost` vs `rejected`) tells you which side observed it.

#### 3.10.3 Rejected variant (receiver / dispatcher side)

Bound by the core dispatcher on the receiver side when an inbound EXECUTE with `Bounds.ChainID` is refused on cap-check. The dispatcher binds the marker BEFORE returning the response.

**Scope (normative):** the rejected variant fires ONLY when the rejected inbound EXECUTE carries a `Bounds.ChainID` (i.e., it is a chain dispatch). Ordinary point-to-point EXECUTE cap-rejections continue to surface via the response only — no rejected marker — because the caller synchronously sees the rejection in the response. Chain-error sinks exist precisely to cover the chain-dispatch case where the originating step does not synchronously observe downstream failures.

Bound under the core dispatcher's `core/chain-errors` component-owned authority (per §3.10.7).

**Reason value (normative).** The `{reason}` segment is the dispatcher's `result.data.code` value verbatim (ENTITY-CORE-PROTOCOL.md §3.3 line 736; same code that goes on the wire response). For cap-rejection, the canonical code is `capability_denied` (ENTITY-CORE-PROTOCOL.md §3.3 line 736's own example; all three reference impls converge to this value as of v1.19 ratification). Impls that emit a different code at the response layer (e.g., the HTTP-family-only `forbidden`) migrate to `capability_denied` as part of the v1.19 spec-text landing.

The `{reason}` IS the `code` — same value, recorded once on the wire and once in the path. The body's `code` field (per §3.10.6) carries the same value denormalized.

#### 3.10.4 Mirror-pointer pattern (SHOULD)

When a rejected-variant marker is bound on the receiver, the receiver SHOULD include the marker's content hash in the EXECUTE_RESPONSE error metadata via the optional `rejected_marker` field on `ErrorData`:

```
ErrorData {
  code:             text             // e.g. 'forbidden'
  message:          text
  rejected_marker:  Hash (omit)      // receiver's marker hash; new in this revision
}
```

The sender then binds a lost-variant marker at `.../lost/{chain_id}/{step_index}/capability_denied/{marker_hash}` (v1.20 path scheme; `{reason}` = `capability_denied` per §3.10.5). The body of the sender's marker SHOULD include `rejected_marker_hash: Hash` (the receiver-side marker hash from the response) so cross-peer audit can walk the pair from either side. **Note (v1.20):** `rejected_marker_hash` body field value equals the terminal `{marker_hash}` segment of the receiver-side marker's path — same value, two surfaces (path tail + body field).

The mirror reference is informational; absence does not invalidate either marker. The `ErrorData.rejected_marker` field is additive — implementations without it produce/consume valid envelopes.

#### 3.10.5 `{reason}` value (normative; v1.19 — collapsed from v1.18 reserved-reasons registry)

**The rule.** `{reason}` is the `code` field of the `system/protocol/error` shape (ENTITY-CORE-PROTOCOL.md §3.3) associated with the failure. Same value across both kinds. Same value across all failure categories. No separate reason vocabulary owned by this section.

For each category of failure, the canonical home for the `code` is per ENTITY-CORE-PROTOCOL.md §3.3 line 742's per-component scoping:

| Failure category | `code` source (canonical home) | Examples |
|---|---|---|
| Handler returns non-2xx response | Handler's own appendix per ENTITY-CORE-PROTOCOL.md §3.3 line 742 (e.g., `EXTENSION-TREE.md` Appendix A for tree handler) | `capability_denied` (ENTITY-CORE-PROTOCOL.md §3.3 line 736), `not_found`, handler-specific codes |
| Continuation engine internal failure | EXTENSION-CONTINUATION Appendix A (this spec; see below) | `on_error_dispatch_failed`, `merge_value_not_map`, `transform_failed`, `chain_construction_invalid` |
| Per-request transport failure | ENTITY-CORE-PROTOCOL.md §6.12 | `recv_timeout`, `connection_broken`, `protocol_error` |
| Connection / handshake failure | ENTITY-CORE-PROTOCOL.md §4.7 | `incompatible_protocol`, `invalid_signature`, etc. (rare in chain-dispatch context; usually surfaces before any chain step exists) |

The marker entity body carries the same `code` value denormalized (per §3.10.6).

**Path-safety (normative).** Codes used as `{reason}` path segments MUST conform to ENTITY-CORE-PROTOCOL.md §1.4 path-segment rules (UTF-8; no null bytes; no empty segments; no embedded `/`). A handler emitting a non-path-safe `code` SHOULD have it sentinel-substituted (`{reason}` = `unspecified_error`) by the dispatching engine with the raw `code` preserved in the marker body's `code` field per §3.10.6.

**Fallback for missing `code`.** Handlers SHOULD always include `code` on error responses per ENTITY-CORE-PROTOCOL.md §3.3. If a handler emits an error response with `status >= 400` but no `code` field, the engine SHOULD bind under `{reason}` = `protocol_error` (ENTITY-CORE-PROTOCOL.md §6.12; the missing-code condition IS a protocol violation by the responding handler, surfaced at the transport-consumer boundary).

**Sibling-path coexistence (v1.16 §3.4 property preserved; v1.20 extended).** Distinct codes at the same `(chain_id, step_index)` coexist as distinct sibling `{reason}` paths. Under v1.20, each `{reason}` subtree then has zero-or-more `{marker_hash}` children — one per distinct observation (each occurrence of the same code at different wall-clock times produces a distinct `{marker_hash}` because the body's `timestamp` field varies; see §3.10.6). The tree IS the event log. Same-content re-binding (same body bytes → same `content_hash` → same path) is genuine `tree:put` no-op idempotency by construction; redelivery under the §3.10.6 timestamp-capture discipline dedupes naturally.

**Deprecation (v1.19).** The v1.13 reason `"forward_dispatch_non2xx"` is deprecated. Impls SHOULD migrate to the code-as-reason rule above. Pre-v1.19 markers under the deprecated reason remain readable as historical state. Subscription patterns matching the deprecated path should migrate to `chain-errors/lost/*/*/*` or to specific Category B canonical codes (e.g., `.../capability_denied`).

**Cap-rejection: canonical 403 code is `capability_denied`** (ENTITY-CORE-PROTOCOL.md §3.3 line 736 example; convergent across all three reference impls as of v1.19 ratification — entity-core-go and entity-core-rust pre-v1.19 conformant; entity-core-py migrates `forbidden` → `capability_denied` as part of the v1.19 single-commit migration).

##### v1.18 reserved-reasons table (DEPRECATED; kept for migration reference only)

The following v1.18 reason values are deprecated by v1.19 and MUST NOT be used in new binds:

| Deprecated `{reason}` | v1.19 replacement | Migration |
|---|---|---|
| `forward_dispatch_non2xx` | `{reason}` = `result.data.code` (handler's actual code) | ~5-10 LoC per impl; single-commit migration; existing markers remain readable |
| `forward_dispatch_403` | `{reason}` = `capability_denied` (canonical 403 code) | Zero — no impl wrote v1.18 reason |
| `cap_denied` | `{reason}` = `capability_denied` (canonical 403 code) | Zero — no impl wrote v1.18 reason |
| `on_error_dispatch_failed` | Same identifier; canonical home moved to EXTENSION-CONTINUATION Appendix A | Zero — same string; sources from canonical home |
| `merge_value_not_map` | Same identifier; canonical home moved to EXTENSION-CONTINUATION Appendix A | Zero — same string; sources from canonical home |

#### 3.10.6 Marker entity body-fields registry (normative)

Marker entities (type `system/runtime/chain-error-lost`, used by both `lost` and `rejected` kinds — the kind distinction lives in the path) carry these reserved body field names. Impls MUST use these names with these types; impls MAY add additional fields for impl-specific context (consumers treat unknown fields as informational, no reactive behavior).

**Reserved across both kinds:**

| Field | Type | Purpose |
|---|---|---|
| `reason` | text | the `{reason}` value matching this marker's path segment (denormalized for in-body inspection without path parsing) |
| `timestamp` | integer | Unix milliseconds when the failure was observed (see timestamp-capture discipline below) |
| `chain_id` | text | for in-body inspection without path parsing |
| `step_index` | text | for in-body inspection without path parsing |

**Reserved on `lost` kind:**

| Field | Type | Purpose |
|---|---|---|
| `target_uri` | text | URI the dispatch was aimed at |
| `target_peer_id` | text | peer ID the dispatch was aimed at |
| `status` | integer | downstream status code (when applicable) |

**Reserved on `rejected` kind:**

| Field | Type | Purpose |
|---|---|---|
| `requesting_peer_id` | text | peer ID that originated the rejected request |
| `attempted_uri` | text | URI the rejected request was aimed at |

**Reserved on `lost` kind WHEN the marker mirrors a peer's rejected marker** (§3.10.4 mirror-pointer pattern):

| Field | Type | Purpose |
|---|---|---|
| `rejected_marker_hash` | Hash | content hash of the receiver-side rejected marker. Stored in the lost marker's body so the cross-peer audit walker can follow the reference without inspecting wire metadata. Body-side companion to wire-side `ErrorData.rejected_marker`. |

Rationale: marker entities are content-addressed. Cross-impl hash convergence on equivalent failures requires the body field names + types to match. Without this registry, three impls would name fields independently and equivalent markers would not cross-impl-hash equal — breaking any cross-peer audit that walks marker pairs by hash.

**Timestamp-capture discipline (normative; v1.20).** The `timestamp` field MUST be captured at failure-origination time (the moment the failure is observed in the engine, transport, or handler), NOT regenerated at the marker-bind site. This makes the v1.20 `{marker_hash}` terminal path segment (§3.10.1) behave correctly:

- **Operational re-occurrence** (target really fails 10 times at 10 different wall-clock moments) — 10 distinct origination timestamps → 10 distinct body bytes → 10 distinct `content_hash` values → 10 distinct paths. Tree records the full event history.
- **Subscription redelivery** (the same logical failure event is redelivered because a downstream notification path retried) — the same origination timestamp is carried through redelivery → bytes-identical body → same `content_hash` → same path → `tree:put` is a genuine no-op. No duplicate markers from redelivery.

Without this discipline, redelivery would multiply markers as if each redelivery were a new occurrence — losing the redelivery-dedup property the v1.20 path scheme provides.

#### 3.10.7 Binding authority is behavioral, not mechanism (normative)

Chain-error markers MUST bind under component-owned authority. The bind authority MUST NOT be derived from the caller's propagated capability.

**Rationale:** a chain dispatch may fail precisely because the caller's propagated cap doesn't authorize the writes the error path needs. If the marker bind used the propagated cap, the cap-rejection variant could not record itself. Component-owned authority guarantees the error surface is reachable regardless of caller cap shape. This generalizes the F11 visibility convention (already landed three-way pre-this-revision; the marker path scheme itself was always pinned by v1.16 §3.4).

**This is a behavioral rule, not a mechanism.** The spec does not mandate the mechanism by which a component realizes this authority. Conformant mechanisms include:

- Explicit cap entity (`internal_scope` grant) seeded at peer construction
- Direct privileged store access at the component / dispatcher boundary (the component's local write path doesn't traverse the cap-check)
- Self-issued cap derived from peer identity at the bind site
- Any equivalent mechanism that ensures the bind authority is not derived from the caller's propagated cap

What the spec requires is the OUTCOME: cap-rejected variants can record themselves; observability bind failures don't depend on caller cap shape; markers are reachable regardless of the conditions being observed. The mechanism is impl-private; impls choose what fits their internal architecture.

For dispatcher-side binds (the `rejected` variant per §3.10.3), the canonical component-owned authority is named `core/chain-errors` — parallel to the per-extension internal_scopes that the subscription / continuation engines own. New cross-cutting core observability concerns SHOULD follow the same pattern (one named owner per concern).

#### 3.10.8 Bind failure visibility (normative)

If a chain-error marker bind itself fails (system error, mis-configured authority, etc.), the failure MUST be surfaced (logged, returned via operational state, or propagated to monitoring) so operators can diagnose stalled chains. The bind MUST NOT silently claim success. F11 visibility convention generalized.

#### 3.10.9 `result.data.code` and `{reason}` are the same value (v1.19; supersedes v1.18 informative)

For all marker kinds and categories, the `{reason}` path segment IS `result.data.code` (or its engine/transport equivalent per §3.10.5). Same vocabulary, same identifier, same value — recorded once on the wire (when applicable) and once in the path.

This was an amendment in v1.19. Prior v1.18 framing treated `{reason}` and `ErrorData.code` as potentially-divergent layers (`code` as HTTP-family analog, `{reason}` as specific cause within that family). That divergence was an artifact of v1.18's invented reserved-reasons registry — it required a separate vocabulary for the registry. With v1.19 collapsing §3.10.5 to the code-as-reason rule, the two are the same value by construction. See the Stage-4 transport-and-observability proposal §1.2b for the absorbed reasoning.

#### 3.10.10 Implementation notes (informative)

- Marker binds are content-addressed; under v1.20 path scheme, distinct observations land at distinct `{marker_hash}` terminal segments under the `(chain_id, step_index, reason)` prefix — flapping targets DO multiply markers within the slot (intentionally: the tree IS the event log), while same-content redelivery dedupes (genuine `tree:put` no-op) per §3.10.1 + §3.10.6 timestamp-capture discipline
- Markers are passive runtime state, not events: subscriptions match per the normal mechanism but the markers themselves have no reactive behavior
- Markers MAY be GC'd after a retention window (impl-defined; v1.9 §3.4 A.1 already authorized this for the lost kind; same applies to rejected)
- Reproducer test for the rejected variant (v1.20 path scheme; v1.19 vocabulary): two-peer setup with restricted caps that do NOT include the chain handler's receive operation; observe inbound dispatch attempt; expect both a receiver-side marker at `.../rejected/{chain_id}/{step_index}/capability_denied/{receiver_marker_hash}` AND a sender-side marker at `.../lost/{chain_id}/{step_index}/capability_denied/{sender_marker_hash}` with `rejected_marker_hash` body field on the sender-side marker equal to `{receiver_marker_hash}` (per §3.10.4 — same value, two surfaces). Both kinds bind under the same `{reason}` segment (`capability_denied`) — the kind segment tells you which side observed it; the `{marker_hash}` segments differ because the two markers' bodies differ (receiver-side has `requesting_peer_id` + `attempted_uri`; sender-side has `target_peer_id` + `target_uri` + `rejected_marker_hash` mirror; both have `timestamp` captured at failure-origination per §3.10.6).

#### 3.10.11 Appendix A reference (v1.19)

See `Appendix A — Continuation engine error codes` at the end of this document for the canonical home of engine-emitted `code` values. Parallel to `EXTENSION-TREE.md` Appendix A's tree-handler codes. Per ENTITY-CORE-PROTOCOL.md §3.3 line 742 per-component scoping.

---

## 4. Dispatch Capability Model

### 4.1 Problem

A continuation at a path can target any handler path. The continuation handler dispatches the EXECUTE. Without scoped authorization, a caller could create a continuation that triggers operations beyond the caller's own grants — privilege escalation through continuation creation.

### 4.2 Solution

The `dispatch_capability` field on continuation entities carries the content hash of a capability token that authorizes the dispatch. The continuation handler uses this capability on the constructed EXECUTE.

**When present**: The continuation handler includes the dispatch capability token in the dispatched EXECUTE's envelope. The dispatch layer verifies it through standard capability verification (ENTITY-CORE-PROTOCOL.md §5.2). The capability MUST grant the continuation's `target` (handler), `operation`, and `resource`.

**When absent**: The advance operation returns an error (`missing_dispatch_capability`). Every dispatching continuation MUST carry explicit dispatch authority. The continuation handler's own grant authorizes managing continuation entities at `system/continuation/*` paths — it is not used for dispatching to target handlers.

- **Caller-created continuations**: The caller provides `dispatch_capability` when invoking `install` (§3.2). The install operation validates the writer's identity is **in** the cap's authority chain via `check_creator_authority` (ENTITY-CORE-PROTOCOL.md §5.5) — the in-chain check of §3.1a — preventing callers from referencing capabilities they don't legitimately hold.
- **System-created continuations** (e.g., inbox handler creates a continuation for delivery infrastructure, subscription engine sets up auto-sync): The creating handler issues a scoped grant — attenuated from its own grant — and invokes `install` with that grant's hash as `dispatch_capability`. The chain roots at the local peer identity *and* the handler is in it as granter, so the §3.1a in-chain check passes by construction (the local case where in-chain ⇔ rooted-at-installer).
- **Cross-peer / remote-target continuations**: When a continuation step's `target` resolves to a remote peer B's handler, the `dispatch_capability`'s authority chain MUST satisfy **both**:
  - **(i) rooted at B's conferred authority** — the chain terminates at a root **B** (the verifying peer at advance) recognizes: the authority B conferred on the installer for the targeted operation/resource. This makes B's advance-time `VerifyChain` / dispatch verification (ENTITY-CORE-PROTOCOL.md §5.2) succeed.
  - **(ii) installer in-chain as the re-attenuation (leaf) granter** — the installer attenuates the B-conferred authority into the narrowly-scoped `dispatch_capability`, appearing as a `granter` in the chain, so the §3.1a in-chain install check passes.
  - **(iii) granted to the dispatching host peer (normative — v1.11)** — the `dispatch_capability`'s **`grantee` MUST be the identity of the peer that performs the deferred dispatch**: the continuation's host peer, which is the author of the dispatched EXECUTE. A continuation is a suspended execution; its host peer's continuation handler reconstructs and signs the dispatched EXECUTE with that peer's keypair — the only key it holds. B's standard `grantee == EXECUTE author` check (ENTITY-CORE-PROTOCOL.md §5.2) therefore requires `grantee` = the host peer. This is **determined by the model, not a configuration choice**: there is no separate handler-grant or delegated wielder — the peer that hosts the continuation is the one that dispatches it. Granting to the *installer* (the self-wielded form) is wrong whenever the installer identity is not the host peer's identity — effectively every cross-peer continuation, since the installer is the caller/admin that set the continuation up while the dispatcher is structurally the host peer. Authoring as the host peer is **not** a ENTITY-CORE-PROTOCOL.md §6.8 escalation: authority is bounded by the scoped, B-rooted, installer-attenuated chain ((i)+(ii)), not by the signing identity.

  Rooting the chain *at the installer* is the local sufficient condition only and is **wrong** for the cross-peer case: a chain rooted at the installer does not terminate at an authority B recognizes, so B's `VerifyChain` fails at advance (this is the original dispatch-capability collapse). The correct shape — **rooted at B's conferred authority, installer in-chain as the re-attenuation leaf granter, granted to the dispatching host peer (the EXECUTE author)** — is the continuation analog of EXTENSION-SUBSCRIPTION's B-rooted caller capability (§1.2/§1.3; the subscription caller cap is likewise B-rooted with its grantee being the wielding caller, validated against the EXECUTE author via the standard ENTITY-CORE-PROTOCOL.md §5.2 dispatch chain), reusing the implemented coherent-capability-authority model (`proposals/implemented/PROPOSAL-COHERENT-CAPABILITY-AUTHORITY.md`, test-vector row 11: "chain-rooted at A with installer C as leaf granter"). Where the remote target is statically determinable at install (no `target_extract`, and the static `target` resolves to a remote peer) implementations SHOULD reject a non-conformant chain at install; where `target` is resolved only at advance (`target_extract` present — see the "Capability interaction" note in §2.2, the cap is checked against the final EXECUTE) the non-conformant chain fails at advance via B's `VerifyChain` (current behavior) and this case is the required-shape contract rather than a new install check.

  **Chain transport (normative).** At advance/dispatch the **full** authority chain — every capability entity from the leaf `dispatch_capability` up to the root B recognizes, **and the signature entity for every link in that chain** — MUST be present in the dispatched EXECUTE envelope's `included` map. §4.3's general rule guarantees only the *leaf* cap, because parent/granter caps (and all per-link signature entities) are referenced from *within* the cap entities, not from the EXECUTE's `data` fields; cross-peer dispatch MUST additionally bundle the transitive chain and its signatures. The installer already holds them (§3.2 step 5 collected and persisted the full chain via `collect_authority_chain`); dispatch re-collects from there and includes all entities. The safe default is to bundle the whole chain — content-addressing makes over-inclusion free (any entity B already holds dedups by hash on ingest), eliminating the "B GC'd a parent → `VerifyChain` fails" failure mode at zero correctness cost. A missing per-link **signature** is a transport-completeness failure of this rule (surfaced at advance as B's `VerifyChain` "no signature found"), **not** an install-time `chain_unreachable` (§8.1) — `chain_unreachable` is entity-*resolvability* of the chain's cap links only; signature *verification* is advance-time at B (ENTITY-CORE-PROTOCOL.md §5.2). Implementations MAY additionally fail-fast at install if a collected link's signature is absent (defense-in-depth SHOULD), but that does not redefine `chain_unreachable`.

  **Root-cap regime — re-attenuation is plain V7, including for role-derived roots (normative clarification — v1.12 / Amendment 4).** The re-attenuation in (ii) is a **standard ENTITY-CORE-PROTOCOL.md §5.6 capability attenuation** of a capability the installer legitimately holds — *not* a role-extension operation, regardless of what kind of cap (i)'s root is. Two consequences, both load-bearing:

  - **Revocation of the root cascades to the `dispatch_capability` for free, via ENTITY-CORE-PROTOCOL.md §5.1 chain-root revocation.** Because the chain *roots at* the (i) authority, B's advance-time `is_revoked` (ENTITY-CORE-PROTOCOL.md §5.1, walk-to-root then check the root's tree binding) revokes the `dispatch_capability` the instant that root's binding is removed at B. When (i) is a **role-derived cap** (EXTENSION-ROLE v2.0 PR-1: role-derived caps are V7 root caps, `parent:null`, `granter=B`), ROLE exclusion / `unassign` / `re-derive` revoke by *deleting the role-derived cap's tree binding* (EXTENSION-ROLE §215/§239 — "V7's `is_revoked` operates on the cap path"); that deletion transitively revokes every `dispatch_capability` rooted at it. No role-specific tracking of the `dispatch_capability` is required or possible — chain-root revocation is exactly the property ROLE itself relies on. `re-derive` (root replaced by a new token hash) correctly revokes `dispatch_capability`s rooted at the *old* hash; the continuation must be re-installed against the new root — fail-safe, not a leak.
  - **EXTENSION-ROLE PR-8.2 / `system/role:delegate` is OUT OF SCOPE here.** That mechanism governs *member-to-member delegation of role authority* — making **another peer act in the role** with role-tracked, re-derivable authority (EXTENSION-ROLE.md §5.6, §433). A cross-peer continuation `dispatch_capability` is categorically different: a single scoped one-shot V7 cap for one deferred mechanical EXECUTE, wielded by the continuation's host peer, that confers no role membership, nothing re-derivable, and **dies with its root** (above). Routing it through `system/role:delegate` would be a category error (wrong entity shape, wrong lifecycle). Therefore raw V7 re-attenuation (the SDK's `MintReattenuated`-class helper, C-3) is conformant for **any** B-rooted root — connect-grant *or* role-derived; they behave identically under this case. An implementation MUST NOT require `system/role:delegate` for a role-derived root, and MUST NOT treat raw re-attenuation of a role-derived root as a role-delegation-policy bypass.

**The capabilities in a cross-peer continuation** — an **instance of the general three-slot model** in ENTITY-CORE-PROTOCOL.md §5.2 ("Cross-peer capability provenance — the three slots"); mirrors EXTENSION-SUBSCRIPTION.md §1.2/§1.3. The dispatch_capability fills: **root** = B's conferred authority; **grantee** = the dispatching host peer (EXECUTE author); **in-chain granter** = the installer (re-attenuation leaf). Read the V7 note first if the root/grantee/granter distinction is unclear — its omission here is exactly what caused the v1.9→v1.11 corrections:

| Capability | Carried in | Authorizes | Rooted at | Installer's place (granter) | Granted to (grantee) | Validated by |
|---|---|---|---|---|---|---|
| Dispatch capability | continuation `dispatch_capability` (+ full chain in envelope `included`) | the deferred EXECUTE to B's handler | **B's conferred authority** | **in-chain as the re-attenuation leaf granter** | **the dispatching host peer (= the EXECUTE author)** | install-time in-chain check (§3.1a, §3.2 step 4) + advance-time `VerifyChain` at B incl. `grantee == author` (ENTITY-CORE-PROTOCOL.md §5.2) |
| (if result returns) result `deliver_token` | result EXECUTE `deliver_token` | delivery back to the installer's inbox | the installer | (is the root) | the result-delivering peer (B's engine) | dispatch chain at the installer |

The dispatch capability is the continuation analog of subscription's B-rooted **caller capability**; the result deliver_token is the analog of the A-rooted **subscription deliver_token**. There is no paradox to reconcile: the chain is B-rooted *and* the installer is in it as the leaf granter — §3.1a's "writer anywhere in the chain" is satisfied without rooting at the writer.

This ensures every continuation's dispatch authority is explicit, inspectable, scoped at creation time, and authorized — the writer must legitimately hold the cap they embed. There is no implicit fallback to handler authority (see ENTITY-CORE-PROTOCOL.md §6.8, no silent escalation).

### 4.3 Capability Lifecycle

The dispatch capability token is created by the caller when setting up the continuation. The caller MUST ensure the capability entity is in the local content store before or at the time the continuation is created. For cross-peer creation via EXECUTE, the capability entity MUST be included in the EXECUTE envelope's `included` map per the general rule in ENTITY-CORE-PROTOCOL.md §3.1 / §3.2 (all entities referenced by hash from the EXECUTE's `data` fields must be present in `included`) — the receiving peer ingests it during normal envelope processing. For local creation, the caller MUST `content_store.put` the capability entity explicitly. The token's TTL bounds the continuation's effective dispatch window — if the token expires, the continuation cannot dispatch and advancement fails.

For **cross-peer dispatch to a remote target**, envelope inclusion is *not* limited to the leaf `dispatch_capability`: the **full** authority chain up to the root the target peer recognizes MUST travel in the dispatched EXECUTE's `included` map — see §4.2 ("Chain transport"). The general ENTITY-CORE-PROTOCOL.md §3.1 / §3.2 rule places only the leaf cap (it alone is referenced from the EXECUTE `data` fields); the transitive parent/granter chain is referenced from *within* the cap entities and MUST be bundled explicitly by the dispatching peer.

---

## 5. Suspended Continuation Lifecycle

### 5.1 Creation

Suspended continuations are created by:

- **Dispatch layer**: Via registered suspension handler (§3.8), on TTL exhaustion, budget exhaustion, or chain depth exceeded. If no suspension handler is registered, suspension does not occur.
- **Continuation handler**: On transient dispatch failure during advancement (§3.3). The handler creates a suspended entity capturing the assembled EXECUTE that could not complete.
- **Handlers**: On unrecoverable errors that should be preserved for later resolution. The handler creates a suspended continuation and returns an error response. The suspended entity captures the handler's target, operation, params, and failure reason.

### 5.2 Visibility

Suspended continuations are tree entities at `system/continuation/suspended/*`. They are:

- **Listable**: `GET system/continuation/suspended/` returns all suspended processes
- **Readable**: `GET system/continuation/suspended/{id}` returns the full suspended state
- **Capability-gated**: Access to `system/continuation/suspended/` SHOULD be restricted to peer administrators and authorized operators

### 5.3 Resume

The `resume` operation on the continuation handler (§3.6) reconstructs the EXECUTE from suspended state and dispatches with fresh bounds. The `resolution` field in the resume request allows the operator to provide additional context (e.g., conflict resolution choice for a merge conflict).

### 5.4 Abandon

The `abandon` operation deletes the suspended entity after verifying its type is `system/continuation/suspended`. The operation is lost. This is equivalent to killing a stopped process.

### 5.5 Automatic Cleanup

Implementations SHOULD periodically clean up suspended continuations that exceed a configurable age limit. Cleanup policy is implementation-defined.

---

## 6. Security Considerations

### 6.1 Privilege Escalation via Continuation

A caller with tree write access to a continuation path could create continuations targeting sensitive handlers. Mitigation: the `dispatch_capability` field is required and provides explicit, scoped authorization for the dispatch target. A continuation without `dispatch_capability` cannot dispatch (advance fails with `missing_dispatch_capability`). The continuation handler's own grant is restricted to managing continuation entities — it does not authorize dispatch to target handlers.

### 6.2 Resource Exhaustion via Chains

A continuation chain with dynamic construction could consume unbounded resources. The dispatch layer tracks chain depth and suspends when the implementation-defined maximum is exceeded (§3.8). The `chain_id` field correlates all operations in a chain. Chain depth is a monotonically increasing counter, not reset by fresh bounds.

### 6.3 Orphaned Continuations

A standing continuation (`remaining_executions: null`) at a path with no active source delivering to it is an orphan — it will never advance. Orphaned continuations are not harmful but waste tree space. Operators SHOULD monitor for orphaned continuations using type queries on `system/continuation` entities.

### 6.5 Privacy + cross-peer observability

Per `GUIDE-INSPECTABILITY.md` v1.2 §9 #4: `system/continuation/**` entities (forward, join, suspended) and `system/runtime/chain-errors/**` markers are **capability-controlled** — bodies carry dispatch capability hash references, captured params (which may transitively reference sensitive entities), target + operation call shapes, and chain causality. The `install-result` operational echo is **public**.

Per §9 #7: **local-namespace** for both `system/continuation/**` and `system/runtime/chain-errors/**`. These path families are per-peer operational state and MUST NOT propagate via subscriptions or revision sync. Cross-peer observation requires explicit operator-grant on `system/inspect/*` (per `GUIDE-INSPECTABILITY.md` §8). Subscription handlers SHOULD refuse subscriptions on these prefixes unless the caller's scope explicitly enumerates a narrower path with operator-class authority.

This subsection is the canonical declaration that resolves the cross-peer propagation question for `system/runtime/chain-errors/*` markers raised in the inspectability cycle's closing memo. Other extensions inheriting the chain-error marker convention from §3.10 inherit this local-namespace declaration; see EXTENSION-SUBSCRIPTION.md §7.4, EXTENSION-INBOX.md §11, EXTENSION-REVISION.md §12 for the parallel extension-side declarations.

Per the privacy-and-cross-peer-observability audit §2.5.

### 6.4 Suspended Continuation Accumulation

Without cleanup, suspended continuations accumulate at `system/continuation/suspended/*`. Implementations SHOULD enforce a maximum count or age limit and clean up expired suspended entities.

---

## 7. Constants

### 7.1 Status Codes

| Code | Meaning |
|------|---------|
| 409 | Conflict — `hash_mismatch` (join slot CAS) or `slot_already_filled` (duplicate slot delivery) |
| 429 | Bounds exceeded (with optional `suspended_at` field) |

### 7.2 Standard Operations

| Operation | Handler | Description |
|-----------|---------|-------------|
| `advance` | `system/continuation` | Advance a continuation with a result |
| `resume` | `system/continuation` | Resume a suspended continuation |
| `abandon` | `system/continuation` | Delete a suspended continuation |

### 7.3 Continuation Types

| Type | Purpose | Created by |
|------|---------|------------|
| `system/continuation` | Forward dispatch on result arrival | Caller |
| `system/continuation/join` | Fan-in accumulation | Caller |
| `system/continuation/suspended` | Saved execution state | System (dispatch layer, continuation handler, or handler) |
| `system/continuation/transform` | Structural navigation spec + dynamic EXECUTE field extraction + bounded field ops | Embedded in continuation |
| `system/continuation/transform-op` | One bounded field operation within a transform's `transform_ops` | Embedded in transform |
| `system/continuation/advance-request` | Input to advance operation | Caller (inbox handler, direct user) |
| `system/continuation/resume-request` | Input to resume operation | Operator |
| `system/continuation/abandon-request` | Input to abandon operation | Operator |

---

## 8. Conformance

**Conformance timing — read before §8.1 (the cross-peer G2 MUST is gated, not a flag-day).** v1.9 is **not** a core-protocol change: it adds no new core primitive. Every mechanism it uses — capability authority chains and `check_creator_authority` (ENTITY-CORE-PROTOCOL.md §5.5), `VerifyChain` (ENTITY-CORE-PROTOCOL.md §5.2), the envelope `included` map (ENTITY-CORE-PROTOCOL.md §3.1/§3.2), content-addressed dedup — is already normative and already shipped; v1.9 only specifies how a continuation *composes* them, mirroring the implemented EXTENSION-SUBSCRIPTION.md §1.2/§1.3 pattern.

The cross-peer / remote-target §8.1 requirement (§4.2 case 3 — B-rooted re-attenuated `dispatch_capability`, full chain bundled) is the required **shape** *when an implementation performs cross-peer continuation dispatch*. It is **gated and sequenced, not a flag-day**:

- An implementation that does **not yet** perform conformant cross-peer continuation dispatch is **not non-conformant** for lacking it. The specified interim behavior is **advance-time failure** (the dispatched cross-peer EXECUTE fails B's `VerifyChain`) — the documented "current behavior," not a regression. Local and system-created continuations are **wholly unaffected** (the chain resolves from the install-persisted store; in-chain ⇔ rooted-at-installer there).
- The **chain-transport** part (§4.3 / §8.1 — the full authority chain travels in the dispatched envelope's `included`) is **purely additive**: over-inclusion is content-addressed-free, no regression, safe to land immediately and independently. A properly B-rooted dispatch cap then verifies at the target today.
- The proof obligation is **workbench-owned and sequenced**: spec → C-3 SDK re-attenuation mint helper (solo, unit-testable) → per-impl dispatch threading → workbench-go V-1 / Phase C end-to-end proof. Flipping the strict cross-peer authorization in lockstep with that sequence is what avoids the all-callers-break-at-once failure mode; doing it before C-3 exists cross-impl is premature.

A conformance suite SHOULD assert the install-time and additive MUSTs (§8.1) now; it SHOULD **not** encode the cross-peer §8.1 G2 MUST as a present-tense check — doing so mis-flags every implementation that is correctly following the sequence above. (Arch resolution; see the continuation cross-peer-and-transform-ops proposal §8 + the V-1 sequencing.)

### 8.1 MUST Implement

- Continuation handler at `system/continuation` with `install`, `advance`, `resume`, and `abandon` operations (§3.1)
- `install` operation as the proper create path for `system/continuation` and `system/continuation/join` entities (§3.2, CT1)
- Install handler MUST verify the embedded `dispatch_capability` via `check_creator_authority` (ENTITY-CORE-PROTOCOL.md §5.5) as an **in-chain** check — the writer's identity (`ctx.execute.data.author`) MUST appear as a granter **anywhere in** the collected authority chain (§3.1a, §3.2 step 4); reject with 403 `embedded_cap_unauthorized` when it does not (CT2). This is *not* a chain-*root* check; the prior "chain root against author" wording was correct only for the local case and is reconciled in §3.1a — implementing it as a root check breaks cross-peer continuations
- Install handler MUST persist the embedded `dispatch_capability` and its full authority chain to local content store after the in-chain check succeeds (§3.2 step 5)
- `transform_ops` (when transforms are supported): MUST implement the agreed op set with total/pure/bounded semantics, apply ops after `extract`/`select` and before the `*_extract` fields, and reject an unrecognized `op` at install (fail-closed) with `400 unknown_transform_op` — never silently skip (§2.2)
- Cross-peer / remote-target dispatch (§4.2 case 3): the `dispatch_capability` chain MUST be rooted at the target peer's conferred authority, with the installer in-chain as the re-attenuation leaf granter (a chain rooted at the installer is non-conformant for a remote target), **and granted to the dispatching host peer — the `grantee` MUST be the identity that authors the dispatched EXECUTE (the continuation's host peer), so the target's `grantee == author` check (ENTITY-CORE-PROTOCOL.md §5.2) passes; granting to the installer (self-wielded) is non-conformant whenever the installer is not the host peer**; and the **full** authority chain (leaf → target-recognized root) MUST travel in the dispatched envelope's `included` map (the leaf-only reading of §4.3 is insufficient cross-peer)
- Install handler MUST surface 404 `chain_unreachable` when any link in the cap's authority chain is missing from envelope `included` and local content store
- Advancement algorithm: resolve continuation at path, dispatch based on type (§3.3)
- Forward continuation advancement algorithm (§3.4)
- `remaining_executions` lifecycle — CAS decrement on successful advancement, do not advance when exhausted (§3.4)
- Exhaustion check — entity at `remaining_executions: 0` MUST NOT advance (§3.4)
- CAS (`expected_hash`) on `remaining_executions` decrement for forward continuations (§3.4)
- No-op when no continuation entity exists at path (§3.3 step 5)
- Type verification on abandon — reject if entity is not `system/continuation/suspended` (§3.8)

### 8.2 SHOULD Implement

- Structural transform processing (extract/select) (§3.6)
- Dynamic EXECUTE field extraction (`resource_extract`, `target_extract`, `operation_extract`) on continuation transforms (§2.2, §3.6)
- `transform_ops` encoding in the SDK and total/analyzable treatment by the static analyzer (§2.2, T-3)
- Cross-impl SDKs SHOULD expose a re-attenuation mint helper (B-rooted `dispatch_capability`, installer as leaf granter, **`grantee` = an explicit dispatching-peer parameter — NOT self-wielded to the installer**) and a dispatch chain-walk + bundle helper (full chain → envelope `included`) (§4.2 case 3, C-3)
- Lost-error markers use the per-occurrence path `system/runtime/chain-errors/lost/{chain_id}/{step_index}/{reason}/{marker_hash}` (v1.20 path scheme; v1.16 per-reason subsegment + v1.20 terminal hash segment) — distinct reasons coexist as sibling `{reason}` subtrees; distinct observations within a reason coexist as sibling `{marker_hash}` children; same-content re-binding is genuine content-addressed no-op (§3.10.1)
- Chain-error marker `{reason}` segment IS `result.data.code` (v1.19 §3.10.5; same value across all kinds and categories; no separate reason vocabulary)
- Lost-error marker on `on_error` delivery failure — bind an informational marker at `.../{chain_id}/{step_index}/on_error_dispatch_failed/{marker_hash}` with `code: "on_error_dispatch_failed"` (Appendix A); no reactive behavior; MAY GC after a retention window (§3.4 + §3.10 + Appendix A; v1.20 path scheme)
- Lost-error marker on no-`on_error` forward dispatch non-2xx — bind an informational marker at `.../{chain_id}/{step_index}/{result.data.code}/{marker_hash}` with `code` = the response's `result.data.code` (e.g., `capability_denied` for 403); same type as the A.1 marker; trigger range status ≥ 400; trigger timing every occurrence (each occurrence binds its own marker at a distinct `{marker_hash}` segment per §3.10.1 + §3.10.6 timestamp-capture discipline); no reactive behavior; MAY GC after a retention window. **v1.13 reason `"forward_dispatch_non2xx"` deprecated as of v1.19; v1.20 path scheme — see §3.4 deprecation note + §3.10.5 + §3.10.1.**
- Lost-error marker on merge-mode non-map value — when `result_merge: true` meets a non-map post-transform value, bind an informational marker at `.../{chain_id}/{step_index}/merge_value_not_map/{marker_hash}` with `code: "merge_value_not_map"` (Appendix A); dispatch proceeds with static-only params; no reactive behavior; MAY GC after a retention window (§3.4, v1.16; Appendix A canonical home as of v1.19; v1.20 path scheme)
- Chain-error rejected variant (cap-rejection) — receiver-side dispatcher binds marker at `system/runtime/chain-errors/rejected/{chain_id}/{step_index}/capability_denied/{receiver_marker_hash}` (the canonical 403 code per ENTITY-CORE-PROTOCOL.md §3.3 line 736); sender side mirrors at `lost/{chain_id}/{step_index}/capability_denied/{sender_marker_hash}` with `rejected_marker_hash` body field per §3.10.4 (value equals `{receiver_marker_hash}` — same value, two surfaces). Per §3.10.3; v1.19 vocabulary; v1.20 path scheme; companion to `ErrorData.rejected_marker` wire field
- `resolve_or_default` fallback to static values when extraction path is absent or navigation fails (§3.6)
- `resource_extract` string-to-targets wrapping (§3.6)
- Transform error handling — pass through original result on navigation failure (§2.2)
- Join continuation advancement (§3.5)
- Join slot exactly-once — reject duplicate delivery to already-filled slot with 409 (§3.5)
- Join CAS via `expected_hash` for multi-process safety (§2.3)
- Suspension handler registration for bounds exhaustion and chain depth (§3.9)
- Chain depth counter in execution context, suspension at configured maximum (§3.9)
- Dispatch failure handling — suspend on transient, error on permanent (§3.4)
- Dispatch capability validation on continuation dispatch (§4)
- Suspended continuation creation on dispatch failure (§3.4, §5.1)
- Suspended continuation cleanup policy (§5.5)
- Capability-gated access to `system/continuation/suspended/` (§5.2)

### 8.3 MAY Implement

- Custom suspended continuation age limits (§5.5)
- Orphaned continuation detection (§6.3)
- Process-local locking optimization for joins (§2.3)

### 8.4 Implementation-Defined

- Suspended continuation ID generation scheme
- Suspended continuation maximum age for cleanup
- Chain depth maximum before forced suspension (SHOULD default to 64)
- Suspension handler registration mechanism
- Cleanup policy for exhausted continuations (`remaining_executions: 0`)
- Internal deliver token generation for same-peer chain dispatch (§3.6 step 4)

---

## Appendix A — Continuation engine error codes (normative; v1.19)

The continuation engine emits `system/protocol/error` shapes (V7 §3.3) when chain-machinery-internal failures occur. Per V7 §3.3 line 742's per-component scoping, this appendix is the canonical home for engine-emitted `code` values — parallel to `EXTENSION-TREE.md` Appendix A for tree-handler codes.

These codes appear as:

- `result.data.code` on engine-synthesized error descriptions (when the engine binds a chain-error marker for an internal failure per §3.10)
- `{reason}` path segment on the marker bound under `system/runtime/chain-errors/lost/{chain_id}/{step_index}/{reason}/{marker_hash}` (v1.20 path scheme; the `code` and `{reason}` values are the same per §3.10.5; the terminal `{marker_hash}` segment per §3.10.1)

### A.1 Reserved codes

| `code` | `status` | Trigger |
|---|---|---|
| `on_error_dispatch_failed` | 500 | An `on_error` deliver-target dispatch failed (transient or permanent). Per v1.9 §3.4 A.1. |
| `merge_value_not_map` | 400 | `result_merge: true` met a non-map post-transform value at chain assembly. Per v1.16 §3.4. |
| `transform_failed` | 400 | A continuation-transform vocabulary evaluation produced an error (e.g., `extract` against a non-map, `transform_ops` evaluation hit an unknown op). Per §2.2 transform contract. |
| `chain_construction_invalid` | 400 | The continuation entity at install time was malformed (missing required fields, type discrimination failed, `dispatch_capability` chain-root check failed). Per §3.2. |

Additional codes MAY be reserved by future spec amendments to this appendix. Impls MAY emit non-reserved codes for impl-specific engine failures; consumers MUST treat unknown codes as informational (no reactive behavior).

### A.2 Path-safety

All codes in §A.1 conform to V7 §1.4 path-segment rules and are safe for use as `{reason}` path segments per §3.10.5. New codes reserved by future amendments MUST also be path-safe.

### A.3 Cross-reference

- Per-request transport error codes (sibling component) — V7 §6.12
- Connection / handshake error codes (sibling component) — V7 §4.7
- Tree handler error codes (handler-component precedent) — `EXTENSION-TREE.md` Appendix A
- Origin proposal — the Stage-4 transport-and-observability proposal §1.2b
