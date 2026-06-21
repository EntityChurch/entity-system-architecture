# Operational State: User Guide

**Status**: Active
**Audience:** Peer implementers and SDK builders. Assumes you've read V7 §3 (entity types) and §6.3 (tree handler); does not assume you've worked out the operational-state model from §3.13 type definitions alone — that's what this guide is for.
**Spec reference:** ENTITY-CORE-PROTOCOL.md §3.13 (operational state types), §6.3 (system data paths), §9.2 (SHOULD conformance). Builds on `proposals/PROPOSAL-OPERATIONAL-STATE-TYPES.md` (implemented v7.9; refreshed for v7.40 rename).
**Related guides:** GUIDE-CAPABILITIES.md (capability model — operational state composes with grants), GUIDE-IDENTITY.md (identity extension — separate concern; this guide does not cover identity-aware peer state).

---

## 1. What operational state is

**Operational state** is the entity vocabulary a peer uses to describe itself: what transports it speaks, how it's configured, what other peers it knows about, what connections are active, and what local names it has for things. It's the answer to "what does this peer expose so other code — handlers, subscribers, syncing peers, the peer's own kernel after a restart — can reason about its current operational reality."

The design choice that distinguishes operational state from how most systems model the same information: **everything is entities written to the tree through the standard emit pathway.** No separate event bus for connection events. No separate config-reload mechanism. No separate registry for known peers. A handler that wants to know whether peer X is connected reads `system/peer/status/{peer_id}` like any other entity. A handler that wants to react to disconnection subscribes to that path. A peer recovering from a crash reads its own tree to remember what it was doing.

This works because entities, the emit pathway, and subscription already exist for every other purpose in the system — operational state is just one more application of those mechanisms, not a new layer.

**Why this matters.** When operational state lives in implementation structs (a `HashMap<PeerId, ConnectionInfo>` somewhere in the kernel), it's invisible to handlers, unsubscribable, unsyncable, and has to be reconstructed by hand on restart. When operational state lives in entities, it's all of those things by default. The cost is minor (a handful of tree writes per connection event); the win is that every peer is self-describing and observable through the same mechanisms used for everything else.

## 2. The five types, walked together

V7 §3.13 defines five types as one surface:

| Type | Stored at | Purpose |
|------|-----------|---------|
| `system/transport` | `system/transport/{protocol}` (local) and `system/peer/transport/{peer_id}/{protocol}` (remote) | What transports a peer speaks; what addresses it listens on or has been observed at. |
| `system/config` | `system/config/{concern}` | Runtime configuration parameters, grouped by concern. |
| `system/peer/status` | `system/peer/status/{peer_id}` | The local peer's observation of another peer's connectivity. |
| `system/peer/alias` | `system/peer/alias/{name}` | Local human-readable names for peers. |
| `system/connection` | `system/connection/{peer_id}` | Active authenticated connection state. |

These are not five disconnected entries. They reference each other:

- `system/peer/status` carries a `connection` field that path-references the corresponding `system/connection` entity.
- The same `system/transport` type is used for both a peer's own transports (`system/transport/{protocol}`) and remote-peer-discovery data (`system/peer/transport/{peer_id}/{protocol}`).
- Every status, alias, and connection entity carries a `peer_id` field of type `system/peer-id`.

**`system/peer` (the V7 entity) is not in the operational-state list — and that's intentional.** The V7 entity `system/peer` (the keypair-bearing entity, formerly `system/identity` pre-v7.40) is *not* operational state. It's the peer's identity-as-cryptographic-fact. Operational state is *about* peers, not the peer entities themselves. There is also no `system/peer` handler — `system/peer/...` is a data subtree accessed via the standard tree handler; operational-state writes go through `tree:put`.

**Where does the V7 `system/peer` entity live in the tree?**

- **Foreign peers' entities are content-addressed; no fixed tree path.** When you encounter a foreign peer (via signature, capability grantee, attestation reference), you have its content_hash and can find the entity in your local content store or in `envelope.included` per V7 §6.5. There is no `system/peer/{peer_id}` or `system/peer/identity/{peer_id}` reservation — content-addressing handles foreign-peer references already.
- **Local peer's own entity at `system/peer/self` (SHOULD).** Per `PROPOSAL-OPERATIONAL-STATE-AND-SDK-CONVERGENCE.md` C2, the local peer's `system/peer` entity SHOULD be bound at `system/peer/self`. Simple lookup; sibling key alongside the operational-state collections (`status/`, `alias/`, `transport/`). SHOULD-tier consistent with the rest of §3.13 — peers that don't populate are still V7-conformant, just lose tree-walkability for the local-peer entity (handlers can still compute the content_hash from peer_id).

**Recommended layout under `system/peer/`:**

```
system/peer/
  self                          → V7 system/peer entity (local keypair entity)  (recommended)
  status/{peer_id}              → operational status (this guide §4)
  alias/{name}                  → local naming (this guide §6)
  transport/{peer_id}/{protocol} → remote transport discovery (this guide §5)
```

`self` is one leaf alongside three operational-state subtrees. No conflict — `status`, `alias`, `transport` are subtree prefixes; `self` is a single key.

This guide covers operational state. Identity-stack concerns — controllers, agents, identifiers, attestations, quorums — live in `EXTENSION-IDENTITY` and its companion guide. The two are deliberately separate: operational state is about running a peer; identity is about who a peer represents and how that representation rotates and recovers.

**Two ways peers are referenced — paths use peer_id, entity fields use content_hash.** This guide writes path placeholders as `{peer_id}` because operational-state paths (and all V7 paths) use peer_id — the Base58 string derived from the peer's public key, format `Base58(key_type || hash_type || hash(pubkey))` per V7 §1.5. So `system/peer/status/{peer_id}` literally means "Base58 string at this path segment."

Entity-internal references — the `signer` field on a signature entity, the `grantee` and `granter` fields on a capability entity, substrate references inside attestation/quorum entities — use content_hash instead, type `system/hash` per V7 §3.5/§3.6. content_hash is the canonical content hash of the referenced `system/peer` entity (per ECF, V7 §1.2).

These are different representations of the same peer, and they're **not interchangeable**:

- Paths are **addresses** (where things live; human-readable; stable across the peer's life unless its key changes; derivable from just the public key).
- Entity-internal references are **content-addressed** (cryptographic verification; distinguishes the specific `system/peer` entity used at a specific moment, which matters across rotations and multi-key cases).

Given one you can derive the other: peer_id is the `peer_id` field of the `system/peer` entity; content_hash is computed by hashing the entity's canonical encoding. The substrate provides `resolve_peer(content_hash) → system/peer entity` for the hash → entity direction; the inverse is just reading the entity's `peer_id` field. See V7 §1.4 (path model) for the explicit convention.

Throughout this guide: when you see `{peer_id}` in a path, it's the Base58 string. When you see `signer`, `grantee`, or similar references on an *entity field*, those are content_hashes. The two layers don't swap.

## 3. Configuration (`system/config`) — the model

> **Status note: configuration patterns are exploratory.** The `system/config` type and the `system/config/{concern}` path convention are stable; the *patterns* for using them — what concerns to define, how to structure parameters, what configuration is and isn't for — are exploratory and will converge with use. The guidance below is starting-point convention, not normative. Treat it as "here's how to use this if you don't have a strong opinion" rather than "here's how everyone must use this." Refinement will follow as more peers populate `system/config` in real deployments.

The `system/config` type holds runtime configuration parameters. Each entity has a `concern` (what it configures) and a map of `parameters`. The path is `system/config/{concern}` — one entity per concern.

```
system/config/connection → {
  type: "system/config",
  data: {
    concern: "connection",
    parameters: {
      keepalive_interval_ms: 30000,
      keepalive_timeout_ms: 10000,
      max_missed_keepalive: 3,
      reconnect_min_ms: 1000,
      reconnect_max_ms: 60000,
      reconnect_backoff: "exponential"
    }
  }
}
```

**The model in one sentence:** configuration is data in the tree, mutable through standard `tree:put`, observable through standard subscription, no separate reload mechanism.

**What this means in practice.**

- A handler that needs the keepalive interval reads `system/config/connection` from the tree. It does not consult a config file, an environment variable, or a startup struct. It reads an entity.
- When configuration changes at runtime (someone writes a new `system/config/connection` entity), the emit pathway fires. Subscribers — including any handler that watches that path — receive the change and react. There is no `reload_config()` call. The tree write *is* the reload.
- When the peer restarts, configuration survives if the tree is persistent. The peer reads `system/config/*` on startup and operates with those parameters. There is no config-file-vs-runtime drift because there is no config file in the protocol model.

**Example concerns.** V7 §3.13 lists `connection`, `capability`, and `bounds` as example concern names ("etc." — not an exhaustive normative list). Plausible content for each, written here illustratively rather than prescriptively:

| Concern | Illustrative content (not normative) |
|---------|----------------------------------------|
| `connection` | Liveness/keepalive parameters, reconnect backoff. Anything governing how connection liveness is measured and how recovery behaves. |
| `capability` | Default TTL for issued grants, scope-defaulting policy, revocation-list refresh interval. Anything governing capability issuance and validation at runtime. |
| `bounds` | Default TTL/budget for incoming requests, max-await-stack depth, timeout policy. Anything governing resource-bound enforcement. |

These are starting-point conventions, not protocol requirements. Specific field names and granularity are exploratory; nothing in V7 fixes them. A peer that doesn't expose runtime configuration is still V7-conformant; it just isn't reactive to runtime config changes.

**Granularity guidance.** The base shape is per-concern: one entity holds all parameters for `connection`. This is simpler to read, atomic when written, and matches the way most config gets thought about. Per-parameter decomposition (e.g., `system/config/connection/keepalive-interval-ms` as its own entity — the path segment is kebab per the namespace axis, even though the in-map parameter key stays `keepalive_interval_ms`) is allowed when fine-grained subscription or compute dependency requires it — for example, if one handler subscribes to "any change in keepalive timing" and another subscribes to "any change in reconnect policy," per-parameter entities let each subscribe to a narrow path. The type definition (`parameters: map_of any`) supports both shapes.

**Domain-handler config is allowed.** V7 §3.13 explicitly opens `system/config/` for domain handlers to define their own entities with domain-specific types. For example, a `local/files` handler MAY store its root-mount configuration at `system/config/local-files/shared` as a `local/files/root-config` entity — not the generic `system/config` shape. The sub-path under `system/config/` is the namespace for the configuring extension or handler; the entity type carried at that path can be domain-specific. This keeps configuration discoverable in one place (the `system/config/` subtree) while letting domain handlers define typed config schemas appropriate to their concerns.

*Naming flexibility — convention not enforced.* Both forms work for placing a handler's config in the tree:

- **Single hyphenated segment** — `system/config/local-files/shared` (V7 §3.13's example). Treats the config as one logical thing at a single tree location; "local-files" reads as a single identity.
- **Nested path** — `system/config/local/files/shared`. Preserves the namespace-as-tree structure; lets you list/walk all `local/...` configs by sub-prefix.

Neither is forbidden. Path segments are slash-separated; nothing in V7 says you can't use slashes throughout a config path. The choice is between "more distinct top-level entity" (hyphenated) and "preserves tree structure" (nested) — both are reasonable. We don't force one over the other; pick what reads well for the configuration concept. This is one of the convention areas explicitly expected to refine as deployments reveal patterns.

**What configuration is NOT.** Configuration is for *parameters that govern peer behavior*. It is not:

- *Capability grants.* Those are `system/capability/grants/...` entities with their own ownership rules.
- *Handler ops.* You don't "configure" what `system/tree:put` does. You configure parameters that affect runtime decisions adjacent to handler execution (how long to wait, how often to retry).
- *Identity state.* If you're using EXTENSION-IDENTITY, identity-aware parameters live in identity entities, not in `system/config`.
- *Application state.* App-defined entities live under `app/{app-id}/...`. Configuration is for the *peer's own* operational behavior.

When in doubt: if a parameter affects how the peer's kernel or system handlers behave at runtime, it's a candidate for `system/config`. If it's about what an application does, it's not.

## 4. Connection lifecycle

Two types together describe connections: `system/peer/status` (the local peer's observation of another peer's connectivity) and `system/connection` (the active authenticated connection itself).

**State machine for `system/peer/status`.** The `status` field cycles through:

```
(unknown) → connected       First successful handshake completes
connected → suspect          Missed keepalive threshold reached
suspect   → connected        Keepalive resumes
suspect   → disconnected     Suspect timeout expires without keepalive
connected → disconnected     Connection closed (graceful or transport failure)
disconnected → connected     Reconnection succeeds
```

Every transition is a tree write through the emit pathway. There is no separate "connection event" mechanism; the write to `system/peer/status/{peer_id}` *is* the event. Subscribers to that path see the full state-transition history as it happens.

**`system/peer/status` and `system/connection` are paired.** When a connection is established:

1. `system/connection/{peer_id}` is written with `status: "active"`, the negotiated transport, the remote `address`, the `established_at` timestamp, and any negotiated `parameters` (protocol version, hash format, compression, encryption).
2. `system/peer/status/{peer_id}` is written with `status: "connected"`, `connected_at: <now>`, `last_seen: <now>`, and `connection: "system/connection/{peer_id}"` (a `system/tree/path`-typed reference).

While the connection is live, `system/peer/status.last_seen` is updated as activity arrives — useful for diagnostics (when did we last hear from this peer?) and as input for keepalive logic.

When the connection enters graceful shutdown (the local or remote peer initiates a clean close), `system/connection/{peer_id}` transitions to `status: "draining"` — connection still exists, no new requests accepted, in-flight work completing. This is observable and gives subscribers a chance to react before the connection is fully closed.

When a connection drops (either after draining or via transport failure):

1. `system/connection/{peer_id}` is updated with `status: "closed"`. Whether the entity is removed entirely or kept as a tombstone is implementation choice (V7 doesn't pin one). One reasonable pattern: **keep as tombstone with `status: "closed"`** so historical lookup and the history extension can reason about prior connections; removal throws away information that's cheap to keep. Either approach is conformant; the convention here is exploratory.
2. `system/peer/status/{peer_id}` is updated with `status: "disconnected"`. The `connection` reference can either be removed or left pointing at the tombstone.

The distinction between the two types: `system/connection` carries connection-specific facts (which transport, what address, when established, negotiated parameters). `system/peer/status` carries the peer-relationship-level summary (are we connected to this peer? when did we last hear from them?). One is *the connection*; the other is *what we know about this peer*. A peer we've connected to many times has one current `system/connection` (or none) and one `system/peer/status` that has accumulated a history of transitions.

**Reading the lifecycle.** A handler that wants to check "is peer X currently connected" reads `system/peer/status/{peer_id}` and looks at the `status` field. If it needs the connection details (which transport, which address), it follows the `connection` path-reference to `system/connection/{peer_id}`. Two reads, two distinct concerns, no mixing.

## 5. Transport advertising and discovery

Transports describe how peers reach each other. The same `system/transport` type serves two purposes:

- **Local advertising.** `system/transport/{protocol}` describes a transport this peer offers — its bound addresses, its status (listening / stopped / error), its protocol parameters.
- **Remote discovery.** `system/peer/transport/{peer_id}/{protocol}` describes a transport this peer has *observed* on a remote peer. Same type, different path location, different meaning.

**Local example:**

```
system/transport/tcp → {
  type: "system/transport",
  data: {
    protocol: "tcp",
    addresses: ["0.0.0.0:4040", "[::]:4040"],
    status: "listening",
    parameters: { frame_limit: 16777216, keepalive_idle_s: 30 }
  }
}

system/transport/websocket → {
  type: "system/transport",
  data: {
    protocol: "websocket",
    addresses: ["ws://192.168.1.42:4041", "wss://peer.example.com/ws"],
    status: "listening"
  }
}
```

**Remote-discovery example:**

```
system/peer/transport/{peer_id}/tcp → {
  type: "system/transport",
  data: {
    protocol: "tcp",
    addresses: ["192.168.1.42:4040"],
    status: "observed"
  }
}
```

The `status` field distinguishes the two roles: `"listening"` for local transports actively accepting; `"observed"` for transports we've seen on remote peers (no implication that they're currently up — just that we know they exist).

**How discovery happens.** The handshake (V7 §4.2 connection setup) is the natural moment to populate remote-transport entries. When peer A connects to peer B, B can write `system/peer/transport/{A_peer_id}/{protocol}` based on A's announced transports. Sync (when EXTENSION-REVISION is registered) propagates further. The protocol does not mandate a specific advertisement mechanism — peers populate from whatever they observe.

**WebSocket address format.** WebSocket transports carry full URLs in `addresses` (e.g., `"ws://192.168.1.42:4041"`, `"wss://peer.example.com/ws"`), not bare host:port pairs. This is because the WebSocket scheme (`ws://` vs `wss://`), the path (`/ws`, `/peer`, etc.), and the port encoding are all part of how a peer reaches the endpoint. Other transports (TCP, QUIC) carry host:port. The `addresses` field is `array_of string`; the format is per-protocol.

**Multi-transport per peer.** A single connected peer is addressed via one `system/connection/{peer_id}` entity. The `transport` field records which transport that connection uses. Connecting to the same peer over both TCP and QUIC simultaneously — multi-transport sessions — is *not* in V1 scope. If you need it, that's EXTENSION-NETWORK territory: a network extension can layer a multi-transport session model on top of the operational-state vocabulary. The base operational-state types stay simple: one connection per peer at a time.

## 6. Aliasing

`system/peer/alias` is local naming. It is the entity-system equivalent of `/etc/hosts`: a private-to-this-peer mapping from human-readable names to peer IDs.

```
system/peer/alias/alice-laptop → {
  type: "system/peer/alias",
  data: {
    name: "alice-laptop",
    peer_id: "12D3KooWAB..."
  }
}
```

Stored at `system/peer/alias/{name}`. Resolution is a tree lookup at the path. Zero network cost, zero protocol participation.

**What aliases are NOT.** This is where most confusion happens, so be explicit:

- **Not URI redirects.** A request to `entity://alice-laptop/system/tree/...` is not protocol-redirected. URI construction uses the *resolved* peer_id; the alias is consulted at the application or SDK layer to resolve the name *before* URI construction. The wire protocol always sees peer_ids, never aliases.
- **Not protocol identifiers.** Two peers can have wildly different aliases for the same peer_id ("alice-laptop" on my peer; "the-design-machine" on someone else's). Aliases are not a shared namespace; they're not an identity claim; they're not part of authentication.
- **Not synced by default.** Each peer maintains its own alias map. Sync of `system/peer/alias/*` is opt-in (via EXTENSION-REVISION with explicit grant). Most deployments will not sync aliases — they're personal, not shared.
- **Not "claim this name" tokens.** Anyone can write any alias to their own tree. The alias maps `name → peer_id` locally; nothing about the alias entity itself proves the named peer endorses the name.

If you find yourself wanting aliases to be authoritative, synced, or globally meaningful, you want something else — group membership, a registry handler, or an identity-extension construct. Not aliases.

## 7. Population locus

Operational-state types are SHOULD-tier conformance: a peer that doesn't populate them is still V7-conformant, but it loses observability/subscribability/recovery benefits. The remaining question is: **when a peer chooses to populate operational state, who actually performs the writes?**

> **EXTENSION-NETWORK status note.** The transport/discovery layer of EXTENSION-NETWORK is in use across implementations today; the continuation-driven reconnection and subscription-restoration layers are partial. Peers using the extension get transport advertising and operational-state population for the layers that have landed; full reconnection orchestration is currently per-impl until the rest of the extension stabilizes. Read the recommendations below as "the canonical direction" rather than "what's universally available today."

**The canonical answer for connection lifecycle: EXTENSION-NETWORK.** EXTENSION-NETWORK exists specifically to manage persistent peer relationships — maintaining connections, detecting failures, reconnecting, restoring subscriptions — and as part of doing that, it writes the operational-state entries:

- On connection established: writes `system/peer/status/{peer_id}` (`"connected"`) and `system/connection/{peer_id}` (`"active"`).
- On keepalive failure: updates `system/peer/status/{peer_id}` to `"suspect"`.
- On disconnect: updates both entries (`status` to `"disconnected"` / `"closed"`).
- On reconnect: reads `system/peer/transport/{peer_id}/*` to choose a transport, then re-populates as above.

If you have EXTENSION-NETWORK installed, you have operational-state population for `system/peer/status` and `system/connection` for free, written through standard tree ops. The lifecycle is observable, subscribable, and debuggable through the same mechanisms as any other entity data — which is the whole point.

**What about peers without EXTENSION-NETWORK?** Three options:

| Option | What it means | When it fits |
|--------|---------------|---------------|
| **Kernel-side** | The peer's kernel writes operational-state entries directly as part of its own connection-lifecycle handling, without going through EXTENSION-NETWORK. | Minimal peers that don't want the full network-extension surface but still want observability. |
| **SDK-side** | The SDK wraps kernel events and writes operational-state entries on the kernel's behalf. | Middle ground when no extension-network and no kernel-side population. |
| **App-side** | The application writes operational-state entries based on its own connection-lifecycle observations. | Last resort. Same surface area as inventing your own ad-hoc registry; no advantage over installing EXTENSION-NETWORK. |

The direction we're converging on: where EXTENSION-NETWORK is available and applicable, let it own the population — that's what it's for. The kernel-side / SDK-side / app-side options aren't fallbacks to choose against the network extension; they're patterns for peers in environments where the network extension isn't available or appropriate (minimal peers, environments where the continuation-driven layers haven't landed yet, etc.). As the extension stabilizes across impls, more peers will naturally land on the network-extension path.

**Other types — locus by type.**

| Type | Who populates | Notes |
|------|---------------|-------|
| `system/peer/status` | EXTENSION-NETWORK (canonical) | Or kernel-side / SDK-side as fallback. |
| `system/connection` | EXTENSION-NETWORK (canonical) | Or kernel-side / SDK-side as fallback. |
| `system/transport` (local) | Likely kernel (when listener binds) | The kernel is in a natural position to know what it's bound to; per-impl confirmation pending. EXTENSION-NETWORK reads but may also write depending on its scope per impl. |
| `system/peer/transport` (remote) | Connection-time (handshake or EXTENSION-NETWORK during reconnection) | Populated from a remote peer's announced transports. |
| `system/config` | Concern-by-concern. The kernel writes baseline `connection` / `bounds` concerns; SDKs or apps populate concerns they own. | Domain handlers populate their own `system/config/{handler-name}/...` entries (per §3). |
| `system/peer/alias` | App or user | Aliases are personal naming choices. Neither kernel nor extension has a view. |

**Per-impl population status.** EXTENSION-NETWORK is the canonical writer where it's installed. Whether each impl ships EXTENSION-NETWORK by default vs. opt-in is per-impl: entity-core-go ships kernel-side population for `system/peer/transport/*` (`core/peer/remote.go:25-53`), with status/connection planned as follow-on after this proposal lands. See `PROPOSAL-OPERATIONAL-STATE-AND-SDK-CONVERGENCE.md` "Decisions made" for the per-type locus picks.

## 8. Composition with extensions

Operational-state types compose with several extensions. The extensions don't need new mechanisms — they observe and write operational state through the standard tree/emit pathways.

**EXTENSION-NETWORK** (canonical, covered in §7). The network extension is the primary writer of `system/peer/status` and `system/connection`, and the primary reader of `system/peer/transport` for reconnection. Its connection-lifecycle continuation graphs are expressed as continuations watching operational-state paths. Operational state is the *vocabulary* the network extension uses; the network extension is the *active agent* that drives state through that vocabulary.

**EXTENSION-SUBSCRIPTION.** Watches tree paths and delivers notifications. Useful patterns:

- `system/peer/status/*` — react to any peer connectivity change.
- `system/peer/status/{peer_id}` — react to one specific peer's connectivity.
- `system/connection/*` — react to connection establishment/closure across all peers.
- `system/config/{concern}` — react to runtime configuration changes (the "config reload" pattern, but as standard subscription).

A handler that needs to keep state synchronized with operational reality subscribes to one of these paths and reacts in its callback. No special API.

**EXTENSION-REVISION** (versioning + sync). Provides versioning, merge, and peer-to-peer synchronization for entity trees. Sync is selective per path. The sync postures below are starting-point conventions — **defaults you can override per deployment**, not normative rules:

- `system/transport/*` — typically MAY sync, to advertise local transports to a synced peer.
- `system/peer/alias/*` — typically NOT synced (aliases are personal). Sync only with explicit intent (e.g., a small group sharing a naming convention).
- `system/config/*` — typically NOT synced (operational parameters tend to be private). Capability-controlled if synced at all.
- `system/peer/status/*` — typically NOT synced (your view of who's connected is your view).
- `system/connection/*` — typically NOT synced (your active connections aren't the other peer's concern).

Rough heuristic: transports are advertisement-shaped (broadcasting "here's how to reach me" is the point); the rest is a peer's local view of its own world. EXTENSION-REVISION doesn't enforce these defaults — they're starting points to adjust per deployment.

**EXTENSION-HISTORY.** Accumulates path-level transition records when a path is history-enabled. Operational-state paths get history "for free" when configured — `system/peer/status/{peer_id}` accumulates a record of every status transition, useful for diagnostics ("when did this peer first go suspect? how often does it disconnect?"). Operational state and history compose without either knowing about the other.

**EXTENSION-CONTINUATION.** Enables execution chaining as entities. EXTENSION-NETWORK uses this internally (its reconnection logic is expressed as continuation graphs over operational-state subscriptions). Application-level continuations can do the same — for example, a continuation that fires on `system/peer/status/{peer_id}` transitions to `"disconnected"` and triggers an app-defined cleanup.

## 9. Recovery

The tree IS the recovery mechanism. A peer that crashes and restarts with a persistent tree reads its own operational state to remember where it was:

- `system/transport/*` — restart listening on the same protocols. (The kernel may have to re-bind sockets, but the *intent* — "I was listening on TCP at 0.0.0.0:4040" — is in the tree.)
- `system/config/*` — apply runtime configuration from the tree, not from a config file. Any config changes made between sessions survive.
- `system/peer/status/*` — the prior view of which peers were connected. Status will need to be reconciled with reality (everyone is "disconnected" until reconnection succeeds), but the *list* of known peers is preserved.
- `system/connection/*` — likely all stale (`status: "active"` from before the crash), to be reconciled — overwritten on reconnect, or marked `"closed"` and tombstoned.
- `system/peer/alias/*` — survives directly; aliases don't depend on runtime state.

**What's NOT in operational state and so doesn't survive restart automatically:** ephemeral subscription handles, in-memory continuation state without a tree binding, application data not written to the tree. If you want it to survive restart, write it as an entity. That's true throughout the system; operational state is just one application of the principle.

**Reconciliation pattern (suggested, not normative).** A reasonable startup sequence:

1. Read `system/config/*` and apply runtime parameters.
2. Read `system/transport/*` and rebind transports. Update entries with current bound addresses (which may have changed).
3. Read `system/peer/status/*` and mark all entries as `"disconnected"` (the prior session ended). Status will update naturally as reconnections happen.
4. Mark `system/connection/*` entries as `"closed"` (or remove them, depending on tombstone choice).
5. Begin handshake/reconnection logic per the network layer.

V7 doesn't pin a specific sequence; this is starting-point convention. The peer comes back online with memory of who it was talking to, configured the same way as before, and a clean reconnection state to drive forward.

## 10. Quick reference

**Reserved paths under `system/`:**

| Path pattern | Type | Who writes |
|--------------|------|------------|
| `system/transport/{protocol}` | `system/transport` | Kernel (when listener binds) |
| `system/peer/self` (SHOULD per V7 §3.13) | `system/peer` (V7 entity) | Kernel at startup |
| `system/peer/transport/{peer_id}/{protocol}` | `system/transport` (reused) | Kernel at handshake; EXTENSION-NETWORK during reconnection |
| `system/config/{concern}` | `system/config` (or domain-specific type) | Kernel for `connection`/`bounds`; concern owner otherwise |
| `system/peer/status/{peer_id}` | `system/peer/status` | EXTENSION-NETWORK (canonical); kernel/SDK as fallback |
| `system/peer/alias/{name}` | `system/peer/alias` | App / user |
| `system/connection/{peer_id}` | `system/connection` | EXTENSION-NETWORK (canonical); kernel/SDK as fallback |

**What's NOT operational state:**

- `system/peer` entity — V7 keypair-bearing entity (foundational identity, not operational).
- `system/identity/...` paths — owned by EXTENSION-IDENTITY.
- `system/attestation/...`, `system/quorum/...`, `system/role/...`, `system/group/...` — owned by their respective extensions.
- `system/handler/...`, `system/type/...`, `system/capability/...` — owned by core protocol mechanisms (handler dispatch, type system, capability system).
- `app/{app-id}/...` — application state.

**When to use which:**

| You want to know... | You read... |
|---------------------|-------------|
| Is peer X connected? | `system/peer/status/{peer_id}.status` |
| What transport am I using to reach peer X? | follow `system/peer/status/{peer_id}.connection` → `system/connection/{peer_id}.transport` |
| What addresses do I listen on? | `system/transport/*` |
| What addresses can I reach peer X at? | `system/peer/transport/{peer_id}/*` |
| What's the current keepalive interval? | `system/config/connection.parameters.<keepalive-field>` (specific field name is per-impl convention; see §3 — not normative) |
| Who is peer ID 12D3KooW...? | `system/peer/alias/*` (search for matching peer_id) |

**When to write which:**

| You want to record... | You write... |
|------------------------|--------------|
| Connection just established | `system/connection/{peer_id}` (active) + update `system/peer/status/{peer_id}` (connected, with `connection` ref) |
| Connection just dropped | `system/connection/{peer_id}` (closed, tombstone) + update `system/peer/status/{peer_id}` (disconnected) |
| Missed keepalive threshold | update `system/peer/status/{peer_id}` (suspect) |
| Transport bound and listening | `system/transport/{protocol}` (listening, with addresses) |
| Discovered remote peer's transport | `system/peer/transport/{peer_id}/{protocol}` (observed) |
| Configuration changed | `system/config/{concern}` (with updated parameters); subscribers react |
| User added a peer alias | `system/peer/alias/{name}` |

---

## References

- ENTITY-CORE-PROTOCOL.md §3.13 — operational state type definitions.
- ENTITY-CORE-PROTOCOL.md §6.3 — system data paths table (lists operational-state paths under tree-handler access).
- ENTITY-CORE-PROTOCOL.md §9.2 — SHOULD-implement conformance entry for operational-state types.
- `extensions/network-peer-extensions/EXTENSION-NETWORK.md` — canonical writer for `system/peer/status` and `system/connection`; reconnection, keepalive, subscription restoration.
- `extensions/standard-peer-extensions/EXTENSION-SUBSCRIPTION.md` — observe operational-state path changes.
- `extensions/standard-peer-extensions/EXTENSION-REVISION.md` — sync semantics (selective per path).
- `extensions/standard-peer-extensions/EXTENSION-HISTORY.md` — path-level transition recording when configured.
- `extensions/standard-peer-extensions/EXTENSION-CONTINUATION.md` — execution chaining; used by EXTENSION-NETWORK internally and by application-level continuations.
- `proposals/PROPOSAL-OPERATIONAL-STATE-TYPES.md` — the original proposal; landed as V7 v7.9; refreshed for the v7.40 rename.
- `proposals/implemented/PROPOSAL-SYSTEM-PEER-RENAME-AND-SUBSTRATE-CLEANUP.md` — the V7 `system/identity` → `system/peer` rename absorbed in v7.40.
- `proposals/implemented/PROPOSAL-OPERATIONAL-STATE-AND-SDK-CONVERGENCE.md` — the proposal that commissioned this guide (C11) and contains the related changes (C1: peer_id vs content_hash, C2: `system/peer/self` reservation). Implemented spec-side. "Decisions made" section documents the population-locus picks and initial-grant-scope decision.
- The SDK-domain-convergence exploration — analysis behind the proposal's decisions; deeper context on alternatives considered.
- GUIDE-CAPABILITIES.md — capability model; operational state composes with grants but capabilities are not covered here.
- GUIDE-IDENTITY.md — EXTENSION-IDENTITY (separate concern; identity is not operational state).
