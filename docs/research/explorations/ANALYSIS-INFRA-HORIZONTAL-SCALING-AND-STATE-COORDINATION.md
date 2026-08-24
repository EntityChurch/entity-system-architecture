# ANALYSIS — horizontal scaling of the infra & how multi-server state is coordinated

**Status:** Analysis — 2026-07-22. **NOT** a proposal; it answers a build question that shapes how we implement the
managed-infra servers: *in a (mostly) stateless server form, how does the infra scale across multiple servers —
and where some state genuinely must be coordinated, how do multiple servers agree on it without a bottleneck?*
Companion to `ANALYSIS-CONNECTION-FLOWS-AND-MINIMAL-INFRA-FOOTPRINT.md`. Author: arch workspace.

**We build this.** These are **our** infra servers — reflector, signaling, relay — that we design and implement
ourselves, entity-native (on content-addressing, subscriptions, the relay). Iroh / libp2p / WebRTC are **reference
designs we learn from** (their proven techniques: rendezvous hashing, DCUtR punch coordination, ICE) — **not**
dependencies we adopt. The point is to build the system and its infra.

**The answer in one line.** The infra scales horizontally because it is architected to have **almost no
globally-coordinated mutable state**: state is either **absent** (reflector, static registry — trivial scale),
**deterministically sharded by a key both parties independently derive** (signaling — no replication, no
coordinator), **per-connection soft state that lives on one server and is rebuildable** (data-relay circuit — no
replication, just routing + failover), or **durably sharded by recipient with *declared* routing** (mailboxes —
the MX record *is* the shard map). The two rules that make this work: **(P1) make state addressable, not
session-local**, and **(P2) declare routing, don't discover it.**

---

## Part A — The four state classes (everything the infra holds falls into one)

The scaling strategy is entirely determined by *which class* a piece of state is in:

| Class | Example | Coordination need | Scaling strategy |
|-------|---------|-------------------|------------------|
| **0 — No state** | STUN reflector; static registry | none | trivial: any instance, any LB, add boxes |
| **1 — Transient shared** | a signaling handshake (2 peers' addrs + offer, seconds) | both peers must meet at the same place | **deterministic sharding** by a key both derive — no replication |
| **2 — Per-connection soft** | a data-relay circuit (lifetime of the connection) | both peers use the *same* relay; rebuildable if it dies | **route both to one relay** + soft-state failover — no replication |
| **3 — Durable sharded** | an offline mailbox (until delivered/TTL) | the recipient + senders must agree where it lives | **shard by recipient peer-id, routing is *declared*** (the MX record) |

There is **no class-4 "globally-replicated mutable state"** anywhere in the design — that is the deliberate
absence that makes it scale. Nothing requires all servers to agree on a changing shared value in real time.

---

## Part B — The two principles that eliminate the coordinator

### P1 — Make state addressable, not session-local
Never key state to "the TCP session that happens to be on server 3's memory." Key it to a value **both parties can
independently compute**: a `content-hash`, a tree `path`, a `peer_id`, or a **rendezvous key** = `hash(sorted(peer_a,
peer_b))` (or the `conversation_id`). Once state is addressable by such a key, **any server can serve it and both
parties can find it without asking a coordinator** — sharding becomes a pure function of the key.

### P2 — Declare routing, don't discover it
Peers should learn *where* a piece of state lives from data they **already share**, not from a lookup service:
- **Mailboxes:** Bob's `system/peer/inbox-relay` MX record *declares* which relay holds his mail → senders route
  there directly. The MX record **is** the shard map; no directory query per message.
- **Signaling / relay pools:** the registry `service-advertisement` lists the **pool** of reflector/signaling/
  relay endpoints; both peers **rendezvous-hash** the shared key into that pool → both pick the same server, from
  data they both already hold. No broker, no session-affinity load balancer.

Together P1+P2 mean the coordination is *computed from shared data*, never *stored in a shared coordinator*.

---

## Part C — Per-role scaling recipe

### C.1 Registry (class 0) — the CDN, no coordination
Static signed files on object storage + CDN. Scaling is the CDN's problem (solved, ∞). Multiple origins are
just replicas of immutable content; content-addressing makes replicas trivially consistent (same bytes → same
hash). **Zero coordination.**

### C.2 STUN reflector (class 0) — truly stateless
Reads a packet's source address, echoes it, forgets it. **No cross-request state at all** → scale by putting N
instances behind anycast / round-robin / any L4 LB. Adding a box needs no coordination. A peer may even reflect
off *several* independent reflectors and take the agreement (free NAT-type detection). **This is the easy one —
genuinely embarrassingly parallel.**

### C.3 Signaling (class 1) — the crux; deterministic sharding, no shared store
This is the one the question is really about. The handshake needs Alice and Bob to exchange ~1–2 KB through the
*same* rendezvous for a few seconds. **Do NOT** put a naive round-robin LB in front (it would split the pair
across servers). Instead — **client-side rendezvous hashing:**

1. The registry `service-advertisement` lists the **signaling pool** `[S1, S2, … Sn]` (Part F).
2. Both peers compute `k = rendezvous_key(alice, bob)` (a value both derive identically — e.g.
   `hash(sorted(peer_a, peer_b))` or the `conversation_id`).
3. Both compute `server = rendezvous_hash(k, pool)` (**highest-random-weight / consistent hashing**) → **the same
   server**, from data they both already hold. Both connect to it; it brokers **locally**; it holds only its own
   shard's sessions, only for the handshake.

**Why this scales with zero shared state:** each server owns a shard of the key space; no server needs any other
server's data; the "coordination" is the deterministic hash both peers ran. **Adding/removing a server** re-maps
only `1/n` of keys (the consistent-hashing property) and only affects in-flight handshakes (which simply retry —
class-1 state is transient and rebuildable). Capacity = sum of the pool; a hot key is one handshake, not a hot
shard.

**Robustness (stale-pool skew):** if Alice and Bob hold slightly different advertisements (one stale), they might
hash to different servers. Two cheap mitigations, in order: **(a)** short advertisement TTL bounds the skew window;
**(b)** a thin **inter-server forward** — a server that receives one side of a key it can also see the other side's
"looking for you" beacon forwards within the pool (a tiny gossip, addresses only). Start with (a); add (b) only if
churn warrants.

**The entity-native alternative (even less bespoke):** model signaling as **entities in a rendezvous namespace** —
Alice PUTs a signed `offer` at `…/signaling/{k}/offer`, Bob **subscribes** the prefix and reacts, PUTs an `answer`;
Alice reacts. Now signaling is **Mode-F relay + SUBSCRIPTION over a sharded namespace** — the system's *own*
coordination substrate, sharded by `{k}` exactly as above, with TTL-reaped ephemeral entities (no durable storage).
We reuse the relay we already build instead of a bespoke signaling protocol. **This is the preferred shape** — it
makes "the signaling server" just a relay serving a sharded ephemeral namespace.

### C.4 Data-relay / circuit (class 2) — one relay per circuit, soft, routed not replicated
A relayed circuit inherently lives on **one** relay (it bridges Alice's and Bob's two connections). Scaling:
- **Route both peers to the same relay** for a given circuit — rendezvous-hash the key into the advertised
  data-relay pool (same mechanism as C.3), or one peer picks and names it in the signaling offer.
- **The circuit is soft state:** if the relay dies, the peers re-establish (re-punch or re-select) — **never
  replicated.** Failover = rebuild, which class-2 state tolerates.
- **Capacity-aware selection:** because this tier is bandwidth-metered (the budget risk), selection MAY consult a
  relay's advertised load/policy and pick the least-loaded within the hash neighborhood. Scale = add relays; the
  hash spreads circuits; no relay needs another's state.

### C.5 Inbox-relay / mailbox (class 3) — sharded by recipient, MX *is* the shard map
A mailbox is durable, so it's the one genuinely stored thing. Scale it like any sharded store:
- **Shard by recipient `peer_id`** — Bob's mailbox lives on the relay(s) his **MX record declares** (P2). Senders
  route there directly; no lookup. Bob's MX may list several (priority-ordered) for **replication/durability** —
  and *those* are the only replicas in the whole design, scoped to one peer's mail, converged by the standard
  content-addressed sync (append-only, idempotent by content-hash → conflict-free).
- **Scale** by adding relays and letting peers' MX records spread recipients across them; **rebalance** by peers
  re-publishing MX (declared, not coordinated). Bound cost with per-peer quotas + TTLs (the reference-deployment
  guide).
- **Prefer static-CDN Mode-S** where latency allows — then the mailbox is CDN objects (class-0 scaling) that the
  recipient polls, pushing even this durable tier onto the cheap infrastructure.

---

## Part D — The general recipe (what "coordinate state across servers" means here)

For *any* new infra state, ask which class it is and apply the matching pattern — **you should almost never reach
for a shared coordinator:**
- **Can it be eliminated?** (reflector) → stateless; scale by adding boxes.
- **Is it transient + shared?** (signaling) → **rendezvous-hash a shared key** into a pool; no replication.
- **Is it per-connection + rebuildable?** (circuit) → **route to one server; soft-fail-over**; no replication.
- **Is it durable?** (mailbox) → **shard by an addressable key; declare the routing (MX-style)**; replicate only
  within that key's scope, converged by content-addressing.

If you ever find yourself wanting all servers to agree on a changing shared value in real time (a global session
table, a live presence map, a shared counter), that is the smell — re-key it so it shards (P1) and declare its
routing (P2), or make it soft. The design has no place that needs consensus on mutable shared state, and keeping
it that way is what keeps the servers stateless-to-scale.

## Part E — Implications for the build + the service-advertisement

- **The `service-advertisement` lists *pools*, not single endpoints** — `reflector: [.. ..]`, `signaling: [.. ..]`,
  `data_relay: [.. ..]` — so peers **client-side rendezvous-hash** into them (Part C.3/C.4). This is a small
  refinement to `PROPOSAL-REGISTRY-SERVICE-ADVERTISEMENT` (the schema already allows arrays; add the **normative
  selection rule: rendezvous-hash the connection key into the advertised pool** so both peers converge). Fold that
  in.
- **Build the signaling server as a relay over a sharded ephemeral rendezvous namespace** (C.3 entity-native
  shape) rather than a bespoke stateful broker — it reuses the relay we build and inherits its scaling.
- **The only server that ever stores durable data is the inbox-relay** (class 3), and even it prefers the CDN
  (Mode-S static). Everything else is stateless (0), transient-sharded (1), or soft (2).
- **Learn-from-Iroh notes (we build our own):** their DCUtR is our punch-coordination (C.3 signaling carries the
  `fire_at`); their relay/DERP is our data-relay (C.4, but entity-native + capability-gated); their QUIC gives
  multiplexing we'd get from our own transport work — all techniques to study, all re-implemented on our substrate.

## References

- `ANALYSIS-CONNECTION-FLOWS-AND-MINIMAL-INFRA-FOOTPRINT.md` (the per-role state/cost table this scales),
  `EXPLORATION-CONNECTIVITY-NAT-WEBRTC-UNIFIED-ARCHITECTURE.md` (the four separable services),
  `PROPOSAL-REGISTRY-SERVICE-ADVERTISEMENT.md` (advertise pools; the rendezvous-hash selection rule to fold in),
  `GUIDE-REFERENCE-DEPLOYMENT.md` (the tiers this scales within).
- Substrate reused for coordination: `EXTENSION-RELAY.md` (Mode F/S), `EXTENSION-SUBSCRIPTION.md` (reactive
  rendezvous namespace), content-addressing (V7 §1.2/§1.7 — replica consistency for free).
