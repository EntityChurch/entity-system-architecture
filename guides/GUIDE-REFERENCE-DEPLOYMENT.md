# GUIDE — reference deployment: standing up efficient, scalable managed infrastructure

**Status:** Draft — 2026-07-22. Operator runbook (user-facing, per the `guides/` charter). Documents the **target
reference deployment** for the managed-infrastructure pattern; per-role **maturity flags** mark what ships today
vs. what rides the connectivity track. Consolidation of already-specced roles + the budget-tiered model — invents
no protocol. Design basis: `docs/research/explorations/EXPLORATION-FULL-STACK-DISCOVERY-INFRA-AND-ENTITY-CHAT.md`
+ `ANALYSIS-CONNECTION-FLOWS-AND-MINIMAL-INFRA-FOOTPRINT.md`.

## 1. The governing principles (why this is cheap by design)

1. **No special infrastructure tier.** Every infra role is an ordinary peer running one system handler
   (`EXTENSION-REGISTRY` §1: "registry is just a peer"; a CDN-hosted peer *is* a Mode-S relay). You are not
   running a bespoke server fleet — you are running peers.
2. **Resolve + introduce, don't discover.** The deployment **names peers** (static registry) and **introduces
   them** (stateless tunnel); it does **not** run open discovery (a DHT). Skipping open discovery is what removes
   the one genuinely stateful, expensive component (`ANALYSIS…` Part A).
3. **The core path is near-stateless and storage-free; the two heavy roles are opt-in.** Registry + reflector +
   signaling + punch establish a direct, transport-encrypted link with **zero bulk storage and only seconds-long
   per-handshake state.** Storage (mailboxes) and relay-bandwidth are **à-la-carte fallbacks**, each tied to one
   capability. Spend on them deliberately.
4. **Valid floor / graceful degradation.** Every tier below is a valid deployment; dropping a tier drops a
   *capability*, never correctness. A budget deployment runs Tier 0, adds Tier 1 cheaply, and offers Tier 2 only
   as budget allows.

## 2. The deployment tiers (spend in this order)

| Tier | You run | Enables | Cost shape | Maturity |
|------|---------|---------|-----------|----------|
| **0 — Static reef** | a **static-CDN registry** (object storage + CDN): signed bindings + `service-advertisement` + optionally peers' trees (Mode-S static) | name resolution; service discovery; publish/read apps (SITE); async pickup of statically-hosted mailboxes | **object storage + CDN egress** — essentially free, infinite scale, **no server** | **Ships** (REGISTRY §7.4; Mode-S static) |
| **1 — + Generic tunnel** | a **stateless STUN reflector** + a **signaling relay** (one small commodity VM, horizontally scalable, no storage) | **direct P2P** for the majority (online, cone NAT ~70–90%) → live chat/file/collab at zero infra storage | **1–few small VMs**, CPU/RAM trivial, bandwidth tiny (addresses + ~KB offers) | Reflector/signaling **proposed** (connectivity track) |
| **2 — + Bounded fallbacks** | **data-relay** (TURN/Mode C) for symmetric-NAT; **inbox-relay** (Mode S live) for offline delivery | the last ~10–20% of NATs; store-and-forward to offline peers | **bandwidth** (data-relay) + **storage** (mailboxes) — **the metered tier; bound it** | Mode S **ships**; Mode C **deferred** |

**A minimal, budget-first deployment is Tier 0 + Tier 1** — a static reef plus one small tunnel VM. That already
delivers live entity-chat between online peers. Add Tier 2 only for the two features (hard-NAT, offline) that
justify their meter.

## 3. Per-role efficiency & scale playbook

### 3.1 Registry — the static reef (Tier 0)
- **Serve signed bindings + the `service-advertisement` as static, content-addressed files from object storage +
  a CDN.** No application server. The only compute is **offline** (signing bindings with the operator key).
- **Scale:** it's a CDN — effectively infinite reads, edge-cached under the binding TTL. Cost = storage +
  egress; both tiny (bindings are bytes).
- **Efficiency levers:** long TTLs on stable bindings (fewer origin fetches); content-addressed immutability (CDN
  caches forever until rotation); the "coral reef" property means **no live peer to keep up** — it works dormant.
- **Rotation/revocation:** publish a new signed binding / a revocation entity; propagation is TTL-bounded (see the
  W7 `revocation_propagation_bound` if you need it reason-about-able).

### 3.2 STUN reflector — stateless echo (Tier 1, core path)
- **Holds nothing** — it reads a packet's source address and echoes it. Per-request only.
- **Scale:** trivially horizontal (no shared state) — front several instances with round-robin/anycast; a single
  small instance already sustains very high request rates (each request is tens of bytes, microseconds of work).
- **Efficiency levers:** UDP, no TLS-per-request overhead needed for the reflection itself (the *result* is used
  in an authenticated punch); rate-limit per source to blunt abuse; co-locate with signaling on the same VM.

### 3.3 Signaling / rendezvous — transient tiny sessions (Tier 1, core path)
- **Holds a handshake session for seconds** (two peers' addresses + a pending offer, ~1–2 KB), then discards it;
  **never bulk data.** ICE-style ~30 s hard timeout caps the lifetime.
- **Scale:** each session is tiny and short-lived → one small VM handles many thousands of concurrent handshakes.
  Horizontal scale via a shared ephemeral store (small Redis) **or** stateless-with-both-peers-polling a
  rendezvous namespace. Prefer the ephemeral-store shape only if you outgrow one box.
- **Efficiency levers:** short session TTLs (reclaim fast); cap offer size; **the payload MAY be
  ENCRYPTION-opaque** so the relay does no crypto work and stores no readable data; rate-limit per peer.
- **Cheapest possible signaling:** for a Tier-0-only deployment, signaling can even ride **Mode-F relay** or a
  rendezvous *namespace* on the static reef (peers poll) — no dedicated server at all, at the cost of latency.

### 3.4 Data-relay (TURN / Mode C) — the bandwidth meter (Tier 2, fallback)
- **This is the budget risk** — it relays *every byte* of the connections that fall back to it, for their
  lifetime. Treat it as metered.
- **Minimize demand:** maximize punch success (good reflection + candidate ordering) so few connections need it;
  the prefer-cheap-path rule (`EXTENSION-REGISTRY.md` §3b.4 — a SHOULD) means peers try it last.
- **Bound the spend:** `policy: metered`/`members` on the advertised relay; per-session bandwidth + duration
  caps; per-peer quotas; drop-or-degrade on budget exhaustion (the valid floor: a dropped relay = "that
  symmetric-NAT peer must be online-and-punchable or fall to store-and-forward," not a broken system).
- **Offload:** a budget deployment MAY advertise a **commodity/community** relay (or none) rather than run its
  own — the advertisement makes this a config choice, not a code change.

### 3.5 Inbox-relay (Mode S) — the storage meter (Tier 2, fallback)
- **Stores queued messages until delivered or TTL** — the cost of *offline* delivery.
- **Bound the spend:** per-peer mailbox quotas; aggressive message TTLs; **prefer static-CDN Mode S** (a static
  host *is* a Mode-S relay — offline mailboxes can be static objects the peer polls, pushing storage onto the
  cheap CDN tier).
- **Avoid it entirely** where the app allows: a both-peers-online chat needs **no** mailbox — offline delivery is
  an explicit, per-deployment (even per-conversation) upgrade, not a baseline cost.

## 4. The seed — what the distribution ships (ties it together)

A fresh peer is connected with **zero live infrastructure** via the preloaded seed (REGISTRY §7):
- the **pinned registry identity** (trust root — "like a root CA shipped with an OS");
- a **resolver-config** pointing at the registry's static `http-poll` endpoint;
- **precedes** — pre-cached signed bindings (the deployed "CDN trio") so first run works offline;
- and the **`service-advertisement`** (`EXTENSION-REGISTRY.md` §3b), so the first resolve also teaches the peer
  the reflector + signaling endpoints — the whole cheap core path in one static fetch. *(Specified as of
  2026-08-14; **implemented in no peer yet** — the endpoints are still out-of-band configuration in practice.)*

## 5. A worked budget deployment (the recommended minimum)

- **Tier 0:** object storage + CDN hosting the registry (signed bindings, `service-advertisement`) and, if you
  host peers, their Mode-S static trees. *Cost: pennies; scales infinitely.*
- **Tier 1:** one small VM running the STUN reflector + signaling relay (co-located; stateless; add instances
  behind round-robin only when you outgrow one). *Cost: one small VM; CPU/RAM/bandwidth trivial.*
- **Tier 2 (optional, metered):** advertise a `data_relay` (your own capped/quota'd, or a commodity one) only if
  you must serve symmetric-NAT peers; advertise an `inbox_relay` only if you need offline delivery — prefer
  static-CDN Mode-S mailboxes.

This deployment delivers **live P2P entity-chat for the majority of users at object-storage + one-small-VM cost**,
degrades gracefully as budget flexes, and scales the cheap tiers effectively without bound.

## References

- `ANALYSIS-CONNECTION-FLOWS-AND-MINIMAL-INFRA-FOOTPRINT.md` (the flows, the state/cost table, the tier
  economics), `EXPLORATION-FULL-STACK-DISCOVERY-INFRA-AND-ENTITY-CHAT.md` §B (the pattern),
  `PROPOSAL-REGISTRY-SERVICE-ADVERTISEMENT.md` (design record — **ratified + folded 2026-08-14** as
  `EXTENSION-REGISTRY.md` §3b, which is the citable source for the service set and prefer-cheap-path).
- Specs: `EXTENSION-REGISTRY.md` §3b (the advertised service set, selection, prefer-cheap-path), §7/§7.4 (seed, coral reef), `EXTENSION-RELAY.md` §3.5 (MX), §3.2 (Mode S static),
  §3.4/§11.1 (Mode C), `EXTENSION-DISCOVERY.md` §3 (mDNS), `EXTENSION-NETWORK.md` §10 (reachability classes).
- Adjacent: `GUIDE-PEER-COMPOSITIONS.md` (named compositions), `GUIDE-NETWORKING-MODEL.md` (the how-it-fits
  narrative), `GUIDE-CROSS-PEER-MESSAGING.md` §6 (durable-messaging roles).
