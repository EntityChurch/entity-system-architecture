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

## §4a Reaching a peer when all you hold is a peer-id

> **Read this section if you are building anything that receives a `peer_id` from somewhere other
> than a name lookup** — a group roster, a capability grant, a `runtime-peer-set`, an inbox-relay
> declaration, a QR code, a link someone pasted. **It is the case the rest of this guide assumes
> away**, and it is where every reachability question in this stack currently converges.

### §4a.1 There are exactly three ways to reach a peer, and they are rungs, not alternatives

`EXTENSION-NETWORK` §10's dispatch ladder tries them in a fixed order, and **Amendment 14 makes the
ordering a MUST** — live first, store-and-forward last. Getting the rung wrong is not a
preference-level mistake: a peer that is reachable *right now* falling to the mailbox is the exact
defect Amendment 14 was written to eliminate.

| Rung | How | The intermediary | Landed in |
|---|---|---|---|
| **1 · Direct** | dial an address the peer published | none | NETWORK §6.5 transport profiles · §10 steps 1–3 |
| **2 · Mediated (live)** | meet the peer at a rendezvous and negotiate a channel | a **signaling pool** | SIGNALING §3–§6.5 · NETWORK §10.3 `establish_live` · §6.5.2d |
| **3 · Store-and-forward** | leave a signed envelope with the peer's relay; it polls | an **inbox-relay** | RELAY §3.5 + Mode S · INBOX · NETWORK §10.2 |

**Listening is not what separates these.** A peer with no listening socket is reachable on rungs 2
and 3, and NETWORK says so directly: §6.5.2c's `websocket` profile *"pushes to a **non-listening**
target down that target's own outbound socket,"* and §6.5.2d is a durable profile that carries **no
`endpoint` at all** — *"the capability + negotiation parameters, not an address."* **A browser peer is
an ordinary peer that lives on rungs 2 and 3**, not a degenerate one. Treat "has no listener" as a
statement about which rung, never about whether.

**And a static, CDN-hosted peer is not a special category either.** RELAY §1 exposes four modes from
one `system/relay` handler, and *"static-CDN-hosted peers fall here"* — Mode S. A relay peer is
*"just a peer running `system/relay`. No special infrastructure role; same substrate as everything
else."*

### §4a.2 The thing all three have in common, and it is the gap

**Rungs 2 and 3 both require you to learn an intermediary before you can use them — and rung 1
requires you to learn an address. From a `peer_id` alone, none of the three has an answer.**

| Rung | What you must learn first | Where it comes from today | From a `peer_id` alone? |
|---|---|---|---|
| 1 · Direct | a transport profile | `binding.transports` (by **name**), or `system/peer/transport/*` from **prior contact** | ❌ |
| 2 · Mediated | the signaling pool — §3.4 is a `MUST — same provider`, and a mismatch is a **silent never-meet** | REGISTRY §3b.1 `services`, which rides a **`ResolutionResult`** | ❌ **name-keyed** |
| 3 · Store-and-forward | the inbox-relay declaration | REGISTRY (*"the A+MX-in-one-zone pattern"*), else a cached copy | ❌ |

**RELAY §3.5 states its own instance plainly**, and it is the most honest sentence in the stack:

> *"A peer with no registry presence and no prior relationship is **reachable by a stranger only via a
> cached copy** — honest parity with a mail server that has no DNS entry."*

**The parity is the part to push on.** A mail server with no DNS entry has **no identity you can hold
either** — there is nothing to look up. Here you hold a `peer_id`: a complete cryptographic identity
that self-certifies every answer it could possibly be given. **The lookup is missing, not the
identity**, and that asymmetry is why this is a gap rather than a design choice.

**So: today, `peer_id` → reachable requires either a name or a prior relationship.** That is the
honest v1 floor. It is not a bug in any one extension — REGISTRY, SIGNALING and RELAY each solve their
own problem correctly, and each solves it **keyed on a name**, because a name is what a directory
indexes.

### §4a.3 What closes it — one record, tracked as R-22 / R-25

**`PROPOSAL-PEER-TRANSPORT-SET` (DRAFT, not ruled).** A `system/peer/transport-set` entity, keyed on
`peer_id`, **signed by the peer**, carrying its transport profiles and — per R-25 — the intermediary
locators for rungs 2 and 3.

**Why one record rather than three:** the property that makes it work is not the schema, it is the
key plus the signature. **Anyone may serve it and serving confers nothing**, because the signature is
what makes it valid — so the lookup can be the cheapest mechanism available ("ask whoever you are
already talking to") rather than a trusted index. That is the same posture as `published-root`, and
the transport-set uses the identical signature carriage on purpose.

**One caveat that is load-bearing:** the locators say **where to find an intermediary**, never what
that intermediary will do for you. An intermediary carries bytes and gets no authority — envelopes
stay signed end-to-end and capability chains pass through unchanged (RELAY §1). A locator that grows
into a credential or a policy has left this record's job; the credential question is
`EXTENSION-REGISTRY` §3b.0a's and stays there.

**Status:** rung 1 is `R-22`, rungs 2–3 are `R-25`, both **v2**, both on `docs/COHORT-OPEN-ITEMS.md`.
**Neither is a `EXTENSION-REGISTRY` v1 blocker** — v1's release line is `COHORT-OPEN-ITEMS` §0a and
none of this touches it.

### §4a.4 What to build against today

- **If you can get a name, use it.** Resolve-by-name returns the binding *and* `services` *and* the
  inbox-relay declaration in one answer. Every rung works. This is the supported path.
- **If you hold only a `peer_id`, you need a prior relationship** — a cached transport profile, a
  session record, an out-of-band rendezvous key (SIGNALING §3's `pair`/`tag`/`secret`/`lobby`).
  **Design for that precondition explicitly rather than discovering it**; the failure mode is a silent
  never-meet, not an error.
- **Do not model a live peer on the inbox-relay.** It is the **offline** mailbox (rung 3). Reaching
  for it while the peer is live is the Amendment 14 defect.
- **If you publish presence rather than an address** (rung 2), **you owe a withdraw path.** Presence
  is a substitute for a listening socket — an assertion to a third party — and an unwithdrawable one
  is a standing invitation to a peer the user has already dismissed. *(Found live in
  `entity-browser-rust` and fixed: a teardown swept the registry row, the route and the dial marker,
  and left the reach intent probing forever.)*

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
| **Name → reachable (all three rungs)** | REGISTRY binding + `services` + inbox-relay | ✅ v1 |
| **`peer_id` → reachable, no name, no prior contact** | **nothing — §4a** | ⏳ **R-22 / R-25** (`PROPOSAL-PEER-TRANSPORT-SET`, DRAFT) |
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
