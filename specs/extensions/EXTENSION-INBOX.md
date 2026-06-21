# Inbox Extension — Normative Specification

**Version**: 5.9

**Status**: Active
**Depends**: ENTITY-CORE-PROTOCOL.md (v7.8+)
**Supersedes**: EXTENSION-CALLBACK.md v4.0
**Proposal**: PROPOSAL-CONTINUATION-SPEC-AMENDMENT.md (A1), PROPOSAL-DELIVERY-AND-INBOX-RENAME.md (D3, D4, D7, D8)

---

> **Path notation.** Paths in this document use peer-relative notation (without leading `/{peer_id}/`). All peer-relative paths resolve to the local peer's namespace: `system/tree` means `/{local_peer_id}/system/tree`. Every path in the entity tree is absolute at rest — rooted at a peer identity. See ENTITY-CORE-PROTOCOL.md §1.4 for the path model. Cross-peer examples use absolute paths with explicit peer identities.

## 1. Overview

The inbox extension enables asynchronous result delivery. When an EXECUTE includes a `deliver_to` field (ENTITY-CORE-PROTOCOL.md §3.2), the handler delivers results as an authenticated EXECUTE to the delivery URI rather than returning them on the connection.

The `system/delivery-spec` type referenced by this extension is defined in ENTITY-CORE-PROTOCOL.md §3.11.

This extension defines transport logic only:

- Inbox delivery and notification types
- `deliver_token` extension field on EXECUTE
- Inbox handler (receive)
- Delivery construction algorithm
- Deliver token scoping
- Write-ahead inbox processing

This extension is independent of EXTENSION-CONTINUATION.md. When EXTENSION-CONTINUATION is also installed, the inbox handler delegates to the continuation handler's `advance` operation for paths that have a continuation entity. When EXTENSION-CONTINUATION is not installed, the inbox handler stores deliveries in the tree.

### 1.1 Async Request-Response Flow

```
Caller                                      Handler Peer
  |                                           |
  |  1. Write continuation entity at          |
  |     inbox path (optional)                 |
  |                                           |
  |  2. Create deliver token:                 |
  |     grants: deliver to inbox URI          |
  |     grantee: handler peer                 |
  |                                           |
  +- 3. EXECUTE {deliver_to, deliver_token} ->|
  |                                           |
  |<-- 4. EXECUTE_RESPONSE {status: 202} -----|  Acknowledged
  |                                           |
  |  (Connection may close)                   +- 5. Process request
  |                                           |
  |<-- 6. EXECUTE {                           |  Delivery
  |       uri: inbox_uri,                     |
  |       operation: "receive",               |
  |       params: {                           |
  |         original_request_id,              |
  |         status, result                    |
  |       }                                   |
  |      } -----------------------------------|
  |                                           |
  +- 7. Inbox handler:                        |
  |     a. Store in tree (write-ahead)        |
  |     b. If continuation installed and      |
  |        continuation entity exists:        |
  |        delegate to continuation handler   |
  |     c. Clean up stored delivery on        |
  |        successful advancement             |
  |     d. Otherwise: message stays in tree   |
  |                                           |
  +- 8. EXECUTE_RESPONSE {status: 200} ------>|  Delivery acknowledged
  |                                           |
```

Step 7 is the key integration point. The inbox handler uses write-ahead processing: store first, process second, clean up on success. If EXTENSION-CONTINUATION is installed and a continuation entity exists at the inbox path, the handler delegates to the continuation handler's `advance` operation. If no continuation entity exists or EXTENSION-CONTINUATION is not installed, the delivery stays in the tree (the inbox is a persistent mailbox).

---

## 2. Type Definitions

### 2.1 Inbox Delivery

```
system/protocol/inbox/delivery := {
  fields: {
    original_request_id: {type_ref: "primitive/string"}
    status:              {type_ref: "primitive/uint"}
    result:              {type_ref: "core/entity"}
  }
}
```

The `result` field is typed as `core/entity` (the materialized form `{type, data, content_hash}`, see ENTITY-NATIVE-TYPE-SYSTEM.md §8.1) because the delivery mechanism is type-agnostic — it carries whatever the handler produced. The value is a full inline entity, not a raw data payload. See §4.1.

### 2.2 Inbox Notification

```
system/protocol/inbox/notification := {
  fields: {
    subscription_id: {type_ref: "primitive/string"}
    event:           {type_ref: "primitive/string"}
    uri:             {type_ref: "system/tree/path"}
    hash:            {type_ref: "system/hash", optional: true}
    previous_hash:   {type_ref: "system/hash", optional: true}
  }
}
```

The `uri` field is the bare tree path (e.g., `system/foo/bar`), not a full entity URI. Receivers needing a URI form construct it from the delivery envelope's peer identity.

### 2.3 Extension Field on EXECUTE

```
deliver_token: {type_ref: "system/hash", optional: true}
```

**Extends**: `system/protocol/execute`

When present, the value is the content hash of a `system/capability/token` entity authorizing the handler peer to deliver to the inbox URI. The token entity MUST be in the envelope's `included` map. `deliver_token` MUST only be present when `deliver_to` is also present. When `deliver_to` is present without `deliver_token`, the handler MUST reject with status 400 (`missing_deliver_token`).

**Revocation.** Revocation of `deliver_token` is handled automatically by dispatch-layer verification per ENTITY-CORE-PROTOCOL.md §5.2 step 4 (integrated `is_revoked` check, per PROPOSAL-COMPUTE-AMENDMENTS-V3 C2). A revoked `deliver_token` fails verification when the handler attempts to re-dispatch the delivery — no inbox-specific revocation check is required.

---

## 3. Handlers

### 3.1 Inbox Handler

```
system/handler := {
  pattern:    "system/inbox"
  name:       "inbox"
  operations: {
    receive: {input_type: "primitive/any"}
  }
}
```

Handler entity at pattern path `system/inbox`. Dispatch resolves any URI under `system/inbox/` to this handler via longest-prefix match (ENTITY-CORE-PROTOCOL.md §6.6).

Single `receive` operation accepts any typed entity. The entity's type carries the semantic information — `system/protocol/inbox/delivery` for async operation results, `system/protocol/inbox/notification` for subscription events, `system/protocol/execute` for raw deferred dispatch, or any domain-specific message type.

### 3.2 Write-Ahead Processing

On receiving a `receive` EXECUTE at an inbox path:

1. Validate capability through standard verification chain (ENTITY-CORE-PROTOCOL.md §5.2)
2. Store delivery in tree (write-ahead)
3. Check for continuation entity at inbox path
4. If EXTENSION-CONTINUATION is installed and a continuation entity exists at the path: delegate to continuation handler (§3.3)
5. If advancement succeeds: clean up stored delivery
6. If no continuation entity exists or EXTENSION-CONTINUATION is not installed: delivery stays in tree (persistent mailbox)
7. Return EXECUTE_RESPONSE — status 200 on success, appropriate error on failure

```
receive(inbox_path, message_entity, ctx):
  ; 0. Capability validation (standard verification chain, §3.2 step 1)
  ;    Already performed by dispatch layer before handler invocation.
  ;    The deliver token on the EXECUTE grants receive on this inbox path.

  ; 1. Store in tree (write-ahead)
  store_path = inbox_path + "/" + generate_id()
  entity_tree.put(store_path, message_entity)

  ; 2. Check for continuation
  continuation = entity_tree.get(inbox_path)
  if continuation is not null and continuation.type in ["system/continuation", "system/continuation/join"]:
    ; 3. Attempt advancement
    ; Extract result and status based on message type.
    ; For inbox/delivery: use .result and .status fields (result may be null).
    ; For other types: use entire .data as result, status 200.
    if message_entity.type == "system/protocol/inbox/delivery":
      extracted_result = message_entity.data.result    ; may be null — that's valid
      extracted_status = message_entity.data.status or 200
    else:
      extracted_result = message_entity.data
      extracted_status = 200

    result = dispatch({
      uri:       "system/continuation"
      operation: "advance"
      resource:  {targets: [inbox_path]}
      params:    {
        type: "system/continuation/advance-request"
        data: {
          result: extracted_result
          status: extracted_status
        }
      }
    })
    if result.advanced:
      ; 4. Clean up stored delivery on success
      entity_tree.put(store_path, null)
    ; else: leave stored — continuation exhausted, error, etc.
  ; else: no continuation, message stays in tree (inbox)

  return {status: 200}
```

Crash at any point leaves consistent state:
- Crash after step 1: message stored, no advancement. Retry-safe.
- Crash after step 3: message stored, continuation advanced. Cleanup on recovery.
- Crash after step 4: complete. Nothing to recover.

### 3.3 Continuation Behavior

When EXTENSION-CONTINUATION is installed, the inbox handler checks for continuation entities before falling back to pure storage.

**Three scenarios**:

| Continuation installed | Entity at path | Behavior |
|----------------------|----------------|----------|
| Yes | `system/continuation` or `system/continuation/join` child path | Delegate to `system/continuation` handler `advance` operation |
| Yes | No continuation entity | Store in tree (write-ahead, §3.2) |
| No | N/A | Store in tree (write-ahead, §3.2) |

**Delegation**: The inbox handler dispatches an EXECUTE to the continuation handler:

```
delegate_to_continuation(inbox_path, message_entity):
  ; Extract result and status from the message (same logic as §3.2)
  if message_entity.type == "system/protocol/inbox/delivery":
    extracted_result = message_entity.data.result    ; may be null
    extracted_status = message_entity.data.status or 200
  else:
    extracted_result = message_entity.data
    extracted_status = 200

  advance_params = {
    type: "system/continuation/advance-request"
    data: {
      result: extracted_result
      status: extracted_status
    }
  }

  dispatch({
    uri:       "system/continuation"
    operation: "advance"
    resource:  {targets: [inbox_path]}
    params:    advance_params
  })
```

If the continuation handler's `advance` operation returns `{advanced: false}` (no continuation entity at path), the inbox handler leaves the message in tree storage.

### 3.4 Concurrency

Concurrent deliveries to the same inbox path are handled by CAS (`expected_hash`) on tree writes:

- **Standing continuations** (`remaining_executions: null`): Read-only continuation, concurrent dispatches succeed independently.
- **One-shot continuations** (`remaining_executions: 1`): First delivery advances, second finds continuation consumed, falls back to tree storage.
- **Counted continuations**: CAS retry on decrement.
- **Join continuations**: CAS retry on slot accumulation.

No locking required. The emit pathway serializes writes per path. CAS detects and resolves conflicts.

### 3.5 Path Identity

Inbox paths SHOULD include unique identifiers:

```
system/inbox/{chain_id}                          ; chain root inbox
system/inbox/{chain_id}/step-2                   ; chain step
system/inbox/watch/{subscription_id}             ; subscription target
system/inbox/work/{queue_name}/{job_id}          ; work queue job
```

Cross-chain contamination is prevented by `expected_hash` (CAS) and `chain_id` on bounds. UUIDs in paths are operational hygiene; CAS is structural correctness.

### 3.6 Delivery observability invariant

Every `receive` EXECUTE accepted by the dispatch layer MUST produce one of:

1. **A write-ahead entity** at `system/inbox/{path}/{id}` per §3.2 step 1, OR
2. **A chain-error marker** under `system/runtime/chain-errors/` per EXTENSION-CONTINUATION.md §3.10.1, OR
3. **A synchronous non-2xx `EXECUTE_RESPONSE`** to the dispatching peer.

Absence of all three is a §9 #8 anomaly signal (per `GUIDE-INSPECTABILITY.md` v1.1 §9 #8). `validate-peer` SHOULD include this as a `CAT-CHAIN-COMPLETION` conformance category check: for each `receive` EXECUTE arriving at the inbox dispatcher, verify within bounded time that one of the three terminal states is observable. NEITHER = conformance failure (silent drop).

This invariant closes the dispatch-routing-side silent-drop class — when an inbox handler short-circuits before the write-ahead store, or the dispatcher routes a `receive` to a non-existent inbox path, no entity appears AND no marker binds AND no sync error returns. F-CIMP-2's failure mode (subscription `deliver_async` wrappers dispatching but the dispatcher not reaching the configured inbox handler) is exactly this class.

The invariant is observability-only — no new handler behavior, no new entity type, no new wire-format change. It pins the implicit completion contract that already exists in the three-path structure of §3.2/§3.3 so inspect tooling and validate-peer can mechanically check it.

When the chain-error marker is bound (option 2), `{reason}` follows EXTENSION-CONTINUATION.md §3.10.5 — it IS the response's `result.data.code` verbatim. For the canonical F-CIMP-2 class (dispatcher routes to a `receive` operation but no handler is registered at the path), the marker `{reason}` is `not_found` (matching ENTITY-CORE-PROTOCOL.md §6.12 transport-code vocabulary), bound at `system/runtime/chain-errors/lost/{chain_id}/{subscription_id_or_request_id}/not_found/{marker_hash}`. Inspect tooling SHOULD find this canonical path when diagnosing "the dispatcher accepted the EXECUTE but no inbox processing happened" cases.

---

## 4. Delivery Construction

When a handler processes an EXECUTE with a `deliver_to` field, it delivers results as follows.

### 4.1 Construction Algorithm

```
construct_delivery(original_execute, result_status, result_data, local_identity):
  delivery_params = {
    original_request_id: original_execute.data.request_id
    status:              result_status
    result:              result_data
  }

  deliver_to = original_execute.data.deliver_to
  deliver_token = lookup(original_execute.data.deliver_token, envelope.included)

  delivery_execute = {
    type: "system/protocol/execute"
    data: {
      request_id: generate_id()
      uri:        deliver_to.uri
      operation:  "receive"
      resource:   {targets: [deliver_to.uri]}
      params:     delivery_params
      author:     local_identity.content_hash
      capability: deliver_token.content_hash
    }
  }

  signature_entity = sign(delivery_execute, local_identity)

  return envelope(delivery_execute, included: [local_identity, signature_entity, deliver_token, ...chain])
```

The operation is always `"receive"` — the message type carries the semantics.

**Result format.** The `result` field in `system/protocol/inbox/delivery` carries the handler's result as a full inline entity `{type, data, content_hash}`, preserving entity identity through the delivery chain. This is consistent with EXECUTE_RESPONSE (ENTITY-CORE-PROTOCOL.md §3.4), where `result` is an entity, not raw data.

The result entity is encoded as an inline CBOR value — a map within the delivery params map. It MUST NOT be byte-string wrapped (CBOR major type 2). Byte-string wrapping produces a different wire format that requires an additional decode step on the receiving end, breaking interop with implementations that expect inline encoding.

When a handler's EXECUTE_RESPONSE provides the result for delivery construction, the full result entity (including its `type` and `content_hash` fields) is placed into the delivery's `result` field without modification. Stripping fields makes the delivery lossy and prevents downstream handlers from reconstructing the entity.

**ECF ordering.** When the inline entity map passes through a decode/reencode cycle (delivery chain, continuation transforms), the ECF key ordering MUST be preserved. Implementations MUST use deterministic CBOR encoding (ECF per ENTITY-CBOR-ENCODING.md) for all entity maps, including inline entities in delivery results. Non-deterministic CBOR encoders could produce different key orderings, changing the entity's content_hash.

### 4.2 Bounds Propagation

Delivery inherits bounds from the originating EXECUTE's context per ENTITY-CORE-PROTOCOL.md §5.9. The dispatch layer manages bounds on the delivery EXECUTE — the handler constructs the delivery, the dispatch layer attaches bounds.

### 4.3 Routing

The constructed EXECUTE routes through standard outbound dispatch. If the delivery URI targets a local path, delivery is dispatched locally. For same-peer dispatch, the capability used MAY be the server's own handler grant for the inbox handler rather than the caller's deliver token.

**Symmetric semantics for handler-initiated sub-dispatch.** A handler that invokes a sub-dispatch (e.g., via the implementation's `hctx.Execute(uri, op, params, deliver_to: X)` equivalent) — whether the target URI is local or remote — MUST follow the same async-spawning semantics as a wire-entry EXECUTE with `deliver_to`. The sub-dispatch handler runs asynchronously and routes its result via inbox delivery at `X`; the synchronous return from the sub-dispatch is the 202 acknowledgement, not the operation result. Implementations MUST NOT silently drop `deliver_to` on sub-dispatches to local handlers; doing so breaks continuation chains whose middle steps target local URIs.

### 4.4 Delivery Failure

If the delivery EXECUTE cannot be delivered:

- The handler SHOULD queue the delivery for retry
- Retry policy is implementation-defined
- If the deliver token expires before successful delivery, the handler MUST discard the queued delivery

### 4.5 Immediate vs Deferred

A handler receiving an EXECUTE with a deliver_to field:

- MUST NOT return the operation result as EXECUTE_RESPONSE on the connection
- SHOULD return EXECUTE_RESPONSE with status 202 and `result: null` to acknowledge receipt
- MUST deliver the operation result via inbox when processing completes

---

## 5. Token Scoping

### 5.1 Requirements

```
deliver_token := system/capability/token where {
  grants: [{
    handlers:   {include: ["system/inbox"]}
    resources:  {include: [inbox_uri]}
    operations: {include: ["receive"]}
  }]
  grantee: delivery_peer_identity_hash
}
```

### 5.2 Scoping Rules

- **Resource**: SHOULD be the exact inbox path. MUST NOT be broader than `system/inbox/*`.
- **Operations**: SHOULD be `"receive"`.
- **Grantee**: MUST be the identity of the peer that will deliver.
- **TTL**: SHOULD be bounded.
- **Delegation**: Deliver tokens SHOULD carry `no_delegation` in `delegation_caveats` unless cross-peer forwarding is intended.

---

## 6. Security Considerations

### 6.1 Stale Dispatch Capabilities

When EXTENSION-CONTINUATION is installed, dispatch capability tokens on continuation entities may expire between continuation creation and delivery arrival. The dispatch layer validates capabilities at dispatch time — expired tokens produce authorization failures. The inbox handler falls back to tree storage (no continuation advancement).

Sender rotation between send and receive falls into the rotation visibility window per EXTENSION-IDENTITY.md §9.3 — receive-time `verify_request` rejects messages whose sender's runtime peer was retired before the message arrived. Once stored, advancement uses the captured signature and is unaffected by subsequent sender rotation.

### 6.2 Deliver Token Replay

Deliver tokens are scoped to specific inbox URIs and grantees. Replay by a different peer fails grantee validation. Replay to a different path fails resource scope validation. TTL bounds the replay window.

---

## 7. Constants

### 7.1 Status Codes

| Code | Meaning |
|------|---------|
| 202 | Accepted (async acknowledgement) |

### 7.2 Standard Operations

| Operation | Handler | Description |
|-----------|---------|-------------|
| `receive` | `system/inbox` | Receive a message at an inbox path |

---

## 8. Write Authorization

All inbox handler tree writes are to the handler's managed namespace (`system/inbox/*`). The handler's own grant authorizes all writes.

| Operation | Write target | Authorization | Capability recorded |
|-----------|-------------|---------------|-------------------|
| receive (store) | `system/inbox/{chain_id}/{id}` | Handler grant | Handler grant |
| receive (cleanup) | `system/inbox/{chain_id}/{id}` (delete) | Handler grant | Handler grant |

The caller's `deliver_token` authorizes *delivery to the inbox* (validated at §3.2 step 1). The tree write that stores the delivery at an inbox path is authorized by the handler's own grant — the handler manages its namespace.

See ENTITY-CORE-PROTOCOL.md §6.8 for the general write authorization model.

---

## 9. Conformance

### 9.1 MUST Implement

- `deliver_token` field on EXECUTE when `deliver_to` is present (§2.3)
- Rejection of EXECUTE with `deliver_to` but without `deliver_token` (§2.3)
- Delivery construction with operation `"receive"` (§4.1) when processing EXECUTE with deliver_to field
- Delivery result carries full inline entity `{type, data, content_hash}` (§4.1)
- Delivery result encoded as inline CBOR, not byte-string wrapped (§4.1)
- Bounds propagation from originating context to delivery (§4.2)
- Standard capability verification on incoming inbox deliveries (§3.2)
- Deliver token grantee validation — delivery peer identity MUST match token grantee
- Inbox handler at `system/inbox` with `receive` operation (§3.1)
- Write-ahead processing: store delivery before continuation advancement (§3.2)
- When EXTENSION-CONTINUATION is installed: delegation to continuation handler's `advance` operation for paths with continuation entities (§3.3)
- When EXTENSION-CONTINUATION is not installed: store delivery in tree (§3.2)
- Backward-compatible storage when no continuation entity exists at inbox path (§3.2)
- Handler-initiated sub-dispatches with `deliver_to` MUST follow the same async-spawning semantics as wire-entry EXECUTEs with `deliver_to`, regardless of whether the target URI is local or remote (§4.3)

### 9.2 SHOULD Implement

- 202 acknowledgement for async operations (§4.5)
- Retry on delivery failure (§4.4)
- Narrow deliver token scoping (§5.2)
- `no_delegation` on deliver tokens (§5.2)
- UUID-based inbox paths (§3.5)
- `expected_hash` (CAS) on inbox write operations (§3.4)
- Default `operation` to `"receive"` when omitted from `system/delivery-spec`

### 9.3 MAY Implement

- Local dispatch optimization for same-peer deliveries (§4.3)
- Backpressure: reject deliveries above inbox depth threshold (429)
- Push-to-pull model switch for high-volume inboxes

### 9.4 Implementation-Defined

- Retry policy for delivery failure (backoff strategy, maximum retries, queue limits)
- Internal deliver token generation for same-peer chain dispatch
- Continuation entity detection mechanism (tree read vs handler registry check)
- Inbox depth limits for backpressure

---

## 11. Privacy + cross-peer observability

Per `GUIDE-INSPECTABILITY.md` v1.2 §9 #4: write-ahead entities at `system/inbox/**`, `system/protocol/inbox/delivery` results, and `system/protocol/inbox/notification` params are **capability-controlled** — bodies carry application-defined delivery payloads + identity references. Payloads MAY carry hash references to **sensitive** types (e.g., `deliver_token` referencing `system/capability/token` per ENTITY-CORE-PROTOCOL.md §5); transitive classification applies — the hash reference itself is `capability-controlled` (reveals "a capability of this hash authorizes delivery here") but a hash without the entity is not bearer.

Per §9 #7:
- **Transport-only** for the arrival hop — a `receive` EXECUTE arrives FROM a remote peer per §3.1; the cross-peer hop IS the propagation surface for the arrival event.
- **Local-namespace** for stored write-ahead entities at `system/inbox/**` after arrival. Subscription-based propagation of `system/inbox/**` MUST be refused at the subscription handler unless the caller's scope explicitly enumerates a per-recipient narrower path with operator-class authority (the "secondary inbox aggregator" pattern, application-defined). The default subscription posture for `system/inbox/**` is refusal — a subscription on the broad inbox prefix would let a third party harvest all inbound traffic.

Chain-error markers bound on inbox dispatch failures per §3.6 are themselves **local-namespace** per EXTENSION-CONTINUATION.md §6.5 (the canonical home for chain-error marker locality).

## 10. Durability Contract — extracted

Durability was extracted from this spec into a separate optional extension. See `EXTENSION-DURABILITY.md` (exploratory, optional). The previously-published §10.1–§10.8 content is preserved verbatim in that file. See the v5.9 changelog entry for rationale.
