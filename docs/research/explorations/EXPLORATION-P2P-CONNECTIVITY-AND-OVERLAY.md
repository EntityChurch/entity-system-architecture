# EXPLORATION — the P2P connectivity substrate & the overlay fast-follow (consolidated design review)

**Status:** Exploration / design review — 2026-07-24. **NOT** a proposal, **NOT** ratified. **Self-contained:**
it consolidates the connectivity + overlay design into this repo so future work references only this document and
the in-repo specs it cites — nothing external. Author: arch workspace at the operator's request.

**What this is.** One consolidated review of the whole *"connect any two peers, then let them form a
self-managing P2P network"* design — enough to confirm we have the right pieces and are moving the right
direction before an implementation plan. It rests on this repo's folded specs, its in-repo proposals, and its
in-repo sibling explorations (§11). Where it describes design not yet in a spec, it anchors that design to the
**reserved seams the folded specs already leave open** (`EXTENSION-ROUTE` `resolve_next_hop`, `EXTENSION-RELAY`
Mode A/C, `EXTENSION-DISCOVERY` deferred backends, `EXTENSION-NETWORK` §10.2) — so nothing here is invented from
nothing, and nothing here depends on a document outside this repo.

**The two-phase shape (the operator's steer):**

- **Phase 1 — the minimal connectivity/discovery layer.** A keyed-hash rendezvous over **near-stateless
  NAT/STUN services**: resolve → introduce → punch, then data flows peer-to-peer. The central services do
  connectivity only, never the data path. **Build this first.**
- **Phase 2 — the overlay (a fast-follow).** The self-organizing P2P infrastructure *on top* — membership,
  provider lookup, feed fan-out — that **offloads steady-state traffic from the connection services to the peer
  network.** Worth getting right; it follows Phase 1, it does not gate it.

These are **two independent layers, not just two phases** (§3a, §5): the connection/discovery layer stands
alone — peers connect and talk with no overlay at all — and the overlay depends on it only to bootstrap, so
either can evolve, or be swapped for a different provider's, without the other. And the infrastructure is
**provider-plural**: anyone can run it — your own private service, a community one, or the entity-church service
as a **universal fallback** — because the design gives no provider special status (§3a).

---

## §1 The frame — one connectivity pattern, many consumers

A chat, a social feed, a site, a multiplayer game, a content/CDN fetch are **different consumers over one
substrate.** What you exchange and why is limitless; the **pattern of connectivity is invariant.** So we build
the connectivity substrate once, and each application is a different *key* and a different *consumer* over it.

Concretely, the substrate is **one rendezvous-by-key primitive** — `announce(K)` / `find(K)` — where the key is
derived per use:

| Mode (`SIGNALING §2.2` tag) | `K` is… | Consumer |
|---|---|---|
| **Direct** (`pair`) | `H` of the two peer-ids (sorted) | message a known peer |
| **Topic / room** (`tag`) | `H` of a public topic label | "join `chess`" → matched with peers on that key |
| **Shared secret** (`secret`) | `H` of an agreed private string | two peers meet on a term only they know — a zero-infra gate |
| **Blind lobby** (`lobby`) | `H` of a well-known constant | "just connect me to anyone here" |
| **Content** *(Phase 2)* | a content hash | "who has this?" — the CDN case, answered by provider-lookup (§5b), not the signaling rendezvous |

You design **one** mechanism; the modes are how `K` is computed. The first four are Phase-1 signaling-rendezvous
keys; their **normative derivation** — one mode-tagged hash, `H("entity:rdv:v1" ‖ mode ‖ SEP ‖ canonical(input))`,
with the cross-peer canonicalization pins (peer-id sort, byte-exact strings, fixed `mode`/`SEP` bytes) — lives
in `PROPOSAL-CONNECTIVITY-SIGNALING-AND-PUNCH` §2.2; this table is the map of consumers over it. A **shared secret
is just an agreed string both peers use at the same time on the same service pool** — mechanically identical to a
`tag`, distinguished only by the secrecy/entropy of the string (§2.2). This sits naturally on two invariants the model
already carries: entities are **content-addressed** and **dedup'd** (`EXTENSION-CONTENT`, `EXTENSION-TREE`), so
content-mode rendezvous — "who has hash X" — is a question the address space is already shaped to answer.

The real design axis is **who answers `find(K)`** — the central services (Phase 1) or the peer overlay
(Phase 2). That split *is* the two-phase plan.

---

## §2 The floor we build on (what is already true)

**Folded / Active in this repo — do not rebuild:**

- **Name resolution.** `EXTENSION-REGISTRY` `resolve(name) → (peer_id, transports, …)`; the registry is a
  **static, near-stateless publisher** (§7.4) that stores **pointers, not bulk data**, and scales like a CDN with
  delegated-manifest sharding (§6a.3). It holds no live session/presence state.
- **Reach an offline or NAT'd peer.** `EXTENSION-RELAY` **Mode S** store-and-forward + the signed
  `system/peer/inbox-relay` MX declaration (§3.5), resolved through the registry; driven by the
  `EXTENSION-NETWORK` §10.2 `dispatch_fallback` seam. This is the **async floor** — message anyone, get a reply,
  today.
- **Public ↔ NAT'd, both directions.** `EXTENSION-NETWORK` §10 `held_connection_client`: a NAT'd peer that holds
  an outbound socket is reachable both ways (the relay/coordinator pushes down the socket it holds).
- **Push subscription, including cross-peer.** `EXTENSION-SUBSCRIPTION` is push (fires on tree change), and
  **cross-peer subscribe/notify already works** — a follower subscribes on a publisher and the publisher pushes
  from its own local tree (§6), with bounded dissemination trees for fan-out (§8). *(This corrects an earlier
  draft: base cross-peer subscription is not missing — see §5 for the one part that is.)*
- **Content model + authenticity.** `tree = path → hash`, content-store dedup, and author authentication by
  signature target-matching (`EXTENSION-IDENTITY`; the V7 §5.2 rule). A replayed post is the *same* dedup'd
  entity — idempotent by construction.
- **Scaling law.** There is **no central poll/heartbeat loop** in the model: a live central presence map is
  explicitly the anti-pattern; liveness is **edge-local + subscription-propagated** (`system/peer/status` on
  transition, per `PROPOSAL-NETWORK-LIVENESS-REACTIVE-BUILDOUT`), delivery prefers **push down a held socket**
  over poll, and any Mode-S poll is a **recipient-sharded mailbox drain**, not a central presence poll.

**In-repo DRAFT (design done, unbuilt):**

- `PROPOSAL-NETWORK-REACHABILITY-FACTS` — reflection (STUN-role) + dial-back (am I publicly reachable?) +
  candidate gathering. The landable-now wedge.
- `PROPOSAL-CONNECTIVITY-SIGNALING-AND-PUNCH` — a `system/signaling` rendezvous carrier + the punch coordination
  messages + TCP-simultaneous-open. Operator calls recorded: stateless rendezvous carrier; TCP-simopen first →
  our-QUIC later; relay/async is an opt-in dedicated-peer tier, never basic infra.
- `PROPOSAL-REGISTRY-SERVICE-ADVERTISEMENT` — the signed advertisement of the reflector/signaling **pools** +
  the rendezvous-hash selection rule (both peers hash the shared key to the same server with zero shared state).

**Reserved seams (folded, empty on purpose) — where Phase 2 plugs in:** `EXTENSION-ROUTE` `resolve_next_hop`
(§4) for computed routing; `EXTENSION-RELAY` **Mode A** (aggregate) and **Mode C** (live circuit), named for
forward-compatibility (§11.1); `EXTENSION-DISCOVERY` internet-scale backends (§8.2/§10). The architecture
anticipated the overlay and left the sockets open.

---

## §3 Phase 1 — the minimal connectivity/discovery layer (keyed-hash over NAT/STUN)

The near-term target: **any two peers connect through a minimal keyed-hash rendezvous, then talk directly.** The
central services do connectivity only.

**The minimal center — three near-stateless services:**

- **Reflector** (STUN-role): "what is my public address?" Stateless.
- **Signaling rendezvous:** two peers deposit/collect opaque signed coordination messages at a shared
  `rendezvous_key`, meeting at the same server via rendezvous-hash. Transient per-handshake state, TTL-reaped.
- **Connector pool:** the advertised set of live on-ramp/relay endpoints (`PROPOSAL-REGISTRY-SERVICE-ADVERTISEMENT`
  §3.1). Soft state.

**The flow (all in-repo primitives):** resolve the peer/key (registry) → gather candidates + learn public
address (reachability-facts) → exchange over the signaling rendezvous → **simultaneous-open punch** → a direct
peer-to-peer transport that the rest of dispatch uses unchanged (a punched connection is an ordinary transport,
per `EXPLORATION-CONNECTIVITY-NAT-WEBRTC-UNIFIED-ARCHITECTURE`). If the punch fails (symmetric NAT, a minority of
pairs), fall back to a relay — the **opt-in** tier, never the basic center.

**Cost profile (per `ANALYSIS-CONNECTION-FLOWS-AND-MINIMAL-INFRA-FOOTPRINT` + `-INFRA-HORIZONTAL-SCALING`):** the
center resolves names and introduces peers with **zero bulk storage and only transient (~seconds, ~1 KB)
per-connection state**, and scales horizontally because it holds **no globally-coordinated mutable state** — each
service is stateless, keyed-hash-sharded, or soft-per-connection. This is exactly the "keyed-hash minimal layer"
of the operator's steer.

**Buildable now.** Phase 1 needs only: reachability-facts + signaling/punch + service-advertisement (all in-repo
DRAFT) + the connector pool. No overlay, no computed routing, no new central state.

### §3a The connection service is a dumb opaque pipe — pluggable providers, two admission modes

The connection service can be **its own standalone thing**, and that is a feature, not a compromise. The reason
is a property already in the design: the signaling carrier **never decodes what it moves** — it holds an *opaque*
signed blob at a hash-derived rendezvous key and hands it back, and reflection is stateless
(`PROPOSAL-CONNECTIVITY-SIGNALING-AND-PUNCH` §2.1, mirroring `EXTENSION-RELAY` §9 opacity). So the service
understands **no entities** — it moves opaque bytes keyed by a hash. The entity-ness (signatures, capabilities)
lives in the *payload* and at the *peers*, never in the carrier. Three consequences:

- **It can be maximally scalable.** No per-request entity runtime, no capability evaluation, no stored state — a
  single node handles introductions at raw-socket throughput. This is why we start here: maximize the scalability
  of one node.
- **A stranger's service is safe to use.** The carrier can neither read nor forge your coordination messages
  (they are signed end-to-end; content self-verifies), so you can rendezvous through infrastructure you do not
  trust — the deferred-trust property doing real work.
- **The two layers are independent.** This connection/discovery layer stands entirely alone; the Phase-2 overlay
  depends on it only to bootstrap and not the reverse. Either can evolve — or be swapped for another provider's —
  without the other.

**Two admission modes, one interface — scope both:**

1. **Open / stateless (the start).** No capability required; admission is **rate-limit only** (per-source,
   per-key). The service is a **standalone component** — it may use the entity system for its own setup /
   identity / config, but the hot path (reflect, deposit/collect at a key) is a dedicated non-entity code path.
   The default, tuned for per-node scale.
2. **Capability-gated (optional).** The *same* interface behind an entity **capability check** — a grant from the
   operator is required to use it (a private or club rendezvous). Same reflect/deposit/collect logic; admission
   adds one capability evaluation. For deployments that want the entity auth model in front of their connection
   layer.

The interface is identical across modes; only admission differs — so "putting it behind the capability model" is
adding a gate, not a redesign. The standalone service and the `system/signaling` entity handler present the same
surface to peers; they differ only in whether the entity runtime sits in the request path. *(Normative home:
`PROPOSAL-CONNECTIVITY-SIGNALING-AND-PUNCH` §2.3.)*

**One node, two deployment lenses — both first-class.** The value is not only the public-fallback case. The same
node is worth running **even if you expose it only to your own managed peers** — a private device mesh (your
phone, laptops, servers punch to each other anywhere, through infrastructure you alone control, no third party in
the introduction path) — *and* it functions as **generic shared infrastructure anyone can use.** Same code, same
wire surface; a private pool + optional capability gate vs. an open public pool is **configuration, not a fork.**
The self-hosted mesh and the universal fallback are the two ends of one spectrum, not two products.

**Pluggable providers + a universal fallback.** A peer selects its connection service from a preference-ordered
set — **your own private service, a community service, or the entity-church service as the universal fallback** —
advertised as pools (`PROPOSAL-REGISTRY-SERVICE-ADVERTISEMENT`); no provider has special status in the design.
One consequence to pin: **both peers of a handshake must meet at the same provider** (they rendezvous at a shared
key on a shared server). So a private/community provider works when both peers share it — named in the
contact/link that introduced them — while a **well-known provider (the church, or any) is the common denominator
that guarantees any two strangers can always meet.** Provider plurality gives isolation, scale, and trust control
to whoever wants it; the shared fallback preserves the "connect with anyone, anywhere" property.

---

## §4 The two-tier reality — native infrastructure, browser leaves

A hard constraint that shapes everything (and is already the established model in
`EXPLORATION-CONNECTIVITY-NAT-WEBRTC-UNIFIED-ARCHITECTURE` + `EXPLORATION-FULL-STACK-DISCOVERY-INFRA-AND-ENTITY-CHAT`):

- A **browser cannot accept inbound**, has no stable address, and has no background life. Its only P2P transport
  is **WebRTC**, which works only after signaling and often needs a relay fallback for symmetric-NAT pairs.
- **WebRTC over the internet is not a separate mystery:** it is the *same* reflect → signal → punch → relay
  triad as native, implemented by the browser's built-in ICE stack and driven by our signaling. Same candidate
  model, same central services.

**Conclusion:** a browser peer is **first-class at the data layer** (two browsers exchange directly over WebRTC
once introduced) but **can never be infrastructure** — never an on-ramp, relay, or overlay node, because it
cannot advertise "connect to me." Therefore:

- **Tier 1 — native / public peers (the Tauri app).** Publicly reachable or hole-punchable, running in the
  background. **All infrastructure — on-ramps, relays, and the whole Phase-2 overlay — lives here.**
- **Tier 2 — browser / mobile leaves.** Data participants that consume Tier 1 and cannot constitute it.

Any provider runs the seed connection services (the entity-church one as the universal fallback, §3a); volunteer
**native** peers grow Tier 1; browsers ride it via a native bridge. A pure-browser network cannot self-bootstrap
— the on-ramp tier is necessarily native. Note the two axes are orthogonal: *tier* (native infra vs. browser
leaf) is about who can host, while *provider* (private / community / fallback) and *mode* (open vs. gated, §3a)
are about who runs a given service and on what terms.

---

## §5 Phase 2 — the overlay (the fast-follow)

The overlay is what lets the peer network **take over steady-state work from the center**: finding peers,
locating content, and fanning out feeds — so the central services stay a thin cold-start/introduction on-ramp.
Each piece plugs into a reserved in-repo seam.

**a) Membership & peer discovery at internet scale** → fills `EXTENSION-DISCOVERY`'s deferred internet backend
(§8.2/§10). Approach: **epidemic membership** — peers propagate known-peer/liveness facts (a natural extension of
the edge-local `system/peer/status` model) so the peer set self-heals without a central roster. Open fork:
**build our own structured lookup vs. ride an existing external DHT** for global "where is peer/key K." Either
way the base case (LAN + registry-name + link) already works today; this is the internet-scale addition.

**b) Provider lookup — "who has content/key K"** → fills `EXTENSION-ROUTE`'s `resolve_next_hop` computed-routing
seam (§4). Approach: **provider records** (key → the set of peers serving it), the CDN primitive. Content
addressing makes this safe: the record points you at *a* holder, and the hash verifies the bytes regardless of
who served them — so an untrusted overlay peer can serve content correctly. Open fork: the routing paradigm
(next-hop vs. hold-the-graph) — scale-dependent, to be chosen at design time.

**c) Feed fan-out at scale — the one remaining relay primitive** → `EXTENSION-RELAY` **Mode A** (aggregate),
named-deferred (§11.1). **This is the only genuinely-missing delivery piece, and it is narrow:** base cross-peer
subscribe/notify already works (§2), and bounded dissemination trees already spread fan-out
(`EXTENSION-SUBSCRIPTION` §8). What is *not* built is an **aggregating relay** — a peer that subscribes
*outbound* to many publishers and republishes a *merged* stream to many followers (the shape a busy account or a
firehose needs). RELAY §11.1a notes the shipped subscription engines are local-tree-only, so this outbound
aggregation is the choreography to add. Needed only at scale; a personal feed does not need it.

**d) The bootstrap → overlay handoff.** Bootstrap centrally **once** (reach one known service), then the overlay
carries membership and lookup; the center becomes the **cold-start + punch fallback**, not the steady-state hub.
The pieces for this are (a)+(b); the one folded line to reconcile is that the registry resolve loop is currently
re-consulted *on transport failure* — the handoff adds an "ask the overlay first, the registry last" path
without losing the registry's fail-closed correctness.

**e) The self-replenishing on-ramp set** — the genuine increment over a static connector pool: a reachable
native peer, once it verifies it is publicly dialable (reachability-facts), advertises itself into the connector
set on a TTL and can serve as an on-ramp/relay for newcomers. The set then self-heals as peers come and go.
*Role differentiation by capacity (which peers become heavier relays) is a design question, not yet specified.*

---

## §6 The social feed — the first real consumer

A social feed is the right first application because it **rides the async floor and needs no punch** — it proves
the substrate on the easy path, and live/low-latency (games, calls) becomes a later upgrade, not a dependency.

- **A personal feed is nearly all built.** Advertise `name → peer`; serve the feed as a signed
  content-addressed subtree with a **signed, monotonic head pointer** (the `site-root` pin pattern in
  `APP-CONVENTION-SEMANTIC-CONTENT-SITE` §2 — a resolve-to-latest pointer, highest-valid-seq wins). Followers
  subscribe (push); offline followers receive via Mode-S + the inbox-relay MX; authorship is signed. Appending a
  post = write a signed entity, advance the signed head pointer. `PROPOSAL-APP-CONVENTION-CHAT` already models the
  append-only content-addressed log this reuses.
- **What the *social* feed adds** — and these are the only real gaps:
  1. **Ordered timeline.** Tree `.list` is hash-ordered; an ISO-date/sequence name-prefix gives a chronological
     *floor* for free (`APP-CONVENTION-SEMANTIC-CONTENT-SITE` §4.2), but a true semantic feed/index (newest-first,
     prev/next, RSS) is a deferred, named application extension — small, Phase-2.
  2. **Fan-out at scale** — the aggregating relay of §5c. Personal feeds don't need it; popular accounts /
     firehoses do.
  3. **Open-follow discovery** — "find and follow a stranger's feed" beyond name/link is the same internet-scale
     discovery of §5a, and it re-uses the folded admission rule (`EXTENSION-DISCOVERY` §1: never silently connect
     to strangers; admit via grant + IDENTIFY).

So the social feed forces exactly the Phase-2 pieces — which is why it is the right driver: it de-risks by
running Phase-1-and-async first, then pulls the overlay in where scale demands it.

---

## §7 The one genuinely open question — who runs the infrastructure

Everything above is either built, designed, or a bounded reconciliation. The **one thing with no design anywhere**
is the **incentive and abuse model for an open overlay**: why a native peer volunteers to run an on-ramp/relay
(bandwidth, the relay-fallback cost for un-punchable pairs), and how an *open* membership/provider overlay resists
**Sybil / eclipse** attacks. Single-peer **DoS hygiene** exists (rate limits, resource ceilings, reflection/
amplification defenses in the connectivity design); an **economic/reputation/anti-Sybil** model does not.

This is a **deployment / product decision, not a spec write**, and it is **deferrable**: the practical path is a
**spectrum, not a fork** — the church runs Tier 1 first (managed); opens it to community-run instances
(federation-style); and lets self-organizing overlay features grow underneath as the incentive and the tech
mature. The same protocol supports the whole spectrum; you slide along it. Keep it in view; don't let it block
Phase 1 or the Phase-2 build.

---

## §8 Where this sits in the workstreams

- Both phases live under **`W-CONNECTIVITY`** (the consumer↔consumer connectivity stack).
- **Phase 1 is the near-term spine already planned** — extend the existing "entity-chat on the async floor" plan
  to "**social feed on the async floor**," and land reachability-facts + signaling/TCP-simopen + the connector
  pool.
- **Phase 2 is a new phase of `W-CONNECTIVITY` (the overlay)** — a fast-follow, off the near-term critical path.
  Its fan-out item is already carried as a deferred follow-on (the Mode-A subscription-fan-out ask in the master
  map).
- **Validation gate (unchanged discipline):** none of it is real until a cross-impl cohort exercises an actual
  punch + an actual end-to-end exchange between two conformant peers.

---

## §9 How much design review remains before an implementation plan

- **Phase 1 — plannable now.** It stands on Active surfaces + in-repo DRAFT proposals; it needs no new central
  state and no overlay. The remaining items are fold-time conventions and build-time research already listed in
  the connectivity proposals (signaling namespace, `fire_at` clock model, symmetric-NAT specifics). Move it to an
  implementation plan.
- **Phase 2 — one bounded reconciliation cycle, not an invention cycle.** The design is largely established; the
  remaining *decisions* are a small, enumerated set (§10) — the aggregating-relay shape, build-vs-ride an external
  DHT for lookup, the routing paradigm, feed-ordering semantics, and the self-replenishing on-ramp mechanism.
  Each is a design choice with a clear home, not an open research problem. None blocks Phase 1.
- **The incentive/anti-Sybil question — product, deferrable.** Not a spec gate; resolved by the managed →
  federated → self-organizing deployment spectrum (§7).

**Net:** Phase 1 is ready to plan. Phase 2 is close — a single reconciliation pass over a bounded decision set —
and can begin in parallel without gating Phase 1.

---

## §10 Open decisions ledger

| Decision | Phase | Kind | Note |
|---|---|---|---|
| Signaling namespace + `fire_at` relative-clock model + symmetric-NAT handling | 1 | fold-time pin / build research | in the connectivity proposals |
| Rendezvous key modes — `pair`/`tag`/`secret`/`lobby`, one mode-tagged derivation + byte-exact string keys | 1 | design — **resolved into `SIGNALING` §2.2** | shared-secret = an agreed string, same mechanism; byte-exact (no normalization), `mode`/`SEP` bytes pinned at fold |
| QR / out-of-band carrier as a secondary signaling path | 1 | operator review | flagged in the signaling proposal |
| Connection service: **open-stateless (default) vs. capability-gated (optional)** modes | 1 | design — scope both | same interface; gate adds one capability check (§3a) |
| Standalone service vs. `system/signaling` entity-handler packaging (hot path non-entity for scale) | 1 | design | entity system for setup; hot path its own thing (§3a) |
| Provider selection + **both-peers-meet-at-the-same-provider** coordination | 1 | design | private/community pools + a well-known fallback (§3a) |
| Self-replenishing on-ramp set (advertise-on-dialable + TTL) | 2 | design | the increment over the static connector pool |
| Internet-scale membership + lookup: **build structured lookup vs. ride an external DHT** | 2 | design fork | fills the DISCOVERY/ROUTE reserved seams |
| Provider-lookup routing paradigm (next-hop vs. hold-the-graph) | 2 | design fork | scale-dependent |
| Aggregating relay (Mode A) — outbound multi-publisher subscribe + merged republish | 2 | design | the only missing delivery primitive; needed at scale |
| Feed ordering: semantic timeline/index above the name-sort floor | 2 | app extension | small, driver = the feed itself |
| Role differentiation by capacity (which peers become heavier relays) | 2 | design | not yet specified |
| Incentive / anti-Sybil for an open overlay | 2+ | product / deployment | deferrable; the managed→federated→self-organizing spectrum |

---

## §11 References (in-repo only — this document is self-contained)

**Folded / Active specs:** `EXTENSION-NETWORK` (§10 dispatch + `held_connection_client`, §10.2 dispatch-fallback,
§5 keepalive), `EXTENSION-REGISTRY` (§7.4 static publisher, §6a.3 sharding), `EXTENSION-RELAY` (§3.2 Mode S, §3.5
inbox-relay MX, §11.1 Mode A/C reserved, §11.1a subscription-engine locality), `EXTENSION-SUBSCRIPTION` (§6
cross-peer, §8 dissemination trees), `EXTENSION-ROUTE` (§4 `resolve_next_hop` seam), `EXTENSION-DISCOVERY` (§1
admission, §8.2/§10 deferred backends), `EXTENSION-CONTENT` / `EXTENSION-TREE` (content-addressing, dedup,
`tree = path → hash`), `EXTENSION-IDENTITY` (author authentication), `APP-CONVENTION-SEMANTIC-CONTENT-SITE` (§2
signed head pointer, §4.2 semantic-feed deferral), `APP-CONVENTION-EMBED`.

**In-repo proposals (DRAFT):** `PROPOSAL-NETWORK-REACHABILITY-FACTS`, `PROPOSAL-CONNECTIVITY-SIGNALING-AND-PUNCH`,
`PROPOSAL-REGISTRY-SERVICE-ADVERTISEMENT`, `PROPOSAL-APP-CONVENTION-CHAT`,
`PROPOSAL-NETWORK-LIVENESS-REACTIVE-BUILDOUT`.

**In-repo sibling explorations:** `ANALYSIS-CONNECTION-FLOWS-AND-MINIMAL-INFRA-FOOTPRINT`,
`ANALYSIS-INFRA-HORIZONTAL-SCALING-AND-STATE-COORDINATION`,
`EXPLORATION-CONNECTIVITY-NAT-WEBRTC-UNIFIED-ARCHITECTURE`,
`EXPLORATION-FULL-STACK-DISCOVERY-INFRA-AND-ENTITY-CHAT`.

**Trackers / guides:** `WORKSTREAMS.md` (W-CONNECTIVITY), `ROADMAP-EXTENSIONS.md` (Stage B),
`GUIDE-NETWORKING-MODEL.md`.

*The one sentence: Phase 1 — a minimal keyed-hash connectivity layer over near-stateless NAT/STUN services — is
ready to plan on already-Active and in-repo-DRAFT surfaces; the Phase-2 overlay that offloads steady-state work
to the peer network is a bounded, mostly-designed fast-follow whose only genuinely-open question (who runs and
secures the open infrastructure) is a deferrable deployment decision, not a spec gate.*
