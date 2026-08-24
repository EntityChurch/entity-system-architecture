# ANALYSIS — connection-establishment flows & the minimal-infrastructure footprint

**Status:** Analysis — 2026-07-22. **NOT** a proposal; it answers a systems-resource question that shapes the
connectivity/infra design: *how far can we go with minimal data storage — just a static-CDN registry + a generic,
near-stateless NAT/tunnel — and what does establishing a direct connection actually require of the managed infra
(what it does, what resources, what per-connection state, over what timeframe)?* Companion to
`EXPLORATION-CONNECTIVITY-NAT-WEBRTC-UNIFIED-ARCHITECTURE.md` + `EXPLORATION-FULL-STACK-DISCOVERY-INFRA-AND-ENTITY-CHAT.md`.
Author: arch workspace. Timeframes/success-rates below are **industry-typical (ICE/WebRTC/libp2p), flagged
non-normative** — the exact handshake is "read the RFCs, don't invent" (NAT proposal §9).

**The answer in one line.** You can go **very far** on `[static-CDN registry + one generic near-stateless
STUN/signaling tunnel]` — that combination resolves names and establishes **direct** connections for the majority
of peers (online, cone-NAT) with **zero bulk storage and only transient (seconds-long, ~1 KB) per-connection
state**. Only **two** cases pull in heavier infra, and both are **opt-in fallbacks, not the core path:** (1)
**offline delivery** needs mailbox *storage*; (2) **symmetric-NAT peers (~10–20%)** need a data-relay's
*bandwidth*. The architecture is cheap precisely because it **resolves names and introduces peers** rather than
running **open discovery** (a DHT — the one genuinely stateful, expensive thing — which chat does not need).

---

## Part A — The clarification that unlocks the whole thing: *finding* ≠ *connecting*, and *resolve* ≠ *discover*

"Discovery" is three different acts with wildly different infra costs. Conflating them is what makes the flows
feel murky:

| Act | What it means | Infra needed | Cost |
|-----|---------------|--------------|------|
| **Local discovery** | "who is on my LAN?" | none (mDNS multicast) | **zero** |
| **Name resolution** | "I know *bob.entitychurch.org* (or a QR/link) — where is Bob?" | static-CDN registry | **near-zero** (static bytes, cached) |
| **Open discovery** | "find *any* peer talking about X" | a DHT / gossip mesh | **high** (stateful, replicated) — **deferred, and chat never needs it** |
| **Connection introduction** | "Alice + Bob both know each other's peer-id — help them open a direct link" | STUN reflector + signaling relay | **near-zero** (stateless / transient) |

**The minimal-infra thesis rests on this:** for the apps we care about (chat, file share, collaboration) you
**already know who you want to talk to** — by name, by QR/link, or from your LAN. So you need **resolution +
introduction**, both cheap, and you can **skip open discovery** (the expensive, stateful one) entirely. The
system is minimal by *choosing not to do internet-scale open search.*

---

## Part B — The connection flows, traced (what happens, who does what, over what timeframe)

### Flow 1 — LAN (mDNS): zero infra
Alice `:scan`s → multicast query on the LAN → Bob's peer answers (SRV+TXT with his `peer_id_hint` + endpoint).
Grant-prompt + IDENTIFY admit. Direct TCP/WS connect. **Managed infra involved: none.** Timeframe: ~ms–1 s.

### Flow 2 — one peer is publicly reachable (a server, or NAT'd-but-holding-a-socket): registry only
1. Alice `resolve("bob.entitychurch.org")` → **static-CDN GET** → signed binding `(bob_peer_id, transports, MX,
   ttl)`, verified against the pinned registry identity. *(1 RTT, TTL-cached; infra = static bytes.)*
2. Bob advertises a listener (`wss://…`) **or** is NAT'd but already holds an outbound socket to a relay
   (`held_connection_client`). Alice dials it (or the relay pushes down Bob's held socket). Direct.
**Managed infra: the static CDN only** (and, for the held-socket case, a relay Bob keeps a socket to — but that's
Bob's choice, one socket, no per-Alice state). Timeframe: 1–2 RTT after resolve.

### Flow 3 — both peers behind (cone) NAT: the punch — the interesting case
1. **Resolve** (as Flow 2, static CDN).
2. **Reflect (STUN):** Alice and Bob each ask a **reflector** "what's my public IP:port?" — the reflector echoes
   the packet's *observed source*. **Stateless**: it holds nothing after the reply. *(1 RTT each; tens of bytes.)*
3. **Signal (rendezvous):** Alice and Bob exchange their candidate addresses + a `punch-sync {fire_at}` via a
   **signaling relay both can reach outbound** (the "matchmaker passes addresses, not bulk data"). The relay
   holds a **transient signaling session** — the two peers' addresses + the pending offer — **for the handshake
   only (seconds; ~30 s ICE consent-freshness timeout), then discards it.** *(a few messages, ~1–2 KB total; no
   bulk data ever crosses it.)*
4. **Punch (peer-to-peer, no infra):** at `fire_at` both peers simultaneously send to each other's public
   addresses; the NAT mappings open; the direct connection establishes. **Managed infra: none** — this is the two
   peers. *(sub-second, a few RTT.)*
5. **Direct connection live:** all data flows peer-to-peer, transport-encrypted (DTLS/TLS). **Managed infra for
   the data path: none.** Ongoing: the peers send periodic keepalives to keep the punched NAT mapping open
   (~15–30 s) — *peer-to-peer, not infra.*

### Flow 4 — punch fails (symmetric NAT, ~10–20%): the data-relay fallback (the one heavy case)
Steps 1–3 as Flow 3, then the punch fails (symmetric NAT re-randomizes ports). Fall back to a **data-relay**
(TURN / RELAY Mode C): the relay holds a **circuit for the connection's lifetime** and **relays every byte**.
**This is the only role that costs bandwidth + long-lived per-connection state.** It is a *fallback*, minimized by
maximizing punch success.

### Flow 5 — recipient is offline: store-and-forward (the other heavy case — storage)
Alice's message can't reach Bob live → `dispatch_fallback` → **RELAY Mode S** stores the signed envelope at
Bob's **inbox-relay** (his MX, resolved via the registry). The relay **holds the message until Bob polls or the
TTL expires** — **mailbox storage.** Bob drains on reconnect. This is the async floor; it costs *storage*, and it
is what you spend if you want *offline* messaging.

---

## Part C — The infra resource & per-connection-state table (the core answer)

| Infra role | Per-connection state | How long held | Data volume | Compute | Scales like |
|---|---|---|---|---|---|
| **Registry (static CDN)** | **none** | — (TTL-cached at the edge) | signed bindings (static files) | **zero** (static serve) | a CDN (∞) |
| **STUN reflector** | **none** | per-request (ms) | tens of bytes | trivial (echo source addr) | ∞ (stateless) |
| **Signaling / rendezvous** | 2 peers' addrs + pending offer | **handshake only (seconds; ~30 s max)** | ~1–2 KB, **no bulk data** | trivial (forward messages) | very high (tiny, short-lived sessions) |
| **Punch** | — (peers do it) | — | — | **zero** | — |
| **Data-relay (TURN / Mode C)** | full circuit | **connection lifetime** | **all traffic (bandwidth)** | proportional to traffic | poorly (bandwidth-bound) — *fallback only* |
| **Inbox-relay (Mode S, store-and-forward)** | the mailbox | until delivered / TTL | **the queued messages** | storage I/O | storage-bound — *offline only* |

**Read this table as the minimalism map:** the top four rows (**registry, reflector, signaling, punch**) are the
*core path* and are **collectively near-stateless and storage-free** — a static CDN + a small generic tunnel. The
bottom two rows (**data-relay, inbox-relay**) are the *only* heavy infra, each tied to *one specific capability*
you can choose to offer or not.

---

## Part D — "How far with minimal storage?" — the explicit envelope

**With just `[static-CDN registry + one generic STUN/signaling tunnel]` you get, at ~zero storage & ~zero
per-connection state:**
- name resolution for every named peer (static, cached, infinite-scale);
- **direct, transport-encrypted P2P connections** for the **majority** of peer pairs (both online, cone NAT —
  industry-typical **~70–90%** punch success);
- and therefore **live entity-chat (and any P2P app) between online peers** — discover/resolve → punch → message
  → render — **without the infra storing any conversation data or holding meaningful per-connection state.**

**What you give up without heavier infra, and what each costs to add:**

| Capability you'd add | Infra it requires | Cost shape | Can you avoid it? |
|---|---|---|---|
| **Offline / async delivery** ("message Bob while he's asleep") | inbox-relay (Mode S) **storage** | storage ∝ queued messages × TTL | Yes — require both peers online (pure-P2P chat needs *zero* storage) |
| **Symmetric-NAT peers** (~10–20%) | data-relay (TURN/Mode C) **bandwidth** | bandwidth ∝ relayed traffic | Only by dropping those peers; or accept the relay for that slice |
| **Open wide-area discovery** ("find peers about X") | DHT / gossip **stateful mesh** | replicated state | Yes — **not needed**; resolve-by-name + QR/link + LAN replace it |

**The punchline:** the *generic near-stateless tunnel* the operator envisioned is not a compromise — it **is** the
intended core, and it is generic in the strongest sense: the reflector and signaling relay are **content-agnostic
and app-agnostic** (they move addresses and opaque offers, never app data), so **one deployment of them serves
chat, file-transfer, collaboration — everything**, and can even be **encrypted-opaque** (the signaling payload can
itself be an ENCRYPTION-wrapped blob the tunnel can't read). Storage and relay-bandwidth are **à-la-carte
upgrades** bolted to the two specific features that need them.

---

## Part E — Timeframes & lifetimes (what data lives how long)

| Datum | Lifetime | Where |
|---|---|---|
| Registry binding | **long** (TTL: minutes–hours), edge-cached, immutable-until-rotated | static CDN |
| Reflected address | **momentary** (the reply) — never stored | reflector holds nothing |
| Signaling session (addrs + offer) | **seconds** (handshake; ~30 s hard timeout) then discarded | rendezvous relay |
| Candidate addresses | **session-ephemeral** — explicitly *not* durable `system/peer/transport` profiles (they'd go stale instantly) | in-message only |
| Punched NAT mapping | **connection lifetime**, kept alive by peer↔peer keepalive (~15–30 s cadence) | the two NATs, not infra |
| Direct connection | connection lifetime | peer↔peer, no infra |
| Relayed circuit (fallback) | connection lifetime | data-relay (the heavy case) |
| Mailbox (offline) | until delivered / TTL | inbox-relay storage (the heavy case) |

**The one durable, operator-run datum in the core path is the registry binding — and it is static, immutable, and
CDN-cacheable.** Everything else in a direct-connection flow is either momentary (reflect), seconds-long
(signal), or peer-to-peer (punch, keepalive, data). That is the whole reason the managed infra can be a **static
reef + a generic stateless tunnel.**

---

## Part F — Implications for the design / next steps

1. **This validates the minimal-infra target** and sharpens the connectivity build order: the **reachability-facts
   wedge** (reflection + AutoNAT) + a **generic signaling tunnel** are *the* core-path infra to build — small,
   stateless, generic, app-agnostic. The data-relay (Mode C) and the inbox-relay (Mode S) are the two
   capability-specific heavy pieces, sequenced behind, offered per-deployment.
2. **The registry service-advertisement (A+MX+SRV)** should advertise *which reflector / signaling tunnel* a
   deployment offers — so a fresh peer learns the whole cheap core path from one static resolve. (This is the
   `PROPOSAL-REGISTRY-SERVICE-ADVERTISEMENT` unit; this analysis says the SRV set = `{reflector, signaling,
   [optional] data-relay, [optional] inbox-relay}`.)
3. **entity-chat's floor is now precisely placed:** two *online* peers behind cone NATs chat with **zero infra
   storage** (registry-resolve → punch → message). "Offline messages" is the explicit, isolable upgrade that
   turns on inbox-relay storage — a per-deployment (even per-conversation) choice, not a baseline cost.
4. **Open item (unchanged, flagged):** the exact punch handshake (connectivity-check / consent-freshness, RFC
   8445 §7 / RFC 7675) and the TCP-simopen-vs-QUIC/Iroh substrate (the G1 buy-vs-build) — this analysis does not
   resolve them; it bounds the *infra economics* around whichever substrate is chosen (the state/resource shape
   is substrate-independent — reflect/signal/punch/relay-fallback are the same silhouette for tcp, QUIC, or WebRTC).

## References

- `EXPLORATION-CONNECTIVITY-NAT-WEBRTC-UNIFIED-ARCHITECTURE.md` (the four separable services; the substrate forks),
  `EXPLORATION-FULL-STACK-DISCOVERY-INFRA-AND-ENTITY-CHAT.md` (the five-layer pipeline; managed-infra pattern).
- Transport encryption (the clarification): `EXTENSION-ENCRYPTION.md` §"Why not just transport encryption?"
  (L54) + §"Orthogonal layers" (L916–920) — transport §4.5 frame encryption (TLS/WSS/DTLS) is hop-by-hop and
  orthogonal to entity-level E2E; both compose.
- Spec anchors: `EXTENSION-REGISTRY.md` §7.4 (coral-reef static registry), `EXTENSION-RELAY.md` §3.5 (MX), §3.4/§11.1
  (Mode C data-relay), `EXTENSION-NETWORK.md` §10 (`held_connection_client`), `EXTENSION-DISCOVERY.md` §3 (mDNS);
  archived `PROPOSAL-NAT-TRAVERSAL-AND-WAN-REACHABILITY.md` §3–§4 (reflection/AutoNAT/candidates/DCUtR punch),
  §9 (the "read the RFCs" open items).
