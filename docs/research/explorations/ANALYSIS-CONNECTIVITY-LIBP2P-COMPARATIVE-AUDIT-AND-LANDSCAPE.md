# ANALYSIS — connectivity: comparative audit vs libp2p + 2026 landscape survey

**Status:** Analysis — 2026-08-01. **NOT** a proposal. Answers an operator steer: *do a serious
comparative audit between libp2p and ourselves, survey the landscape, and confirm we're still on
track (not accidentally re-implementing libp2p).* Companion to
`EXPLORATION-CONNECTIVITY-NAT-WEBRTC-UNIFIED-ARCHITECTURE.md` (the design),
`ANALYSIS-CONNECTION-FLOWS-AND-MINIMAL-INFRA-FOOTPRINT.md` (the infra economics), and
`GUIDE-NETWORKING-MODEL.md` §4 (the existing ours↔ICE↔libp2p role table). Author: arch workspace.

**Provenance.** Internal decomposition read from the folded spec surface (`EXTENSION-SIGNALING.md`
v1.0, `EXTENSION-NETWORK.md` §6.7/§10.3, `EXTENSION-RELAY.md`, the three connectivity proposals,
`SDK-OPERATIONS` §11.6.9). External landscape verified against live 2026 sources (libp2p.io,
iroh.computer, W3C/RFC-Editor, project repos); licensing / for-profit / hosted-relay facts checked
explicitly. Items that could not be verified are flagged **[unverified]**.

---

## The verdict in one line

**We are not re-implementing libp2p.** At the *role* level we are deliberately building the same
proven shapes libp2p and WebRTC/ICE build (that's correct — "decomposition we follow, not depend
on"), but with three **deliberate reductions/substitutions** — no DHT, entity-native
security/framing instead of Noise+yamux+multistream, single-server-role signaling — and one
**deliberate deferral** (our-QUIC). The one place the "re-implementing libp2p" charge would become
fair is the our-QUIC + full-symmetric-NAT roadmap, and that is exactly what's deferred/refused. **On
track — with the scope line being: stop at cone-NAT punch + relay floor; treat our-QUIC as the single
genuine buy-vs-build reconsideration point.**

## Three corrections to our own record (surfaced by this audit — do not skip)

The exercise's value is where it contradicts our prior assumptions. Three do:

1. **py-libp2p is NOT abandoned.** It is actively maintained — v0.6.0 (Feb 2026), v0.7.0 (~mid-2026),
   three named maintainers, commits into 2026, self-labeled "steadily progressing toward production
   readiness" (still pre-production). **This corrects a claim made earlier this cycle that
   "py-libp2p is effectively abandoned."** That claim was wrong and must not be used as a reason for
   build-our-own. (The real three-impl argument survives — see §5 — but on different grounds.)

2. **Iroh is NOT "usable free only at small scale."** iroh itself is dual **MIT/Apache-2.0** OSS, and
   its relay server is **also OSS with published binaries** — **self-hosting is first-class and free
   at any scale.** The small-scale limit applies *only* to n0's shared *public* relays (dev/test
   default), not to iroh. So the stated refusal reason ("for-profit, usable free only at small
   scale") is **half right and half wrong**: n0's commercial hosted-relay motion is real and
   confirmed (paid "Iroh Services", $19/mo + $0.27/relay-hr), but adopting iroh does **not** force a
   paid dependency. If we keep refusing Iroh, the honest grounds are **soft lock-in** (hardcoded
   default hosted relays + a *non-standard forked QUIC* (`noq`, n0's Quinn hard fork) + single-org
   stewardship + commercial motion), **not** "paywalled beyond small scale." *"VC-backed" is
   **[unverified]*** (a ~$4.29M Form D traces to one self-flagged-unconfirmed source; not on
   EDGAR/Crunchbase). Say "commercial founder-backed startup monetizing hosted relays," not
   "VC-backed."

3. **libp2p is no longer Protocol-Labs-controlled, and its paid steward exited go/js.** PL → spun out
   the IPFS/libp2p foundations (2023) → engineering became **Interplanetary Shipyard** → **Shipyard
   announced end of active go-libp2p + js-libp2p maintenance on 2025-09-30**, handing to community
   maintainers. Releases have continued (go-libp2p v0.47, Jan 2026), so community maintenance is
   functioning, but the **named successor steward is [unverified]**. Net: libp2p is *not* a
   for-profit-locked dependency (good, permissive, community-run), but it now carries its own
   **maintenance-continuity risk** — which cuts *for* studying-not-depending, and makes libp2p a
   safer *reference* than Iroh (community vs single-org) but a riskier *dependency* than it was.

**None of these changes the recommendation. All three change the *reasons* we give for it.**

---

## §1 The role-level comparison — ours ↔ libp2p ↔ WebRTC/ICE

Every connectivity role maps cleanly onto both reference systems. This is the point: we are not
inventing mechanisms, we are re-decomposing proven ones onto entity-native primitives.

| Role | Ours (pin) | libp2p | WebRTC/ICE | Note |
|---|---|---|---|---|
| **Name → peer-id** | REGISTRY (static-CDN) + DISCOVERY (mDNS) | Kademlia **DHT** + rendezvous + mDNS | app-defined | **We deliberately skip the DHT** (§4) |
| **Reflection (public mapping)** | `network:observe-address` — NETWORK §6.7.1 | Identify observed-addr | **STUN** (RFC 8489, obs. 5389) | ours cites RFC 5389; 8489 supersedes (§6 hygiene) |
| **Reachability / NAT-type** | `network:check-reachability` dial-back-observed-only + multi-reflector — §6.7.1/§6.7.2 | **AutoNAT v1/v2** | ICE (implicit) | dial-back-to-observed-only = the AutoNAT amplification rule |
| **Candidate gathering** | `host`/`srflx`/`relay`, session-ephemeral — §6.7.3 | multiaddrs | **ICE candidates** (RFC 8445) | ours: candidates never durable profiles |
| **Signaling carrier** | rendezvous mailbox, pluggable carrier — `EXTENSION-SIGNALING` §2.2/§4 | **Circuit-Relay-v2** as coordination substrate (no dedicated signaling server) | out-of-band, **app-defined** SDP | we chose an *explicit* carrier; ICE leaves it undefined |
| **Rendezvous key** | mode-tagged SHA-256, byte-exact, 4 modes — §3 | rendezvous namespaces (loose) | — | our "silent-never-meet" guard is stricter-pinned |
| **Punch coordination** | connect-request/response/punch-sync — §6.1 | **DCUtR** | ICE connectivity checks | 1:1 shape match |
| **Punch mechanism** | **TCP simultaneous-open** (now); QUIC hole-punch (deferred) | QUIC (pref.) / TCP simopen | ICE checks (browser) | see §4 substrate |
| **Firing sync** | `fire_at` = delay-from-receipt, `d≥rtt/2` — §7.2 | DCUtR RTT sync | — | clock-domain pin |
| **Controlling/controlled role** | initiator = signaling-client — §7.4.1 | (DCUtR initiator) | **ICE controlling/controlled** (RFC 8445) | 1:1 |
| **Consent / conn-check** | deferred to RFC 8445 §7 + RFC 7675 (not re-derived) | ICE checks | RFC 8445/7675 | correctly *not invented* |
| **Transport security** | entity-layer: capability handshake + ENCRYPTION | **Noise / TLS 1.3** | DTLS-SRTP | we substitute our own |
| **Framing / mux** | entity dispatch layer (already frames) | multistream-select + **yamux** (mplex deprecated) | SCTP data channel | **we need neither** — dispatch is the frame |
| **Relay fallback (data)** | RELAY **Mode S** (v1) / **Mode C** (deferred) | **Circuit-Relay-v2** | **TURN** (RFC 8656) | Mode S async floor ships; Mode C = live circuit deferred |
| **Peer identity** | entity peer-id (Ed25519, content-addressed) | PeerId (multihash pubkey) | none (DTLS fingerprint) | ours is the same idea, entity-native |
| **Browser leg** | `webrtc` profile + SDP/ICE schema — DRAFT | WebRTC transport | **is** WebRTC | ours DRAFT, unbuilt (§H) |

**Reading:** the left column is a complete connectivity stack at the role level. Nothing is missing
vs the reference systems *except by choice* (DHT) or *by deferral* (our-QUIC, Mode-C, browser).

## §2 What we substitute (and why it's less code than libp2p, not more)

Three libp2p subsystems have **no analogue in our stack because our substrate already provides the
function** — this is where "are we re-implementing libp2p" is most clearly *no*:

- **multistream-select + yamux muxer → the entity dispatch layer.** libp2p must negotiate protocols
  and multiplex streams over one connection. Our connection already carries capability-gated EXECUTE
  dispatch with its own framing; there is nothing to add. (libp2p itself deprecated mplex; muxing is
  real ongoing surface there.)
- **Noise/TLS handshake → the capability handshake + ENCRYPTION extension.** Peer auth and E2E are
  entity-native and already landed; a punched socket runs the *same* handshake a dialed socket does
  (`EXTENSION-SIGNALING` §7.4).
- **Kademlia DHT → resolve-by-name (static CDN) + QR/link + mDNS.** See §4.

Net: our "new" connectivity code is **the coordination dance + reflection facts + a keyed mailbox** —
DCUtR's *idea* re-expressed, not DCUtR-the-subsystem, and none of libp2p's transport-security-mux
tower.

## §3 Landscape positioning — who actually punches, who's a company, what's the license

From the verified survey. The distinction that matters: **relay-only ≠ NAT-traversal.**

| System | Company or protocol | Hole-punches? | License | Note for us |
|---|---|---|---|---|
| **libp2p** | protocol (community-maintained; Shipyard exited go/js 2025-09) | **yes** (DCUtR) | MIT (+Apache; go Apache **[unverified]**) | our primary reference; safe to mirror, risky to depend |
| **WebRTC/ICE** | **standard** (W3C Rec 2025-03 + IETF RFCs) | **yes** (ICE) | royalty-free, no vendor | the browser leg *is* this; mirror freely |
| **Iroh / n0** | **for-profit company** (commercial hosted relays; VC **[unverified]**) | yes | MIT/Apache (incl. relay) | refuse as *dependency* (soft lock-in), study as prior art — free-at-any-scale correction, §corrections |
| **Tailscale** | **for-profit** ($160M Series C **[secondary]**) | **yes** (DERP + punch) | client BSD-3; Headscale OSS control | prior art for relay+punch coordination |
| **Nebula / Defined Networking** | tool (MIT) + company | **yes** (lighthouse) | MIT | cert/PKI overlay; prior art |
| **Hyperswarm / Holepunch** | company (Tether-backed **[unconfirmed]**) | **yes** (HyperDHT/UDX) | MIT / Apache | the one adjacent that punches over a DHT |
| **Nostr** | protocol | **no** (dumb relays) | — | relay/store-forward only; not traversal |
| **AT Protocol / Bluesky** | protocol (for-profit PBC steward) | **no** (federation) | MIT/Apache | PDS→relay→appview; not P2P |
| **WireGuard** | tool (Donenfeld) | **no** (tunnel only) | GPLv2 / MIT | the crypto-tunnel layer, no traversal of its own |

**Takeaways:** (a) mirroring **WebRTC/ICE + libp2p DCUtR** is mirroring a royalty-free standard and a
community protocol — the most durable references available, materially safer than Iroh (single-org,
forked non-standard QUIC). (b) Every serious P2P stack that punches uses the **same reflect → signal →
simultaneous-open → relay-fallback** silhouette we do — strong evidence our decomposition is right,
not idiosyncratic. (c) The systems that *don't* punch (Nostr, ATProto) are exactly the ones that
accept relay-only — which is precisely our async Mode-S floor, already shipped.

## §4 The three deliberate divergences (the anti-scope-creep evidence)

1. **No DHT / no open-discovery.** libp2p's single most expensive, stateful subsystem (Kademlia) we
   omit entirely. The minimal-infra analysis is explicit: for chat/file-share/collab you *already
   know who you want* (name, QR, LAN), so resolution + introduction (both near-stateless) suffice and
   open discovery is unneeded. This is a **scope reduction vs libp2p**, the opposite of creep, and the
   reason our core infra is "static CDN + stateless tunnel" not "a replicated mesh."
2. **Single-server-role signaling** (client is the conformance surface; server single-impl by design,
   `EXTENSION-SIGNALING` §2.1) vs libp2p folding signaling into Circuit-Relay-v2. Ours is a
   dumb keyed mailbox — smaller, app-agnostic, self-hostable.
3. **Punched connection = ordinary transport** (NETWORK §10.3). No new transport type for native
   punch; only *establishment* (simopen) and *keepalive* are new. libp2p carries a full transport
   abstraction tower; we reuse the one we have.

## §5 The build-our-own case — restated on correct grounds

With py-libp2p-is-dead removed, the case still holds, on three surviving grounds (strongest first):

1. **Entity-native model mismatch (decisive).** Everything here is capability-gated EXECUTE dispatch
   over a content-addressed tree with local-view authority. libp2p brings its own identity, muxing,
   security, addressing, and discovery — adopting it means bolting a foreign networking model
   alongside ours, not integrating one. The connectivity we need is *thin* precisely because our
   substrate already does auth/framing/identity; libp2p would re-supply all of it.
2. **The three-independent-impls thesis.** The conformance model's value is three *independent*
   ground-up impls (the cohort-consistent-vs-independent-convergence distinction the project
   polices). Adopting libp2p collapses "three independent implementations" into "three bindings of
   one upstream." This holds **regardless of py-libp2p's health** — it's about independence, not
   maturity. (py-libp2p being pre-production is a *practical* wrinkle, not the argument.)
3. **Supply-chain / "the system and its infra are the point."** AGENTS-STANDARD's minimal-deps floor;
   the project's thesis is a from-scratch protocol. Plus libp2p's own **maintenance-continuity risk**
   (Shipyard exit) now cuts against depending on it.

What is **weakened** and should be dropped from our talking points: "py-libp2p is abandoned" (false),
and "Iroh is paywalled beyond small scale" (false — self-host is free at any scale).

## §6 Honest risk register — where we actually stand

| Risk | Severity | Reality |
|---|---|---|
| **The punch has never crossed a real NAT.** Only the *meet* half (rendezvous, key, signed coordination) is cross-impl green; the entire punch half (choreography, fire_at, srflx-from-punch-socket, conn-check) is **folded-but-never-run** — every punch to date is loopback (`EXTENSION-SIGNALING` §11.5). | **HIGH — the load-bearing unknown** | This, not spec completeness, is the real state. Blocked on live infra (two boxes behind real NATs), not design. No claim of working traversal until a cross-impl run over real NAT. |
| **our-QUIC is the true "re-implementing libp2p" boundary.** Building our own QUIC transport + full symmetric-NAT traversal + dual-hole sequencing *is* where the charge becomes fair. | **MED — trajectory risk** | Correctly deferred/refused today. This is the single point to *genuinely* reconsider buy-vs-build (see §7). |
| **Single-impl server + never-run punch** = two unproven surfaces on one gate | MED | Client is conformance surface by design, but selection/interop (§2.2, §3.1.1) can't be proven until go/py clients + a cross-impl gate exist. |
| **srflx-gatherer + dial-back unbuilt everywhere** (only Go observe-address responder exists) | MED | The cheapest reachability wedge is still unbuilt 3-way; it gates the punch's `srflx`. |
| **Browser leg (WebRTC) is DRAFT, nothing built** | LOW (scoped) | Relay is the pinned floor for native↔browser until it folds (§7.3.1) — a correct floor, not a gap. |
| **GUIDE-NETWORKING-MODEL points at the archived NAT proposal**, predates the fold | LOW (hygiene) | Re-point §7 refs at the current 3-unit split (SIGNALING v1.0 / NETWORK §6.7 / WEBRTC-TRANSPORT). |
| STUN cited as RFC 5389; current is **RFC 8489** (obsoletes 5389) | trivial | Binding semantics stable; note 8489 at next NETWORK §6.7 touch. |

## §7 Where we're going — the recommendation

**Confirmed on track.** Concretely:

1. **Finish the cone-NAT floor, then stop.** Do rung-3 (wire PunchIo/Carrier/LiveEstablish into a
   real in-process punch), then the reachability wedge (srflx gatherer + dial-back, 3-way), then a
   **real cross-NAT run on two boxes** — that last is the only thing that converts "folded" to
   "works." Target = cone-NAT punch (~70–90% of pairs) + **relay Mode-S/Mode-C fallback** for the
   rest. Do **not** chase symmetric-NAT direct punch (correctly refused — relay is the answer).
2. **Treat our-QUIC as the one real buy-vs-build reconsideration.** It's the large undertaking and the
   point where "our own stack" gets heavy. When a consumer forces it, *re-run this audit for that one
   transport*: options are (a) build-our-own on our substrate (current plan; study Iroh's noq/DCUtR
   as prior art, don't depend), (b) a thin QUIC lib (quinn/quiche) *as a transport dependency only*,
   under our handshake/dispatch — a far narrower dependency than libp2p-whole. Defensible either way;
   decide with a consumer in hand, not speculatively.
3. **Keep the references honest.** libp2p (DCUtR/AutoNAT/Circuit-Relay) + WebRTC/ICE RFCs are the
   study set; Iroh is prior-art-only for QUIC+traversal (refuse as dependency on soft-lock-in grounds,
   not the false "paywalled" one). Update our proposal prose that carries the corrected reasons.

**Bottom line for the operator's question:** you have not drifted into re-implementing libp2p. You've
built a *thinner* thing (no DHT, no transport-security-mux tower) on the same proven punch silhouette,
with the heavy part (our-QUIC) correctly fenced off. The genuine risk is not scope — it's that the
punch is still unproven against a real NAT. Ship the cone-NAT + relay floor, prove it on two boxes,
and revisit buy-vs-build only at the our-QUIC line.

## References
- Internal: `EXTENSION-SIGNALING.md` v1.0 (§2–§11), `EXTENSION-NETWORK.md` §6.5/§6.7/§10.2/§10.3,
  `EXTENSION-RELAY.md` §3/§10/§11, `PROPOSAL-CONNECTIVITY-SIGNALING-AND-PUNCH.md`,
  `PROPOSAL-CONNECTION-NODE.md`, `PROPOSAL-NETWORK-REACHABILITY-FACTS.md`,
  `PROPOSAL-EXTENSION-WEBRTC-TRANSPORT.md`, `SDK-OPERATIONS` §11.6.9,
  `EXPLORATION-CONNECTIVITY-NAT-WEBRTC-UNIFIED-ARCHITECTURE.md`,
  `ANALYSIS-CONNECTION-FLOWS-AND-MINIMAL-INFRA-FOOTPRINT.md`, `GUIDE-NETWORKING-MODEL.md` §4.
- External (verified 2026): libp2p.io + libp2p/specs (DCUtR, AutoNAT-v2); ipshipyard.com 2025
  maintenance update; iroh.computer (pricing, DERP, noq); W3C WebRTC Rec 2025-03; RFC 8445 (ICE),
  8489 (STUN), 8656 (TURN), 8838 (trickle), 7675 (consent), 8866 (SDP); tailscale.com; slackhq/nebula;
  holepunchto/hyperdht. Unverified items flagged inline.
