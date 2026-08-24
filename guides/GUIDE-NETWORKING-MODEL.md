# GUIDE: The networking model — how information travels & how to deploy

> **Status: forward-looking plan / proposal-for-feedback.** This guide describes how
> we see networking working across scales — *not* a claim that every piece is implemented. It
> exists so deployers have a reference architecture (you're not inventing this) and reviewers
> have something concrete to push on. Each section marks **what ships in v1** vs **what's a named
> proposal on the roadmap.** If a piece reads wrong to you, that's the point — tell us.

This is the map. Detail lives in the linked explorations and proposals.

---

## §1 The mental model — four concerns, kept separate

Getting a message from peer **A** to peer **D** in a real network involves four *separate*
questions. The whole design rests on not conflating them:

| # | Question | Answered by | Analogy |
|---|---|---|---|
| 1 | **Who is "D"?** (name → peer-id) | REGISTRY / DISCOVERY | DNS |
| 2 | **To reach D, who's the next hop?** | **ROUTE** (the routing table) | a router's routing table (RIB/FIB) |
| 3 | **Move it one hop.** | **RELAY** (the forwarding plane) | SMTP / IP forwarding |
| 4 | **How do I actually open a connection to that next hop?** (incl. through NAT) | **NETWORK** (transport + reachability) | IP / TCP / NAT traversal |

Those four are all **addressed** — there is a specific D. Information also moves in two
**unaddressed** ways, where there is no single recipient:

| # | Question | Answered by | Analogy |
|---|---|---|---|
| 5 | **Spread this to whoever cares.** | **GOSSIP** (scatter) — *roadmap* | epidemic / anti-entropy |
| 6 | **Many peers serve the same bytes.** | **CONTENT** (blob manifest + chunks) | BitTorrent |

The recurring mistake in P2P systems is to mash these together (one blob that "handles
networking"). We keep them as distinct planes so each does one job and can evolve independently.

> ### The boundary that matters most: RELAY is addressed messaging, not data expansion
>
> **RELAY answers "carry *this* to *D*."** That is peer-to-peer messaging — one sender, one
> destination, an opaque signed envelope, a hop budget. It is the SMTP-shaped problem, and
> **it works without any of concerns 5 and 6**, exactly as SMTP works without gossip or
> BitTorrent. That is why a LAN or VPN deployment is complete today (§3).
>
> **Expanding shared data across a network is a different problem** and has two different homes:
> **GOSSIP** for unaddressed spread of updates and membership, and **CONTENT** for
> many-peers-serve-the-same-bytes. Neither is relay's job, and pushing either into relay is how
> the forwarding plane grows a routing algorithm and a replication policy it should not own.
>
> **The one genuine seam is RELAY Mode A (aggregate)** — a persistent multi-publisher
> intermediary. It is addressed like a relay and multi-source like dissemination, which is why
> §3 groups it with GOSSIP under *federated*, and why its home is an open design question rather
> than a settled one.
>
> **Concern 6 is further along than it looks.** `EXTENSION-CONTENT`'s blob manifest is *"a
> content-addressed chunk list — analogous to a torrent file. Any peer with the chunks can serve
> them,"* with per-chunk hash verification and resumable transfer already specified. Multi-peer
> content distribution does **not** wait on GOSSIP; what it lacks is peer/chunk *discovery*
> (which peers hold which chunks), not the transfer model.

---

## §2 How a message gets from A to D

Walking the planes in order, for A sending to D:

1. **Resolve the name** (if A has a name, not a peer-id) → REGISTRY/DISCOVERY gives A the
   peer-id of D. *(v1 ✓)*
2. **Pick the next hop** → A consults its **route table** (`system/route` entities): "to reach D,
   next hop is N" — or "D is direct." Three sources, in precedence order:
   **a source route the originator set > the route-table lookup > a trivial direct/no-route
   default.** *(source-route + table: v1.x, in review)*
3. **Forward one hop** → **RELAY** moves the opaque, signed envelope to N and decrements a hop
   budget (`ttl_hops`). N repeats from step 2 with *its* table until a hop is the terminal hop
   (next == D), which delivers the original bytes to D **byte-identical to a direct send** — D
   needs no relay extension to receive. *(single-hop v1 ✓; multi-hop v1.x, in review)*
4. **Open the connection** → at each hop, **NETWORK** turns a peer-id into a reachable endpoint
   (`system/peer/transport/{peer_id}/*`) and dials it. If the next hop is behind NAT, this is
   where reachability/traversal lives — see §4. *(direct dial v1 ✓; NAT traversal: roadmap)*

**ROUTE stores the table; RELAY reads it to forward; NETWORK opens the wire; producers
(the peer / DISCOVERY / GOSSIP) fill the table.** Nobody computes routes inside relay; relay
never opens its own routing algorithm; the table is just entities everyone can read and update.

> **"But every peer has its own table — isn't that a distributed route graph you could traverse
> locally?"** Yes — and *how much* of the graph each peer holds is the whole routing design (the
> four classic paradigms: source-routing / distance-vector / link-state / DHT, trading graph-
> knowledge against scale). ROUTE is paradigm-neutral storage; the paradigm is a *producer* choice,
> scale-dependent. Full treatment: the distributed-route-graph and routing-paradigms exploration.

---

## §3 Deployment topologies — what each scale needs

The same primitives compose into very different networks. For each, what it uses and whether v1
covers it:

### Single LAN / small mesh — ✅ ships in v1
A handful of peers on one network (home, office, a cluster). Peers find each other by **mDNS
DISCOVERY**; they're mutually reachable (no NAT between them); messages go direct, or through one
gateway peer via a **default route** (`{match:"*", via: gateway}`). Source routes handle "I know
the path." **No NAT traversal, no gossip, no DHT needed.** Fully covered today.

### VPN / subnet / gateway — ✅ ships in v1 (needs the route table)
"Route through a specific peer to reach the other side of my subnet" / "route over my VPN to
reach the work network." Each relay holds its **own** route table; A's table says
`D → forward via Gateway`, the gateway's table says `D → deliver`. This **hop-by-hop** routing is
the load-bearing case for the route table (source-routing alone doesn't cut it — A shouldn't need
to know the far side's internal topology). Peers inside a tunnel are usually mutually reachable,
so NAT traversal rarely bites here.

### Federated / multi-organization — ⏳ roadmap
Organizations running their own peer clusters, sharing selectively. Adds **Mode A aggregate**
relay (subscribe to N publishers, serve a unified stream) and **GOSSIP** (epidemic dissemination
of updates/routes). Both need cross-peer subscription/choreography the v1 substrate doesn't have
yet. Named proposals; not v1.

### Global P2P / browser- and mobile-heavy — ⏳ roadmap (with an honest v1 floor)
Internet scale: no global topology knowledge, churn, and **NAT everywhere** (home users, phones,
browser tabs). Adds **computed routing** (DHT/Kademlia — the deferred `resolve_next_hop` escape
hatch, since you can't enumerate a static table to millions of peers), **NAT traversal** (§4),
and **GOSSIP** for membership/route dissemination.
**v1 floor that *does* work at this scale:** a NAT'd peer can still **receive** messages via
store-and-forward through a public relay (RELAY Mode S — drop at the relay's inbox, the NAT'd
peer polls on its outbound connection). What's deferred is *direct* peer-to-peer and *live*
relayed circuits between two NAT'd peers (§4).

| Topology | Primitives | v1? |
|---|---|---|
| LAN / small mesh | DISCOVERY (mDNS) + RELAY (single-hop) + ROUTE (default route) + source-route | ✅ |
| VPN / subnet / gateway | + ROUTE (hop-by-hop tables) + RELAY (multi-hop) | ✅ (in review) |
| Federated / multi-org | + RELAY Mode A + GOSSIP | ⏳ named proposals |
| Global P2P / browser+mobile | + computed routing (DHT) + NAT traversal + Mode C + GOSSIP | ⏳ roadmap; async-via-relay floor ✅ |

---

## §4 WAN reachability & NAT traversal (the honest part)

Most internet peers (home, mobile, browser) sit behind NAT and **cannot accept inbound
connections** — so two NAT'd peers can't directly dial each other. The fix ("hole punching") has
both peers punch outbound holes simultaneously, coordinated by a public matchmaker; when that
fails (symmetric/carrier NAT), data falls back to a public relay. This is exactly WebRTC's
`ICE/STUN/TURN` and libp2p's `Identify/AutoNAT/Circuit-Relay/DCUtR` — two proven systems with the
same shape. Full mechanics: the NAT-traversal and WAN-reachability exploration.

**Where it lives (it's a composition, not one extension):**
- **NETWORK** gains reachability awareness — observed-address ("how do I look from outside?"),
  NAT-detection, and ICE-style candidate gathering. *(the bulk of the new work; it's pure
  transport)*
- A **small punch-coordination protocol** uses RELAY as its signaling channel.
- **RELAY** is the **fallback data path** (a public relay = TURN) and the signaling carrier —
  **relay is not where the punch happens.** Mode S (async) is the v1 floor; Mode C (live
  bidirectional circuit, deferred) is the real-time fallback.

**Do you need a server outside the network?** For *introductions*: yes — a lightweight public
matchmaker any two NAT'd peers can reach outbound (any bootstrap/relay/discovery peer; it passes
addresses, not bulk data). For the *data path*: no, if punching succeeds (peers talk directly);
yes only as the fallback relay when it fails. On a single LAN: neither — mDNS introduces and peers
are mutually reachable. A deployable global network therefore always has *some* public on-ramp
peers, even though most traffic ends up direct.

**v1 status (stronger than it first looks):** a NAT'd peer talking to a **public** peer works fully
in v1, *both directions* — the NAT'd peer holds an outbound socket and the public peer pushes down it
(NETWORK §10 `held_connection_client` class; this is how RELAY Mode S delivers). What's roadmap is
only the case where **both** peers are NAT'd and want a **direct** link: direct hole-punch + live
relayed circuits. Async store-and-forward delivery to any NAT'd peer works now (Mode S). Full design:
`proposals/PROPOSAL-NAT-TRAVERSAL-AND-WAN-REACHABILITY.md` (+ Mode C for the live fallback).

---

## §5 Status board — v1 vs roadmap

| Capability | Extension / home | Status |
|---|---|---|
| Name → peer-id | REGISTRY / DISCOVERY | ✅ v1 |
| LAN peer-finding (mDNS) | DISCOVERY | ✅ v1 |
| Single-hop forward; store-and-poll; raw-frame delivery | RELAY (Mode F / Mode S) | ✅ v1 |
| Per-hop capability enforcement | RELAY §5.2 | ✅ v1 |
| Source-routed multi-hop (originator names the path) | RELAY `route` field (§3.1.1 source 1) | ✅ v1.1 |
| Routing table (next-hop store, hop-by-hop) | **ROUTE** (`system/route`) | ✅ v1 (storage plane) |
| Per-hop cap enforcement across a source route | RELAY §5.2 — every hop independently enforces `relay-forward`, so a route cannot conscript relays | ✅ v1.1 |
| Chunked blob transfer, per-chunk verification, resumable | CONTENT (blob manifest) | ✅ v1 |
| **Torrent-like multi-peer content distribution** | CONTENT (blob manifest = chunk list; any peer with chunks can serve) | ✅ transfer model / ⏳ **chunk-peer discovery** |
| Direct transport dial; transport profiles | NETWORK §6.5/§10 | ✅ v1 |
| Async delivery to NAT'd peers | RELAY Mode S (held-outbound) | ✅ v1 |
| **Bidirectional NAT'd↔public** (push down held outbound socket) | NETWORK §10 `held_connection_client` | ✅ v1 |
| NAT traversal / hole punching (NAT'd↔NAT'd direct upgrade) | NETWORK reachability + coordination + RELAY fallback | ⏳ named proposal (`PROPOSAL-NAT-TRAVERSAL-AND-WAN-REACHABILITY`) |
| Live bidirectional relay circuit | RELAY Mode C | ⏳ deferred |
| Computed routing (DHT/Kademlia) | `resolve_next_hop` escape hatch | ⏳ deferred |
| Federated aggregate streams | RELAY Mode A | ⏳ deferred |
| Gossip / epidemic dissemination | GOSSIP | ⏳ stub proposal |
| Declarative replication policy | (was V2.0 `system/replicate`) | ⏳ tracked (composable today via REVISION+SUBSCRIPTION+GROUP) |

✅ = ships / shipped · 🔄 = proposal under cohort review, expected v1.x · ⏳ = named on the roadmap, not built

---

## §6 For deployers & reviewers

- **Deploying a LAN or VPN/gateway network?** Everything you need is v1 (or v1.x in review).
  Use mDNS for discovery, a default route to your gateway, hop-by-hop route tables for subnets.
- **Deploying at global scale today?** You can run public peers and reach NAT'd peers
  asynchronously via relay now; budget for the NAT-traversal + DHT work landing post-release, and
  expect to run a few public bootstrap/relay peers as on-ramps.
- **Reviewing the design?** The places to push hardest: (1) is the relay/route/gossip boundary
  the right decomposition for *your* network? (2) is ROUTE-as-pure-storage (the peer/discovery/
  gossip fill it; relay reads it) the right call, or should routing own more? (3) does the NAT
  composition (NETWORK reachability + relay fallback, punch as a thin coordination protocol) match
  how you've seen it work? We expect to revisit ROUTE's shape especially once real deployments
  tell us what the dominant route-management mechanism actually is.

---

## §7 Refs

**Specs in this corpus:** `specs/extensions/` — `EXTENSION-RELAY`, `EXTENSION-ROUTE`,
`EXTENSION-NETWORK`, `EXTENSION-REGISTRY`, `EXTENSION-DISCOVERY`, `EXTENSION-SIGNALING`,
`EXTENSION-CONTENT`, `EXTENSION-SUBSCRIPTION`.

**Roadmap items whose design records are not in this corpus.** The GOSSIP and NAT-traversal
proposals were authored before the spec corpus was published and did not cross that split — so
those two rows in §5 are **named on the roadmap without a design document you can read here.**
That is a corpus gap, not a design gap; treat the §5 rows as the current statement of intent
until the records are pulled in or re-authored.
