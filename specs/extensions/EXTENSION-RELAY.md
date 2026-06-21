# EXTENSION-RELAY

**Version**: 1.2
**Status**: Active
**Tier:** Operational — Tier 2b (network), per `core-protocol-domain/specs/SYSTEM-ARCHITECTURE.md` §13.1.
**Authors:** Architecture team.

**Two prior arch rulings are load-bearing and unchanged by this landing:**
- `reviews/DESIGN-STATIC-TRANSPORT-AS-RELAY.md` — **static IS relay**: a static CDN is a passive intermediary; the protocol cannot tell it from active forward.
- `reviews/ANALYSIS-RELAY-CAPABILITY-CHAIN.md` — **relay is transport, not authority**: Alice→Charlie's capability chain passes through relay Bob unchanged; Bob enforces only his own ordinary "may you relay through me" cap. **No V7 amendment.**

Landscape grounding: the relay-and-aggregator pattern study (Nostr / AT Protocol / ActivityPub / libp2p circuit-relay / SMTP-NNTP → four canonical modes).

---

## §1 Concept

A **relay** is an intermediary that carries opaque, signed, capability-bearing envelopes between two endpoints. The origin's capability chain passes through unchanged; the intermediary is transport. Four canonical modes are exposed by the single `system/relay` handler:

- **Mode F — Forward.** Active forwarding peer carries the envelope toward destination. Push-shaped, transient, single-author per envelope, routed.
- **Mode S — Store-and-poll.** Passive intermediary; sender PUTs, receiver polls. Pull-shaped, persistent, single-author per envelope, addressed-by-namespace. (Static-CDN-hosted peers fall here.)
- **Mode A — Aggregate.** Persistent multi-publisher intermediary; subscribes to many publishers; serves a unified stream or queryable view. (Nostr / ATProto relay pattern.) **Deferred from v1** — cross-peer subscription dependency (§11.1a).
- **Mode C — Circuit.** Active relay maintaining a virtual circuit between two peers that can't dial each other directly. (libp2p circuit-relay-v2 / TURN.) **Deferred from v1** (§11.1); the static-host Mode S inbox covers most NAT cases.

A relay peer is **just a peer running `system/relay`**. No special infrastructure role; same substrate as everything else. Operators install whichever modes their deployment supports and advertise them.

**v1 ships Mode F (forward) + Mode S (store-and-poll).** Mode A and Mode C entity types and operations are named for forward-compatibility; their normative spec text is deferred to follow-on proposals.

This spec defines the v1 modes, the entity types per mode, the handler operation surface, the capability model (per the ANALYSIS ruling), and composition with the landed INBOX, CONTINUATION, NETWORK, REGISTRY, and DISCOVERY extensions plus the in-flight STORAGE-SUBSTITUTE-HTTP / BRIDGE-HTTP / ENCRYPTION proposals.

---

## §2 Modes table

| Mode | Push/Pull | Persistent? | Multi-author? | Routing | Stateful? |
|---|---|---|---|---|---|
| F (Forward) | push | transient | single-per-envelope | routed-to-destination | transparent |
| S (Store-and-poll) | put-then-poll | persistent | single-per-envelope | addressed-by-namespace | transparent |
| A (Aggregate, deferred) | publish-then-subscribe | persistent | multi-publisher | broadcast or filtered | may add stream ordering |
| C (Circuit, deferred) | bidirectional | transient | single-per-circuit | routed via reservation | transparent |

Operators select modes per deployment; the spec exposes the mechanism, not the policy.

---

## §3 Entity types

### §3.0 Representation conventions

- **`peer_id`** is the Base58 peer identifier per **V7 §1.5** (`key_type || hash_type || digest`, Base58-encoded). It is **NOT** a bare `system/hash`. Relay routing, namespace addressing, and `put_by`/`source_peers` all use this canonical Base58 form. (Routing keyed on the wrong representation silently fails to match — the cross-impl trap REGISTRY/DISCOVERY pinned.)
- **`<system/hash>`** denotes a content hash in the **bare 2-key form** `ECF({type: "system/hash", data: <33-byte H>})` per V7 §1.5 / EXTENSION-TREE.
- **Timestamps** are **integer milliseconds since the Unix epoch.**
- **Signatures** are carried per **V7 §5.2 target-matching** and are reachable at the invariant-pointer `system/signature/{hex(content_hash)}`. Relay entities do **not** carry `refs: { signature }` blocks.

### §3.1 Mode F — Forward

```
type: "system/relay/forward-request"
data: {
  destination:     <peer_id>,            ; terminal recipient (Base58, §3.0)
  route:           [<peer_id>] | null,   ; v1.1 — the remaining relay hops, in order, ending
                                         ;   at `destination`; the originator's SOURCE ROUTE.
                                         ;   null/absent → single-hop (use `next_hop`) or
                                         ;   table-routed (no route + no next_hop → §3.1.1).
                                         ;   omitempty: a v1.0 single-hop request encodes
                                         ;   byte-identically (the field is absent when null).
  next_hop:        <peer_id | null>,     ; single-hop shorthand / route-table output (Base58).
                                         ;   When `route` is present and non-empty, `next_hop`
                                         ;   is advisory and MUST equal route[0] if both set
                                         ;   (else invalid_request/400, pre-dispatch).
  ttl_hops:        u32,                  ; relay-transport hop budget; decremented per hop
  envelope_inner:  <bstr, 33 bytes>      ; the content hash (bare system/hash digest form,
                                         ;   1 format byte + 32 digest) of the carried inner
                                         ;   envelope; the inner entity travels as opaque raw
                                         ;   bytes in the materialized envelope's `included` set
}
```

**`route` is the forward path (v1.1).** It lists the relay hops *after* the immediate
dispatch target, ending at `destination`. The originator sends the `forward-request` EXECUTE
to the **first** relay; that relay is **not** in `route`. `route` is the inverse of V2.0's
`path: [peer_id]` (where you've been) — it is where you are going. `route` and `next_hop` are
both **absent** for the table-routed case (§3.1.1 reads the local route table per
`EXTENSION-ROUTE.md`); precedence is **source route > route table > direct** (§3.1.1).

**`envelope_inner` is a `data` field carrying the raw 33-byte content hash** — not a
`refs` entry (cohort convergence R6/R7, 3-impl). The relay reads the hash from `data`
to address the carried entity and forwards the inner entity's raw bytes verbatim from
the `included` set; it never decodes them. Content-store dedup is preserved (the inner
is still content-addressed by the hash).

**The carried inner envelope is a full materialized `system/envelope` — `{root, included}`
— not a bare root** (§2.3 ruling; the inner must carry its own signatures/caps so the
terminal hop delivers a self-contained, independently-verifiable message; a bare root
would arrive unverifiable). The relay treats it as opaque bytes regardless (§9).

The inner envelope MAY be encrypted (per the ENCRYPTION proposal); the relay sees only an opaque content-addressed blob it never decodes (§9). The relay MUST decrement `ttl_hops` on each forward and MUST reject (fail-closed) if it is 0 on receipt.

**`ttl_hops` vs V7 `bounds.ttl`.** `ttl_hops` bounds the *relay transport path* length and lives on the *relay* envelope. It is distinct from V7 `bounds.ttl`, which bounds the *computation/dispatch chain* (with `await_stack` cycle detection) and travels *inside* the inner envelope — which the relay cannot read. A single logical dispatch may cross N relay hops. **The relay decrements only `ttl_hops` and never touches the inner envelope's bounds.**

Async response correlation is **not** a relay field — it is carried inside the opaque inner envelope (INBOX `deliver_to` / `deliver_token`); see §6.2.

#### §3.1.1 Per-hop next-hop determination + intermediate vs terminal hop (the dispatch shape)

On receiving a `forward-request`, a relay first determines its **next hop** from exactly
three sources, consulted in this precedence order (the relay is the *forwarding plane*; the
choice of path is the *routing plane* — `EXTENSION-ROUTE.md` §3):

1. **Source route** (`route`, §3.1) — if `route` is present and non-empty, `next = route[0]`.
   The originator dictated the path. (If `next_hop` is also set it MUST equal `route[0]`;
   a mismatch is `invalid_request`/400 **pre-dispatch**, before any forward.)
2. **`next_hop` shorthand** — else if `next_hop` is set, `next = next_hop` (the degenerate
   single-hop / single-element route; v1.0 behavior).
3. **Route table** — else (both absent) consult the local route table per `EXTENSION-ROUTE.md`
   §3 (exact `match` or `"*"` default, lowest metric, expiry-skipped). The table yields
   `Deliver` (terminal), `Forward(via)` (`next = via`), or no match → `no_route`/502. A
   peer with no route table installed at all applies the trivial **direct-or-`no_route`**
   default (deliver iff `destination` is directly reachable, else `no_route`/502) — exactly
   v1.0 behavior.

A relay is at the **terminal hop** when `next == destination` (the destination is the chosen
next hop / directly reachable from this relay). Otherwise it is at an **intermediate hop**.

- **Intermediate hop.** The relay forwards a `forward-request` to `next` (with `ttl_hops`
  decremented) as a `system/relay:forward` EXECUTE. The next relay is another RELAY peer.
  When the inbound request carried a **source route**, the forwarded request MUST set
  `route' = route[1:]` (pop the head) and `next_hop' = route'[0]` (or null if `route'` is
  empty); `destination` and `envelope_inner` are unchanged. (Intermediate forwards MUST
  populate `next_hop'` to `route'[0]` so a downstream receiver that reads `next_hop` first
  resolves correctly — cross-impl trap.)

**Worked example A→B→C→D** (D = destination): A → B `{destination: D, route: [C, D],
ttl_hops: 8}`; B sees `next = C ≠ D` → intermediate, forwards B → C `{destination: D,
route: [D], next_hop: D, ttl_hops: 7}`; C sees `next = D == destination` → terminal, delivers
the bare inner envelope to D. D verifies the inner signature exactly as on a direct connection.

**TTL is the loop bound.** `ttl_hops` MUST be ≥ the route length for the route to complete;
on receipt with `ttl_hops == 0` the relay rejects `ttl_exhausted`/400 (fail-closed, no partial
delivery). An explicit finite path + TTL is **self-bounding** — no separate loop-detection
state is required (distinct from the inner envelope's `bounds.await_stack`, which detects
request/response cycles a relay cannot see — §3.1).
- **Terminal hop — raw-frame forwarding (§2.1 ruling).** The relay **writes the inner envelope's raw bytes verbatim into the destination's inbound frame** and dispatches them as a normal inbound message — exactly the bytes the destination would have received on a direct connection. The relay **MUST NOT decode-then-re-encode** the inner envelope (which is what `system/envelope` `{root, included}` already is — a complete message ready to deliver). It reads `data.envelope_inner` (the hash) only to locate the bytes; it copies the bytes; it does not parse them. The relay envelope (`forward-request`) is consumed by the terminal relay and is **not** delivered onward.

Raw-frame (not decode-then-redispatch) is required to deliver true byte-identity-to-direct: a decode/re-encode round-trip can be byte-identical for ECF-canonical inputs but is **not guaranteed** to be, and §3.1.1's promise is exactness. This resolves the Rust (raw-frame) vs Python (decode-then-redispatch) divergence in favor of raw-frame; an impl whose dispatch hookpoint takes an inner entity whose `.Data` is raw bytes writes `.Data` verbatim rather than decoding it.

This is required by the cap-chain ruling (§5.1): the destination verifies the inner envelope's signature + capability chain *exactly as on a direct connection*, and therefore **MUST NOT need the RELAY extension installed merely to receive** a forwarded message. The terminal relay delivering its own relay-typed wrapper, or a re-encoded inner, would break the transparency guarantee and the byte-identical-to-direct property (§9). For a browser/NAT'd destination the "dispatch" is the push down the held outbound session (§6.2.1); the verbatim inner-envelope bytes are what is pushed.

### §3.2 Mode S — Store-and-poll

```
type: "system/relay/store-entry"
data: {
  namespace:       <path>,               ; where the receiver polls
  expires_at:      <timestamp | null>,   ; ms since epoch; standard cap-style expiry
  put_by:          <peer_id>,            ; the authenticated session/connection peer (Base58, §3.0)
  envelope_inner:  <bstr, 33 bytes>      ; content hash of the carried inner envelope
                                         ;   (data field, raw 33-byte bstr — as §3.1)
}
```

**Both the store-entry and the inner envelope are tree-bound** (the relay indexes everything it serves in the tree, so the standard tree-handler cap check governs per-namespace access):

- the **store-entry** at `system/relay/store/{namespace}/{entry_hash_hex}` (`{entry_hash_hex}` = the hex33 form of the entry hash, path-safe per REGISTRY §6.3);
- the **inner envelope** at `system/relay/store/{namespace}/inner/{inner_hash_hex}` — nested under the same namespace subtree, so a single namespace-scoped tree-read cap covers both fetches. Tree-binding is path→hash (PRIMER invariant #1); the inner's bytes still live once in the content store, so dedup is preserved and two namespaces may point at the same inner hash.

`:put` (and the §6.2.1 `:fallback`) MUST establish both bindings. The receiver polls `system/relay:poll` against `namespace`; the relay returns enumerated store-entry hashes (§4.2), which the receiver fetches via `tree:get` on the namespace-scoped paths — **`system/content` is NOT a receive-side dependency** (§4.2). STORAGE-SUBSTITUTE-HTTP is the first deployed Mode S — a static-CDN-hosted peer's tree IS a Mode S relay namespace (§6.5).

**`put_by` is placement-identity, not authorship.** `put_by` MUST equal the
**authenticated session/connection peer** — the peer that placed this entry over this
connection (§2.2 ruling). The relay sets/verifies it from the authenticated connection
identity and rejects a mismatch (`put_by_mismatch`/400, §4.3). Using the
session/connection peer (not the wire-EXECUTE author) is what makes `put_by` mean
*placement*: on a cross-peer dispatch the two differ, and placement is the connecting
peer. This makes `put_by` trustworthy for abuse attribution, rate-limiting, and GC. It
is **NOT** an authorship claim: the actual author is whoever signed the *inner*
envelope (V7 §5.2), which the relay cannot read. A poller MUST derive authorship from
the inner envelope's signature, never from `put_by`. (In the §6.2.1 fallback path the
forwarding relay places the entry on the origin's behalf, so `put_by` = the relay's
session identity, while authorship remains the origin's inner-envelope signature —
exactly why the two must stay distinct.)

### §3.3 Mode A — Aggregate (deferred from v1)

Entity type named for forward-compatibility; not normatively specified (Mode A deferred — §11.1a):

```
type: "system/relay/aggregate-subscription"
data: {
  source_peers:    [<peer_id>],          ; publishers being aggregated (Base58)
  source_subtrees: [<path_pattern>],     ; per-publisher subtree filter
  filter:          <filter expression | null>,   ; per-event filter (opaque to relay)
  emit_to:         <namespace>           ; where the aggregated stream surfaces
}
```

### §3.4 Mode C — Circuit (deferred to follow-on)

Named for forward-compatibility; not normatively specified:

```
type: "system/relay/circuit-reservation"  ; reservation slot
type: "system/relay/circuit-dial"         ; connect through reserved slot
```

### §3.5 The inbox-relay declaration — the MX-equivalent (closes Q2)

A peer publishes a signed declaration of *where its mail is stored when it is
unreachable* — the DNS-MX analog. This is what gives the §6.2.1 Mode-S fallback a
resolvable, self-certifying target, closing the v1.0 Q2 gap (which-relay-holds-my-mail).

```
type: "system/peer/inbox-relay"
data: {
  relays: [
    { relay:     <peer_id>,            ; the relay holding my mail (Base58, §3.0)
      namespace: <path>,               ; where to put it (default: my own peer_id)
      priority:  u32 }                 ; lower = preferred (MX-priority; backups higher)
  ],
  expires_at:  <timestamp | null>      ; ms since epoch; null = until superseded
}
; SIGNED by the declaring peer per V7 §5.2; signature reachable at
; system/signature/{hex(content_hash)}. No refs: block.
```

**Authored by the peer; served by an always-on holder.** The peer authors and signs
the declaration (it is the source of truth, self-certifying). But it MUST be *served*
for resolution by a holder that is up when the peer is down — **the MX record lives in
the directory, not on the mail server**. Authority travels with the signature, not the
host (V7 §1.4 universal namespace), so any holder serves it authoritatively without
being trusted: it can serve or withhold the signed bytes, but cannot forge or redirect.

**Resolution (always-on first):** REGISTRY is the primary home — a name resolution
returns the inbox-relay declaration alongside the name→peer binding (the A+MX-in-one-zone
pattern; the registry serves a synced copy referenced by the binding's current head).
Secondary holders: the relay named in the declaration (itself always-on), and a
local/relationship cache. The peer's own tree is the authoritative origin but **not**
the fallback-path source (it's offline exactly when needed). A resolver MUST verify the
declaration's signature against the resolved `peer_id` and reject (fail-closed) on
mismatch, and MUST try `relays` in ascending `priority` order. A peer with no registry
presence and no prior relationship is reachable by a stranger only via a cached copy —
honest parity with a mail server that has no DNS entry.

(Full design + security audit + the 6 cross-impl test vectors incl. the end-to-end
`INBOX-RELAY-FALLBACK-1`: `proposals/implemented/PROPOSAL-PEER-INBOX-RELAY-MX-EQUIVALENT.md`.)

---

## §4 Handler operations

```
system/relay:forward             ; Mode F — accept envelope; forward to next hop
system/relay:put                 ; Mode S — accept envelope; store at namespace
system/relay:poll                ; Mode S — receiver polls a namespace for entries
system/relay:advertise           ; All — relay announces supported modes + limits
```

Deferred — Mode A: `system/relay:subscribe` / `:unsubscribe`. Mode C: `system/relay:reserve-circuit` / `:dial-circuit`.

### §4.1 `system/relay:advertise` (all modes)

A relay announces its capabilities via an entity at `system/relay/advertise/{relay_peer_id}`:

```
type: "system/relay/advertise"
data: {
  modes:           [<mode_id: "F"|"S">],             ; v1 modes; "A"/"C" when they land
  endpoints:       [<dial-able endpoint per NETWORK §6.5>],
  limits: {
    max_envelope_size:    u64 (optional),
    max_storage_bytes:    u64 (optional, Mode S),
    forward_rate_limit:   u32 (optional, Mode F; envelopes/sec)
  },
  caps_required:   [<cap_path>],                     ; what cap is needed to use this relay
  expires_at:      <timestamp | null>                ; ms since epoch
}
```

The advertise entity MUST be signed by `relay_peer_id` per V7 §5.2; its signature is reachable at `system/signature/{hex(advertise.content_hash)}` (no `refs:` block — §3.0). Consumers query this entity (standard tree fetch) to see what's available and what cap they need. *Finding* the relay's `peer_id` is DISCOVERY's / REGISTRY's job — §6.7.

### §4.2 Operation wire schemas (Mode F + Mode S)

Each op is an EXECUTE against the `system/relay` handler carrying a request entity; the relay returns a result entity or a V7 ERROR (§4.3). The carried **inner envelope stays opaque** throughout (§9) — these schemas describe only the relay-envelope surface.

**`system/relay:forward` (Mode F).** Request IS a `system/relay/forward-request` (§3.1). Result:

```
type: "system/relay/forward-result"
data: {
  status:          "forwarded" | "queued-fallback" | "rejected",
  next_hop:        <peer_id | null>,         ; hop actually used, if forwarded
  stored_at:       <path | null>             ; namespace, if queued-fallback (§6.2.1)
}
```

`ttl_hops` is decremented before forward; `ttl_exhausted` if 0 on receipt. If the destination has no live session, the relay falls back to Mode S and returns `queued-fallback` with `stored_at` set (§6.2.1). Async response correlation, if any, rides inside the inner envelope (§6.2) — the relay-level result is only the immediate ack.

**`system/relay:put` (Mode S).** Request IS a `system/relay/store-entry` (§3.2). Result:

```
type: "system/relay/put-result"
data: {
  status:      "stored",
  stored_at:   <path>,                       ; system/relay/store/{namespace}/{hash}
  entry_hash:  <system/hash>,                ; hash of the stored store-entry
  expires_at:  <timestamp | null>            ; ms since epoch
}
```

**`system/relay:poll` (Mode S).** Poll is a **relay-owned** operation over the namespace subtree `system/relay/store/{namespace}/*`; the cursor and entry ordering are the relay's (NOT INBOX's — §6.1):

```
type: "system/relay/poll-request"
data: {
  namespace:   <path>,
  since:       <cursor | null>,              ; relay-owned cursor; null = from start
  limit:       <uint | null>                 ; null = backend default
}

type: "system/relay/poll-result"
data: {
  entries:     [<system/hash>],              ; hashes of store-entry entities
  cursor:      <opaque>,                      ; pass back as `since` to continue
  has_more:    bool
}
```

`poll` returns store-entry **hashes** (pointers), not inline bytes. The receiver then does **two `tree:get`s on the namespace-scoped relay paths** (§3.2): it fetches the store-entry at `system/relay/store/{namespace}/{entry_hash_hex}`, reads its `envelope_inner` field (the inner hash), and fetches the inner envelope at `system/relay/store/{namespace}/inner/{inner_hash_hex}`. Both are content-addressed and preserve content-store dedup.

**`system/content` is NOT a relay receive-side dependency.** A receiver consumes Mode-S traffic with `tree` + `relay` only. The "content-addressed two-hop discipline (NETWORK §6.5.3.1)" is **path→hash→bytes**, realized two ways by transport — neither the `system/content` extension: over a **live session** a single materializing `tree:get` (V7 §1.7) on the relay-tree path returns the entity bytes directly; over a **static CDN** (STORAGE-SUBSTITUTE-HTTP, §6.5) `TREE_GET` returns the bound `system/hash` pointer (Amendment 6) and the consumer second-hops `CONTENT_GET /content/{hex33(H)}`. `CONTENT_GET` (a transport route / core hash-fetch) is distinct from `system/content:get` (an extension handler op); the inner bytes are reached by tree resolution plus a core hash-fetch, never by installing `system/content`. This is also why the static-CDN Mode-S deployment — which has no live `:poll` handler — consumes through the same tree paths: the surface is uniform across live and static Mode S.

**Cursor ordering is impl-defined; the contract is resumability, not a specific order.**
A relay MAY order entries FIFO/insertion-order, lexically by hash, or otherwise — all
conformant (cohort: Go uses insertion-order, Python lexical-by-hash; both pass). The
only requirement is that passing `cursor` back as `since` makes forward progress and
eventually drains the namespace without loss or unbounded repetition. **Conformance
tests MUST assert resumability (cursor round-trips and drains), NOT a FIFO order.**

**Empty namespace is not an error.** A poll against a namespace the caller is authorized to read that currently holds no entries MUST return `{entries: [], has_more: false}` at status 200 — *not* `namespace_not_found`. This is the normal steady state for a freshly-created inbox (e.g. a reconnecting destination polling its own peer-id namespace before anything is queued, §6.2.1). `namespace_not_found` (§4.3) is reserved for deployments that require explicitly-provisioned namespaces, polled against one that was never provisioned.

### §4.3 Error taxonomy

All relay ops fail via the standard V7 ERROR shape (status centralized per V7 §3.3; codes domain-scoped). Errors are **fail-closed**: on any error the op performs no partial effect (no forward, no store, no dequeue) — consistent with the v7.75 substrate-resilience posture (bounded resources, deliver-or-signal, never silently drop).

**Reused V7 floor codes** (do not reinvent):

| Status | Code | Meaning | Emitted by |
|---|---|---|---|
| 403 | `capability_denied` | caller lacks the §5.2 cap for this op/namespace (V7 §3.3 / v7.71 authz contract) | all |
| 413 | `payload_too_large` | exceeds advertised `max_envelope_size` (**V7 §4.10(a) floor**) | forward, put |

**Relay-owned codes** (conditions V7 has no floor for; the relay owns its code domain per V7 §3.3, as DISCOVERY owns `discovery_scan_overflow`):

| Status | Code | Meaning | Emitted by |
|---|---|---|---|
| 400 | `ttl_exhausted` | `ttl_hops` reached 0 on receipt | forward |
| 400 | `invalid_request` | `route` and `next_hop` both set but `next_hop ≠ route[0]` (§3.1.1, rejected pre-dispatch) | forward |
| 502 | `no_route` | no source route, no `next_hop`, and no route-table match / destination not directly reachable (§3.1.1) | forward |
| 429 | `rate_limited` | exceeds advertised `forward_rate_limit` | forward |
| 400 | `namespace_invalid` | malformed namespace path | put, poll |
| 404 | `namespace_not_found` | namespace not provisioned (deployments requiring explicit provisioning only; empty ≠ not-found, §4.2) | poll |
| 507 | `storage_full` | exceeds advertised `max_storage_bytes` | put |
| 400 | `expired_on_arrival` | `expires_at` already past at put time | put |
| 400 | `put_by_mismatch` | `store-entry.put_by` ≠ authenticated session/connection peer (§3.2) | put |
| 502 | `no_inbox_relay` | destination unreachable *and* declared no inbox-relay; nothing to store-and-forward to (§3.5/§6.2.1) | forward |

`expired_on_arrival` is **400, not 410** by design: `:put` is a *creation* op, and an already-expired entry is a dead-on-arrival request (a client error in the submitted content) — not a 410 Gone, which would imply a previously-existing resource that is now gone. Nothing was ever stored, so there is no resource to be Gone.

---

## §5 Capability model

Per `reviews/ANALYSIS-RELAY-CAPABILITY-CHAIN.md`, authoritative.

### §5.1 The two orthogonal permission relationships

When Alice → relay Bob → Charlie:

1. **Alice ↔ Charlie** — the actual authority. Alice holds a capability that lets her do this operation on Charlie. Roots at Charlie; verified by Charlie on receipt. **Bob constructs no link in this chain.**
2. **Alice ↔ Bob** — the right to *use Bob as a relay*. Roots at Bob; ordinary handler-permission; orthogonal to (1).

Bob cannot forge Alice's signature; cannot escalate; worst he can do is *drop* or *delay*. Confidentiality from Bob is ENCRYPTION's job, orthogonal.

### §5.2 The per-mode relay-side caps

Bob enforces a separate ordinary cap per mode op:

```
system/capability/relay-forward       ; Mode F — may forward envelopes
system/capability/relay-put           ; Mode S — may put entries at a namespace
system/capability/relay-poll          ; Mode S — may poll a namespace
system/capability/relay-advertise     ; All — may publish advertise entity (typically operator-only)
system/capability/relay-subscribe     ; Mode A (deferred) — may open subscriptions
```

Scoped per V7 §5.4 patterns. Deployments can grant any subset:
- Public-CDN-style: `relay-poll: include all` (open read); `relay-put: include operator`.
- Friend-network forwarding: `relay-forward: include trusted contacts`.

### §5.3 V7 §5.8 registry placement

RELAY belongs in the V7 §5.8 **non-site** column. Relay's cross-peer data flow is transport of an already-constructed chain (Alice→Charlie), already covered by V7 §6.5 dispatch. **No V7 amendment required.** A one-line documentation clarification in V7 §5.8 naming RELAY as a non-site is sufficient (deferrable to V7's next routine errata).

### §5.4 Per-mode cap-scope notes

- **Mode F:** the inbound envelope's destination is opaque to Bob; Bob's cap check is *"may this peer forward through me"* + optional rate limits. The next hop comes from one of three sources (§3.1.1) — **source route** (originator-dictated, §3.1), **route table** (`EXTENSION-ROUTE.md`), or the trivial **direct** default; the spec exposes the slot, not the algorithm (Kademlia, link-state, gossip-learned, static are deployment/routing-extension choices, never relay's). **The per-hop `relay-forward` cap applies to every hop of a source route.** Source-routing does **not** let an originator conscript arbitrary relays: each relay in `route` independently enforces `relay-forward`, so a route can only traverse relays that grant the forwarder the cap (a mid-route relay without the grant → `capability_denied`/403 at that hop; the route does not complete). This is the abuse/DoS bound, identical to single-hop. The route table cannot conscript either — a `forward` route to a relay that won't grant `relay-forward` fails the same way.
- **Mode S:** Bob's cap check is *"may this peer put at this namespace"* and *"may this peer poll at this namespace."* The two are independent (typical: public read + restricted write). No INBOX cap is in the path (Mode S is self-contained — §6.1). **The namespace-scoped tree-read cap that gates `:poll` visibility also governs the receiver's post-poll `tree:get`s** on the store-entry and inner-envelope paths under `system/relay/store/{namespace}/*` (§3.2/§4.2) — receive-side needs `tree` + `relay`, not `system/content`.
- **Mode A / Mode C (deferred):** publish/subscribe and reserve-circuit caps respectively.

### §5.5 Default grants + fallback authority

- **Self-poll default grant.** The `relay-poll` cap is enforced on the *relay* (the peer running Mode S), not on the destination. On first install, **a peer running Mode S SHOULD install a default grant authorizing every requesting peer P to `relay-poll` at namespace = P's own `peer_id`** — i.e. "any peer may always read the fallback inbox the relay holds *for that peer*." This is what lets a reconnecting destination retrieve its own Mode-S fallback inbox (§6.2.1) on any relay that hosts one for it, with no operator action. It mirrors EXTENSION-DISCOVERY §4.1's default-grants-on-first-install posture and the seed-policy discipline. (It does **not** widen read access to anyone else's namespace — P may poll only namespace = P.)
- **Fallback store runs under forward authority.** When a relay falls back from `:forward` to a Mode-S store for an unreachable destination (§6.2.1), the store is performed under the **forwarder's `relay-forward` grant** — the caller does NOT need a separate `relay-put` for the fallback. The fallback namespace (= destination `peer_id`) is the relay's choice, not the caller's, so no caller-side put authority is implicated.

---

## §6 Composition with other extensions

### §6.1 INBOX (`EXTENSION-INBOX.md` v5.9)

Mode S is **inbox-shaped but self-contained**. It is structurally a mailbox at a passive store, but it does **not** delegate to the INBOX operation:

- Landed INBOX exposes exactly one op, `receive` (a write-ahead delivery into a tree path, optionally delegating to a CONTINUATION at that path). It has **no `poll`, no cursor, no pagination.**
- Mode S `:put` and `:poll` are therefore **RELAY's own operations** over `system/relay/store/{namespace}/*`. The poll cursor and entry ordering are relay-owned.
- **Optional backing:** a deployment MAY back a Mode S namespace with an INBOX path (the put-shape matches INBOX `receive`), but that is a deployment composition with its own caps, out of RELAY v1 scope. RELAY neither requires nor delegates to the INBOX op.

### §6.2 CONTINUATION (`EXTENSION-CONTINUATION.md` v1.20) — async response correlation

**Async response correlation is carried inside the opaque inner envelope, not in any relay field.** The relay is genuinely unaware of it. The mechanism reuses INBOX `deliver_to` / `deliver_token` (V7 §3.2 + INBOX §3.3):

1. Alice's inner envelope is an EXECUTE carrying `deliver_to` (a reply URI in Alice's own tree) + `deliver_token` (a cap authorizing delivery to that inbox path). Alice installs a `system/continuation` entity at the reply path.
2. The relay forwards the inner envelope opaquely; Charlie ingests and verifies it directly (Bob invisible to the cap-chain check — §5.1).
3. Charlie's response is delivered as an authenticated EXECUTE to `deliver_to`.
4. At Alice's reply path, INBOX `receive` finds the continuation entity and delegates to the continuation handler's `advance` (INBOX §3.3) — that is the resumption. CONTINUATION advances because a result *landed at its path*; it does not depend on how the result arrived.

The relay does NOT carry per-request state for response correlation. (Legacy v4.0-Rust's PendingRelayRegistry is retired; its replacement is INBOX delivery + a CONTINUATION at the reply path — not a relay-level correlation field.)

#### §6.2.1 Delivery to a browser / NAT'd destination

NETWORK §14: browser peers are always connection initiators and cannot accept inbound connections. The rule:

> **Mode F delivery to a browser/NAT'd destination reuses the destination's live outbound session** (NETWORK §14 Browser Peer Considerations; the §6.5.1b peer-initiated dispatch model; the §10 `held_connection_client` reachability class): the relay pushes down the already-open socket the destination holds to the relay. **If the destination has no live session, Mode F falls back to Mode S** (store at a namespace; destination polls on reconnect), returning `forward-result {status: queued-fallback}`.

**Fallback target (resolved via the inbox-relay declaration, §3.5).** The forwarding
relay resolves the destination's `system/peer/inbox-relay` declaration (§3.5) and stores
at the highest-priority reachable `(relay, namespace)`. This is the MX-record lookup —
the destination *published* where to leave its mail. If the destination declared no
inbox-relay, the relay MAY store at the **default convention** — namespace =
destination `peer_id` on the current forwarder (`system/relay/store/{destination_peer_id}/{hash}`)
— which works when the forwarder is itself a relay the destination will poll; if neither
a declared relay nor a usable default yields a reachable store target, surface
`no_inbox_relay` (502, never a silent drop). On reconnect the destination polls
`system/relay:poll {namespace: <its own peer_id>}` (or the namespace it declared). The
store runs under the forwarder's `relay-forward` authority and the destination polls
under its self-poll default grant (§5.5).

**Scope limit — namespace is derivable; the holding relay is not.** The *namespace* needs no side-channel (it IS the destination's identity, fixed by convention). But **which relay(s) hold a fallback inbox for a destination** is **not** discoverable in v1: a third party forwarded via a relay the destination may not know about. v1 fallback therefore works when the destination knows which relay(s) to poll — a configured home relay, or a relay the sender and destination both already use. General "any relay might have queued for me" discovery is the same family as Mode A / a registry-of-relays and is **deferred** (§11.1a). Because the holding-relay problem is unsolved in v1, a per-relay namespace override would only add a second unknown (now the destination must also learn each relay's namespace) for no benefit — so the override is **dropped from v1**. The forward path, when a fallback-relay-discovery driver arrives, is a derivable `{configured_prefix}/{destination_peer_id}` suffix so the destination still walks N relays at one known suffix (Q5).

#### §6.2.2 Registering the NETWORK §10.2 dispatch-fallback seam (sender/responder-side escalation)

§6.2.1 covers the forward path **once a relay is already carrying the message**. It does not cover the **originating peer's own dispatch** deciding to *use* store-and-forward when its direct delivery fails — the "deliver to an offline stranger / respond to an offline caller" case. That decision is the NETWORK §10.2 `dispatch_fallback` seam, and **RELAY is its v1 implementation.**

When RELAY is installed, it registers `dispatch_fallback(peer_id, execute)` at NETWORK §10 step 4. The policy:

1. **Resolve** the target's `system/peer/inbox-relay` declaration (§3.5) — via REGISTRY (landed v1.0; the A+MX-in-one-zone resolution) or a configured declaration — **verifying its signature against the target's identity before use** (§3.5 resolver MUST; same fail-closed verify R8's SIG-1 vector gates). No resolution path ⇒ the seam returns `null` (→ NETWORK's unchanged terminal; no new infrastructure required).
2. If this peer holds `relay-put` authority at the declared `(relay, namespace)` **or shares a relay with the target**, **Mode-S `:put` directly**.
3. Else **Mode-F forward** through a relay this peer can forward through (`relay-forward` grant) targeting the destination; that relay performs the §6.2.1 MX store under forward authority (§5.5). **Prefer rung 3 (Mode-F-through-a-relay)** over rung 2 where both are available — it carries the §6.2.1 fallback + §5.5 authority story without the sender needing put rights; fall to direct `:put` only where the sender already shares the relay.
4. Target declared no inbox-relay and no forward path exists ⇒ return `null` (→ NETWORK's `502`, surfaced as RELAY's existing `no_inbox_relay`, §4.3).

**v1 limit — which relay to forward through.** Rung 3 requires the sender to know *a* relay that grants it `relay-forward`: v1 = a configured home relay (the same "holding-relay is not discoverable" limit §6.2.1 already documents). General relay discovery stays deferred (Mode A / registry).

**Conformance gates the outcome, not the rung.** The MUST is: a RELAY peer's §10 dispatch to an offline target with a resolvable, signature-verified inbox-relay declaration escalates to store-and-forward (the envelope lands at the target's inbox; the target polls and verifies the sender signature **byte-identical to a direct delivery**, §9). The rung-2-vs-rung-3 choice is policy. No new capability, no new error code. The forcing function is the cross-impl `INBOX-RELAY-FALLBACK-1` flow (Go build-tested GREEN; cohort mirrors at impl time). Per NETWORK Amendment 11 / `proposals/implemented/PROPOSAL-DISPATCH-FALLBACK-SEAM-SENDER-SIDE-STORE-AND-FORWARD.md`.

### §6.3 NETWORK (`EXTENSION-NETWORK.md` Amdt 10)

Active modes use NETWORK's transport primitives. **NETWORK §6.5 transport profiles** apply directly; **NETWORK §10 outbound dispatch** resolves the target's reachability. Mode S backed by a static CDN uses BRIDGE-HTTP for put/get translation; the **NETWORK §6.5 `http-poll` transport profile** (§6.5.3) specifies the wire shape, and the two-hop pointer discipline is **NETWORK §6.5.3.1** (Amendment 6, normative). Mode S backed by a live peer uses NETWORK directly (no bridge).

### §6.4 REGISTRY (`EXTENSION-REGISTRY.md` v1.0)

**Registry-peer-as-Mode-S-publisher.** A registry peer publishes its binding entities at `system/registry/binding/{binding_hash}` (REGISTRY §3); consumers fetch via Mode S (static-CDN-hosted or live). This is REGISTRY §8.1 verbatim — no special registry transport.

**Aggregator-as-meta-registry.** A peer running Mode A subscribed to N registry peers' binding subtrees serves the combined view. A consumer MAY add the aggregator peer as one backend entry in its own (peer-local) resolver chain — `resolver-config` is peer-local and not synced (REGISTRY §4), so the aggregator is *consumed*, it does not install itself into another peer's config.

> **v1 deferral.** Depends on RELAY **Mode A**, deferred from v1 (§11.1a). REGISTRY §8.2 independently marks it v1-deferred behind Mode A. Cross-registry federation lands when Mode A lands. See `EXTENSION-REGISTRY.md §8.2`.

### §6.5 STORAGE-SUBSTITUTE-HTTP (`PROPOSAL-EXTENSION-STORAGE-SUBSTITUTE-HTTP.md`, in-flight)

STORAGE-SUBSTITUTE-HTTP IS Mode S relay for content. From the CONTENT side: local store misses → consult substitute sources, one being a static-CDN-hosted peer. From the RELAY side: that peer is a Mode S relay holding content at a namespace. **Same wire shape; same trust discipline.** v1 RELAY documents it as the first deployed Mode S; no re-specification.

**Mechanism A, not BRIDGE-HTTP** (the recurring conflation). The static-CDN Mode-S surface is **HTTP-as-storage-transport** (NETWORK §6.5 `http-poll`: the bytes on the wire ARE entity bytes, hash-verified) — the **read** side already exists in the `http-poll` serving routes (`TREE_GET`/`CONTENT_GET`, §6.5.3.1). It is **NOT BRIDGE-HTTP**, which is Mechanism B (fetch *foreign* HTTP content — HTML/JSON — and wrap it as `system/bridge/http/fetched`; not on the v1 critical path). A truly-static CDN is **read-only over the protocol**: the publisher writes to the static origin out-of-band (deployment action), and a receiver polls + fetches over `http-poll` — so the read flow is exercisable today with no proposal landed. A *protocol-mediated* write to an HTTP origin (if wanted) is **STORAGE-SUBSTITUTE-HTTP**'s surface (in-flight), still not BRIDGE-HTTP.

### §6.6 ENCRYPTION (`PROPOSAL-EXTENSION-ENCRYPTION.md`, in-flight)

Confidentiality from the relay is ENCRYPTION's job. The relay carries an opaque content-addressed inner-envelope ref; ENCRYPTION wraps the inner content so the relay (and any observer of its storage) sees only ciphertext. Orthogonal to transport mechanics.

### §6.7 DISCOVERY + REGISTRY relay-finding (`EXTENSION-DISCOVERY.md` v1.0, `EXTENSION-REGISTRY.md` v1.0)

RELAY's `advertise` entity describes a relay's *capabilities*; it does not solve *finding* the relay. Relay-finding is delegated:

- **REGISTRY** — a relay reachable by a name; the consumer resolves the name → relay `peer_id`, then fetches `system/relay/advertise/{peer_id}`.
- **DISCOVERY** — a relay on the local network MAY announce itself via DISCOVERY `:announce` (mDNS DNS-SD `_entity-core._udp.local.`), after which a consumer's `:scan` surfaces it as a candidate; the consumer then fetches its advertise entity. This announce-a-relay option is **design-space**, not a mandate — a relay reachable by name or static endpoint needs no DISCOVERY participation.

The clean-separation boundary holds: REGISTRY = name→peer, DISCOVERY = find unknown peers, RELAY = transport between found peers. RELAY assumes the relay peer is already reachable by one of these means.

### §6.8 ROUTE (`EXTENSION-ROUTE.md` v1) — the routing plane

RELAY is the **forwarding plane** (move the envelope one hop); ROUTE is the **routing plane** (the stored table answering "to reach D, who's next?"). When a `forward-request` arrives with **no source route**, RELAY reads the local `system/route` table and applies ROUTE §3's documented match (exact-or-`*` default, lowest metric, expiry-skip) to pick the next hop (§3.1.1 source 3). RELAY **consumes** the table; it never *computes* or *populates* routes — production (manual config / DISCOVERY-learned / GOSSIP-learned) is the peer's job (ROUTE §4). The per-hop `relay-forward` cap still gates each routed hop (a `forward` route to a relay that won't grant it fails `capability_denied`/403; ROUTE §5). ROUTE is **optional**: RELAY works without it (single-hop + source route), and gains table routing when a table is present.

---

## §7 Mode S signed mutable pointer (tracked blocker)

For Mode S to serve "current state of publisher's tree" use cases, a signed mutable pointer at a named location (the IPNS-equivalent) is required. **This lives in CONTENT, not RELAY** — it's a "current head of tree" question at the content layer (`reviews/DESIGN-STATIC-TRANSPORT-AS-RELAY.md §3`; likely a small CONTENT amendment).

> **Tracked blocker.** Mode S **content-addressed fetch-by-hash** works fully in v1. The **"give me the publisher's latest" (no hash specified)** cases are **gated on the CONTENT signed-mutable-pointer work** and are **not deliverable in RELAY v1.** The cohort should not assume the latest-pointer case ships with Mode S v1.

---

## §8 GC posture (per `core-protocol-domain/guides/GUIDE-GC.md`)

- **Mode F entries:** transient; GCed once forwarded (or after a small bounded retry window). No persistent state.
- **Mode S entries:** persistent; honor `expires_at`; operator-configured `relay_store_retention` knob (default unlimited); per-namespace eviction policy optional.
- **Mode-S fallback entries** (queued-fallback, §6.2.1): persistent until polled or `expires_at`; same retention knob.
- **Advertise entities:** persistent; renewed on relay restart; respect `expires_at`.
- **Mode A subscriptions / aggregated backlog** (deferred): operator-configured retention; default unlimited.

Knobs exposed; default values conservative (unlimited / off / largest); operators pick policy.

---

## §9 Envelope opacity (the central transport invariant)

There are two envelopes; only one is decoded.

- The **relay envelope** (`forward-request` / `store-entry`) IS decoded — that is how the relay reads its outer routing fields (`destination`, `next_hop`, `namespace`, `ttl_hops`).
- The **inner envelope** (the carried payload, addressed by the `data.envelope_inner` 33-byte hash, §3.1) MUST be held as opaque bytes. Implementations MUST NOT decode it as part of forward, store, aggregate, **or terminal-hop delivery**. Routing decisions MUST be based solely on the relay envelope's outer fields. Carry the inner as raw bytes — e.g. Go's `cbor.RawMessage` — keyed by its content hash; never decode it in the relay path; at the terminal hop write those raw bytes verbatim into the destination frame (§3.1.1). (Rationale: decode-and-re-encode can be byte-identical for ECF-canonical inputs but is not *guaranteed* to be; holding — and forwarding — the inner as an opaque content-addressed blob preserves both bit-fidelity and content-store dedup. This is the single discipline behind the §2.1 raw-frame ruling: there is no point in the relay path, *including terminal delivery*, where the inner is decoded.)

---

## §10 Cross-impl conformance

### §10.1 MUST implement

A conformant RELAY v1 **implementation** MUST implement **both** Mode F (forward) and Mode S (put/poll) — both are part of the v1 floor and both MUST be testable. A **deployment** MAY advertise and enable any subset of modes (a static-CDN relay enables only Mode S; a forwarding relay enables only Mode F). Conformance is gated on the implementation's mode support; mode enablement is deployment policy.

- `system/relay:forward` (Mode F): forwarding to a designated next hop, with `ttl_hops` decrement and reject-at-zero, the intermediate-vs-terminal-hop dispatch shape (§3.1.1 — terminal hop forwards the inner envelope's raw bytes verbatim, no decode/re-encode), and Mode-S fallback for unreachable destinations (§6.2.1). **Source-routed multi-hop (v1.1, §3.1.1):** when `route` is present, pop the head per hop (`route' = route[1:]`, `next_hop' = route'[0]`, `ttl_hops−1`), enforce `relay-forward` at every hop, reject `next_hop ≠ route[0]` pre-dispatch (`invalid_request`/400). A single-element `route` MUST behave identically to the equivalent `next_hop` single-hop request, and a v1.0 single-hop request (no `route`) MUST encode byte-identically (omitempty).
- `system/relay:put` + `system/relay:poll` (Mode S): namespace addressing, relay-owned cursor (resumable; order impl-defined, §4.2), empty-namespace-returns-empty (§4.2), `put_by == authenticated session/connection peer` verification (§3.2).
- `system/relay:advertise`: entity creation + publication, signed per V7 §5.2 (no `refs:` block).
- Per-op cap enforcement (§5.2); self-poll default grant + fallback-under-forward-authority (§5.5); fail-closed error surfacing (§4.3).
- Envelope opacity (§9).

### §10.2 SHOULD implement

- Rate limiting per advertised limits (`rate_limited`/429).
- Retention policy per operator config.

### §10.3 MAY implement

- **Computed / mesh routing algorithms for Mode F** (Kademlia/DHT live-lookup, link-state shortest-path, BGP-like) — behind the deferred `resolve_next_hop` resolver seam (`EXTENSION-ROUTE.md` §4). Source-routed multi-hop is **v1.1 MUST** (§10.1), not MAY; only *self-routing / mesh / DHT* is deferred. Most "smart" backends merely *populate the `system/route` table* (a producer) and need no new read path.
- Mode A (aggregate) — deferred from v1 (§11.1a). When implemented: `:subscribe` / `:unsubscribe`, multi-source aggregation, aggregation semantics.
- Mode C (circuit) when the follow-on proposal lands.

### §10.4 MUST NOT

- Forge signatures on relayed envelopes.
- Decode the inner envelope (addressed by the `data.envelope_inner` hash) in the forward / store / aggregate / terminal-delivery path (§9).
- Re-encode or substitute the inner envelope's bytes between receipt and forward/store.
- Construct cap-chain links on behalf of relayed peers (§5.3).
- Inject content into the inner envelope.
- Decrement or otherwise touch the inner envelope's V7 `bounds` (the relay decrements only its own `ttl_hops` — §3.1).

---

## §11 What this extension does NOT cover (v1 scope cuts)

### §11.1 Mode C (circuit relay) — deferred

Circuit relay for NAT traversal is structurally distinct (bidirectional virtual circuit; reservation discipline; bandwidth accounting). Defer to a follow-on proposal when a driver materializes; the static-host Mode S pattern covers most NAT cases via Mode S inbox addressing (§6.2.1). Entity types named for forward-compatibility (§3.4); operations named (§4); spec text deferred.

### §11.1a Mode A (aggregate) — deferred from v1

**RELAY v1 ships Mode F + Mode S only.** Mode A's "aggregator subscribes to N publisher peers' subtrees" requires **cross-peer subscription initiation** that does not exist in current substrate — subscription engines (the landed EXTENSION-SUBSCRIPTION and the cohort impls) are local-tree-only; cross-peer subscription is new choreography layered above. Mode A is the federation answer (Nostr / ATProto) and the substrate for aggregator-as-meta-registry (§6.4); both depend on a cross-peer subscription mechanism that deserves its own proposal. **RELAY v1 explicitly does NOT deliver registry federation.** REGISTRY §8.2 inherits this deferral.

**Transport-fallback dependency.** Mode F's "forward to `next_hop`" is itself a transport call; if it fails, RELAY needs the fallback loop the transport-composition exploration will specify (try alternate transports / re-resolve via registry / give up cleanly). Mode F's routing-on-failure is gated on `EXPLORATION-TRANSPORT-COMPOSITION-AND-FALLBACK.md` (Phase 2). v1 Mode F ships with explicit `next_hop` required plus the Mode-S fallback (§6.2.1) until that lands.

### §11.2–§11.6 Other cuts

- **Routing algorithms** (§11.2): the next-hop *decision* is split from forwarding (§3.1.1). **Source-routed multi-hop ships in v1.1** (originator dictates the path; §3.1). The **route table** (`EXTENSION-ROUTE.md`) is the stored-table read. **Computed/mesh routing** (DHT, link-state, gossip-learned next-hop) is deferred to a routing extension behind the `resolve_next_hop` resolver seam (`EXTENSION-ROUTE.md` §4) — relay defines the slot, never the algorithm. *(Corrects the v1.0 blanket "smarter routing is Phase-2": only self-routing/mesh is deferred; source-routing is not.)*
- **Aggregation semantics** (§11.3, Mode A deferred): mechanism exposed; semantics per-deployment.
- **Anti-abuse / rate limiting** (§11.4): operator concern; relay advertises limits; enforcement per-impl.
- **Signed mutable pointer** (§11.5): CONTENT-layer; tracked blocker (§7).
- **Cap-chain delegation through relay** (§11.6): that is `delegation`, not `relay` — already specified (V7 capability delegation). Not in v1 RELAY.

---

## §12 Configurations

- **Public CDN-style Mode S:** Mode S handler; `relay-poll: include all`; backed by static host via BRIDGE-HTTP per STORAGE-SUBSTITUTE-HTTP. Already deployed via CDN trio.
- **Friend-network Mode F forwarding:** Mode F handler; `relay-forward: include trusted contacts`; Mode-S fallback for offline contacts (§6.2.1).
- **Private hybrid (small team):** Mode F for live messaging + Mode S for offline inbox.
- **Public Mode A aggregator** (federation backbone): deferred to when Mode A lands (§11.1a).

---

## §13 Open questions (informative)

- **Q1: Mode A subscription wire shape** (deferred with Mode A).
- **Q2: Mode S retention defaults** — operator-config knob (§8); defaults conservative.
- **Q3: Mode F multi-hop — RESOLVED (§3.1.1).** Source-routed multi-hop ships in **v1.1** (the originator names the path in `route`). Absent a source route, the relay reads the local route table (`EXTENSION-ROUTE.md`) or applies the trivial direct-or-`no_route` default. **Computed/mesh routing** (a relay independently finding the path with no source route and no stored table — DHT/Kademlia/link-state) stays deferred to a routing extension behind the `resolve_next_hop` seam (`EXTENSION-ROUTE.md` §4); not RELAY's job.
- **Q6: Onion-routing wire form (open — flagged by Go, arch ruling pending before onion ships).** The v1.1 `route` is a cleartext `[peer_id]`. An onion variant (each hop sees only its own next hop, encrypted to its key) changes the CBOR element type from `peer_id` to `bstr` — *not* a forward-compatible field add. Two paths: (a) a `route_form: "clear" | "onion"` discriminator so the element type is inferable at decode, or (b) a **separate** `onion_route: [bstr]` wire field. Either works for the impls; the ruling is deferred to the Mode-C + ENCRYPTION cycle (§2.4 of the source-route proposal; onion is out of v1.x scope).
- **Q4: Mode A re-publication** (deferred) — the aggregator does NOT re-sign; receivers verify against the original publisher's signature.
- **Q5: Fallback-relay discovery — RESOLVED (§3.5).** Which-relay-holds-my-mail is now resolvable via the destination's published, self-certifying `system/peer/inbox-relay` declaration (the MX-equivalent), served always-on by REGISTRY. The remaining deferral is only multi-hop *routing* when no declared relay is reachable (Mode A / mesh, §11.1a) — a genuinely beyond-email capability, not the rendezvous gap.

---

## §14 Cross-references

- `reviews/ANALYSIS-RELAY-CAPABILITY-CHAIN.md` — cap-chain ruling (relay = transport).
- `reviews/DESIGN-STATIC-TRANSPORT-AS-RELAY.md` — static-IS-relay ruling.
- `reviews/ANALYSIS-AND-REVIEW-EXTENSION-RELAY-PRE-IMPL.md` — pre-impl scrutiny + case studies + revision grounding.
- `explorations/EXPLORATION-RELAY-AND-AGGREGATOR-PATTERN.md` — landscape + primitive extraction.
- `EXTENSION-INBOX.md` (v5.9), `EXTENSION-CONTINUATION.md` (v1.20), `EXTENSION-NETWORK.md` (Amdt 10), `EXTENSION-REGISTRY.md` (v1.0), `EXTENSION-DISCOVERY.md` (v1.0), `EXTENSION-ROUTE.md` (v1, routing plane) — composition surfaces.
- `proposals/implemented/PROPOSAL-RELAY-SOURCE-ROUTED-MULTIHOP-AND-ROUTING-BOUNDARY.md` — v1.1 source-route fold; `explorations/EXPLORATION-INFORMATION-TRAVEL-RELAY-ROUTING-GOSSIP.md` — the relay/routing/gossip taxonomy that grounds the boundary.
- `proposals/PROPOSAL-EXTENSION-STORAGE-SUBSTITUTE-HTTP.md`, `proposals/PROPOSAL-EXTENSION-BRIDGE-HTTP.md`, `proposals/PROPOSAL-EXTENSION-ENCRYPTION.md`, `proposals/PROPOSAL-STATIC-PEER-HOSTING-UMBRELLA.md` — in-flight dependencies.
