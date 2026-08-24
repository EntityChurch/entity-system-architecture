# EXTENSION-ROUTE

**Version**: 1.0
**Status**: Active
**Depends**: ENTITY-CORE-PROTOCOL.md (v7.40+) — the only prerequisite; the table is ordinary tree-bound, cap-scoped `system/route` entities (§1).
**Related**: EXTENSION-RELAY.md (the *consumer* — reads this table per §3 when a `forward-request` has no source route); EXTENSION-DISCOVERY.md / gossip backends (*producers* — ROUTE accepts cap-gated writes and prescribes no route-computation algorithm, §1)
**Tier:** Operational — Tier 2b (network), sibling of RELAY / NETWORK / REGISTRY / DISCOVERY, per `core-protocol-domain/specs/SYSTEM-ARCHITECTURE.md` §13.1.
**Authors:** Architecture team.
**Companion:** `EXTENSION-RELAY.md` v1.1 (the consumer — reads this table when a `forward-request` has no source route, §3.1.1 source 3); `proposals/PROPOSAL-RELAY-SOURCE-ROUTED-MULTIHOP-AND-ROUTING-BOUNDARY.md` (names the resolver seam this spec demotes to the deferred computed-routing escape hatch).

---

## §1 Concept

EXTENSION-ROUTE is a **storage plane**: it holds a peer's **routing table** — a set of
`system/route` entities saying *"to reach destination D, the next hop is N (or D is direct)."*
That is the whole job. ROUTE **stores** routes and defines how the table is **read**; it does
**not** compute routes, does **not** decide how the table is populated, and owns **no** resolver
registry.

Three clean roles, along seams that already exist:

| Role | Who | What |
|---|---|---|
| **Store** | **EXTENSION-ROUTE** (this) | the table = `system/route` entities; tree-bound; cap-scoped; inspectable |
| **Consume** | **RELAY** (`EXTENSION-RELAY.md` §3.1.1) | when a `forward-request` has no source route, relay reads the local table and applies the documented match (§3) to pick the next hop |
| **Produce** | the **peer** / **DISCOVERY** / **GOSSIP** — *not* ROUTE | how routes get computed/populated: manual config, discovery-learned, gossip-learned, an operator controller. ROUTE accepts cap-gated writes; it does not prescribe the algorithm |

ROUTE is kept deliberately lean precisely because we **do not yet know** the dominant
route-management mechanism (static config? gossip-learned? DHT?) — so ROUTE provides the
data-coordination layer (how routes are referred to, stored, updated, read) and leaves the
*production* of routes to composition, to be revisited once peer feedback reveals what real
deployments converge on. This is the expose-knobs-not-values discipline.

**Design-space posture.** *Producing* routes is design space, not a theorem — there are many
correct ways (static config, DHT/Kademlia, link-state, gossip-learned, source-dictated). This
spec specifies only the **stable, neutral data layer**: the `system/route` entity, how relay
reads it, and the configure cap. It deliberately does **not** specify (or rank) how the table
is filled.

Historically this is `system/routes` (V1.0 routing tables; V2.0 Layer-5 `system/routes` as a
*separate* extension from `system/relay`). This spec restores that separation, scoped to storage.

---

## §2 The route entity (the load-bearing v1 deliverable)

A peer's routing table is a set of **route entities**, tree-bound and cap-scoped:

```
type: "system/route"
data: {
  match:       <peer_id> | "*",      ; the destination this route covers; "*" = default route
                                     ;   peer_id is Base58 per V7 §1.5; "*" is the literal
                                     ;   string token (primitive/string), NOT a peer-id
  action:      "deliver" | "forward",
  via:         <peer_id> | null,     ; REQUIRED iff action="forward"; the next hop (Base58)
  metric:      u32 | null,           ; lower = preferred when multiple routes match; null = 0
  expires_at:  <timestamp | null>,   ; ms since epoch; null = until superseded
}
; SIGNED by the configuring authority per V7 §5.2; signature reachable at the invariant-pointer
; system/signature/{hex(content_hash)}. No refs: block.
```

Stored at `system/route/{id}` (tree-bound, so per-route cap-scoping flows through the standard
tree handler — same discipline as the RELAY receive-side ruling). The table is the entities; a
read is a standard `tree:get` over the `system/route/*` subtree. Inspectable, signed,
cap-configurable like any other substrate state. **This entity shape + the match (§3) + the cap
(§5) is the entire v1 conformance surface** — no algorithm, no in-process registry.

**Representation conventions** (mirroring RELAY §3.0): `peer_id` is Base58 per V7 §1.5 (not a
bare `system/hash`); timestamps are integer ms since the Unix epoch; signatures are carried per
V7 §5.2 target-matching at the invariant-pointer, never in a `refs:` block. The `match` value
`"*"` reflects to `primitive/string` — only `via` is a peer-id (cross-impl trap: do not decode
`"*"` as a peer-id).

---

## §3 How relay reads the table (the match + precedence)

When RELAY needs a next hop and has no source route, it reads the local route table and applies
this **documented match** (ROUTE defines the semantics; relay performs the read — there is no
separate resolver object in v1, "read the table" *is* the resolution):

1. Gather route entities whose `match` is exactly `destination` or `"*"`, **not expired**.
2. Pick the lowest `metric` (exact `match` outranks `"*"` on ties — longest-match-wins,
   degenerate over a non-hierarchical peer-id space). `metric: null` = 0.
3. `action="deliver"` → terminal hop (deliver here); `action="forward"` → forward one hop to
   `via`; **no match** → `no_route`/502 (fail-closed; or RELAY §6.2.1 Mode-S fallback first).

**Precedence in the forward op** (relay-owned, RELAY §3.1.1): explicit **source route** (in the
envelope) **>** **route-table lookup** (this match) **>** built-in **direct/no_route** default
(if no table is present at all). An originator-dictated path always wins; the table is consulted
only when the originator left the path open; the trivial direct-or-`no_route` default applies
when there is no ROUTE table at all. So ROUTE is **optional**: relay works without it (single-hop
+ source-route), and gains hop-by-hop table routing when a table is present.

**Why exact-match + default, not prefix/CIDR.** Peer-ids are content hashes — a flat,
non-hierarchical space with no aggregation structure (unlike IP CIDR). Prefix globbing on Base58
strings (V1.0's `"Qm123*"`) is expressible but semantically meaningless (no topology follows the
prefix). The honest base is **exact destination → next hop, plus one default route** — exactly
V1.0's `{peer_pattern:"*", via:"PeerD"}` default-route shape, restored. This covers the
configured-mesh and VPN/gateway cases the release needs (hop-by-hop local tables: each relay's
own table says where next).

---

## §4 What ROUTE does NOT do — and the deferred computed-routing escape hatch

**ROUTE does not produce routes.** Populating the table — manual config, discovery-learned,
gossip-learned, an operator's controller — is **out of scope**. ROUTE accepts cap-gated writes
to `system/route/*` (§5) and stops there. "Your peer figures out how its routes work; we don't
do that."

**The deferred escape hatch — *computed* routing.** A static stored table cannot express a
*computed* next-hop (a DHT/Kademlia live lookup by XOR-distance, a link-state shortest-path).
Those genuinely need a *function*, not stored entities. **That** — and only that — is where a
pluggable in-process resolver seam earns its place:

```
; DEFERRED — named, not built. A future computed-routing extension MAY register:
resolve_next_hop(destination: peer_id, ctx) -> Deliver | Forward(next_hop) | NoRoute
; pure decision, no inner-envelope access, no blocking I/O on the hot path (resolve from
; cached state; refresh out-of-band); TTL is the loop backstop. Precedence: source-route >
; resolver > table > direct.
```

Two things keep this lean: (a) it is **deferred** (no driver for the v1 release — LAN/VPN/gateway
need only the stored table); and (b) **most "smart" backends populate the same `system/route`
table** rather than replace the read path — link-state and gossip-learned routing *write routes*
that relay reads exactly as in §3. Only true *live-lookup* routing (DHT) needs the resolver
function. So even when computed routing arrives, the §2/§3 storage path is unchanged for the
common case. The spec does not rank backends; a deployment installs the one its topology calls for.

---

## §5 Capability model

- `system/capability/route-configure` — may write/expire `system/route` entities (operator /
  the peer's own admin authority). Per-route cap-scoping via the tree path. This is the only cap
  ROUTE defines — it guards *who may populate the table*.
- Reading the table needs no extra caller cap — it is relay's local read of substrate state; the
  authority that matters is (a) who could *configure* routes (above) and (b) the per-hop
  `relay-forward` cap each downstream relay still enforces (RELAY §5.2). **The table names the
  path; relay's per-hop cap decides whether each hop is permitted.** A route to a relay that
  won't grant `relay-forward` simply fails at that hop (`capability_denied`/403) — a route entity
  cannot conscript the relays it names, exactly as source-routing cannot.

---

## §6 Composition

- **RELAY** (`EXTENSION-RELAY.md` v1.1) — the consumer. Reads the table per §3 when no source
  route; precedence in §3. RELAY §3.1.1 forwards to a `forward` route's `via` with `ttl_hops−1`;
  a `deliver` route → terminal raw-frame; no match → `no_route`/502 (or §6.2.1 Mode-S fallback
  first).
- **NETWORK** (`EXTENSION-NETWORK.md`) — orthogonal layer below. A route names a *peer-id* next
  hop; NETWORK §10 resolves that peer-id to a reachable *endpoint*
  (`system/peer/transport/{peer_id}/*`) and dials it (and, for NAT'd next hops, is where
  reachability/traversal lives). **ROUTE = which peer; NETWORK = how to reach that peer.** Never
  conflate.
- **GOSSIP** (`proposals/PROPOSAL-EXTENSION-GOSSIP.md`, stub) — a *future* route-**production**
  source: learned routes disseminated epidemically, which a peer then *writes into its
  `system/route` table* for relay to read. GOSSIP fills the table; ROUTE stores it; RELAY reads
  it. Distinct planes.
- **DISCOVERY / REGISTRY** — find peers / resolve names, and a route-production source: a peer
  learns reachable neighbors via mDNS/scan and *writes routes* the same way. Same pattern:
  DISCOVERY/GOSSIP produce, ROUTE stores, RELAY consumes. ROUTE assumes the destination peer-id
  is already known; finding it is DISCOVERY/REGISTRY's job.

---

## §7 Cross-impl conformance

### §7.1 v1 conformance floor (storage plane only)

1. The `system/route` entity (§2) — shape, tree-binding at `system/route/{id}`, signature per
   V7 §5.2, `route-configure` cap.
2. The documented match relay applies (§3) — exact+default match, metric tie-break, expiry-skip,
   precedence (source-route > table > direct), no-match → `no_route`.

### §7.2 Conformance vectors (cohort; Go authored + build-tested GREEN 8/8)

- `ROUTE-EXACT-1` — exact `match` → forward to `via`.
- `ROUTE-DEFAULT-1` — `match: "*"` default route.
- `ROUTE-METRIC-TIEBREAK-1` — lowest metric wins; exact outranks `*` on ties.
- `ROUTE-EXPIRED-SKIP-1` — expired route is skipped.
- `ROUTE-NOROUTE-1` — no match → `no_route`/502.
- `ROUTE-DELIVER-1` — `action="deliver"` → terminal hop.
- `ROUTE-PERHOP-CAP-1` — routed hop without `relay-forward` → `capability_denied`/403, route
  does not complete.
- `ROUTE-ABSENT-TABLE-1` — no table → relay's trivial direct-or-`no_route` default (RELAY §3.1.1).

**Cross-impl traps (from the Go build-test):**
use `hash.Bytes()` not the padded `Digest[:]` for the tree path (else non-canonical
130-char paths under SHA-256); `"*"` is a string token not a peer-id; the route-table resolver
runs **only** when both `route` and `next_hop` are absent (precedence, §3); route-table state can
leak across test runs — use fresh ephemeral peer-ids per check (no `TreeRemove` on the client yet).

### §7.3 Deferred (named, not built)

- **Computed routing — the `resolve_next_hop` resolver seam** (§4): DHT/Kademlia live-lookup,
  link-state shortest-path; their own follow-on proposals. (Link-state/gossip-learned routing
  that merely *populates* the table needs no resolver — it is a producer, §4.)
- **Route production** (manual controller, discovery-learned, gossip-learned) — out of ROUTE's
  scope by design; the peer's / DISCOVERY's / GOSSIP's job.
- **Prefix/topological aggregation** — design-space; only with a structured backend, never
  Base58 string globs.
- **Metric semantics beyond tie-break** (real distance/cost models) — backend-defined.

---

## §8 Cross-references

- `EXTENSION-RELAY.md` v1.1 §3.1.1 / §5.2 / §6.8 / §9 — the consumer (forwarding plane).
- `EXTENSION-NETWORK.md` §10 — transport resolution (the distinct layer below: peer-id → endpoint).
- `proposals/implemented/PROPOSAL-EXTENSION-ROUTE.md` — source proposal (history).
- `proposals/implemented/PROPOSAL-RELAY-SOURCE-ROUTED-MULTIHOP-AND-ROUTING-BOUNDARY.md` — the
  source-route sibling + the resolver-seam naming.
- `explorations/EXPLORATION-INFORMATION-TRAVEL-RELAY-ROUTING-GOSSIP.md` — the
  relay/routing/gossip taxonomy (§3 boundary; §2.2/§2.3 the `system/routes` lineage).
- `explorations/EXPLORATION-DISTRIBUTED-ROUTE-GRAPH-AND-ROUTING-PARADIGMS.md` — why
  ROUTE is paradigm-neutral storage (distance-vector forwarding is the universal base; link-state
  / DHT / source-route are producer/backend choices on top).
- Historical: `v1.0-core-revision/reviews/review-01/NETWORK-TOPOLOGY-EXPLORATION.md`,
  `v2.0-core-revision/protocol-layers.md` (Layer-5 `system/routes`).
