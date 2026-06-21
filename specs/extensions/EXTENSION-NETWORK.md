# Network Extension — Normative Specification

**Version**: 1.4

> **Amendment 11 — dispatch-fallback seam at §10 step 4 (store-and-forward escalation; new §10.2).** The §10 ladder's step-4 terminal (queue/502) cannot originate delivery to a peer with no live or recurring session that is offline/NAT'd right now. Amendment 11 names a `dispatch_fallback(peer_id, execute) → {ok, result} | null` seam consulted once at step 4 **before** the terminal; the store-and-forward policy lives in RELAY (§6.2.1), never in NETWORK — same layering boundary as the relay→routing `resolve_next_hop` seam. **Additive / v1.x:** `null` (no RELAY installed) ⇒ byte-identical to the pre-seam terminal; non-RELAY v1 floor unchanged. Two cohort-convergence corrections fold in with it: **(a)** the step-4 terminal is restated as impl-variable — `queue_pending` (§8 outbox) is an OPTIONAL rung an impl MAY interleave only if it implements §8 (Rust + Python ship no §8 outbox; their terminal is a bare error), so "byte-identical when unset" means "behaves as this impl's terminal does today"; **(b)** a normative insertion-site MUST — the seam is consulted at the caller holding both `peer_id` and the `execute` envelope, never inside connection-resolution (3-impl independent convergence). Conformance gates the outcome (offline-target delivery lands at the inbox; target polls + verifies signature as direct), not the policy's internal rung choices. No V7 change, no wire change, no new cap/error code. Cohort review converged 3-way before fold (Go build-tested `INBOX-RELAY-FALLBACK-1` PASS; Rust + Python confirmed seam-vs-inline fit against their dispatch ladders — Rust as a hard crate-DAG constraint).

> **Amendment 10 — `serve_scope` MUST cover the trie-node closure when `signed_pointer` is advertised (§6.5.6 `published-set` bullet).** Cross-impl-run absorption: three impls picked three publisher serving-scope shapes (Go = whole-store; Rust = namespace-only; Python = closure-of-signed-root). Only Python's shape lets the PEER-MANIFEST §1.1 walk-from-signed-root complete: CHAMP trie nodes are hash-linked, not path-bound (V7 §1.7), so the §6.5.6 `published-set` path-binding check 404s on interior nodes — and the consumer's `CONTENT_GET` walk from `root_hash` halts. **Resolution:** when a publisher advertises a `signed_pointer` in the same `http-poll` profile, the `serve_scope` content-face MUST also resolve the transitive trie-node closure reachable from `published-root.root_hash` (root + interior + leaf-bound content + `published-root` entity + signature entity). Publishers serving `published-set` without `signed_pointer` (content-only mirror) retain path-bound-only shape — unchanged. `whole-store` trivially covers this case. Cohort impact: Rust either implements closure-scope when keeping `signed_pointer` advertised, or drops `signed_pointer` from the advertised profile to stay namespace-only conformant.

> **Amendment 9 — reserved-word table extensibility hook + length-floor rule (§6.5.6 / G4).** L5 application conventions (sites / repos / spaces and future L5 patterns) need to register URL projection prefixes at the §6.5.6 demux layer so the legacy web can address them. The v0.4 review's first instinct — enumerate the L5 entries inside NETWORK — was a layer violation: NETWORK is the entity system's *networking* extension; it owns the URL routing mechanism and its own reservations, but it has no business cataloguing L5 applications. Amendment 9 makes the layer separation explicit: **(a)** the reserved-word table is **explicitly extensible** — other specs MAY register additional first-segment literals, and the §6.5.6 demux consults the union; **(b)** any registered reserved word MUST satisfy the **length-floor rule** (strictly shorter than the Ed25519 peer-id minimum encoded length) so the literal-then-parse-by-string demux algorithm stays correct regardless of how many words are registered; **(c)** NETWORK does NOT enumerate the registered extensions — each owner spec lists its own. The change is to §6.5.6 G4 (the reserved-words bullet); no new wire surface, no demux algorithm change, no behavior change for existing impls. Companion: `APP-CONVENTION-SEMANTIC-CONTENT-SITE v0.5` registers `sites` as the SITE convention's URL projection prefix in the L5/applications domain.

> **Amendment 8 — session state entity (R6) + reachability-class outbound dispatch (R5); folds the §8 round (Q1 `priority`, Q6 `advertised_at`→optional, Q2 profile-id=final-segment) and drops the R2 inbound-reuse SHOULD.** R6 makes §6.1 literal: per-peer session state (held capability + handshake bookkeeping) becomes a tree entity at `system/peer/session/{remote_peer_id}` (new §6.6), so the held capability survives reconnect and is inspectable, instead of living in per-connection memory. R5 rewrites §10 around the reachability-class table: the dispatcher reads `system/peer/session/{peer}:held_capability` to skip the handshake, resolves the target's transport profiles by `(priority asc, profile-id lex)`, and branches push / queue by the target's reachability class. Q1 adds an optional `priority: uint` to the live profile entities (lower = preferred; default 100; reserved `primary` unset → 0). Q6 relaxes `advertised_at` from MUST to OPTIONAL (it was advisory-only — §6.5.1a D3 — so requiring it was ceremony and a cross-impl footgun). Q2 pins `profile-id` = the final path segment. R2's "SHOULD reuse the accepted inbound socket" prose is dropped to pure impl-detail (post-R6 it carries no auth content). **Landed behind a 3-way-green gate: R6 validated 7/7; Q1 field round-trip + the §10 priority-selection-ordering (Q5) gate all 3-way green (Go/Rust/Py).** Amendment 8 covers PROPOSAL-TRANSPORT-FAMILY-LIVE-REACHABILITY-AND-SESSION-LIFECYCLE §8.9–§10 plus the staged R5–R6 landing. NOT in scope: R4 poll-fallback `delivery_mode` (Phase-2; deferred), REGISTRY, WS/WebRTC build, the cap-axis capability-model ruling (separate track).

> **Amendment 7 — live-transport reachability framing; the §6.5.2c half-duplex caveat corrected (§6.5.1b new + §6.5.2c).** Impl review of HTTP-live bidirectional delivery (entity-core-go `24de569`) found the §6.5.2c parenthetical *"subscribe needs a duplex transport: tcp/websocket"* to be **false**, not merely ambiguous. Subscription/async/peer-initiated delivery is a *fresh EXECUTE from the source to the target's inbox* (EXTENSION-INBOX §2) — the source dials the target — so the constraint is the **target's reachability**, never per-connection duplex. An HTTP-listener target receives deliveries over an ordinary half-duplex source→target POST. WebSocket's real distinction is narrower: it also pushes to a **non-listening** target (browser). Rewrites the §6.5.2c parenthetical to the reachability framing and adds informative §6.5.1b (duplex taxonomy + "active connection" is direction-agnostic for full-duplex transports ⇒ inbound sockets are reusable for outbound dispatch). **Wording / convergence-intent only — no wire change; no behavioral change for tcp / http-listener / ws-server targets (impls already behave this way).** Two items are explicitly NOT in this amendment and travel as a proposal + cross-impl feedback round: (1) capability held per-peer-session vs per-connection (§6.1 leans per-peer; impls lean per-connection); (2) the poll-fallback delivery mode for targets that can neither listen nor hold a duplex socket (Phase-2; new-extension-vs-application-logic open). R1+R2 are this amendment; R3 cap-lifecycle + R4 poll-fallback are proposed/deferred.

> **Amendment 6 — TREE_GET leaf is a hash pointer, not the dereferenced entity (§6.5.3 / §6.5.3.1 / §6.5.6).** Impl feedback on Amendment 5 surfaced an internal contradiction: §6.5.3 step 5 (two-hop) vs §6.5.3.1 leaf bullet (one-hop, dereferenced entity). **Resolved two-hop.** The `http-poll` `TREE_GET` leaf route (`{peer_id}/{path}{tree_leaf_suffix}`) returns the **bound content hash** as a `system/hash` value (bare 2-key `ECF({type:"system/hash", data:H})`), NOT the dereferenced wire entity. The consumer reads `H` and fetches the bytes with a second hop `CONTENT_GET /content/{hex33(H)}`. This is exactly **`tree:get mode:"hash"` (V7 §1.7) over HTTP** — no new type, no V7 bump. One-hop is non-conformant: it materializes a separate entity copy at every tree path bound to `H`, defeating the V7 §1.7 content-store dedup invariant (same content at N paths ⇒ one copy) that a static CDN cannot recover. Serving mode (§6.5.6) returns the same pointer (uniform consumer code path live + static). Also grounds `serve_scope` in V7 §1.4's *Local view* / *Authority layers* model (pointer, not restatement). Cohort impact: one function each (live peers' `serveTreeEntity` → return hash; `validate-peer serving_mode` → assert pointer + second-hop deref; workbench-go → wrap its 33-byte payload in the ECF envelope).

> **Amendment 5 — HTTP pathing standardization (§6.5.3 / §6.5.3.1 / §6.5.6).** Read-surface pathing: listings are **named objects** (`{path}{tree_listing_suffix}`, default `.list`, distinct from `tree_leaf_suffix`) — **no trailing slash** (it doesn't survive static CDNs); append-one/strip-one bijection; `MANIFEST_GET` body+status defined; three-prefix endpoint (+`manifest_url_prefix`); first-segment **literal-or-peer-id-parse** demux with reserved words `{content, manifest, peers}`; canonical status table (200/400/404/405, +414 MAY; no 501; no 3xx); `serve_scope` is a **capability token** (one ACL machinery, evaluated by the live-surface cap evaluator with the published cap as the effective context); listing scope-gating (filtered `count`, empty-in-scope ⇒ 200, post-scope offset); pagination via optional `next_page` on `system/tree/listing` (V7 §3.9). Landed at Rev e after two-round cohort convergence (rust/py/egui/go). CORS + MIME are deployment config (`RUNBOOK-CDN-BROWSER-DEPLOYMENT`), not impl code.
**Status**: Active
**Depends**: ENTITY-CORE-PROTOCOL.md (v7.9+), EXTENSION-INBOX.md (v5.0+), EXTENSION-CONTINUATION.md (v1.2+), EXTENSION-SUBSCRIPTION.md (v3.3+)
**Optional**: EXTENSION-REVISION.md (v2.0+) — reconnection catch-up via revision pull
**Encoding**: ENTITY-CBOR-ENCODING.md (ECF)

---

> **Path notation.** Paths in this document use peer-relative notation (without leading `/{peer_id}/`). All peer-relative paths resolve to the local peer's namespace: `system/tree` means `/{local_peer_id}/system/tree`. Every path in the entity tree is absolute at rest — rooted at a peer identity. See ENTITY-CORE-PROTOCOL.md §1.4 for the path model. Cross-peer examples use absolute paths with explicit peer identities.

## 1. Overview

The network extension manages persistent peer relationships: maintaining connections, detecting failures, reconnecting, restoring subscriptions, and draining pending deliveries. It builds on the reactive pipeline (inbox + continuation + subscription) to express connection lifecycle as entity-native processes.

This extension defines:

- Session vs connection distinction
- Keepalive mechanism
- `maintain-peer` operation and its continuation graph
- Reconnection with backoff
- Subscription restoration after reconnection
- Pending delivery queue
- Graceful close with reason codes

### 1.1 Scope

This extension covers:

- **Connection lifecycle** — liveness detection, failure, reconnection
- **Session management** — relationships that survive connection drops
- **Subscription restoration** — re-establishing notification delivery after reconnect
- **Pending delivery** — queuing outbound messages during disconnection

This extension does **not** cover:

- Peer discovery (mDNS, DNS-SD, seed peers — `EXTENSION-DISCOVERY`, shipped)
- Relay / store-and-forward (`EXTENSION-RELAY`, shipped)
- NAT traversal / WAN reachability — the *reachability facts* (observed-address, dial-back,
  candidate gathering) land as an **amendment to this extension**; the punch-coordination dance is a
  thin protocol over RELAY signaling. See `proposals/PROPOSAL-NAT-TRAVERSAL-AND-WAN-REACHABILITY.md`.
  (Note: the asymmetric-NAT case — reaching a NAT'd peer from a public peer — is *already* covered by
  the `held_connection_client` reachability class, §10.)
- Group coordination (future EXTENSION-NETWORK-GROUP)
- Transport negotiation (ENTITY-CORE-PROTOCOL.md §4)

### 1.2 Design Principles

**Entity-native lifecycle.** Connection lifecycle is expressed as continuation graphs in the entity tree. Subscriptions watch operational state entities; continuations react to state changes. The lifecycle is inspectable, subscribable, and debuggable through the same mechanisms as any other entity data.

**Wire interoperability.** This extension describes entity-native peer lifecycle using continuations and subscriptions. Imperative implementations (reconnection in connection manager code) interoperate — the wire protocol is the same regardless of internal execution mechanism. Operational state types (v7.9 §3.13) enable cross-implementation observability.

**Session over connection.** An authenticated relationship (session) outlives any individual transport connection. Subscriptions, pending deliveries, and capability grants are tied to sessions, not connections. Connection drops are recoverable; session termination is deliberate.

---

## 2. Type Definitions

### 2.1 Maintain Request

```
system/network/maintain-request := {
  fields: {
    peer_id:     {type_ref: "system/peer-id"}
                 ; Peer to maintain a relationship with
    address:     {type_ref: "primitive/string", optional: true}
                 ; Initial address (e.g., "192.168.1.100:4040")
    reconnect:   {type_ref: "primitive/bool", optional: true}
                 ; Auto-reconnect on disconnect (default: true)
    resubscribe: {type_ref: "primitive/bool", optional: true}
                 ; Auto-restore subscriptions on reconnect (default: true)
    keepalive:   {type_ref: "system/network/keepalive-config", optional: true}
                 ; Override keepalive defaults
    backoff:     {type_ref: "system/network/backoff-config", optional: true}
                 ; Override reconnection backoff defaults
  }
}
```

### 2.2 Backoff Configuration

```
system/network/backoff-config := {
  fields: {
    min_ms:   {type_ref: "primitive/uint", optional: true}
              ; Minimum delay between reconnection attempts (default: 1000)
    max_ms:   {type_ref: "primitive/uint", optional: true}
              ; Maximum delay (default: 60000)
    strategy: {type_ref: "primitive/string", optional: true}
              ; "exponential" (default), "linear", "constant"
  }
}
```

### 2.3 Keepalive Configuration

```
system/network/keepalive-config := {
  fields: {
    interval_ms:  {type_ref: "primitive/uint", optional: true}
                  ; Ping interval (default: 30000)
    timeout_ms:   {type_ref: "primitive/uint", optional: true}
                  ; Pong timeout (default: 10000)
    max_missed:   {type_ref: "primitive/uint", optional: true}
                  ; Missed pongs before failure (default: 3)
  }
}
```

### 2.4 Maintain Result

```
system/network/maintain-result := {
  fields: {
    peer_id:       {type_ref: "system/peer-id"}
    session_id:    {type_ref: "primitive/string"}
                   ; Identifier for the maintenance session
    subscriptions: {array_of: {type_ref: "primitive/string"}, optional: true}
                   ; Subscription IDs created for lifecycle monitoring
    chain_id:      {type_ref: "primitive/string"}
                   ; Process chain_id for the lifecycle continuation graph
  }
}
```

### 2.5 Release Request

```
system/network/release-request := {
  fields: {
    peer_id: {type_ref: "system/peer-id"}
             ; Peer to release
    reason:  {type_ref: "primitive/string", optional: true}
             ; "shutdown", "idle", "migration" (default: "shutdown")
  }
}
```

### 2.6 Release Result

```
system/network/release-result := {
  fields: {
    peer_id:    {type_ref: "system/peer-id"}
    cleaned_up: {array_of: {type_ref: "system/tree/path"}}
                ; Tree paths removed during cleanup
  }
}
```

### 2.7 Status

```
system/network/status := {
  fields: {
    maintained_peers: {array_of: {type_ref: "system/network/peer-summary"}}
    pending_count:    {type_ref: "primitive/uint"}
                      ; Total queued outbound messages across all peers
  }
}
```

### 2.8 Peer Summary

```
system/network/peer-summary := {
  fields: {
    peer_id:        {type_ref: "system/peer-id"}
    session_id:     {type_ref: "primitive/string"}
    status:         {type_ref: "primitive/string"}
                    ; "connected", "disconnected", "reconnecting"
    pending_count:  {type_ref: "primitive/uint"}
                    ; Queued outbound messages for this peer
    subscriptions:  {type_ref: "primitive/uint"}
                    ; Active subscription count on this peer
  }
}
```

### 2.9 Close Request

```
system/network/close-request := {
  fields: {
    peer_id: {type_ref: "system/peer-id"}
    reason:  {type_ref: "primitive/string"}
             ; "shutdown", "idle", "error", "migration"
  }
}
```

### 2.10 Pending Delivery

```
system/network/pending-delivery := {
  fields: {
    peer_id:    {type_ref: "system/peer-id"}
    sequence:   {type_ref: "primitive/uint"}
                ; Monotonic sequence for ordering
    execute:    {type_ref: "system/protocol/execute"}
                ; The EXECUTE to deliver
    created_at: {type_ref: "primitive/uint"}
                ; ms since epoch
    expires_at: {type_ref: "primitive/uint", optional: true}
                ; ms since epoch; delivery abandoned after expiry
  }
}
```

Pending deliveries are stored at `system/outbound/{peer_id}/{sequence}`.

---

## 3. Handler

### 3.1 Handler Manifest

```
system/handler := {
  pattern:    "system/network"
  name:       "network"
  operations: {
    maintain-peer: {
      input_type:  "system/network/maintain-request"
      output_type: "system/network/maintain-result"
    }
    release-peer: {
      input_type:  "system/network/release-request"
      output_type: "system/network/release-result"
    }
    status: {
      input_type:  null
      output_type: "system/network/status"
    }
    close: {
      input_type:  "system/network/close-request"
      output_type: null
    }
  }
  internal_scope: [
    {handlers: {include: ["system/tree"]}, resources: {include: ["system/*"]}, operations: {include: ["get", "put"]}},
    {handlers: {include: ["system/subscription"]}, resources: {include: ["system/*"]}, operations: {include: ["subscribe", "unsubscribe"]}},
    {handlers: {include: ["system/protocol/connect"]}, resources: {include: ["*"]}, operations: {include: ["hello", "authenticate"]}}
  ]
}
```

### 3.2 Capability Model

```
system/capability/grant-entry := {
  handlers:   {include: ["system/network"]}
  resources:  {include: ["system/network/*"]}
  operations: {include: ["maintain-peer", "release-peer", "status", "close"]}
}
```

The network handler is a system handler — typically only the local peer admin has grants for it. Remote peers do not call `maintain-peer` on another peer.

---

## 4. Operations

### 4.1 Maintain Peer

Creates a continuation graph that manages the lifecycle of a peer relationship.

```
EXECUTE system/network  operation: "maintain-peer"
  resource: {targets: ["system/network"]}
  params: {
    type: "system/network/maintain-request"
    data: {
      peer_id: "12D3...",
      address: "192.168.1.100:4040",
      reconnect: true,
      resubscribe: true
    }
  }
```

The handler:

1. Connects to the peer if not already connected (handshake via `system/protocol/connect`)
2. Creates operational state entities (v7.9 §3.13):
   - `system/peer/status/{peer_id}` → `"connected"`
   - `system/connection/{peer_id}` → connection details
3. Creates lifecycle subscriptions:
   - Subscription on `system/peer/status/{peer_id}` → detects disconnect
   - Subscription on `system/connection/{peer_id}` → detects connection state changes
4. Creates lifecycle continuations (when `reconnect: true`):
   - Continuation at `system/inbox/network/{peer_id}/on-disconnect` → dispatches reconnection
   - Continuation at `system/inbox/network/{peer_id}/on-reconnect` → dispatches subscription restoration
5. Starts keepalive (§5)
6. Returns session info

```
handle_maintain_peer(ctx, params):
  peer_id = params.peer_id
  session_id = generate_id()
  chain_id = "network/maintain/" + session_id

  ; 1. Connect if needed
  current_status = ctx.entity_tree.get("system/peer/status/" + peer_id)
  if current_status is null or current_status.data.status != "connected":
    connect_result = ctx.execute("system/protocol/connect", "hello", {
      address: params.address or resolve_address(peer_id)
    })
    if connect_result.status != 200:
      return error(502, "connection_failed")

  ; 2. Create reconnection continuation (if reconnect enabled)
  if params.reconnect != false:
    backoff = params.backoff or {min_ms: 1000, max_ms: 60000, strategy: "exponential"}

    ; Continuation that triggers reconnection on disconnect notification
    reconnect_continuation = {
      type: "system/continuation"
      data: {
        target:               "system/protocol/connect"
        operation:            "hello"
        resource:             {targets: ["system/protocol/connect"]}
        params:               {address: params.address or resolve_address(peer_id)}
        result_field:         null        ; trigger mode
        remaining_executions: null        ; standing
        on_error: {
          uri: "system/inbox/network/" + peer_id + "/on-reconnect-backoff"
        }
        dispatch_capability:  ctx.handler_grant.content_hash
      }
    }
    ctx.entity_tree.put(
      "system/inbox/network/" + peer_id + "/on-disconnect",
      reconnect_continuation
    )

    ; Backoff continuation for retry
    backoff_continuation = {
      type: "system/continuation"
      data: {
        target:               "system/network"
        operation:            "maintain-peer"    ; retry the whole flow
        resource:             {targets: ["system/network"]}
        params:               params             ; same config
        result_field:         null
        remaining_executions: 1                  ; one-shot per retry
        dispatch_capability:  ctx.handler_grant.content_hash
      }
    }
    ctx.entity_tree.put(
      "system/inbox/network/" + peer_id + "/on-reconnect-backoff",
      backoff_continuation
    )

  ; 3. Create subscription restoration continuation (if resubscribe enabled)
  if params.resubscribe != false:
    resubscribe_continuation = {
      type: "system/continuation"
      data: {
        target:               "system/network"
        operation:            "restore-subscriptions"    ; internal operation
        resource:             {targets: ["system/network"]}
        params:               {peer_id: peer_id}
        result_field:         null
        remaining_executions: null
        dispatch_capability:  ctx.handler_grant.content_hash
      }
    }
    ctx.entity_tree.put(
      "system/inbox/network/" + peer_id + "/on-reconnect",
      resubscribe_continuation
    )

  ; 4. Subscribe to peer status changes
  disconnect_sub = ctx.execute("system/subscription", "subscribe", {
    resource: {targets: ["system/peer/status/" + peer_id]},
    params: {
      events: ["updated"],
      deliver_to: {uri: "system/inbox/network/" + peer_id + "/on-disconnect"},
      deliver_token: create_deliver_token("system/inbox/network/" + peer_id + "/on-disconnect")
    }
  })

  ; 5. Start keepalive
  start_keepalive(peer_id, params.keepalive or defaults)

  return {
    status: 200,
    result: {
      peer_id:       peer_id,
      session_id:    session_id,
      subscriptions: [disconnect_sub.subscription_id],
      chain_id:      chain_id
    }
  }
```

### 4.2 Release Peer

Tears down the continuation graph and optionally closes the connection.

```
handle_release_peer(ctx, params):
  peer_id = params.peer_id
  reason = params.reason or "shutdown"

  cleaned_up = []

  ; Stop keepalive
  stop_keepalive(peer_id)

  ; Remove lifecycle continuations
  for path in list_paths("system/inbox/network/" + peer_id + "/"):
    ctx.entity_tree.put(path, null)
    cleaned_up.append(path)

  ; Remove lifecycle subscriptions
  for sub in find_subscriptions_by_prefix("system/peer/status/" + peer_id):
    ctx.execute("system/subscription", "unsubscribe", {
      params: {subscription_id: sub.subscription_id}
    })

  ; Drain or discard pending deliveries based on reason
  if reason == "shutdown":
    discard_pending(peer_id)
  else:
    ; Leave pending for potential future reconnection
    pass

  ; Close connection if reason is shutdown
  if reason == "shutdown":
    close_connection(peer_id, reason)
    ctx.entity_tree.put("system/peer/status/" + peer_id, {
      type: "system/peer/status",
      data: {peer_id: peer_id, status: "disconnected"}
    })

  return {
    status: 200,
    result: {peer_id: peer_id, cleaned_up: cleaned_up}
  }
```

### 4.3 Status

Returns a summary of all maintained peer relationships.

```
handle_status(ctx):
  peers = []
  for peer_path in list_paths("system/peer/status/"):
    peer_status = ctx.entity_tree.get(peer_path)
    peer_id = peer_status.data.peer_id

    pending = count_paths("system/outbound/" + peer_id + "/")
    subs = count_subscriptions_for_peer(peer_id)

    peers.append({
      peer_id:       peer_id,
      session_id:    lookup_session_id(peer_id),
      status:        peer_status.data.status,
      pending_count: pending,
      subscriptions: subs
    })

  total_pending = sum(p.pending_count for p in peers)

  return {
    status: 200,
    result: {maintained_peers: peers, pending_count: total_pending}
  }
```

### 4.4 Close

Initiates a graceful connection close with a reason code.

```
handle_close(ctx, params):
  peer_id = params.peer_id
  reason = params.reason

  ; Send close notification to remote peer (best-effort)
  ctx.execute_remote(peer_id, "system/protocol/connect", "close", {
    reason: reason
  })

  ; Update local state
  ctx.entity_tree.put("system/connection/" + peer_id, {
    type: "system/connection",
    data: {
      peer_id:        peer_id,
      transport:      "",
      address:        "",
      status:         "closed",
      established_at: 0
    }
  })

  ; Subscription preservation depends on reason
  if reason == "shutdown":
    ; Remote peer should delete subscriptions from us
    pass
  elif reason == "idle" or reason == "migration":
    ; Subscriptions preserved — connection may resume
    pass

  return {status: 200, result: null}
```

---

## 5. Keepalive

Application-level keepalive detects connection liveness independent of transport-level mechanisms.

### 5.1 Mechanism

The keepalive is an EXECUTE/EXECUTE_RESPONSE exchange on the connection handler:

```
EXECUTE system/protocol/connect  operation: "ping"
  params: {
    type: "system/network/ping"
    data: {timestamp: 1709740800000, sequence: 42}
  }

EXECUTE_RESPONSE
  result: {
    type: "system/network/pong"
    data: {timestamp: 1709740800000, sequence: 42, server_time: 1709740800050}
  }
```

*Implementation note:* WebSocket (RFC 6455 §5.5.2-3) has its own ping/pong mechanism at the transport level. Some WebSocket libraries send transport pings automatically. Transport-level pings detect TCP connectivity; application-level pings (this section) detect protocol-level liveness — a peer may be TCP-alive but protocol-unresponsive (e.g., handler deadlock, event loop stall). Implementations MUST NOT disable application-level keepalive because WebSocket transport pings are active. The two mechanisms are complementary.

### 5.2 Ping Type

```
system/network/ping := {
  fields: {
    timestamp: {type_ref: "primitive/uint"}
               ; Sender's clock, ms since epoch
    sequence:  {type_ref: "primitive/uint"}
               ; Monotonic sequence number
  }
}
```

### 5.3 Pong Type

```
system/network/pong := {
  fields: {
    timestamp:   {type_ref: "primitive/uint"}
                 ; Echoed from ping
    sequence:    {type_ref: "primitive/uint"}
                 ; Echoed from ping
    server_time: {type_ref: "primitive/uint"}
                 ; Responder's clock, ms since epoch
  }
}
```

### 5.4 Failure Detection

```
keepalive_loop(peer_id, config):
  interval_ms = config.interval_ms or 30000
  timeout_ms  = config.timeout_ms or 10000
  max_missed  = config.max_missed or 3
  missed = 0
  sequence = 0

  loop:
    sleep(interval_ms)

    ; Skip ping during active message exchange
    if recent_activity(peer_id, interval_ms):
      missed = 0
      continue

    sequence += 1
    result = execute_with_timeout(peer_id, "system/protocol/connect", "ping", {
      timestamp: now(), sequence: sequence
    }, timeout_ms)

    if result is timeout or result is error:
      missed += 1
      if missed >= max_missed:
        ; Connection failed
        update_peer_status(peer_id, "suspect")
        ; After brief grace period, mark disconnected
        sleep(timeout_ms)
        if not reconnected(peer_id):
          update_peer_status(peer_id, "disconnected")
          ; This tree write fires the disconnect subscription
          ; → continuation handles reconnection (§4.1)
        return
    else:
      missed = 0
      update_last_seen(peer_id, now())
```

**Adaptive pinging.** When the connection is actively exchanging messages, keepalive pings are suppressed. Any successful message exchange resets the missed counter. Keepalive only fires during idle periods.

---

## 6. Session Management

### 6.1 Session vs Connection

A **connection** is a transport-level construct: a TCP socket, a QUIC stream, a Bluetooth channel. It has remote address, buffer state, and protocol-level sequence numbers.

A **session** is a protocol-level relationship: an authenticated peer identity, held capability tokens, active subscriptions, and pending deliveries. Sessions are identified by `peer_id` and persist in the entity tree. The session's authentication state — the capability this peer holds to dispatch to the remote, and the handshake cap it minted in return — is recorded at **`system/peer/session/{remote_peer_id}`** (§6.6, Amendment 8). That entity, not per-connection memory, is the durable answer to "do I already hold a valid capability to talk to this peer, or must I re-handshake?"

Sessions outlive connections. Sessions are identified by `peer_id`, not by transport or address. On reconnection, a peer MAY use a different transport than the original connection (e.g., reconnecting via WebSocket after a TCP connection drop). Transport selection uses the remote peer's `system/transport/*` entities. The `system/connection` entity (v7.13 §3.13) records the current transport for the active connection.

When a connection drops:

- The connection entity status updates to `"closed"` (v7.13 §3.13)
- The peer status updates to `"disconnected"`
- Subscriptions remain in the tree (they're entities, not connection state)
- Pending deliveries accumulate at `system/outbound/{peer_id}/`
- Capability tokens remain valid (subject to TTL) — the `system/peer/session/{peer_id}` entity (§6.6) is **NOT** deleted on disconnect; that persistence is the point

When a new connection is established with the same `peer_id`:

- The session resumes implicitly — same peer identity means same session
- Subscription restoration re-activates notification delivery (§7)
- Pending deliveries drain to the new connection (§8)

### 6.2 Session Resumption

Session resumption is implicit, based on `peer_id` matching during handshake:

```
on_connection_established(peer_id, connection):
  ; Check for existing session state
  existing_status = entity_tree.get("system/peer/status/" + peer_id)

  if existing_status is not null and existing_status.data.status == "disconnected":
    ; Session resumption — restore state
    entity_tree.put("system/peer/status/" + peer_id, {
      type: "system/peer/status",
      data: {
        peer_id:      peer_id,
        status:       "connected",
        connected_at: now(),
        last_seen:    now(),
        connection:   "system/connection/" + peer_id
      }
    })
    ; This tree write fires the reconnect subscription
    ; → continuation handles subscription restoration + pending drain
  else:
    ; New session
    entity_tree.put("system/peer/status/" + peer_id, {
      type: "system/peer/status",
      data: {
        peer_id:      peer_id,
        status:       "connected",
        connected_at: now(),
        last_seen:    now(),
        connection:   "system/connection/" + peer_id
      }
    })
```

*Implementation note:* Reconnection continuations SHOULD resolve the remote peer's current transport and address at reconnection time rather than using a cached address from the original connection. The remote peer's transport entities at `system/peer/transport/{peer_id}/*` reflect currently available transports and may have changed during the disconnection period.

### 6.3 Self-Authenticating Messages

EXECUTE envelopes with valid capability chains are self-authenticating (ENTITY-CORE-PROTOCOL.md §4.6). On reconnection, a peer MAY send an EXECUTE with a previously-held capability token instead of performing a full handshake. If the receiver validates the signature and capability chain, the connection is authenticated:

```
Zero-RTT reconnection:
1. Open connection
2. Send EXECUTE with previously-held capability token
3. Receiver validates: signature valid, capability chain roots at local_peer_id
4. If valid: connection authenticated, process request
5. If invalid: respond with 403, sender falls back to full handshake
```

The "previously-held capability token" in step 2 is read from `system/peer/session/{remote_peer_id}.held_capability` (§6.6) — the cap the remote granted this peer at the original handshake. Because it lives in the tree, it survives the connection drop and (for a persisted identity) a process restart, so zero-RTT reconnection is available without re-deriving the cap from connection state.

This is an optimization, not a requirement. Implementations MAY always perform full handshake on reconnection.

### 6.4 Composition with Identity Rotation

*(Informative.)*

Sessions in this extension are keyed by `peer_id` (§6.1). The identity-rotation surfaces defined in EXTENSION-IDENTITY interact with this layer as follows:

- **Op rotation** (EXTENSION-IDENTITY §5.6, `rotate_operator`): invisible to NETWORK. The deployment's runtime peers retain their `peer_id`s; sessions are unaffected.
- **Public_X rotation** (handoff or compromise-recovery; identity-recognition territory): invisible to NETWORK. Sessions are unaffected.
- **Runtime peer keypair rotation** (rare; rotation typically retires a peer rather than re-keying): produces a new `peer_id`. The rotated peer is structurally a new peer to NETWORK; the old session terminates and a new session is required, with its own `maintain-peer` continuation.
- **Runtime peer retirement** (typical flow; EXTENSION-IDENTITY §5.9, `revoke_peer`): terminates that peer's sessions. Contacts opening new sessions to the deployment connect to a different runtime peer per the registry's resolved endpoints; NETWORK requires no special handling beyond normal session establishment to the new endpoint.

NETWORK does not subscribe to identity rotation events; the layering is one-directional. Identity manages rotation and attestation; NETWORK observes the resulting peer_id population through normal session lifecycle.

### 6.5 Transport Profiles

*Amendment 1.* §6.1, §6.2 reference `system/peer/transport/{peer_id}/*` as the namespace for "what transports does this peer speak." This section defines the entity-type and lands two initial profiles (`quic` and `http-poll`) that v1 needs. Sibling profiles for SMTP/IMAP, Git-over-HTTP, Nix substituter, Tor onion services, etc., are expected from forthcoming bridge specs.

A transport profile entity advertises one way that this peer can be reached. The dispatcher in §10 consults the set during peer dispatch — active connection → held-capability authentication (§6.6) → transport-profile resolution by reachability class (`(priority asc, profile-id lex)`) → queue (§8) → (post-corridor) REGISTRY lookup.

**A transport MAY be store-and-forward.** Nothing in this spec assumes a transport binding requires simultaneous liveness of both peers. Sessions outlive connections (§6.1); pending delivery (§8) handles async traversal; signed mutable pointers and signature verification cover the static / asynchronous transport family. The `freshness` field on each profile names the model.

#### 6.5.1 Profile Entity-Type Definition

```
type: "system/peer/transport/<profile-name>"     ; profile-name is the transport family
data: {
  peer_id:            system/peer-id, ; the peer this profile is for (Base58 id per V7 §2.8 — matches the {peer_id} path segment below, NOT a content hash)
  transport_type:     string,         ; matches the profile-name suffix
  endpoint:           transport-specific (object),  ; e.g. {host, port, alpn} for quic; {base_url} for http-poll
  supported_ops:      string[],       ; closed enum {EXECUTE, TREE_GET, CONTENT_GET, MANIFEST_GET} + reserved SUBSCRIBE (D-13; PROPOSAL-EXTENSION-NETWORK-TRANSPORT-FAMILY §7). Descriptive (what the wire physically carries), never a grant.
  freshness:          "live" | "async" | "static-immutable+signed-pointer",
  nonce_required:     bool,           ; true for live handshake; false for store-and-forward
  cap_flow:           "egress" | "ingress" | "both",   ; with respect to the peer this profile describes
  poll_interval_ms?:  int,            ; for async/static; informative cadence for pollers
  signed_pointer?:    Path,           ; for static; where to fetch the publisher's signed mutable root pointer
  priority?:          uint,           ; lower = preferred; default 100; reserved id `primary` unset → 0 (Q1, §6.5.1a)
  advertised_at?:     time            ; OPTIONAL (Amendment 8, Q6); when this profile was last asserted by the peer
}
```

Profile entities live at `system/peer/transport/{peer_id}/{profile-id}` where `{profile-id}` is a per-peer-unique identifier (e.g. `primary`, `cdn-mirror`, `backup-relay`). A peer MAY have multiple profiles of the same type (e.g. two HTTP mirror URLs for redundancy).

The MUST list:

- `peer_id`, `transport_type`, `endpoint`, `supported_ops`, `freshness`, `nonce_required`, `cap_flow` — all required.
- `poll_interval_ms`, `signed_pointer` — required when `freshness` is `async` or `static-immutable+signed-pointer`; absent when `freshness` is `live`.
- `advertised_at` — **OPTIONAL** (Amendment 8, Q6). It is advisory-only (§6.5.1a D3); requiring an advisory field was ceremony and a cross-impl footgun. `omitempty`; consumers treat absence as "no advisory info" and proceed.
- `priority` — **OPTIONAL** (Amendment 8, Q1); `uint`, lower = preferred, default 100. See §6.5.1a selection.

#### 6.5.1a Selection, self-publication, and field rules

*Amendment 2 (cohort Chunk-C cross-impl rollup).*

**Profile selection (D1).** When a peer publishes more than one profile of the wanted `transport_type`, a consumer builds a **deterministic ordered candidate list** and attempts each in order until one connects. The list is sorted by **`(priority asc, profile-id lex)`** (Amendment 8, Q1):

- **`priority`** — lower value = more preferred (DNS-SRV semantics). Default `100` when unset. The reserved profile-id **`primary`**, when its `priority` is unset, has an implicit `priority` of `0` (preserving the prior "primary first" convention verbatim). An explicit `priority` is always authoritative.
- **`profile-id`** — the **final path segment** of `system/peer/transport/{peer_id}/{profile-id}` (Amendment 8, Q2), independent of whether the location-index layer presents relative or absolute paths. Selection operates on the segment, not on a prefixed form. Used as the lexicographic tie-break among equal-priority profiles, stable across every location-index backend (SQLite collation, in-memory map, dict).

**`advertised_at` MUST NOT be used as a selection key** (it is wall-clock and skew-prone — D3 below). Redundant profiles (mirrors) are ordered fallback via the sort, not a single deterministic pick. (Weight-based load-balance among equal-priority mirrors — SRV `weight` — is NOT v1; equal-priority profiles stay an ordered, lex-tie-broken fallback list. Noted as future.)

Back-compatibility: existing single-`primary` deployments and lex-fallback deployments behave identically — `primary`-unset → 0 sorts first as before; all-unset → all 100 → pure lex order as before. The earlier "name the profile so it lex-sorts after TCP" hack (G1) is no longer needed: set `priority`, name freely.

**Self-publication (D1).** A peer **SHOULD** publish a profile entity for each transport it accepts on, at `system/peer/transport/{peer_id}/{profile-id}`. RECOMMENDED, not REQUIRED — a peer may be reachable only via REGISTRY / manifest / out-of-band (e.g. a browser peer that cannot self-host its tree). **Consumers MUST NOT assume the self-published path exists**; absence falls through to other discovery.

**`transport_type` consistency (D5).** The `transport_type` field MUST match the entity-type suffix (`system/peer/transport/<X>` ⇒ `transport_type: "<X>"`). Decoders MUST reject a mismatch (fail closed). The field is retained (self-describing for tooling that holds the data without the tree path), but the entity-type is authoritative.

**`advertised_at` clock (D3).** Wall-clock epoch ms, **informational only**, judged against the consumer's local clock for advisory staleness. Not a selection key; no logical clock in v1 (deliberate — staleness is advisory, not a correctness input).

**Live-transport `endpoint` shape (D4).** Every **live** transport profile (`tcp`, `websocket`, `http`) carries `endpoint: { url: "<scheme>://..." }` — a single scheme-prefixed `url` field (`tcp://host:port`, `wss://host/path`, `https://host/path`). Per-profile variant shapes (`{base_url, route}`, `{scheme, host, port, path}`) are NOT permitted. `http-poll` is the exception: it advertises lookup *prefixes* (`tree_url_prefix`/`content_url_prefix`), not a single dial endpoint.

#### 6.5.1b Duplex, reachability, and connection reuse (informative)

*Amendment 7.* This note pins cross-impl convergence intent for peer-initiated outbound dispatch. It introduces no new wire format or required behavior; it makes explicit what §6.1 (sessions outlive connections), §6.3 (held-cap reconnection), and the §10 dispatch fallthrough already imply.

**Peer-initiated dispatch is general.** Subscription delivery is one instance of "peer B sends an EXECUTE to peer A": continuation chain-advancement, async inbox delivery, pending-delivery drain, and a long-running handler on B choosing to EXECUTE on A all resolve the same way. The source consults the **target's** transport profiles (§6.5.1a D1 order) and either reuses an active connection or dials one. Because each direction consults the *target's* profiles independently, the two directions need not share a transport ("B→A over `http`, A→B over `tcp`" is ordinary).

**Duplex taxonomy.** Whether one connection serves both directions is a property of the transport:

| Transport | One connection | Server→client push on it | Bidirectional dispatch needs |
|---|---|---|---|
| `tcp` | full-duplex | yes | ONE connection, reused both ways |
| `websocket` | full-duplex | yes — *incl. to a non-listening client (browser)* | ONE connection, reused both ways |
| `webrtc` (future) | full-duplex | yes — *NAT-traversing* | ONE connection, reused both ways |
| `http` | half-duplex | no | each side runs a listener (two connections), OR poll-fallback |
| `http-poll` | one-way fetch | no (no session) | consumer polls; publisher never dials |

**Inbound-connection reuse is an impl-detail optimization** *(Amendment 8 — was a SHOULD in Amendment 7; dropped to impl-detail per proposal R2/§7.1 #4).* For full-duplex transports (`tcp`/`websocket`/`webrtc`) an established connection is bidirectional whether dialed or accepted, so an impl MAY reuse an accepted inbound connection for outbound dispatch rather than dialing a second one. Post-R6 this carries no authentication content (the outbound EXECUTE is signed with the `held_capability` from §6.6, independent of which socket carries the bytes), so it is left to the implementation — not a normative SHOULD and not conformance-gated. For `http` the inbound (server) and outbound (client) sides are necessarily distinct.

**Half-duplex targets that cannot listen** (a CLI/browser client with no listener and no duplex socket) cannot be dialed at all. Delivery to them requires the **poll-fallback**: the source queues to a source-hosted path the target drains via `TREE_GET`. This inverts the standard subscriber-hosted-inbox model and is **Phase-2** — not specified or conformance-gated in v1. See the exploration referenced in Amendment 7.

**Capability lifecycle across connections is per-peer-session** (Amendment 8, R6): the cap is held by the `system/peer/session/{peer_id}` entity (§6.6), not by the connection, so it survives drops and a second connection (any transport) reuses it. This resolves the cap-direction question — the held cap is the dispatcher's auth, independent of which socket carries the bytes.

#### 6.5.2 Profile: `system/peer/transport/quic` (aspirational)

An **optional / future** live transport — **no reference impl ships QUIC yet**. The real default live transport is `tcp` (§6.5.2a). Profile shape retained for when an impl adds QUIC.

```
type: "system/peer/transport/quic"
data: {
  peer_id:        <peer_id>,
  transport_type: "quic",
  endpoint: {
    host:  "peer.example.com",
    port:  4433,
    alpn:  "entity-core/1"
  },
  supported_ops:  ["EXECUTE"],          ; full duplex; all V7 message types
  freshness:      "live",
  nonce_required: true,                  ; V7 §4.6 handshake nonce required
  cap_flow:       "both",
  priority?:      100,                   ; OPTIONAL; default 100 (Amendment 8, Q1)
  advertised_at?: <time>                 ; OPTIONAL (Amendment 8, Q6)
}
```

Test-impl correspondence: a conforming impl SHOULD publish a `quic` profile for its peer-id **if and when** it accepts QUIC connections.

#### 6.5.2a Profile: `system/peer/transport/tcp` (the real default live transport)

TCP is the **actual default** live transport across all three reference impls (Go, Rust, Py). Builds on the existing `"tcp"` transport-value + framing (ENTITY-CORE-PROTOCOL.md §3.13/§1.6); this profile makes it discoverable. Impls SHOULD publish a `tcp` profile for their peer-id.

```
type: "system/peer/transport/tcp"
data: {
  peer_id:        <peer_id>,
  transport_type: "tcp",
  endpoint:       { url: "tcp://host:port" },
  supported_ops:  ["EXECUTE"],           ; full duplex; carries server-push (subscribe)
  freshness:      "live",
  nonce_required: true,
  cap_flow:       "both",
  priority?:      100,                   ; OPTIONAL; default 100 (Amendment 8, Q1)
  advertised_at?: <time>                 ; OPTIONAL (Amendment 8, Q6)
}
```

#### 6.5.2b Profile: `system/peer/transport/websocket`

The **browser-capable full-duplex** live transport — the only browser transport that carries server→client push. Real in Rust (native + WASM); built on the existing `"websocket"` transport-value + framing (ENTITY-CORE-PROTOCOL.md §3.13/§1.6, v7.13).

```
type: "system/peer/transport/websocket"
data: {
  peer_id:        <peer_id>,
  transport_type: "websocket",
  endpoint:       { url: "wss://host/ws" },
  supported_ops:  ["EXECUTE"],           ; full duplex; carries server-push
  freshness:      "live",
  nonce_required: true,
  cap_flow:       "both",
  priority?:      100,                   ; OPTIONAL; default 100 (Amendment 8, Q1)
  advertised_at?: <time>                 ; OPTIONAL (Amendment 8, Q6)
}
```

#### 6.5.2c Profile: `system/peer/transport/http` (live — EXECUTE over POST)

The normal protocol over HTTP request/response: POST an EXECUTE envelope; the response body is the EXECUTE-RESPONSE. **POST-only** (nothing in the protocol is idempotent — no GET sub-mode). **Half-duplex per connection** — a single HTTP connection is connector-driven (client POSTs, server responds) and carries no server→client push. This does **not** block subscription / async / peer-initiated delivery: such delivery is a *fresh* EXECUTE from the source to the **target's** inbox (EXTENSION-INBOX.md §2 — the source dials the target), so what it requires is that the *target be reachable* — publish a live profile (`tcp`, `http`, or `websocket` listener) the source can dial — not that any connection be duplex. A target running an HTTP listener receives deliveries over an ordinary half-duplex source→target POST; two peers that each run an HTTP listener form a full-duplex *pair* across two independent connections. Reachability is per-peer and asymmetric: a delivery B→A consults A's profiles, A→B consults B's, so "B reaches A over HTTP while A reaches B over TCP" is the ordinary case, not a special one. `websocket` (§6.5.2b) is distinguished only in that it *additionally* pushes to a **non-listening** target (e.g. a browser page) down that target's own outbound socket. A target that can neither listen nor hold a duplex socket falls to the poll-fallback (Phase-2 — source-hosted queue the target drains via `TREE_GET`; see §6.5.1b). A **wrapper, NOT BRIDGE-HTTP** — the bytes on the wire ARE entity envelopes (Mechanism A), not foreign content. The browser linchpin (a browser can POST but has no raw socket).

**Body framing (MUST).** The HTTP request and response body is the **bare ECF-encoded envelope** (ENTITY-CORE-PROTOCOL.md §5.3). HTTP message framing — `Content-Length` or `Transfer-Encoding: chunked` — delimits it. The ENTITY-CORE-PROTOCOL.md §1.6 **TCP length prefix MUST NOT be applied**: that 4-byte prefix is *stream-transport framing* (TCP delimiting a medium with no message boundaries), and ENTITY-CORE-PROTOCOL.md §1.6 states framing is per-transport — "other transports define their own framing." HTTP is message-oriented and frames the body natively, so an inner prefix is redundant and creates two conflicting length authorities. This keeps the body a clean CBOR document that composes with `fetch`/`curl`/CDNs/proxies — the browser-and-ecosystem reachability that is the `http` profile's entire reason to exist. (Contrast `websocket` §6.5.2b, which *does* reuse the ENTITY-CORE-PROTOCOL.md §1.6 prefix per its explicit V7 v7.13 blessing — that reuse is stated, not a default, and does not extend to HTTP.) A bad body decode is the substrate-level `400` (entity-protocol errors instead travel *inside* the response envelope under a `200`).

The framing *mechanism* (Content-Length/chunked, no length-prefix) is shared by `http` and `http-poll`; the body *payload* differs by route: the `http` EXECUTE route carries a MaterializedEnvelope (`{root, included}`); the `http-poll` `CONTENT_GET` route carries a single bare-hashable entity `ECF({type, data})` per §6.5.3, the `TREE_GET` **leaf** route carries a `system/hash` pointer `ECF({type: "system/hash", data: H})` (the bound hash, two-hop — §6.5.3.1, Amendment 6; NOT the dereferenced entity), and the `TREE_GET` **listing** route carries a `system/tree/listing` wire entity `ECF({type, data, content_hash})`.

```
type: "system/peer/transport/http"
data: {
  peer_id:        <peer_id>,
  transport_type: "http",
  endpoint:       { url: "https://host/entity" },
  supported_ops:  ["EXECUTE"],           ; half-duplex per connection; delivery TO this peer needs it reachable (HTTP listener), not duplex — §6.5.2c / §6.5.1b
  freshness:      "live",
  nonce_required: true,
  cap_flow:       "both",
  priority?:      100,                   ; OPTIONAL; default 100 (Amendment 8, Q1)
  advertised_at?: <time>                 ; OPTIONAL (Amendment 8, Q6)
}
```

##### 6.5.2c.1 HTTP-substrate conventions (Amendment 4)

A live `http` peer threads V7's session identity across stateless POSTs and follows these conventions:

- **Session header `X-Entity-Session`** (MUST agree cross-impl). Server-allocates on the first POST that lacks it; returns it in the response header; client echoes it thereafter. The session ID is **server-allocated and opaque to the client** (the client never parses it) — its *format* is impl-defined and is NOT a cross-impl constant; it SHOULD carry ≥128 bits of unguessable entropy.
- **Status discipline (SHOULD).** `2xx` ⇔ a response envelope is present, *even if it carries an entity-protocol error* (errors travel inside the envelope). Non-`2xx` ⇔ no envelope: decode failure → `400`, dispatcher failure → `502`, wrong method → `405` (`Allow: POST`; `Allow: GET` on the `http-poll` routes), unknown path → `404`.
- **Default path `/entity`** (conventional, not normative — operators advertise the actual path in `endpoint.url`). **Content-Type `application/cbor`** (SHOULD set).
- **Idle-session eviction is an operator knob** (conservative default; NOT a cross-impl constant — two peers needn't share a TTL, since a client re-handshakes on eviction).
- **Hex strictness (MUST).** Content-hash hex on the `http-poll` routes is the strict 66-char wire form (§3.5; §6.5.3). A 64-char digest-only request → `400` (no back-compat shim — no deployed legacy data; clean-break). A hash whose algorithm byte the peer cannot verify → `400` (fail-closed; forward-compatible with SHA-384/512).

#### 6.5.3 Profile: `system/peer/transport/http-poll`

The static-HTTP transport binding for **Mechanism A** (HTTP-as-storage-transport, per `GUIDE-EXTENSION-DEVELOPMENT.md §3.7`). A publisher peer using this profile has published its tree + content to a static HTTP origin (typically a CDN-backed bucket); consumers poll and fetch the bytes **directly** — inline HTTP GET + content-hash verification, NO `EXTENSION-BRIDGE-HTTP` involvement. The bytes-on-wire ARE entity-encoded; the substrate's only role is byte storage. (BRIDGE-HTTP is Mechanism B — foreign-substrate fetch of HTML/JSON/arbitrary content to wrap as `system/bridge/http/fetched`; it is NOT on this path. See §6.5.5 for the two consumer modes and EXTENSION-BRIDGE-HTTP.md §1 (planned) for the scope boundary.)

**Three-prefix URL space (Amendment 5).** The profile carries THREE independently-configured URL prefixes (per the Amendment-5 pathing standardization):

- `tree_url_prefix` — tree-path-keyed URLs. **Every form is a concrete object key (no trailing slash):** `{tree_url_prefix}/{peer_id}/{path}{tree_leaf_suffix}` serves the entity at that path; `{tree_url_prefix}/{peer_id}/{path}{tree_listing_suffix}` serves the listing; `{tree_url_prefix}/{peer_id}{tree_listing_suffix}` the peer root; `{tree_url_prefix}/peers{tree_listing_suffix}` the all-peers (universal-tree-root) listing. See §6.5.3.1.
- `content_url_prefix` — content-hash-keyed URLs: `{content_url_prefix}/{layout-path}/{hash}` serves the bytes for that hash.
- `manifest_url_prefix` — the singular signed manifest: `{manifest_url_prefix}` (terminal; no suffix, no trailing slash).

These three prefixes MAY be **co-located** on one origin (demuxed by first path segment per §6.5.6 — reserved literals `content`/`manifest`/`peers`, else a parseable peer-id ⇒ tree; in co-located form `tree` has no segment word) OR be **entirely separate origins** (tree on one CDN, content on a dedup bucket, etc.). Decoupling supports multi-peer-shared-domain deployments (peer-scoped prefixes) and cross-peer content-store deduplication (hash-keyed, peer-agnostic).

```
type: "system/peer/transport/http-poll"
data: {
  peer_id:        <publisher-peer-id>,
  transport_type: "http-poll",
  endpoint: {
    tree_url_prefix:    "https://shared.example.com/peers/<peer_id>",
                                       ; URLs: {tree_url_prefix}/{tree-path}
    content_url_prefix: "https://shared.example.com/peers/<peer_id>/content",
                                       ; OR shared-dedup: "https://shared.example.com/content"
                                       ; URLs: {content_url_prefix}/{layout-path}/{hash}
    content_layout:     "flat" | "sharded-2-flat" | "sharded-2-4" | "sharded-2-2"
                                       ; pure hash-layout; NO peer-namespacing concern
                                       ;   flat:           /{hash}
                                       ;   sharded-2-flat: /{hash[0:2]}/{hash}
                                       ;   sharded-2-4:    /{hash[0:2]}/{hash[2:4]}/{hash}
                                       ;   sharded-2-2:    /{hash[0:2]}/{hash[2:4]}/{hash}  (alias for sharded-2-4)
    tree_leaf_suffix:   ".bin"         ; default ".bin"; consumers append for ENTITY (leaf) URLs
                                       ;   (operator-overridable; append-one/strip-one per §6.5.3.1)
    tree_listing_suffix: ".list"       ; default ".list"; consumers append for LISTING URLs
                                       ;   MUST differ from tree_leaf_suffix (Amendment 5)
    manifest_url_prefix: "https://shared.example.com/peers/<peer_id>/manifest"
                                       ; singular signed manifest; terminal, NO suffix, NO trailing slash
  },
  supported_ops:  ["TREE_GET","CONTENT_GET","MANIFEST_GET"],  ; D-13 closed enum; partial publisher MAY advertise any non-empty subset (content-only mirror = ["CONTENT_GET"]); distinct anchors: TREE_GET→hash-chain-from-root, CONTENT_GET→hash, MANIFEST_GET→signature. No EXECUTE; no writes; no subscribe (consumer polls).
  freshness:      "static-immutable+signed-pointer",
  nonce_required: false,                 ; static has no session; signatures self-authenticate
  cap_flow:       "egress",              ; with respect to the publisher: they push to the static store (egress); consumers fetch
  poll_interval_ms: 60000,               ; informative; consumers may poll less or more often
  signed_pointer: "system/peer/published-root",   ; canonical signed root pointer for staleness/freshness
  advertised_at?: <time>                 ; OPTIONAL (Amendment 8, Q6)
}
```

**Multi-peer-shared-domain example (the load-bearing case).** When a single domain owner hosts content from multiple peers, each peer's profile embeds its peer-ID literally in the prefix strings:

```
For peer A:
  endpoint.tree_url_prefix:    "https://shared.example.com/peers/<A>"
  endpoint.content_url_prefix: "https://shared.example.com/peers/<A>/content"   ; OR shared: "https://shared.example.com/content"

For peer B:
  endpoint.tree_url_prefix:    "https://shared.example.com/peers/<B>"
  endpoint.content_url_prefix: "https://shared.example.com/peers/<B>/content"   ; OR shared: "https://shared.example.com/content"
```

No template-substitution at consume time — the peer_id is part of the literal prefix string the publisher author writes into their transport profile entity. Consumer uses the prefix as-is.

**Deployment scenarios** (see audit doc §2 for the full stress-test):
- S1 — single peer dedicated domain: `https://peer-a.example.com` + `/content`
- S2 — single peer on CDN with path prefix: `https://cdn.example.com/my-peer` + `/content`
- S3 — multi-peer shared domain (per-peer content): `https://shared.example.com/peers/<peer_id>` + `/content`
- S4 — multi-peer shared with deduplicated content: `https://shared.example.com/peers/<peer_id>` (tree) + `https://shared.example.com/content` (shared)
- S5 — peer publishing to multiple domains: multiple profiles per peer
- S6 — subdomain-per-peer: `https://peer-a.cdn.example.com` + `/content`

All work cleanly under the two-prefix model.

The consumer-side flow:

1. Consumer needs a binding/content from a publisher peer P.
2. Local content-store / tree miss; dispatcher (§10) falls through.
3. Dispatcher resolves `system/peer/transport/{P}/*` → finds `http-poll` profile with `tree_url_prefix` + `content_url_prefix` + `content_layout` + `signed_pointer`.
4. For hash-keyed lookups: dispatcher builds URL = `{content_url_prefix}/{layout-path}/{hash}`, performs an **inline HTTP GET** of the bytes, computes the content hash over the body, verifies it equals `{hash}` (MUST; mismatch → discard, do not ingest), then ingests via `system/content:ingest` into `target_namespace`. No `system/bridge/http:get`; no `system/capability/bridge-http-fetch` cap — the trust anchor is the content hash.
5. For tree-path lookups (less common; typically consumers walk refs from known root): dispatcher builds URL = `{tree_url_prefix}/{tree-path}{tree_leaf_suffix}` and fetches the **bound hash** — the body is a `system/hash` pointer (`ECF({type:"system/hash", data:H})`, §6.5.3.1), NOT the dereferenced entity; reads `H` from `data`; falls through to step 4 (`CONTENT_GET`) for the actual content. **Two hops** — the tree route resolves `path → hash`, the content route resolves `hash → bytes` (ENTITY-CORE-PROTOCOL.md §1.7); collapsing them would duplicate entity bytes per-path on a static CDN.
6. For mutable-binding lookups, consumer fetches the signed pointer first; verifies signature; then fetches the resolved target via step 4.

**This is Mechanism A, not BRIDGE-HTTP.** The profile names the endpoint and op-set; the consumer (dispatcher-driven in Mode A1, operator-tool in Mode A2 per §6.5.5) performs the fetch + hash-verify **inline**. There is no BRIDGE-HTTP handler and no bridge cap surface on this path — the bytes are already entity-encoded and the content hash is the sole trust anchor. BRIDGE-HTTP (Mechanism B) is a structurally-distinct surface for foreign content (HTML/JSON) and is not used here. Empirically validated by workbench-go's `entity-fetch` (stdlib `net/http` + hash-verify, no BRIDGE-HTTP import).

**Why this lands in NETWORK.** `system/peer/transport/*` is the general transport-pluggability namespace; the `http-poll` profile is one transport among many (alongside `quic`, future libp2p/etc.). Landing per-profile entities here keeps the transport slot consistent and independent of any bridge extension.

##### 6.5.3.1 Route response semantics (Amendment 4)

The `http-poll` routes (`CONTENT_GET`, `TREE_GET` entity + listing, `MANIFEST_GET`) are content-addressed fetch over HTTP — the consumer trusts the math, not the host. Validated three-way (Go/Rust/Py via `validate-peer -category serving_mode`). Per the Amendment-5 pathing standardization.

- **`CONTENT_GET` body (MUST).** A `GET` for content hash `H` returns the **bare hashable form `ECF({type, data})`** — the **2-key** CBOR map of the entity whose `content_hash` is `H` — `Content-Type: application/cbor`. This is **NOT** the 3-key wire entity carrying `content_hash` (that value hashes `{type, data}` and so cannot appear in its own preimage). The body **MUST satisfy pure-body rehash**: a consumer computes `0x00 ‖ SHA-256(body)` and accepts only if it equals `H` (Mechanism A — verifiable by a CDN/browser with no ECF decoder). Serving the 3-key entity, the inner `data`, or a chunk's raw payload is non-conformant. Raw-bytes/rendering delivery (reassembled blob + MIME) is a consumer-side concern (CONTENT §5.3 descriptor), not this route.
- **`TREE_GET` leaf (hash pointer) vs listing — named objects, no trailing slash (MUST; Amendment 5; leaf-body per Amendment 6).** Two configurable suffixes (§6.5.3), REQUIRED distinct: `tree_leaf_suffix` (default `.bin`) and `tree_listing_suffix` (default `.list`). There is **no trailing-slash listing form** (it does not survive static CDNs — index-document convention + per-CDN slash normalization; and a normalizing `foo/`→`foo` would 404 a listing by the no-redirect rule below). Every read URL is a concrete object key:
  - `{tree_prefix}/{peer_id}/{path}{tree_leaf_suffix}` ⇒ the **bound content hash** — a `system/hash` value in the **bare 2-key form `ECF({type: "system/hash", data: <33-byte H>})`**, `Content-Type: application/cbor`. The tree is `path → hash` (ENTITY-CORE-PROTOCOL.md §1.7; EXTENSION-TREE.md §1); this route returns the hash, **not** the dereferenced entity. It is exactly `tree:get mode:"hash"` (ENTITY-CORE-PROTOCOL.md §1.7) exposed over HTTP. The consumer reads `data` (the bound hash `H`) and fetches the entity bytes with a second hop `CONTENT_GET {content_prefix}/{hex33(H)}` (§6.5.3 step 5). **Two-hop, normative (Amendment 6).** Returning the dereferenced wire entity `ECF({type, data, content_hash})` here is **non-conformant** — it materializes a separate copy of the entity at every tree path bound to `H`, defeating the content-store dedup invariant (ENTITY-CORE-PROTOCOL.md §1.7: same content at N paths ⇒ one copy) that a static CDN cannot recover. Body is the bare 2-key form (one hash, no ambiguity), not 3-key: a path-addressed pointer has no useful self-`content_hash`, and a 3-key body would carry two hashes (the bound `data` and the pointer's own `content_hash`), forcing the consumer to disambiguate. **`tree_leaf_suffix` is still `.bin` by default** — the suffix names "the leaf at this path," and the leaf *is* its bound hash.
  - `{tree_prefix}/{peer_id}/{path}{tree_listing_suffix}` ⇒ the **listing** (below). `{tree_prefix}/{peer_id}{tree_listing_suffix}` ⇒ the peer-root listing; `{tree_prefix}/peers{tree_listing_suffix}` ⇒ the all-peers (universal-tree-root) listing.
  - **Append-one / strip-one (MUST):** append exactly one suffix; the server strips exactly one recognized suffix; the suffix *identity* selects entity vs listing. Total bijection for any name (entity `foo`⇒`foo.bin`, listing `foo`⇒`foo.list`, entity `foo.bin`⇒`foo.bin.bin`, listing `foo.bin`⇒`foo.bin.list`) — no publish-time check.
  - A bare path with no recognized suffix ⇒ `404`; `{peer_id}{tree_leaf_suffix}` (entity at a peer-id root) ⇒ `404` (roots are directories, ENTITY-CORE-PROTOCOL.md §1.4); bare `peers` (no listing suffix) ⇒ `404`. **No redirects** — no URL form depends on a trailing slash, so there is nothing to canonicalize.
  - A path that is both a bound entity and a parent of children answers the leaf-suffix form with the entity and the listing-suffix form with the listing (mirrors TREE §2.2; MUST NOT impose a file-or-directory exclusivity).
- **Listing body (MUST).** The **existing `system/tree/listing` entity** (ENTITY-CORE-PROTOCOL.md §3.9) — wire entity in ECF, `Content-Type: application/cbor`, shape `{path, entries: {name → {hash?, has_children}}, count, offset, next_page?}`. **Mutable view:** carries its own `content_hash` but MUST NOT be `immutable`-cached (re-rendered on subtree change). `count` is the **in-scope filtered** total (never the raw subtree total — §6.5.6 scope-gating + TREE §1176). **Pagination** is the optional `next_page` content hash: if more entries remain, the head listing carries `next_page` = the hash of the next `system/tree/listing` page, fetched via `CONTENT_GET /content/{hex33}` (immutable, content-addressed). The publish pipeline MUST bind each chain page into the served content namespace (EXTENSION-CONTENT.md §6.4.2) or it is unreachable. Live peers MAY also honor `?offset=&limit=` over the **scope-filtered** child set (independent of the `next_page` chain — not a seek into it). No JSON form.
- **`MANIFEST_GET` body (MUST).** `GET {manifest_prefix}` returns the publisher's signed manifest as a **wire entity `ECF({type, data, content_hash})`** (`system/peer/published-root` / static-handshake manifest per `PROPOSAL-PEER-MANIFEST-STATIC-HANDSHAKE`), `Content-Type: application/cbor`, carrying `content_hash` + signature ref. Singular/terminal: no suffix, no trailing slash; `GET {manifest_prefix}/` ⇒ `404`; none published ⇒ `404`. The manifest is **mutable** (revocation lives here) ⇒ MUST NOT be `immutable`-cached; its revocation primitive is defined in `PROPOSAL-PEER-MANIFEST-STATIC-HANDSHAKE` (this section pins only the cache bound).
- **Hash hex = 66, format-code included (MUST).** The content hash in the URL (`/content/{hex(H)}`), the EXTENSION-CONTENT.md §6.4.2 tree binding key, and the `ETag` all use the ENTITY-CORE-PROTOCOL.md §3.5 convention: **lowercase hex of the full content hash *including the format-code byte* — 66 chars beginning `00` for ECFv1-SHA-256** (`hex33`), NOT the 64-char digest-only hex. This keeps URL ⇄ binding parity (`in_scope(H)` = `tree:get({ns}/{same-hex})`, no boundary conversion) and preserves the algorithm discriminator for crypto-agility. (See EXTENSION-CONTENT.md §6.4.2.)
- **Sharded `content_layout` (`flat` / `sharded-2-flat` / `sharded-2-4` / `sharded-2-2`) slices the same 66-hex `{hash}`.** Slices are positions in that string: `[0:2]` = the format-code byte (`00` for ECFv1-SHA-256) = the **algorithm partition** (one bucket today; auto-diversifies the instant a non-SHA-256 hash ships); deeper slices shard the digest. `sharded-2-4` → `/00/{d0}/{wire}`. The shard prefix is a slice of the leaf id (git-style CAS); characters repeating between dir path and leaf are by construction. One definition of `{hash}` everywhere — no separate digest-sliced layout family.
- **Cache-Control (MUST).** `CONTENT_GET` content is **immutable** (the hash addresses fixed bytes forever) → `Cache-Control: immutable, max-age=<long>`. `TREE_GET` bindings are **mutable** (a path may be rebound to different content) → impls **MUST NOT** emit `immutable` on `TREE_GET` (it would make caches serve stale bindings after a rebind); omit it or use a revalidation-friendly directive. `ETag: "{hex33(H)}"` on both, where **`H` is the addressed content hash**: for `CONTENT_GET` the URL hash; for the `TREE_GET` leaf the **bound hash carried in the pointer's `data`** (the hash the path currently resolves to — Amendment 6), NOT the pointer entity's own content hash; for a listing the listing entity's `content_hash`. A leaf `ETag` thus changes exactly when the path is rebound, which is the correct mutable-binding cache key. *(Rust surfaced the immutable rule; Go/Python drop `immutable` on tree-get.)*
- **Status codes (MUST — all `http-poll` read routes; Amendment 5).**
  - `200` — leaf hash-pointer (`system/hash`, §6.5.3.1) · listing (incl. empty in-scope, `entries={}` `count=0`) · content bytes · manifest found.
  - `400` — malformed: 64-char digest-only hash, unverifiable algorithm byte (§6.5.2c.1), suffix/slash on a hash, malformed percent-encoding, **an encoded slash (`%2F`) inside a path component** (path components are `/`-delimited; reject rather than recover as a literal).
  - `404` — not found AND no recognized suffix · out-of-scope (**identical** to not-held — T4 presence-oracle) · `{manifest_prefix}/` (terminal) · bare unknown first segment · `{peer_id}{leaf_suffix}` (root is a directory) · bare `peers`.
  - `405` — wrong method: `Allow: GET` on read routes; `Allow: POST` on the EXECUTE route.
  - `414` — **MAY**, request URL exceeds the operator-configured cap (RECOMMENDED 8 KB per RFC 7230 §3.1.1; tree paths are uncapped per ENTITY-CORE-PROTOCOL.md §1.4, so a length bound is the parser-DoS guard).
  - `501` is **not** a conformant steady-state response on any route in a shipped peer. **No `3xx`** — no URL form depends on a trailing slash.

#### 6.5.4 Discovery and freshness

How a consumer learns of a publisher's transport profiles is out of scope of this section. For v1: out-of-band (well-known URL, hard-coded, manually configured). Post-corridor: the EXTENSION-REGISTRY.md extension will define peer-ID → endpoint-set resolution.

Freshness of a profile entity is the publisher's concern. Stale profile entries (publisher advertised then dropped a transport) are handled by the consumer's poll cycle on the signed pointer + retry semantics on the bridge call (`network_error` if endpoint unreachable; consumer attempts other profiles).

#### 6.5.5 Consumer modes (A1 dispatcher-driven, A2 operator-driven)

A Mechanism A consumer (per `GUIDE-EXTENSION-DEVELOPMENT.md §3.7` — HTTP-as-storage-transport, NOT HTTP-as-foreign-substrate) MAY operate in either of two modes. Both are conformant. The conformance contract is the same: hash-verify on every fetched byte stream + signature-verify on the manifest (when present per `EXTENSION-STORAGE-SUBSTITUTE-HTTP.md §3-RES.4` (planned)).

**Mode A1 — Dispatcher-driven (peer-resident).** Consumer is a full peer participating in dispatch (§10). On local content-store miss the dispatcher resolves `system/peer/transport/{publisher_peer_id}/*` → finds the `http-poll` profile entity (§6.5.3) with `tree_url_prefix`, `content_url_prefix`, `content_layout`, `tree_leaf_suffix`, and (optionally) `signed_pointer`; builds the URL; performs GET + hash-verify (no BRIDGE-HTTP import — Mechanism A); ingests via `system/content:ingest` into `target_namespace`.

**Mode A2 — Operator-driven (verifying client).** Consumer is a standalone tool (CLI fetcher, mirror utility, debugging client); NOT a peer; no dispatch participation. Receives URL prefix + publisher peer-id + path directly (CLI flags / config / env); builds the URL by hand using the same convention (`{tree_url_prefix}/{path}{tree_leaf_suffix}` for tree-leaves; `{content_url_prefix}/{layout-shape}/{hash}` for content); performs GET + hash-verify by hand; output is the entity bytes (inspection / mirroring / debugging). No `target_namespace`; no ingest.

The trust anchor in both modes is the content-hash + (when present) the manifest signature. The rest of §6.5's profile-entity / dispatcher / ingest apparatus is REQUIRED for A1 and OPTIONAL for A2.

**Conformance.** An impl SHOULD provide Mode A1. It MAY additionally provide Mode A2 (operator tool) for deployment / debugging / mirroring. Implementations MUST NOT publish manifests inconsistent between modes — the URLs A1 builds via dispatch MUST match what A2 receives via CLI for the same `(publisher_peer_id, path)` pair, since §6.5.3's URL-construction rules are common to both.

> **Scope note.** §6.5.5 names the two consumer *shapes*; it does NOT specify the substrate transport-composition mechanism (registration / selection / translation / fallback loop with registry re-resolve / mix-and-mash / extensibility). That mechanism is a separate forthcoming exploration. A1 vs A2 is about who holds the URL, not how transports compose.

#### 6.5.6 Serving mode — a live peer exposing `http-poll` routes (Amendment 4)

A live peer MAY expose `http-poll` content/tree/manifest routes from its own store on the **same HTTP listener** as its `http` EXECUTE route, composing both profiles per §6.5 ¶721 (validated three-way). This collapses the "live peer + separate CDN" two-process story into one box (dev loop, embedded deployments, CDN origin).

- **Route demux (G2) — literal-or-peer-id-parse (MUST; Amendment 5).** One listener. Demux the first path segment **in order** (this is NOT a length threshold — there is a 9–45-char gap between the reserved words and a peer-id; the *check* is literal-then-parse): (1) literal `content` → `GET {content-prefix}/{hex33(H)}` content-by-hash; (2) literal `manifest` → `MANIFEST_GET`; (3) literal `peers{tree_listing_suffix}` (`peers.list`) → all-peers (universal-tree-root) listing — bare `peers` ⇒ `404`; (4) else **parse as a peer-id** — success → `TREE_GET {tree-prefix}/{peer_id}/{path}{suffix}` (leaf **hash-pointer** or listing by suffix, §6.5.3.1 — the leaf-suffix route returns the bound `system/hash`, NOT the dereferenced entity, identically to the static surface so consumers have one code path; Amendment 6); (5) `POST {http-path}` → EXECUTE; (6) else `404`. **Tree has no reserved word** — a parseable-peer-id first segment *is* the tree signal (co-located drops the `tree/` segment). The reserved literals `{content, manifest, peers}` can't collide with a peer-id (a peer-id parse of those short strings fails). No separate HTTP server is required.
- **`serve_scope` exposes the local view (grounding — ENTITY-CORE-PROTOCOL.md §1.4).** A serving peer publishes a scoped projection of its **local view of the universal tree** (ENTITY-CORE-PROTOCOL.md §1.4 *Local view* + *Authority layers*): paths under `/{any_pid}/...` it holds — its own authoritative namespace and any cached/mirrored remote namespaces. `serve_scope` declares *which slice of that local view* this listener answers for; it is not a claim of cross-peer authority (the keyholder for `{pid}` remains canonical — ENTITY-CORE-PROTOCOL.md §1.4 layer 2). A `peers.list` enumerating multiple peer-ids is the **normal universal-tree-root view** (every peer's root is a set of peer-ids it holds), not a multi-tenant feature. The cross-layer discipline that every path-taking layer be peer-id-agnostic — and its conformance guard — lives in `GUIDE-SERVING-MODE.md` (the model itself is ENTITY-CORE-PROTOCOL.md §1.4, not restated here).
- **Served-set scope (`serve_scope`) — a capability token (Amendment 5).** The read routes carry **no request auth** (the client may not speak the protocol) — hash-knowledge (content) / path-presence (tree) is the read authority. The security lever is **which entities the routes answer for**, and `serve_scope` is a literal `system/capability` token the publisher generates, evaluated by **the same cap evaluator the live-EXECUTE surface uses** (`check_permission` / `check_path_permission`). Because the request is unauthenticated, the **published `serve_scope.cap` IS the effective cap** passed to the evaluator: where the live surface asks "does the connection's cap-set permit `get(path)`?", the serving surface asks "does `serve_scope.cap` permit `get(path)`?" — one ACL machinery, no drift. Impls MUST pass the published cap as the effective cap (MUST NOT reach into capability internals or synthesize a connection context). The published-set / whole-store distinction below is expressible as cap shapes, in EXTENSION-CONTENT.md §6.4.1 topology terms:
  - **`published-set` (SHOULD default)** = EXTENSION-CONTENT.md §6.4.1 namespace-scoped topology. A `CONTENT_GET(H)` resolves iff EXTENSION-CONTENT.md §6.4.2 Hash Tree Presence holds (`tree:get({namespace}/{hex33(H)}) ≠ null`). Populate by binding shareable hashes under a content namespace (`system/content/{ns}/{hex33(H)}`) and/or the closure of a published subtree.
    - **Signed-root closure (MUST when `signed_pointer` advertised; Amendment 10).** A publisher that advertises a `signed_pointer` (i.e. participates in the §6.5.3 / PEER-MANIFEST-STATIC-HANDSHAKE.md §1.1 (planned) walk-from-signed-root threat model) MUST also resolve, under the same `serve_scope`, the **transitive trie-node closure reachable by hash-link from `published-root.root_hash`** — the CHAMP trie root node + all interior nodes + all leaf-bound content hashes + the `published-root` entity + its `system/signature` entity at the ENTITY-CORE-PROTOCOL.md §5.2 invariant-pointer path. Trie nodes are hash-linked, not path-bound (ENTITY-CORE-PROTOCOL.md §1.7), so the bare path-binding check above does not reach them; without the closure, `CONTENT_GET(H)` 404s on interior trie nodes and the hash-chain walk from `root_hash` cannot complete — the `signed_pointer` machinery is non-functional in its premised security model. Mechanism is impl choice (precompute closure at publish time and stamp into the served-set, or compute lazily via a `trie_reachable_from(root_hash, H)` check); the contract is "MUST resolve." A publisher who serves `published-set` WITHOUT advertising `signed_pointer` (content-only mirror, no signed-root walk) retains the path-bound-only shape — unchanged. `whole-store` trivially covers this case.
  - **`whole-store` (explicit opt-in)** = EXTENSION-CONTENT.md §6.4.1 single-trust-domain topology; serves any stored hash. MUST be explicit, documented, default-off (EXTENSION-CONTENT.md §6.4.1 marks multi-party use of this "security-defective").
  - A serving impl SHOULD be able to bound its served set to a published-set. **Out-of-scope and not-held MUST return an identical `404`** (T4 presence-oracle mitigation).
- **Two faces of the published set.** `serve_scope` has a **tree-face** (which tree paths `TREE_GET` resolves) and a **content-face** (which hashes `CONTENT_GET` resolves via EXTENSION-CONTENT.md §6.4.2). `TREE_GET` over poll is **published-scope-gated, not unsupported** — it resolves iff the path is within the served subtree; `501` is never the normative shape. A peer publishes a subtree; both its tree paths and the content reachable from it are served under the same `serve_scope`.
- **Listings are scope-gated (MUST; Amendment 5).** A listing enumerates only children within `serve_scope`; `count` MUST be the **in-scope filtered** total (never the raw subtree total — a discrepancy leaks hidden-path existence, TREE §1176). An out-of-scope or non-existent prefix returns `404` **identical** to not-held (T4). An **in-scope** prefix with no children returns `200` + `entries={}` + `count=0` (in-scope-ness is the access boundary; an empty published directory is legitimately observable). Live `?offset=&limit=` MUST run **post-scope** (offset numbering over raw children leaks an offset oracle).
- **G4 — reserved words (normative; Amendment 5; extensibility hook added Amendment 9).** The co-located demux reserves the literal first-segment words **`{content, manifest, peers}`** (§6.5.6 demux); operators MUST pick a live EXECUTE `POST` path that is none of the three and not a parseable peer-id. The reserved-word table is **explicitly extensible**: other specs MAY register additional first-segment literals for their own purposes (e.g. an L5 application convention reserving a URL projection prefix). The §6.5.6 demux consults the union of NETWORK's reservations and registered extensions. **Length-floor rule (normative; Amendment 9).** Any registered reserved word MUST be strictly shorter than the Ed25519 peer-id minimum encoded length (which sets the floor of the "9–45-char gap" observation above) so that a parse-as-peer-id can never succeed on a reserved word — preserving the literal-then-parse-by-string demux algorithm unchanged regardless of how many words are registered. NETWORK does NOT enumerate the registered extensions; each owner spec lists its own.

Request-side *who*-restriction (where a deployment needs it) is **foreign auth** — OAuth / HTTP Basic / mTLS / API gateway via a reverse proxy — a deployment concern outside this spec (a future `bridge-oauth` only on a driver). Serving mode is for mostly-public sharing by design. Operator guidance (closure computation, the `whole-store` caveat + the T2/T3 confidentiality audit, foreign-auth context, content-store hygiene) lives in `GUIDE-SERVING-MODE.md`.

### 6.6 Session State Entity (`system/peer/session/{peer_id}`)

*Amendment 8.* §6.1 states that a session's held capability tokens "persist in the entity tree." This section makes that literal: per-peer authentication state is a tree entity, not per-connection memory. The entity is the **durable per-peer AUTH record** — it answers exactly one question for §10 dispatch: *do I already hold a valid capability to dispatch to this peer, or must I re-handshake?* It is **not** the liveness or reachability record; those are the other three entities of the per-peer set (§6.1, ENTITY-CORE-PROTOCOL.md §3.13, §6.5):

| Dispatch question | Answered by |
|---|---|
| Do I hold a valid cap (skip handshake)? | **`system/peer/session/{peer}`** — `held_capability` + `expires_at` |
| Is there a live connection to reuse? | `system/connection/{peer}` (ENTITY-CORE-PROTOCOL.md §3.13) |
| What transport / poll-fallback? | `system/peer/transport/{peer}/*` profiles + reachability classes (§6.5.1b) |
| Connection lifecycle? | `system/peer/status/{peer}` (§6.2, ENTITY-CORE-PROTOCOL.md §3.13) |

**Schema:**
```
type: "system/peer/session"
path: {local_peer}/system/peer/session/{remote_peer_id}
data: {
  remote_peer_id:       text,                    ; the peer this session authenticates
  remote_identity_hash: bstr(33),                ; system/hash of the remote identity entity
  remote_public_key?:   bytes,                    ; OPTIONAL denorm — see note below
  held_capability:      { hash: bstr(33), chain: [bstr(33), ...] },   ; cap I wield to dispatch to remote
  minted_capability?:   { hash: bstr(33), chain: [bstr(33), ...] },   ; cap I issued to remote (granter bookkeeping)
  granted_at:           uint,                     ; epoch ms — = last handshake
  expires_at?:          uint                      ; epoch ms, omitempty — cap validity window
}
```

**Field semantics:**

- **`held_capability`** (REQUIRED) — the capability the *remote* granted *this peer* at handshake. §10 dispatch reads this to authenticate an outbound EXECUTE and skip re-handshake. `chain` is an array of `system/hash` pointers, **leaf→root, length ≥ 1** (33-byte hashes per the ENTITY-CORE-PROTOCOL.md §3.5 wire form), resolved through the content store at reuse — entities are referenced by hash, never inlined (ENTITY-CORE-PROTOCOL.md §1.7 dedup model). `chain` carries **only** `system/capability/token` content-hashes — no signature/identity/granter padding (a verification bundle does not belong here; the verifier resolves the chain from the content store).
- **`minted_capability`** (OPTIONAL) — the *connection-handshake* cap *this peer* issued *to the remote*, recorded for R3a idempotency (a re-dial returns the same cap rather than minting a churny new one) and for revocation. **It is NOT a reverse-delivery cap.** In a bidirectional pair, A's `minted_capability` for B is the *same cap entity* as B's `held_capability` from A — one cap, recorded from both ends. Back-direction **delivery** authorization is the per-delivery `deliver_token` (granted at subscribe/continuation time, EXTENSION-INBOX / EXTENSION-SUBSCRIPTION), unchanged by this amendment. The session entity moves ONLY the handshake cap; it MUST NOT be built to solve generic back-direction dispatch.
- **`granted_at`** — handshake time. There is deliberately **no** `last_active` field: per-message liveness is `system/peer/status.last_seen`'s job (keepalive-updated, ENTITY-CORE-PROTOCOL.md §3.13); putting it here would force a tree write per message (write amplification → subscription/revision/history fan-out) on the auth record.
- There is deliberately **no** `status` field: connection lifecycle is `system/peer/status`'s job, and cap validity is derivable from `expires_at` (+ revocation check). The auth record needs no separate status.
- **`remote_public_key`** (OPTIONAL) — a denormalization. `peer_id` is a *hash* of the key, not the key, so the pubkey is not trivially derivable without the identity entity; 32 bytes for signature-verify-without-fetch is cheap. Peers MAY omit it and dereference `remote_identity_hash`.

**Lifecycle:**

- **Persistence (MUST).** The entity persists across `disconnected` status — **a disconnection MUST NOT delete it.** Surviving reconnect (and, for a persisted identity, process restart / WASM page-reload) is the entire purpose. (An ephemeral-keypair browser peer gets a new `peer_id` on reload by the identity model and therefore inherits no session — that is correct, not a gap; see ENTITY-CORE-PROTOCOL.md §1.5 / EXTENSION-IDENTITY.)
- **Grants change → mint fresh + overwrite in place.** One entity per peer, mutable. A runtime widen/narrow of the grant rejects the cached cap, re-mints, and updates `held`/`minted` hashes in place — preserving R3a's "same grants ⇒ same cap."
- **No self-session.** A peer never writes `system/peer/session/{local_peer_id}`; local dispatch short-circuits (the in-memory degenerate row, §10).
- **Writes fire ordinary subscription events.** The entity is a real tree write; it hits revision/history and fires events like any write. Subscriptions observing the session subtree MUST scope their patterns accordingly.

---

## 7. Subscription Restoration

After reconnection, subscriptions that were active before the disconnect need their notification delivery re-activated.

### 7.1 Subscription Persistence

Subscription entities at `system/subscription/*` are tree entities (EXTENSION-SUBSCRIPTION.md §2.1). They survive connection drops because they are stored in the tree, not in connection state. However, the notification delivery path may be broken — the deliver token's target peer was unreachable.

### 7.2 Restoration Algorithm

```
restore_subscriptions(peer_id):
  ; Find subscriptions with delivery targets on the reconnected peer
  for sub_path in list_paths("system/subscription/"):
    sub = entity_tree.get(sub_path)
    if sub.data.deliver_uri starts with "entity://" + peer_id:
      ; Re-validate deliver token
      token = content_store.get(sub.data.deliver_token)
      if token is null or is_expired(token):
        ; Token expired during disconnect — subscription is dead
        entity_tree.put(sub_path, null)
        continue

      ; Re-register in notification index
      notification_index.register(sub)

  ; Find subscriptions we hold on the remote peer
  ; These need to be re-validated with the remote
  for sub in local_remote_subscriptions(peer_id):
    result = execute_remote(peer_id, "system/subscription", "subscribe", {
      resource: {targets: [sub.pattern]},
      params: {
        events:        sub.events,
        deliver_to:    sub.deliver_to,
        deliver_token: sub.deliver_token    ; Re-use existing token if valid
      }
    })
    if result.status != 200:
      ; Remote rejected — subscription lost, subscriber must re-create
      log_warning("subscription_restoration_failed", sub.subscription_id)
```

### 7.3 Gap Detection

Between disconnect and reconnect, tree changes may have occurred that produced no notifications. After restoration, the subscriber SHOULD reconcile:

1. For each subscription path pattern, GET the current entity hash
2. Compare with the last known hash from the most recent notification
3. If different, an intermediate change was missed — process accordingly

For revision-enabled prefixes, this gap is automatically handled by revision pull (EXTENSION-REVISION.md §7.1).

---

## 8. Pending Delivery

When a handler emits an EXECUTE targeting a remote peer that is currently disconnected, the message is queued for later delivery.

### 8.1 Queue Storage

Pending deliveries are stored as entities in the tree:

```
system/outbound/{peer_id}/{sequence}  → system/network/pending-delivery
```

The sequence number is monotonically increasing per peer, providing delivery ordering.

### 8.2 Queueing

```
on_outbound_dispatch(peer_id, execute):
  connection = entity_tree.get("system/connection/" + peer_id)

  if connection is null or connection.data.status != "active":
    ; Queue for later delivery
    sequence = next_sequence(peer_id)
    pending = {
      type: "system/network/pending-delivery",
      data: {
        peer_id:    peer_id,
        sequence:   sequence,
        execute:    execute,
        created_at: now(),
        expires_at: execute.bounds.ttl_absolute or (now() + 3600000)    ; 1h default
      }
    }
    entity_tree.put("system/outbound/" + peer_id + "/" + sequence, pending)
    return QUEUED

  ; Connection active — send directly
  send(connection, execute)
  return SENT
```

### 8.3 Drain

On reconnection, queued messages are delivered in sequence order:

```
drain_pending(peer_id):
  for path in sorted(list_paths("system/outbound/" + peer_id + "/")):
    pending = entity_tree.get(path)

    ; Check expiry
    if pending.data.expires_at is not null and now() > pending.data.expires_at:
      entity_tree.put(path, null)    ; Expired — discard
      continue

    ; Deliver
    result = send(connection, pending.data.execute)
    if result is error:
      ; Connection failed again during drain — stop, will retry on next reconnect
      return

    ; Delivered — remove from queue
    entity_tree.put(path, null)
```

### 8.4 Queue Limits

Implementations SHOULD impose limits on pending delivery queues:

| Limit | Recommended | Behavior on exceed |
|-------|-------------|-------------------|
| Max messages per peer | 1000 | Reject new queuing, return error to sender |
| Max total bytes per peer | 10MB | Reject new queuing |
| Max age | 1 hour | Expire on drain (§8.3) |

---

## 9. Graceful Close

### 9.1 Reason Codes

| Reason | Meaning | Subscription behavior |
|--------|---------|----------------------|
| `shutdown` | Peer is shutting down | Delete subscriptions from this peer |
| `idle` | Connection idle, may resume | Preserve subscriptions with timeout |
| `error` | Connection error occurred | Preserve subscriptions with timeout |
| `migration` | Moving to different address | Preserve subscriptions, expect reconnect |

### 9.2 Close Flow

```
Initiator                           Responder
    |                                    |
    +-- EXECUTE connect/close ---------->|
    |   params: {reason: "idle"}         |
    |<---- EXECUTE_RESPONSE (200) -------+
    |                                    |
    [transport close]                    |
```

The close operation is best-effort — if the connection is already broken, the close message may not arrive. The responder detects connection loss via keepalive failure (§5) and applies default behavior (preserve subscriptions with timeout).

*Implementation note:* When closing a WebSocket connection, the entity protocol close (this section) SHOULD complete before the WebSocket close frame (RFC 6455 §5.5.1) is sent. Sending the WebSocket close frame transitions the connection to a closing state in which application messages may not be delivered. Implementations SHOULD: (1) send the EXECUTE connect/close message, (2) await the EXECUTE_RESPONSE, (3) then send the WebSocket close frame.

---

## 10. Outbound Dispatch

When a handler constructs an EXECUTE targeting a remote peer, the dispatch layer routes it by the target's **reachability class** (Amendment 8, R5). The dispatcher composes the four per-peer entities (§6.6 session, ENTITY-CORE-PROTOCOL.md §3.13 connection, §6.5 transport profiles, §6.2 status): it reuses a live connection if one exists, else authenticates with the held capability and resolves the target's profiles, else queues.

```
dispatch_remote(peer_id, execute):
  ; 0. Local short-circuit — never route self through the table.
  if peer_id == local_peer_id:
    return dispatch_local(execute)

  ; 1. Active connection wins — most-live. For full-duplex transports this MAY
  ;    be an accepted (inbound) connection reused for outbound (§6.5.1b, impl-detail).
  connection = entity_tree.get("system/connection/" + peer_id)
  if connection is not null and connection.data.status == "active":
    return send(connection, execute)

  ; 2. Held capability → authenticate without re-handshake (§6.3, §6.6).
  session = entity_tree.get("system/peer/session/" + peer_id)
  if session is not null and session.data.held_capability is not null
     and not expired(session.data.expires_at):
    execute = sign_with(execute, session.data.held_capability)   ; zero-RTT eligible

  ; 3. Resolve the target's reachability class from its profiles (§6.5.1a D1:
  ;    sort (priority asc, profile-id lex)) and dispatch by class:
  profiles = resolve_profiles(peer_id)            ; ordered candidate list
  for p in profiles:
    switch reachability_class(p):
      full_duplex_listener, half_duplex_listener:
        r = connect(peer_id, p.endpoint); if r.ok: return send(r.connection, execute)
      held_connection_client:                     ; non-listening peer holding a duplex socket to us
        if held_socket(peer_id): return push(held_socket(peer_id), execute)
      pollable_client:                            ; non-listening, no held socket → poll-fallback
        ; R4 / delivery_mode poll path — Phase-2; not specified or gated in v1.
        ; Until R4 lands, fall through to queue (step 4).
        continue
      static_publisher:
        continue                                  ; consumer-fetch target, never a dispatch target

  ; 4. No live profile connected. Consult the registered store-and-forward
  ;    fallback seam (§10.2) BEFORE the step-4 terminal. Null if none registered
  ;    (e.g. no RELAY installed) ⇒ byte-identical to the pre-seam terminal below.
  ;    Consulted HERE — at the dispatch caller holding BOTH peer_id and the execute
  ;    envelope — never inside connection-resolution (which holds only peer_id).
  fb = dispatch_fallback(peer_id, execute)        ; §10.2 seam — null if unregistered
  if fb is not null and fb.ok:
    return fb.result                              ; e.g. {status: queued-fallback, stored_at}

  ; 4t. Step-4 terminal (impl-variable). The §8 outbound queue is an OPTIONAL rung
  ;     an impl MAY interleave here iff it implements the §8 outbox; an impl with no
  ;     §8 outbox falls straight to the unreachable error. "Byte-identical to v1 when
  ;     the seam is unset" means: behaves exactly as THIS terminal does today,
  ;     whatever the terminal is in this impl.
  if entity_tree.get("system/peer/status/" + peer_id) is not null:
    return queue_pending(peer_id, execute)        ; §8 outbound queue — §8-conditional
  return error(502, "peer_unreachable")
```

**Reachability classes (the decision table):**

| Target class | Dispatch |
|---|---|
| full-duplex listener (`tcp`/`websocket`/`webrtc`) | reuse active connection (incl. accepted inbound — §6.5.1b); else dial + push |
| half-duplex listener (`http`) | dial its HTTP listener; push (each side runs a listener — §6.5.2c) |
| held-connection client (non-listening, holds a duplex socket to us) | push down its held socket |
| pollable client (non-listening, no held socket) | poll-fallback — source-hosted queue the target drains (R4 / `delivery_mode: poll`; **Phase-2**, not in v1; until then → queue, §8) |
| static publisher (`http-poll`) | n/a — a consumer-fetch target, not a dispatch target |

**Precedence — most-live wins.** When a peer matches multiple classes (e.g. it is both a full-duplex listener and pollable), the order is: active connection > listener-dial > held-socket > **store-and-forward fallback (§10.2)** > outbox-queue. The pseudocode encodes this; the table is the per-class action once a class is selected.

**Held-capability authentication (§6.6, §6.3).** Step 2 reads `system/peer/session/{peer_id}.held_capability` and signs the outbound EXECUTE with it, so the dispatcher skips the handshake when it already holds a valid cap. If the receiver rejects the cap (403), the sender falls back to a full handshake (§6.3). The cap lives in the tree (§6.6), so this works across reconnects and restart.

**Address resolution** (step 3 `resolve_profiles`) consults, in order:
1. `system/peer/alias/{name}` → `peer_id` (if a name was given)
2. `system/peer/transport/{peer_id}/*` → the profile set, sorted per §6.5.1a D1
3. Static configuration at `system/config/bootstrap`
4. *(post-corridor)* REGISTRY lookup — out of scope until the REGISTRY extension lands.

### 10.2 The dispatch-fallback seam (store-and-forward escalation)

The §10 ladder's step-4 terminal (`queue_pending`/`502`) only helps a peer that has, or will re-establish, a session with the target it drains into. It does **nothing** for the case that matters on the open internet: **delivering to a peer you have no live or recurring session with, that is offline or NAT'd right now.** NETWORK names a seam for that escalation; the store-and-forward *policy* lives in an extension (RELAY), never in NETWORK.

**Why a seam, not an inline branch.** NETWORK is **below** RELAY in the layering (RELAY is a network-peer-extension built on NETWORK); NETWORK §10 therefore cannot call into RELAY. This is the identical boundary already used for routing — the substrate exposes the slot, the extension plugs the algorithm (mirror of the relay→routing `resolve_next_hop` seam, `EXTENSION-ROUTE.md` §4). In the strictest impl this is not a preference but a compile-time constraint (an inline "if RELAY present" guard would invert the crate dependency graph).

**Signature.**

```
dispatch_fallback(peer_id, execute) → { ok: bool, result } | null
```

- Consulted **once**, at §10 step 4, after direct dispatch is exhausted and **before** the step-4 terminal (queue/502).
- Returns `null` when no fallback is registered (a peer without RELAY installed). `null` ⇒ the existing terminal runs unchanged — the **additive, no-regression** property. A peer needs no RELAY to *receive* (EXTENSION-RELAY.md §3.1.1) and symmetrically needs none to *dispatch directly*; it gains store-and-forward *send* only by installing RELAY.
- **Insertion-site discipline (normative MUST).** The seam is consulted at the dispatch caller that holds **both** the destination `peer_id` **and** the `execute` envelope it must store/forward — never inside the connection-resolution function (which holds only `peer_id`). All three reference impls converged on this independently; a reference impl that installs the seam inside connection-resolution would have to thread the envelope through and is non-conformant to this rule.

**v1 posture.** The seam is **v1.x**, additive. For non-RELAY peers the v1 floor is unchanged (queue/502, per each impl's terminal — see step 4t). For RELAY peers the new MUST is small: §10 consults the registered `dispatch_fallback` before the terminal; RELAY registers the inbox-relay-resolution behavior behind it (EXTENSION-RELAY.md §6.2.1). Conformance gates the **outcome** (offline-target delivery lands at the target's inbox; the target polls and verifies the sender signature exactly as on a direct delivery), not the rung choices inside the registered policy. No V7 change, no wire-format change, no new capability or error code (reuses `no_inbox_relay`/502, `capability_denied`/403).

**The forward-looking win.** Naming the seam now gives both deferred axes a single home at §10 step 4: the asynchronous floor (store-and-forward, *deliver later* — this seam's RELAY policy) and, later, the live-connection punch (NAT hole / WebRTC, *deliver now*). A mature dispatcher tries live first and store-and-forward last; both escalate from the **same** step-4 site rather than as parallel systems.

---

## 11. Write Authorization

All network handler tree writes are to the handler's managed namespace (`system/network/*`). The handler's own grant authorizes all writes.

| Operation | Write target | Authorization | Capability recorded |
|-----------|-------------|---------------|-------------------|
| maintain-peer | `system/network/peers/{id}/*` | Handler grant | Handler grant |
| release | `system/network/peers/{id}/*` (cleanup) | Handler grant | Handler grant |

Network operations manage connection infrastructure. All writes are handler-authorized.

See ENTITY-CORE-PROTOCOL.md §6.8 for the general write authorization model.

---

## 12. Conformance

### 12.1 MUST Implement

- Handler at `system/network` with `maintain-peer`, `release-peer`, `status`, `close` operations (§3, §4)
- Keepalive ping/pong exchange (§5)
- Peer status tracking via `system/peer/status/*` entities (v7.9 §3.13)
- Connection state tracking via `system/connection/*` entities (v7.9 §3.13)
- Graceful close with reason codes (§9)
- Pending delivery queueing when remote peer is disconnected (§8)
- Pending delivery drain on reconnection (§8.3)
- Session state entity at `system/peer/session/{peer_id}` carrying `held_capability`, persisted across `disconnected` (MUST NOT delete on disconnect) (§6.6, R6)
- Reachability-class outbound dispatch: resolve the target's profiles by `(priority asc, profile-id lex)` and dispatch by class; authenticate via `system/peer/session/{peer}.held_capability` when held (§10, R5)

### 12.2 SHOULD Implement

- Subscription restoration after reconnection (§7)
- Adaptive keepalive suppression during active exchange (§5.4)
- Exponential backoff for reconnection (§2.2)
- Self-authenticating zero-RTT reconnection via `held_capability` from the session entity (§6.3, §6.6)
- Pending delivery queue limits (§8.4)
- Pending delivery expiry (§8.3)

### 12.3 MAY Implement

- Linear or constant backoff strategies (§2.2)
- Custom keepalive intervals (§2.3)
- Subscription gap detection and reconciliation (§7.3)
- One-shot connection for unknown peers (§10)

### 12.4 Implementation-Defined

- Keepalive default intervals
- Backoff default parameters
- Pending delivery queue size limits
- Subscription preservation timeout on `idle`/`error` close
- Address resolution strategy
- Connection pooling
- Concurrent reconnection handling

---

## 13. Types Installed

| Type | Description |
|------|-------------|
| `system/peer/session` | Per-peer session auth record: held/minted capability + handshake bookkeeping (§6.6, R6) |
| `system/peer/transport/tcp` | Live TCP transport profile (§6.5.2a) |
| `system/peer/transport/websocket` | Live WebSocket transport profile — browser-capable (§6.5.2b) |
| `system/peer/transport/http` | Live HTTP transport profile — EXECUTE over POST (§6.5.2c) |
| `system/peer/transport/http-poll` | Static HTTP transport profile — CDN/static hosting (§6.5.3) |
| `system/peer/transport/quic` | Live QUIC transport profile — aspirational (§6.5.2) |
| `system/network/maintain-request` | Input for maintain-peer operation |
| `system/network/maintain-result` | Output of maintain-peer operation |
| `system/network/release-request` | Input for release-peer operation |
| `system/network/release-result` | Output of release-peer operation |
| `system/network/status` | Network status summary |
| `system/network/peer-summary` | Per-peer status within summary |
| `system/network/close-request` | Input for close operation |
| `system/network/pending-delivery` | Queued outbound message |
| `system/network/backoff-config` | Reconnection backoff parameters |
| `system/network/keepalive-config` | Keepalive parameters |
| `system/network/ping` | Keepalive ping |
| `system/network/pong` | Keepalive pong |

---

## 14. Informative: Browser Peer Considerations

Browser peers (running in WASM environments) differ from native peers in several ways that affect network extension behavior:

| Aspect | Native Peer | Browser Peer |
|--------|-------------|--------------|
| **Inbound connections** | Accepts via listener (TCP, WebSocket) | Cannot accept — browser sandbox prevents listening |
| **Connection initiation** | Can connect to any peer via any transport | Can only connect to peers with WebSocket listeners |
| **Identity persistence** | Filesystem keypair | IndexedDB or localStorage; ephemeral if storage unavailable |
| **Background execution** | Continuous process | Page visibility affects timers; service workers possible but limited |
| **Reconnection** | Any available transport | WebSocket only |

**Initiation asymmetry.** Browser peers are always connection initiators. They cannot accept inbound connections. This means:

- Subscription delivery to a browser peer flows through the browser's outbound connection (which the browser initiated). If the browser disconnects, the native peer cannot reconnect — it queues pending deliveries (§8) and waits for the browser to reconnect.
- The `maintain-peer` operation (§4.1) runs on the browser peer. The browser is responsible for reconnection. The native peer's role is to accept the reconnection and resume the session (§6.2).

**Identity model.** Browser peers SHOULD use persistent keypairs stored in IndexedDB for session continuity across page reloads. Ephemeral keypairs (new identity per session) are acceptable but capability grants do not survive page reload. The peer identity model (ENTITY-CORE-PROTOCOL.md §3.5, §4) is unchanged — the keypair source is an implementation decision.

**Timer reliability.** Browser timers (setTimeout/setInterval) are throttled when the page is not visible. Keepalive intervals (§5) may be delayed in background tabs. Implementations SHOULD account for timer throttling when configuring keepalive `timeout_ms` and `max_missed` — more lenient values prevent false disconnect detection for browser peers. Native peers serving browser clients SHOULD tolerate delayed pong responses.
