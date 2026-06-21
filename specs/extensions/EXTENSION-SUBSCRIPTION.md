# Subscription Extension — Normative Specification

**Version**: 3.16

**Status**: Active
**Depends**: ENTITY-CORE-PROTOCOL.md (v7.31+), EXTENSION-INBOX.md (v5.0+)
**Related**: EXTENSION-NETWORK.md (v1.0+) — subscription restoration after reconnection
**Proposal**: PROPOSAL-SUBSCRIPTION-BOUNDED-FANOUT.md (S1-S3), PROPOSAL-COHERENT-CAPABILITY-AUTHORITY.md (SB1-SB3, GR1)

---

> **Path notation.** Paths in this document use peer-relative notation (without leading `/{peer_id}/`). All peer-relative paths resolve to the local peer's namespace: `system/tree` means `/{local_peer_id}/system/tree`. Every path in the entity tree is absolute at rest — rooted at a peer identity. See ENTITY-CORE-PROTOCOL.md §1.4 for the path model. Cross-peer examples use absolute paths with explicit peer identities.

## 1. Overview

The subscription extension enables peers to receive notifications when entity tree paths change. Subscriptions are EXECUTE operations with inbox delivery — a subscription is a standing delivery that fires on each matching change event.

This extension depends on the inbox extension (EXTENSION-INBOX.md). Subscription notifications are delivered as inbox EXECUTEs through the standard dispatch chain.

This extension defines:

- Subscribe and unsubscribe operations
- Subscription entity storage
- Notification construction and delivery
- Subscription lifecycle (creation, renewal, termination)

### 1.1 Coherent Capability

`system/subscription` entities are load-bearing: their `deliver_token` field authorizes future async deliveries by the subscription engine. Use `system/subscription:subscribe` to create them. The subscribe operation validates the deliver_token's chain root against the subscriber's identity (§3.1) before persisting.

Direct `tree:put` to `system/subscription/*` is permitted but bypasses the subscribe operation's validation; subscriptions created that way are not guaranteed to satisfy the `deliver_token` authority invariant. Application-level capability grants should cover `system/subscription:subscribe` rather than direct `tree:put` to the namespace. Direct `tree:put` is appropriate for system-extension code and administrative/bootstrap contexts. See ENTITY-CORE-PROTOCOL.md §6.3 for the kernel-vs-handler principle.

### 1.2 The three capabilities in a subscribe flow

A subscribe request involves three distinct capabilities, each playing a different role and rooted at a different peer in cross-peer scenarios. Confusing them is a common source of design and review confusion. This is an **instance of the general three-slot model** in ENTITY-CORE-PROTOCOL.md §5.2 ("Cross-peer capability provenance — the three slots") — read that first if the root / grantee / in-chain-granter distinction is unclear; the caller capability is the canonical "root = resource owner B, grantee = EXECUTE author A" case:

| Capability | Carried in | Authorizes | Issued by | Validated by |
|---|---|---|---|---|
| Caller capability | Outer EXECUTE `capability` field | The subscribe operation itself, including the pattern resource scope | The subscribee's namespace authority (target peer that owns the data being observed) | Dispatch chain (`verify_request`, `check_permission`) |
| Subscription deliver_token | `params.deliver_token` | Future async dispatch from the subscription engine to the subscriber's inbox | The subscriber's namespace authority (sender peer that owns the inbox) | Subscribe handler (`grants_access` + chain-root check, §3.1) |
| Inbox EXECUTE-level deliver_token | EXECUTE `deliver_token` field (EXTENSION-INBOX.md §2.3) | A specific async delivery on a specific EXECUTE | The deliverer's namespace authority | Dispatch chain at the receiver |

For Alice (peer A) subscribing to Bob's data (peer B):

- Caller capability is B-rooted (B granted A subscribe access).
- Subscription deliver_token is A-rooted (A authorizes B's engine to deliver to A's inbox).
- The two caps root at different peers; they meet only in the subscribe request envelope.

The subscribe handler at peer B validates the caller cap via the standard dispatch chain, and validates the deliver_token's authority chain (per §3.1) against the EXECUTE author at subscribe time. The integrity validation of the deliver_token (signature chain, scope, grantee match, revocation) at delivery time happens at peer A via the standard ENTITY-CORE-PROTOCOL.md §5.2 dispatch chain — that's separate from and unchanged by the subscribe-time check.

---

## 2. Type Definitions

### 2.1 Subscription Entity

Stored at `system/subscription/{subscription_id}` on the server:

```
system/subscription := {
  fields: {
    subscription_id:     {type_ref: "primitive/string"}
    pattern:             {type_ref: "system/tree/path"}       ; Path pattern to watch
    events:              {array_of: {type_ref: "primitive/string"}}
    deliver_uri:         {type_ref: "system/tree/path"}
    deliver_operation:   {type_ref: "primitive/string"}       ; Typically "receive"
    subscriber_identity: {type_ref: "system/hash"}            ; Subscriber's identity hash
    deliver_token:       {type_ref: "system/hash"}            ; Hash of capability token
    include_payload:     {type_ref: "primitive/bool", optional: true}   ; Persisted from the subscribe request; engine reads it at delivery (§4.2). Default false.
    created_at:          {type_ref: "primitive/uint"}         ; ms since epoch
    limits:              {type_ref: "system/subscription/limits", optional: true}
  }
}
```

The subscription entity is the source of truth. Internal subscription registries or indexes are caches over these entities.

> **Informative — Tree Storage**: Subscription entities at `system/subscription/*` are regular tree entities — stored via tree `put`, accessible via tree `get`, listable via trailing-slash `get` on `system/subscription/` (EXTENSION-TREE.md §2.2). The notification index is an implementation-level cache over these tree entities, not a separate store.

### 2.2 Notification

The params type for a notification inbox EXECUTE:

```
system/protocol/inbox/notification := {
  fields: {
    subscription_id: {type_ref: "primitive/string"}
    event:           {type_ref: "primitive/string"}           ; "created", "updated", "deleted"
    uri:             {type_ref: "system/tree/path"}            ; Tree path that changed
    hash:            {type_ref: "system/hash", optional: true}   ; New entity hash (absent on delete)
    previous_hash:   {type_ref: "system/hash", optional: true}   ; Previous hash (absent on create)
  }
}
```

Notifications report what changed and where. By default they carry only `hash` / `previous_hash`, not entity data. When the subscription sets `include_payload` (§2.3), the server MUST bundle the changed entity into the delivery envelope's `included` map (§4.2) — so the subscriber has the bytes atomically with the notification and needs no follow-up cross-peer GET. This is what makes the cross-peer mirror recipe a single hop: the subscriber applies the change locally with `tree:put` + CAS (`expected_hash = previous_hash`), no fetch (see `proposals/PROPOSAL-CONVERGENT-MIRRORING.md`). Absent/`false`, notifications stay lean (the "tell me when, I'll decide whether to read" case).

**`include_payload` precise semantics (normative).**

- **What is bundled:** exactly the **direct entity** at `notification.hash` — *not* its transitively-reachable closure. A receiver that needs referenced entities resolves them separately. (A future "include closure" mode may be added if a driver appears; today it is the direct entity only.)
- **Removed events** (`notification.hash` absent — the binding was removed upstream): nothing is bundled (there is no entity); the notification carries `previous_hash` only.
- **Source-side resolution failure:** if the source cannot resolve the changed entity at delivery time (GC, a not-yet-committed write, a transient race), it MUST deliver the notification **hash-only** (omit the entity from `included`) and SHOULD debug-log — it MUST NOT fail-stop the subscription. Hash-only delivery is always valid; the receiver's chain MAY fall back to a cross-peer GET in that case.
- **Backward-compatible:** `include_payload` is additive and opt-in (default `false`). Subscribers that do not set it see exactly the prior delivery shape — no entity in `included`, hashes-only notification. The server attaches the entity only when the option is set.

**Mirror recipe — CAS pins for convergence (normative).** A receiver mirroring a source path applies each transition to its local mirror path with a single continuation: extract the entity from `included`, then `tree:put` with `expected_hash = notification.previous_hash`. CAS makes a stale lap fail (409 `hash_mismatch`) instead of rolling state back — this is the amplification fix.

- **Bootstrap (first write to a fresh mirror path):** a *created* event carries `previous_hash` absent/zero; the recipe threads it as the **zero hash** into `expected_hash`, which ENTITY-CORE-PROTOCOL.md §3.9 treats as **CAS-create** (succeed only if the path is unbound). The first write is a clean create, and a stale created-lap arriving after the path is bound fails CAS rather than overwriting.
- **Removed events:** `notification.hash` absent ⇒ a `tree:put` with the entity absent (remove), still under `expected_hash = previous_hash` (the prior bound hash). CAS applies to removes (ENTITY-CORE-PROTOCOL.md §3.9), so a stale removal does not regress a newer binding.

### 2.3 Subscribe Request

The params type for a subscribe operation:

```
system/subscription/request := {
  fields: {
    events:          {array_of: {type_ref: "primitive/string"}, optional: true}
    deliver_to:      {type_ref: "system/delivery-spec"}
    deliver_token:   {type_ref: "system/hash"}            ; Hash of capability token for notification delivery
    include_payload: {type_ref: "primitive/bool", optional: true}   ; Bundle the changed entity in the notification envelope's included (default false)
    limits:          {type_ref: "system/subscription/limits", optional: true}
  }
}
; The subscription pattern comes from the EXECUTE's resource field (resource.targets),
; not from params. Same pattern as tree get/put.
; If events is absent, defaults to ["created", "updated", "deleted"]
; If limits is absent, server applies its own defaults.
```

**`include_payload` requires read authorization (normative).** Subscribing is a *distinct* capability from reading. The subscribe check (§4.x dispatch) verifies the caller's capability grants the `subscribe` operation on `system/subscription` for the resource — it does **not** by itself grant read access to the entity content at those paths (that is the `get` operation on `system/tree`). A caller may legitimately hold `subscribe` without `get` — e.g. cache-invalidation or coordination subscribers that react to the *fact* of a change without reading its content. Therefore, when `include_payload` is set, the subscribe handler MUST additionally verify the caller's capability covers the tree read — `check_path_permission("get", resource_path, caller_capability, "system/tree", local_peer_id)` — and MUST reject the subscribe with **403 `payload_unauthorized`** if it does not. `include_payload` does **not** bypass read authorization; it moves enforcement from a subscriber-side `tree:get` pull (which the caller would otherwise need `get` to perform) to a server-side check *before* the content is pushed — the net authorization is identical. A caller with `subscribe` but not `get` still receives lean (hashes-only) notifications by omitting `include_payload`.

The `deliver_token` field carries the content hash of a `system/capability/token` authorizing the server to deliver notifications to the inbox URI. The token entity MUST be in the envelope's `included` map — this follows the general rule in ENTITY-CORE-PROTOCOL.md §3.1 / §3.2: all entities referenced by hash from the EXECUTE's `data` fields must be present in the envelope's `included` map. This is the standard deliver token from EXTENSION-INBOX.md §5. It is distinct from the EXECUTE-level `deliver_token` defined in EXTENSION-INBOX.md §2.3 — that field is for immediate async inbox delivery, while this field is in the subscribe request params for the server to store with the subscription entity.

### 2.4 Subscription Limits

```
system/subscription/limits := {
  fields: {
    max_events:       {type_ref: "primitive/uint", optional: true}   ; Auto-unsubscribe after N notifications
    max_duration_ms:  {type_ref: "primitive/uint", optional: true}   ; Maximum lifetime in ms
    rate_limit:       {type_ref: "primitive/uint", optional: true}   ; Max notifications per minute
    notification_budget: {type_ref: "primitive/uint", optional: true} ; Budget per notification dispatch
  }
}
```

Limits constrain the subscription's resource consumption. If the subscriber requests limits, the server MAY apply stricter limits but MUST NOT apply more permissive ones. If the subscriber omits limits, the server applies its own defaults.

| Field | Exhaustion behavior |
|-------|-------------------|
| `max_events` | Subscription auto-terminates after N notifications delivered |
| `max_duration_ms` | Subscription auto-terminates after duration (independent of deliver token TTL) |
| `rate_limit` | Notifications exceeding the rate are dropped (not queued) |
| `notification_budget` | Per-notification budget for the delivery dispatch chain |

### 2.5 Unsubscribe Request

```
system/subscription/cancel := {
  fields: {
    subscription_id: {type_ref: "primitive/string"}
  }
}
```

### 2.6 Subscription Redirect

Returned by the subscribe handler when the server is at capacity for the requested prefix:

```
system/subscription/redirect := {
  fields: {
    reason:       {type_ref: "primitive/string"}
                  ; "at_capacity" — subscriber limit reached for this prefix.
    prefix:       {type_ref: "system/tree/path"}
                  ; The prefix that is at capacity.
    alternatives: {array_of: {type_ref: "system/hash"}, optional: true}
                  ; Identity hashes of current subscribers that may accept subscriptions.
                  ; The redirected subscriber MAY connect to any of these peers
                  ; and attempt to subscribe on them instead.
                  ; Optional — server MAY omit to avoid disclosing subscriber list.
    capacity:     {type_ref: "primitive/uint", optional: true}
                  ; The configured limit, for informational purposes.
  }
}
```

When `alternatives` is present, it lists identity hashes of peers currently subscribed to this prefix on this server. These peers receive notifications from this server and may accept downstream subscriptions. The list SHOULD contain at most 10 entries. The server SHOULD randomize the order to distribute load. A server that does not wish to disclose its subscriber list MAY omit `alternatives` entirely — the redirect still communicates "at capacity" without leaking subscriber information.

The server MUST NOT include the subscriber's own identity in the alternatives list. The server SHOULD NOT include peers with status "suspect" or "failed" in the alternatives.

### 2.7 Subscription Capacity

A peer MAY advertise its subscription capacity using the existing `system/config` type (ENTITY-CORE-PROTOCOL.md §3.13, O2):

```
system/config := {
  ; At path: system/config/subscription
  data: {
    concern: "subscription",
    parameters: {
      max_subscribers_per_prefix: {type_ref: "primitive/uint", optional: true}
        ; Maximum direct subscribers per watched prefix.
        ; Default: implementation-defined. SHOULD be documented.
        ; A value of 0 means no limit.
      current_subscriber_prefixes: {type_ref: "primitive/any", optional: true}
        ; Map of prefix → current subscriber count.
        ; Updated on subscribe/unsubscribe.
        ; Informative — peers MAY read this to check capacity before subscribing.
    }
  }
}
```

Stored at: `system/config/subscription`. No new type needed — this uses the open-ended `parameters` field of the existing config type.

The `current_subscriber_prefixes` field is a map where keys are path prefixes and values are current subscriber counts. The field is informative — it MAY lag behind the actual state. The authoritative capacity check is the redirect response (§3.1).

---

## 3. Operations

### 3.1 Subscribe

The subscription extension registers a dedicated handler at `system/subscription`:

```
system/handler := {
  pattern:    "system/subscription"
  name:       "subscriptions"
  operations: {
    subscribe:   {input_type: "system/subscription/request"}
    unsubscribe: {input_type: "system/subscription/cancel"}
  }
}
```

Subscribe EXECUTEs target `system/subscription` with the `resource` field carrying the paths being subscribed to. Standard dispatch (ENTITY-CORE-PROTOCOL.md §6.5) routes to the subscription handler and enforces capability access — the caller's capability MUST grant the `subscribe` operation on handler `system/subscription` for the resource paths in the EXECUTE. This is the same capability model as all other handlers: `handlers` scope identifies the handler, `resources` scope identifies the data paths. No additional access check is needed within the handler.

```
EXECUTE
  uri:       "system/subscription"
  operation: "subscribe"
  resource:  {targets: ["local/files/*"]}
  params:    {
    type: "system/subscription/request"
    data: {
      deliver_to:     {uri: "system/inbox/file-watcher"}
      deliver_token:  <hash>
    }
  }
```

The subscription pattern is `resource.targets[0]` — the same field that authorizes the operation at dispatch time. No duplication between authorization and semantics.

The handler stores the subscription entity with the provided deliver token. Notifications are delivered later via the inbox mechanism (EXTENSION-INBOX.md) — the stored deliver token authorizes the delivering peer to send to the inbox URI. The subscription and inbox extensions are thus complementary: subscriptions define *what* to watch; inbox defines *how* to deliver.

**Revocation.** Revocation of stored subscription deliver tokens is handled automatically by dispatch-layer verification per ENTITY-CORE-PROTOCOL.md §5.2 step 4 (integrated `is_revoked` check, per PROPOSAL-COMPUTE-AMENDMENTS-V3 C2). When a subscription fires and the handler attempts to deliver, the dispatch chain catches revoked tokens automatically — no subscription-specific revocation check is required.

```
handle_subscribe(ctx, params):
  ; params is system/subscription/request

  ; 1. Validate deliver token exists in envelope
  deliver_token = lookup(params.deliver_token, ctx.included)
  if deliver_token is null: return error(400, "missing_deliver_token")

  ; 2. Validate deliver token grants delivery to the inbox URI
  if !grants_access(deliver_token, params.deliver_to.uri, params.deliver_to.operation or "receive"):
    return error(403, "deliver_token_insufficient")

  ; 2a. SB1 (v3.10): R1 creator-authorization check on deliver_token.
  ; Uses check_creator_authority (V7 §5.5) which collects the full authority
  ; chain via collect_authority_chain, verifies reachability, then checks
  ; whether the subscriber's identity appears as granter anywhere in the chain.
  ; This is at SUBSCRIBE TIME — distinct from delivery-time integrity validation
  ; (V7 §5.2 verify_request validates signature, scope, grantee, revocation
  ; when the delivery EXECUTE arrives at the receiver).
  (found, chain, err) = check_creator_authority(deliver_token, ctx.execute.data.author, ctx)
  if err == ChainUnreachable:
    return error(404, "chain_unreachable",
      "Deliver token's authority chain is not fully resolvable")
  if not found:
    return error(403, "embedded_cap_unauthorized",
      "Subscriber identity not in deliver_token authority chain")

  ; 2b. Persist deliver_token + chain to local content store so future
  ; notification dispatch can resolve the cap by hash without the chain
  ; needing to travel again. Chain was already collected — no re-walk.
  for entity in chain:
    ctx.content_store.put(entity)

  ; 3. Resolve limits (subscriber request vs server defaults; server may tighten)
  limits = apply_server_limits(params.limits or server_defaults)

  ; 3a. Check subscriber capacity for this prefix (§2.7)
  prefix = ctx.resource.targets[0]
  if max_subscribers_per_prefix > 0 and subscriber_count(prefix) >= max_subscribers_per_prefix:
    ; At capacity — return redirect response
    alternatives = list_subscriber_peers(prefix)   ; MAY be empty or omitted
    return {
      status: 303,
      result: {
        type: "system/subscription/redirect",
        data: {
          reason:       "at_capacity",
          prefix:       prefix,
          alternatives: alternatives,
          capacity:     max_subscribers_per_prefix
        }
      }
    }

  pattern = ctx.resource.targets[0]                    ; From EXECUTE resource field

  ; 3b. include_payload read-authorization (§2.3): content delivery requires
  ;     read access — the subscribe grant alone does not authorize it.
  if params.include_payload:
    if not check_path_permission("get", pattern, ctx.caller_capability,
                                 "system/tree", ctx.local_peer_id):
      return error(403, "payload_unauthorized",
                   "include_payload requires tree:get on the subscribed resource")

  ; 4. Create subscription entity
  subscription_id = generate_id()
  events = params.events or ["created", "updated", "deleted"]

  subscription = {
    type: "system/subscription"
    data: {
      subscription_id:     subscription_id
      pattern:             pattern                      ; From EXECUTE resource target
      events:              events
      deliver_uri:         params.deliver_to.uri
      deliver_operation:   params.deliver_to.operation or "receive"
      subscriber_identity: ctx.execute.data.author    ; system/hash
      deliver_token:       deliver_token.content_hash  ; system/hash
      include_payload:     params.include_payload or false
      created_at:          now()
      limits:              limits
    }
  }

  ; 5. Store subscription entity
  ctx.entity_tree.put("system/subscription/" + subscription_id, subscription)

  ; 6. Store associated entities (deliver token, identities in delegation chain)
  for entity in ctx.included: ctx.content_store.put(entity)

  ; 7. Register in notification index (implementation detail)
  notification_index.register(subscription)

  return {
    status: 200
    result: {
      subscription_id: subscription_id
      pattern:         pattern
      events:          events
      limits:          limits              ; Effective limits (may differ from requested)
    }
  }
```

### 3.2 Unsubscribe

```
handle_unsubscribe(ctx, params):
  ; params is system/subscription/cancel

  subscription = ctx.entity_tree.get("system/subscription/" + params.subscription_id)
  if subscription is null: return error(404, "subscription_not_found")

  ; Only the original subscriber or admin can unsubscribe
  if not hash_equals(subscription.data.subscriber_identity, ctx.execute.data.author):
    return error(403, "not_subscription_owner")

  ; Remove subscription entity
  ctx.entity_tree.put("system/subscription/" + params.subscription_id, null)
  notification_index.remove(params.subscription_id)

  return {status: 200, result: null}
```

### 3.3 Inbox Handler — Notification Delivery

Subscription notifications are delivered through the inbox handler's `receive` operation (EXTENSION-INBOX.md §3.1). The notification entity type (`system/protocol/inbox/notification`) carries the semantic information — no separate operation is needed.

Processing follows the standard inbox `receive` pattern (EXTENSION-INBOX.md §3.2). The inbox handler receives the notification EXECUTE, validates the capability, and processes it through write-ahead storage. Continuation behavior — what the subscriber does with the notification — is determined by EXTENSION-INBOX.md §3.3 (delegation to continuation handler when EXTENSION-CONTINUATION is installed, tree storage otherwise).

---

## 4. Notification Delivery

### 4.1 Trigger

Entity tree changes produce internal change notifications. The notification layer matches changes against registered subscription patterns.

```
on_tree_change(event_type, uri, new_hash, previous_hash, emission_context):
  for subscription in notification_index.match(uri):
    if event_type in subscription.data.events:
      deliver_notification(subscription, event_type, uri, new_hash, previous_hash, emission_context)
```

`emission_context` carries the bounds context from the tree write that triggered the change. Used for `chain_id` inheritance in notification bounds (§4.5).

Subscription's position in the emit pathway consumer ordering (after compute, last synchronous consumer) is specified in SYSTEM-COMPOSITION.md §2.2. The cascade depth threshold at which same-peer notification delivery is suppressed is specified in SYSTEM-COMPOSITION.md §3.2 (RECOMMENDED default: 8).

### 4.2 Construction

```
deliver_notification(subscription, event_type, uri, new_hash, previous_hash, emission_context):
  params = {
    subscription_id: subscription.data.subscription_id
    event:           event_type
    uri:             uri
  }
  if new_hash != null:      params.hash = new_hash
  if previous_hash != null: params.previous_hash = previous_hash

  ; Determine delivery capability based on inbox target.
  ;
  ; The deliver token (granter = subscriber, grantee = server) is verified
  ; at subscribe time (§3.1 step 2) to confirm the subscriber authorized
  ; delivery to the inbox URI. For actual delivery dispatch, the capability
  ; depends on where the inbox handler lives:
  ;
  ; Same-peer: The inbox handler is on this peer. The server dispatches
  ;   locally using its own handler grant for system/inbox. The handler
  ;   grant's granter is the local peer, so VerifyChain passes the root
  ;   trust check. The deliver token is not used at dispatch time — it
  ;   served its authorization purpose during subscription creation.
  ;
  ; Cross-peer: The inbox handler is on the subscriber's peer (or a
  ;   third party). The deliver token accompanies the EXECUTE to the
  ;   remote peer, where the token's granter (the subscriber) matches
  ;   the remote peer's local_peer_id and VerifyChain passes.

  deliver_uri = subscription.data.deliver_uri

  if is_local(deliver_uri):
    ; Same-peer delivery — use server's handler grant
    delivery_capability = handler_grant("system/inbox")
    capability_chain = {}
  else:
    ; Cross-peer delivery — use subscriber's deliver token
    delivery_capability = content_store.get(subscription.data.deliver_token)
    capability_chain = resolve_delegation_chain(delivery_capability)

  ; Construct delivery EXECUTE per EXTENSION-INBOX.md §4.1
  delivery = {
    type: "system/protocol/execute"
    data: {
      request_id: generate_id()
      uri:        deliver_uri
      operation:  subscription.data.deliver_operation    ; From deliver_to.operation at subscribe time; typically "receive"
      resource:   {targets: [deliver_uri]}               ; Authorization target — see note below
      params:     params
      author:     local_identity.content_hash            ; system/hash
      capability: delivery_capability.content_hash       ; system/hash
    }
  }

  ; Create signature entity (target-matching)
  signature = {
    type: "system/signature"
    data: {
      target:    delivery.content_hash
      signer:    local_identity.content_hash
      algorithm: "ed25519"
      signature: sign(delivery.content_hash)
    }
  }

  ; Build included map
  included = {local_identity, signature, delivery_capability, ...capability_chain}
  ; include_payload (§2.3): when the subscription opted in, bundle the changed
  ; entity so the subscriber has the bytes atomically with the notification and
  ; needs no follow-up GET. Opt-in — absent/false keeps notifications lean
  ; (hashes only). When set, the entity MUST be attached if resolvable.
  ; A subscription carries include_payload only if the subscriber passed the
  ; read-authorization check at subscribe time (§2.3, 403 payload_unauthorized
  ; otherwise) — so attaching content here does not bypass read access.
  if subscription.data.include_payload and new_hash != null:
    changed_entity = content_store.get(new_hash)
    if changed_entity != null: included.add(changed_entity)

  dispatch(envelope(delivery, included))
```

**Same-peer delivery shortcut.** For same-peer delivery, the subscription engine MAY dispatch notifications via the server's handler grant for `system/inbox` and SHOULD NOT construct a delivery token. The delivery-token mechanism applies to cross-peer delivery where subscriber and server are distinct principals. The deliver token validated at subscribe time (§3.1 step 2) confirms authorization intent; same-peer dispatch does not require it at delivery time.

**Resource target is deliver_uri, without suffix.** The resource target MUST be `deliver_uri` as stored on the subscription entity — no subscription_id, event type, or other suffix appended. The inbox handler uses the resource target path to locate continuations stored at that path. Appending a suffix causes the path to not match the continuation location, breaking advancement.

Capability scoping for notification delivery uses the `deliver_token` (validated at subscribe time, §3.1 step 2), not the resource target path. The resource target's authorization role is defense-in-depth at the dispatch layer, where `deliver_uri` is sufficient scope.

### 4.3 Pattern Matching

Subscription patterns use the same matching rules as capability patterns (ENTITY-CORE-PROTOCOL.md §5.4):

- Exact path: `local/files/config.json` — matches only that path
- Subtree: `local/files/*` — matches any path under `local/files/`

### 4.4 Entity Inclusion

The server MAY include the changed entity in the notification envelope's `included` map. Inclusion decision is implementation-defined. Strategies:

| Strategy | Behavior |
|----------|----------|
| Never include | Subscriber always fetches. Minimal bandwidth. |
| Always include | Subscriber has entity immediately. Higher bandwidth. |
| Size threshold | Include if entity < N bytes. Exclude large entities. |
| Subscriber hint | Subscribe request includes `include_entities: true/false`. |

The subscriber detects inclusion by checking `envelope.included[params.hash]`.

### 4.5 Notification Budget Model

Notification delivery uses **independent budget** — not the writer's remaining budget. This is the critical design distinction:

```
on_tree_change(event_type, uri, new_hash, previous_hash, emission_context):
  for subscription in notification_index.match(uri):
    if event_type in subscription.data.events:
      ; Each notification gets its own bounds — NOT derived from emission_context.budget
      notification_bounds = {
        ttl:            peer_default_ttl                     ; Fresh TTL
        budget:         subscription.data.limits.notification_budget or peer_default_budget
        chain_id:       emission_context.chain_id            ; Inherited for causal tracing
        cascade_depth:  emission_context.cascade_depth       ; Inherited for cross-peer cascade tracking
        visited:        []                                   ; Fresh visited list
      }
      deliver_notification(subscription, event_type, uri, new_hash, previous_hash, notification_bounds)
```

**Rationale**: A tree write that triggers N subscriptions should not divide the writer's budget among N notifications. The writer pays for its write; each subscription pays for its own notification delivery from its own budget allocation. This prevents a popular path from exhausting the writer's budget.

`chain_id` and `cascade_depth` ARE inherited from the emission context. `chain_id` links notifications to the causal chain that triggered them, enabling end-to-end tracing. `cascade_depth` carries the current cascade depth to the receiving peer, ensuring cross-peer cascades accumulate depth rather than resetting to 0 at each peer boundary (see SYSTEM-COMPOSITION.md §3.4). Budget, TTL, and visited are independent because the notification is a new dispatch chain branching from the write.

### 4.6 Limits Enforcement

Before delivering each notification, the server checks subscription limits:

```
check_subscription_limits(subscription):
  limits = subscription.data.limits
  if limits is null: return ALLOW

  if limits.max_events != null:
    if subscription.delivered_count >= limits.max_events:
      terminate_subscription(subscription, "max_events_reached")
      return DENY

  if limits.max_duration_ms != null:
    if now() - subscription.data.created_at >= limits.max_duration_ms:
      terminate_subscription(subscription, "max_duration_reached")
      return DENY

  if limits.rate_limit != null:
    if subscription.recent_delivery_rate > limits.rate_limit:
      return DENY   ; Drop notification, do NOT terminate subscription

  return ALLOW
```

`delivered_count` and `recent_delivery_rate` are internal tracking state, not stored on the subscription entity. `max_events` and `max_duration_ms` termination deletes the subscription entity. `rate_limit` drops individual notifications without terminating.

### 4.7 Chain-error marker emission on outbound dispatch failures

The subscription engine's outbound notification dispatches participate in the chain-error convention (EXTENSION-CONTINUATION.md §3.10). When a notification delivery fails terminally — limit-exceeded suppression (§4.6), transport failure, or capability rejection — the subscription engine MUST bind a `lost` marker at:

```
system/runtime/chain-errors/lost/{chain_id}/{subscription_id}/{reason}/{marker_hash}
```

where `{step_index}` is `{subscription_id}` (the trigger is a tree change rather than a chained EXECUTE, so no original-request-id is available). `{chain_id}` is inherited from the source change's chain causality per §4.5. `{reason}` follows EXTENSION-CONTINUATION.md §3.10.5 (= the `code` value from the responding handler's error appendix, or `rate_limited` / `max_events_reached` / `max_duration_reached` for limit suppression).

This is the §9 #8 completion contract (per `GUIDE-INSPECTABILITY.md` v1.1 §9 #8) for subscription-emitted chains:
- **SUCCESS** = receiver-side write-ahead entity (EXTENSION-INBOX.md §3.2 step 1).
- **FAILURE** = the marker above.
- **ANOMALY** = neither present after a tree-change-trigger that §4.1 matched.

The marker is informational; it MUST NOT trigger advancement, retry, or any reactive behavior beyond surfacing the failure for inspect tooling and `validate-peer`'s `CAT-CHAIN-COMPLETION` conformance check. Per the chain-error marker convention, same-reason re-binding is idempotent; the path discriminator on `{reason}` keeps distinct failure modes coexistent rather than overwriting.

Rationale: F-CIMP-2 (Cohort B) was an outbound subscription dispatch failure that surfaced session-later as cross-impl attribution thrash rather than at validate-peer time, because the spec did not state whether the §3.10 chain-error marker convention applied to subscription engine outbound EXECUTEs. This subsection closes that gap.

Per `proposals/implemented/PROPOSAL-CHAIN-PARTICIPATION-INVARIANTS.md` §2.3.

---

## 5. Subscription Lifecycle

### 5.1 Creation

1. Subscriber creates a continuation entity at its inbox path (optional, local)
2. Subscriber creates a deliver token scoped to the inbox path
3. Subscriber sends EXECUTE to `system/subscription` with `operation: subscribe`, pattern, delivery spec, and deliver token
4. Server stores subscription entity, registers in notification index
5. Server returns subscription_id

### 5.2 Active Period

Notifications fire on tree changes matching the subscription pattern. Each notification is an independent inbox delivery EXECUTE.

**Per-subscription ordering (normative; v3.15).** Within a single subscription, deliveries MUST arrive in tree-change order — the order in which the underlying tree changes were committed at the publisher. Subscribers MAY rely on this ordering: for a single subscription's matching changes, a notification with `previous_hash = H_n` arrives before a notification reporting the next change in that subscription's stream.

**Cross-subscription ordering is impl-defined.** Across different subscriptions (distinct `subscription_id` values), notification ordering is not guaranteed — the delivery substrate MAY parallelize delivery across subscriptions (e.g., shard-by-`subscription_id` worker pools, per-subscription queues) to scale throughput. This is the **recommended impl pattern for production deployments** serving multiple concurrent subscribers; workbench-go's Stage 5 K-worker microbench measured near-linear scaling to CPU count (K=4 → 3.8×, K=8 → 5×); shard-by-`subscription_id` preserves the within-subscription ordering MUST while parallelizing across subscriptions.

**Rationale.** Within-subscription ordering is the typical consumer expectation (matches state-synchronization, file-change, and audit-log use cases); it makes §5.5 gap detection meaningful (the `previous_hash` chain is well-formed within a subscription). Cross-subscription parallelism is the load-scaling lever — workbench-go's Stage 5 work surfaced 1/N per-subscriber degradation under single-loop delivery, fixed by sharded parallel workers. Per `proposals/implemented/PROPOSAL-STAGE-5-SUBSCRIPTION-POSITIONS.md` §1.

**K-tuning advisory (v3.16, informative; round-3 evidence strengthened).** Under FNV-style or general-purpose hash distribution of `subscription_id` across `K` shards, collisions are common when the steady-state subscription count `N` is close to `K`. **Workbench-go round-3 5-seed re-measurement** at `K=8, N=4` random sub_ids: **3/5 seeds at 2K/sec hub hit a 2:1 collision; 5/5 seeds at 5K/sec hub hit collision (one seed hit a 3:1 collision)**. This matches P-theory: `P(any 2:1 collision) ≈ 67%` at K=8, N=4 (1 - 8!/(8-4)!/8^4). **The K>2N rule of thumb is operationally important, not a curiosity** — production deployments with N≥4 subscribers per hub will routinely observe ~50% per-spoke throughput spread without it. **Operators expecting uniform per-subscription throughput SHOULD configure `K > 2*N`** for the expected steady-state subscription count (e.g., K=16+ for N=8). This is tuning guidance, not a normative requirement — correctness is unaffected by hash collisions (within-sub FIFO + cross-sub parallelism still hold; collisions only cause throughput variance, not ordering or delivery loss). Round-robin or counter-mod-K assignment can give uniform distribution at the cost of subscription-ID-to-shard stability across restarts; impls choose. Per round-2 finding F10 (`entity-workbench-go/docs/architecture/reviews/FEEDBACK-ARCH-STAGE-5-ROUND-2.md` §2) + round-3 5-seed empirical validation (`entity-workbench-go/docs/architecture/reviews/STAGE-5-TOPOLOGY-VALIDATION.md` §1).

### 5.3 Renewal

The deliver token's expiry bounds the subscription's lifetime. To maintain a long-lived subscription:

```
renew_subscription(subscription_id, new_deliver_token):
  ; Send a new subscribe EXECUTE to system/subscription
  ; with the same pattern and inbox URI but a fresh deliver token.
  ; Server detects matching (subscriber, pattern, deliver_uri)
  ; and updates the existing subscription's deliver_token atomically.
```

The server identifies a renewal by matching: same subscriber identity + same pattern + same deliver URI. On match, the server updates the subscription entity's `data.deliver_token` rather than creating a duplicate. The subscription ID is preserved across renewals — only the token rotates.

### 5.4 Termination

A subscription terminates when:

| Trigger | Behavior |
|---------|----------|
| Explicit unsubscribe | Subscriber sends EXECUTE with `operation: unsubscribe`. Server deletes subscription entity. |
| Deliver token expiry | Server validates token before each delivery. On expiry, server deletes subscription entity. |
| `max_events` reached | Notification count equals limit. Server deletes subscription entity (§4.6). |
| `max_duration_ms` elapsed | Subscription age exceeds limit. Server deletes subscription entity (§4.6). |
| Delivery failure | After implementation-defined retries, server MAY delete subscription entity. |
| Server cleanup | Server MAY evict subscriptions with no successful delivery for a configurable period. |

On termination, the server deletes the subscription entity from `system/subscription/`. The subscriber detects termination by absence of notifications and can re-subscribe.

### 5.5 Gap Detection

Notifications are best-effort. The subscriber can detect gaps:

- `previous_hash` in notification should match the last known hash for that URI
- If `previous_hash` doesn't match, an intermediate change was missed
- Subscriber can fetch current state via GET to reconcile

For guaranteed consistency, subscribers SHOULD periodically reconcile via GET on subscribed paths.

### 5.6 Connection Drops

Subscription entities survive connection drops. Subscriptions are tree entities stored at `system/subscription/*` — they are not connection state. When a transport connection to a subscriber's peer fails:

- The subscription entity remains in the tree
- The notification index MAY deactivate the subscription (implementation optimization)
- Notifications for deactivated subscriptions are dropped (at-most-once delivery)
- The deliver token remains valid (subject to its own TTL)

On reconnection:

- Subscription entities are still present — the notification index re-activates them
- Notifications resume from the point of reconnection (not from the point of disconnection)
- Changes that occurred during the disconnection window produce no retroactive notifications
- The subscriber SHOULD use gap detection (§5.5) to reconcile missed changes

When EXTENSION-NETWORK.md is installed, subscription restoration is handled by the network handler's `maintain-peer` continuation graph (EXTENSION-NETWORK.md §7). Without the network extension, implementations handle restoration through their own reconnection logic — the normative requirement is that subscription entities in the tree are the source of truth, and reconnection to a peer whose subscriptions are still valid resumes notification delivery.

**Deliver token expiry during disconnect.** If a deliver token expires while the subscriber is disconnected, the subscription is terminated on the next delivery attempt (§5.4). The subscriber must re-subscribe with a fresh deliver token after reconnecting.

### 5.7 Subscriber Process Restart (v3.15)

§5.6 covers **transport disconnect/reconnect**: the subscriber's process stays up, the TCP connection drops, and the publisher's subscription entity reactivates on reconnection. This subsection covers **subscriber process restart**: the subscriber's process restarts entirely, losing any in-memory inbox handler registrations.

**Publisher-side state is durable (unchanged from §5.6).** Subscription entities persist at `system/subscription/*` on the publisher across the subscriber's restart. The publisher continues attempting delivery to the subscribed inbox URI per its retention/cleanup policy (§5.4 Termination triggers).

**Subscriber-side handler restoration is application-level (normative; v3.15).** On subscriber process restart, the substrate MUST NOT automatically re-register inbox handlers for prior subscription entities — the MUST NOT keeps the substrate's scope predictable so application/SDK helpers can build restoration with known semantics. Re-registration is application-level responsibility — typical implementations expose an SDK helper (e.g., `RestorePriorSubscriptions(channel)`) that, on boot:

1. Scans local tree for `system/subscription/*` entities created by this peer's identity (or matching a configured filter).
2. Re-registers an inbox handler for each via the standard inbox-handler registration mechanism (EXTENSION-INBOX.md §3).
3. Returns the count of restored handlers for operational visibility.

Until the SDK helper runs, the publisher's delivery attempts land at an inbox URI with no registered handler — deliveries fail (per ENTITY-CORE-PROTOCOL.md §5.2 dispatch chain return); the publisher's retention policy (§5.4) eventually evicts the subscription if delivery failure persists.

**Gap detection (§5.5) covers missed-while-down notifications.** After handler restoration, the first received notification's `previous_hash` SHOULD be compared against the subscriber's local last-known hash for that URI; mismatch indicates missed intermediate changes; reconcile via GET on the subscribed paths.

**EXTENSION-NETWORK alternative.** When EXTENSION-NETWORK is installed, its `maintain-peer` continuation graph (§7) MAY perform automatic subscriber-side restoration. The substrate-level requirement above applies when EXTENSION-NETWORK is not installed or its auto-restoration is disabled.

**Rationale.** Subscriber-side restoration is a recovery pattern, not a core substrate property. Content-addressed durable state (the subscription entity in the tree) is the spec boundary; in-memory handler registration is process-lifetime state. Distinguishing the two gives the substrate a clean persistence model + lets recovery patterns layer on top via SDK / EXTENSION-NETWORK / application logic without conflating spec scope. Per `proposals/implemented/PROPOSAL-STAGE-5-SUBSCRIPTION-POSITIONS.md` §2; workbench-go surfaced the gap during Stage 5 restart-equivalence work (`entity-workbench-go/docs/architecture/reviews/STAGE-5-RESTART-MESH.md`).

---

## 6. Cross-Peer Delivery

### 6.1 Third-Party Delivery

The delivery URI does not have to point back to the subscriber:

```
Peer A subscribes on Peer B:
  pattern:      "local/sensors/*"
  deliver_uri:  "entity://peer_c/system/inbox/sensor-data"
  deliver_token: grants Peer B → Peer C delivery
```

Peer B delivers notifications directly to Peer C. Peer A is not in the delivery path.

### 6.2 Capability Requirements

For third-party delivery, the deliver token MUST authorize the server (Peer B) to deliver to the third party (Peer C). The subscriber (Peer A) MUST have delegation authority — either:

- Peer C pre-issues a deliver token to Peer A for this purpose
- Peer A delegates from a broader capability on Peer C

The delegation chain is included in the subscribe EXECUTE's envelope. The server stores the deliver token with the subscription and uses it for every delivery.

---

## 7. Security Considerations

### 7.1 Subscription Enumeration

Subscriptions at `system/subscription/*` are entities in the tree. Access to `system/subscription/` SHOULD be capability-gated. Only the peer admin or the subscription's subscriber identity SHOULD be able to list or read subscription entities.

### 7.2 Notification Flooding

A subscriber could create subscriptions on high-frequency paths to generate excessive delivery traffic. Defenses:

- **Subscription limits** (§2.4) — `rate_limit` and `max_events` bound per-subscription cost
- **Server-imposed limits** — server SHOULD impose maximum subscription count per subscriber identity
- **Independent budget** (§4.5) — writer's budget is not consumed by notifications, preventing a popular path from becoming a denial-of-service vector against writers
- **Notification budget** — `notification_budget` on the subscription limits the cost of each notification's delivery chain
- **Subscriber capacity** (§2.7) — server SHOULD advertise subscription capacity at `system/config/subscription`. When `max_subscribers_per_prefix` is configured and the current subscriber count for a prefix equals or exceeds the limit, the server SHOULD respond to new subscribe requests with a redirect (§3.1 step 3a)

### 7.3 Stale Subscriptions

Subscriptions with expired deliver tokens MUST NOT accumulate. The server MUST validate the deliver token before each delivery attempt and delete the subscription on expiry.

---

### 7.4 Privacy + cross-peer observability

Per `GUIDE-INSPECTABILITY.md` v1.2 §9 #4:
- Subscription entity bodies (`system/subscription/{id}`) and notification delivery params (`system/protocol/inbox/notification`) are **capability-controlled** — carry subscriber identity, deliver_token hash references, and pattern being watched.
- Subscription request / cancel / redirect in-flight types are **capability-controlled**; subscription `limits` and `system/config/subscription` are **public** (resource budgets and advertised capacity, designed to be discoverable).
- `system/subscription/redirect` `alternatives` field is **capability-controlled** and SHOULD be omitted in privacy-sensitive deployments (formalizes §2.6's MAY-omit convention).

Per §9 #7:
- **Local-namespace** for subscription entity state at `system/subscription/{id}` — held at the server peer; subscription enumeration via cross-peer subscription on `system/subscription/**` is NOT a sanctioned discovery mechanism. (Formalizes the §2.6 redirect-alternatives privacy posture as a declaration rather than a per-server option.)
- **Convergent** for notification delivery — the cross-peer surface IS the propagation channel for the subscribed prefix's content. Observers see notifications targeted at their inbox path, not enumeration of all subscriptions. Convergence model: per-notification CAS-pinned application via the mirror recipe (§2.2).

Outbound dispatch failures bind chain-error markers per §4.7; those markers are themselves **local-namespace** per EXTENSION-CONTINUATION.md §6.5 (the canonical home for chain-error marker locality).

Per `proposals/implemented/AUDIT-PRIVACY-AND-CROSS-PEER-OBSERVABILITY.md` §2.2.

## 8. Dissemination Trees (Informative)

When a peer serves a popular prefix, bounded fan-out (§2.7, §3.1 step 3a) naturally produces a dissemination tree. This section describes the convention.

### 8.1 Tree Formation

A dissemination tree forms from bounded fan-out:

```
1. Source peer S publishes data at prefix P
2. Subscriber A subscribes to S for prefix P → accepted (S has capacity)
3. Subscriber B subscribes to S for prefix P → accepted
4. ... (up to K subscribers)
5. Subscriber K+1 subscribes to S for prefix P → redirect to [A, B, ...]
6. K+1 picks A, subscribes to A for prefix P → accepted (A has capacity)
7. A now receives notifications from S AND serves them to K+1
```

A receives data from S via its subscription. When A's local tree changes (notification received, written to local tree, local emit fires), K+1's subscription on A fires. The data propagates: S → A → K+1.

With K=10, tree depth is O(log_10(N)) for N total subscribers:

| N (subscribers) | Direct fan-out | With K=10 tree | Tree depth |
|---|---|---|---|
| 10 | 10 per source | 10 per node | 1 |
| 100 | 100 per source | 10 per node | 2 |
| 1,000 | 1,000 per source | 10 per node | 3 |
| 10,000 | 10,000 per source | 10 per node | 4 |

### 8.2 Intermediate Peer Setup

For a peer to serve as an intermediate node in the dissemination tree, it needs:

1. **A subscription on its upstream peer** for the prefix
2. **A mechanism to write received notification data to the local tree** (so downstream subscriptions fire on local emit)
3. **Capacity to accept downstream subscriptions** (advertised via `system/config/subscription`)

How the intermediate peer writes received data to its local tree is implementation-specific. A continuation at the subscription's inbox path is one approach (see EXTENSION-CONTINUATION.md). An imperative handler that processes notifications and writes to the tree is equally valid. The normative requirement is: notification data from the upstream peer ends up in the local tree, local emit fires, and downstream subscriptions trigger.

### 8.3 Client Behavior on Redirect

When a subscribe request returns status 303 with a `system/subscription/redirect` result:

1. If `alternatives` is present: pick an alternative peer from the list (random or by preference).
2. If `alternatives` is absent: the server declined without suggestions. The subscriber MAY query the source peer's subscriber list via other means, or retry later.
3. Connect to the alternative peer (if not already connected).
4. Subscribe on the alternative peer. If it also responds with 303: follow its redirect, or try another alternative from the original list.
5. If no alternatives have capacity: retry later or accept the latency cost.

Subscribers SHOULD limit redirect following to a maximum depth of 10 hops. If no peer accepts the subscription within the redirect depth limit, the subscriber SHOULD back off and retry after a delay.

Reading the alternative peer's `system/config/subscription` before subscribing is an optional optimization — the redirect response (step 3a) is the authoritative mechanism for capacity enforcement.

### 8.4 Self-Repair

When an intermediate peer goes offline:

1. Downstream subscribers detect the connection drop (network extension)
2. The maintain-peer continuation triggers reconnection attempts
3. If the intermediate peer doesn't recover within the timeout:
   a. Downstream subscribers read the alternatives from the redirect they originally received (cached locally)
   b. Or: downstream subscribers re-subscribe to the original source, which issues a new redirect with updated alternatives
   c. Or: downstream subscribers discover another intermediate peer via the source's subscriber list

The repair uses existing network extension mechanisms. No new repair protocol is needed. The tree rebalances incrementally as subscribers find new upstreams.

### 8.5 Sync Integration

Subscription dissemination and sync dissemination are structurally identical. Both propagate tree state changes through a chain of peers. A peer that serves as a dissemination node for subscriptions can also serve as a sync intermediary for the same prefix.

Peers SHOULD use the same topology for both subscription dissemination and sync propagation when targeting the same prefix. Configuring both through the same upstream peer avoids redundant connections and simplifies the tree structure.

### 8.6 Recommended K Values

The max subscribers per prefix (K) is configurable per peer via `system/config/subscription`. Recommended defaults:

| Peer profile | K value | Rationale |
|---|---|---|
| Resource-constrained (mobile, IoT) | 3-5 | Minimal per-node load |
| Standard peer | 10-20 | Balance between depth and load |
| Hub/infrastructure | 50-100 | Shallow trees, higher capacity |

No system-wide consensus on K is needed. Each peer sets its own limit.

---

## 9. Constants

### 9.1 Event Types

| Event | Description |
|-------|-------------|
| `created` | Entity stored at a path that had no entity |
| `updated` | Entity replaced at a path that already had an entity |
| `deleted` | Entity removed from a path |

### 9.2 Standard Operations

| Operation | Handler | Description |
|-----------|---------|-------------|
| `subscribe` | `system/subscription` | Create a subscription on a path |
| `unsubscribe` | `system/subscription` | Remove a subscription |
| `receive` | `system/inbox` | Deliver a change notification |

---

## 10. Write Authorization

All subscription handler tree writes are to the handler's managed namespace (`system/subscription/*`). The handler's own grant authorizes all writes.

| Operation | Write target | Authorization | Capability recorded |
|-----------|-------------|---------------|-------------------|
| subscribe | `system/subscription/{id}` | Handler grant | Handler grant |
| unsubscribe | `system/subscription/{id}` (delete) | Handler grant | Handler grant |
| renewal | `system/subscription/{id}` (update) | Handler grant | Handler grant |

The caller's capability authorizes *calling the subscription handler with the subscribe operation* (validated at dispatch, ENTITY-CORE-PROTOCOL.md §6.5). The tree write that stores the subscription entity is authorized by the handler's own grant.

**Delivery authorization.** When the subscription engine delivers notifications:

- **Same-peer delivery**: The subscription engine constructs a delivery EXECUTE using the inbox handler grant (resolved from `system/capability/grants/system/inbox`). This is autonomous — no external caller.
- **Cross-peer delivery**: The subscription engine uses the subscriber's stored `deliver_token` (provided at subscribe time, stored on the subscription entity). This token authorizes delivery to the remote inbox.

See ENTITY-CORE-PROTOCOL.md §6.8 for the general write authorization model.

---

## 11. Conformance

### 11.1 MUST Implement

- Subscription handler at `system/subscription` with `subscribe` and `unsubscribe` operations (§3.1)
- Subscription entity storage at `system/subscription/*` (§2.1)
- Subscribe operation with subscription_id response (§3.1)
- Unsubscribe operation (§3.2)
- Notification construction with subscription_id, event, uri (§4.2)
- Independent notification budget — notifications MUST NOT consume the writer's budget (§4.5)
- `chain_id` inheritance from emission context to notification (§4.5)
- Deliver token validation before each notification delivery (§5.4)
- Subscription deletion on deliver token expiry (§5.4)
- Subscribe handler MUST verify the deliver_token's authority chain root against the EXECUTE author's identity via `check_creator_authority` (ENTITY-CORE-PROTOCOL.md §5.5); reject with 403 `embedded_cap_unauthorized` on mismatch (§3.1, SB1)
- When `include_payload` is set, the subscribe handler MUST verify the caller's capability covers tree `get` on the resource (`check_path_permission("get", resource_path, caller_capability, "system/tree", local_peer_id)`); reject with 403 `payload_unauthorized` if not. `subscribe` permission alone does NOT authorize payload delivery — content delivery requires read access (§2.3, v3.13)
- Subscribe handler MUST persist the deliver_token and its full authority chain to local content store after the chain-root check succeeds (§3.1)
- Generalized rule: any handler accepting a `deliver_token` (or equivalent embedded delivery cap) in its EXECUTE input AND that **stores, persists, or forwards** the token for later use MUST validate the token's authority chain against the EXECUTE author's identity at the time of receipt (GR1). Inbox `receive` is out of scope — the deliver_token serving as the dispatch capability on a delivery EXECUTE is validated by ENTITY-CORE-PROTOCOL.md §5.2 dispatch chain; no separate GR1 check is required at receive.
- `max_events` enforcement — terminate subscription at limit (§4.6)
- `max_duration_ms` enforcement — terminate subscription at limit (§4.6)
- Notification delivery as authenticated inbox EXECUTE per EXTENSION-INBOX.md
- Per-subscription ordering MUST be preserved (§5.2; v3.15): for a single subscription, deliveries arrive in tree-change order. Cross-subscription ordering is impl-defined — parallelism across subscriptions is allowed and recommended.
- The substrate MUST NOT automatically re-register inbox handlers for prior subscription entities on subscriber process restart (§5.7; v3.15). Subscriber-side restoration is application-level — application/SDK code is responsible for restoration via the existing publisher-side subscription persistence + §5.5 gap detection, or via EXTENSION-NETWORK's `maintain-peer` when installed. The MUST NOT ensures consumers can rely on a known substrate scope when building restoration helpers.

### 11.2 SHOULD Implement

- Delivery substrate SHOULD parallelize across subscriptions when serving multiple concurrent subscribers (e.g., shard-by-`subscription_id` worker pools) to scale throughput beyond single-goroutine ceilings (§5.2; v3.15). Workbench-go's Stage 5 K-worker microbench measured near-linear scaling to CPU count (K=4 → 3.8×, K=8 → 5×); shard-by-`subscription_id` is the recommended topology since it preserves the within-subscription ordering MUST. **K-tuning (v3.16, informative):** under hash-based shard assignment, operators expecting uniform per-subscription throughput SHOULD configure `K > 2*N` where `N` is the steady-state subscription count, to bound hash-collision throughput variance (round-2 finding F10; see §5.2 K-tuning advisory).

- Subscription renewal detection by (subscriber, pattern, deliver_uri) match (§5.3)
- `rate_limit` enforcement — drop excess notifications (§4.6)
- Maximum subscription count per subscriber identity (§7.2)
- Capability-gated access to `system/subscription/` (§7.1)
- `hash` and `previous_hash` fields in notifications for gap detection (§5.5)
- Server-side limit tightening when subscriber requests are too permissive (§3.1)
- Subscription capacity advertisement at `system/config/subscription` (§2.7)
- Redirect response (status 303) when subscriber count exceeds `max_subscribers_per_prefix` (§3.1 step 3a)
- Redirect depth limit of 10 hops for subscribers following redirects (§8.3)

### 11.3 MAY Implement

- Entity inclusion in notification envelopes (§4.4)
- Custom entity inclusion strategies (§4.4)
- Inbox handler notification processing via `receive` operation (§3.3)

### 11.4 Implementation-Defined

- Notification delivery retry policy
- Maximum queued notifications per subscription
- Subscription cleanup period for failed deliveries
- Entity inclusion strategy
- Subscription ID generation scheme
- Default values for subscription limits (notification_budget, max_events, max_duration_ms, rate_limit)
- Server-imposed limit maximums
- Notification continuation behavior (what subscriber does on receipt) per EXTENSION-INBOX.md §3.3
- Default `max_subscribers_per_prefix` value (§2.7)
- Intermediate peer data propagation mechanism (§8.2)
- Subscriber-side handler restoration mechanism on process restart (§5.7; v3.15) — typical: SDK helper at boot; alternative: EXTENSION-NETWORK `maintain-peer` auto-restoration.
- Shard-pool topology + shard count `K` for the §11.2 SHOULD parallelization (§5.2 K-tuning advisory; v3.16) — typical: hash-based assignment (FNV / DefaultHasher) with `K = min(NumCPU, 8)`; alternatives: round-robin counter-mod-K, hybrid sticky mapping, or other equivalent. Operators may override per the K > 2N rule of thumb for uniform distribution under known steady-state `N`.
