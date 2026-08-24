# EXPLORATION — connectivity: NAT traversal + WebRTC, the unified architecture (how it all fits)

**Status:** Exploration / architecture synthesis — 2026-07-21. **NOT** a proposal (it maps the design and names
the decisions; the ratifiable proposals follow — Part H). Author: arch workspace. **Goal (operator steer):**
*pull NAT + WebRTC together into one coherent design; make sure it's right and that we understand how it all fits
together* — before choosing a build entry point.

**The finding in one line.** Connectivity is **not one feature** — it is **one seam** (NETWORK §10.2
`dispatch_fallback`) with a **layered escalation ladder** behind it, most of whose rungs are *already Active*; the
"NAT/WebRTC" gap is precisely the **live symmetric NAT'd↔NAT'd rung**, and NAT-punch and WebRTC are **two
substrates under one model** — same reachability facts, same signaling-as-a-separable-service, same relay
fallback, different punch mechanism (native simultaneous-open vs the browser's built-in ICE stack) and different
transport profile (`tcp`/`quic` vs `webrtc`). The whole thing composes on seams the current spec *already
reserved*; nothing here re-architects, and the load-bearing reframe is that **a punched connection is just an
ordinary transport**.

**Provenance (read-only; three exhaustive surveys + two archived DRAFTs).** Current spec: `EXTENSION-NETWORK.md`,
`EXTENSION-RELAY.md`, `EXTENSION-INBOX/CONTINUATION/SUBSCRIPTION.md`, `EXTENSION-DISCOVERY.md`,
`guides/GUIDE-NETWORKING-MODEL.md`. Archived DRAFTs (the internal legacy corpus,
read-only): `PROPOSAL-NAT-TRAVERSAL-AND-WAN-REACHABILITY.md` (31 KB, the
mature design-space map) + `PROPOSAL-EXTENSION-WEBRTC-TRANSPORT.md` (the transport-profile sketch) + the Iroh /
browser-case-study explorations. Every claim is pinned `(§/line, file)`; line numbers are current-state.

---

## Part A — The unifying frame: one seam, one escalation ladder

Every connectivity question resolves at **one site**: NETWORK §10 step 4, the `dispatch_fallback(peer_id,
execute) → {ok,result} | null` seam (`EXTENSION-NETWORK.md` §10.2, L1325; Amendment 11). The dispatcher, having
failed to reach the target directly, consults this seam once before its terminal. **Everything — store-and-forward
today, NAT-punch/WebRTC later — plugs in here** (§10.2 states it verbatim: L1334 *"later, the live-connection
punch (NAT hole / WebRTC, deliver now)… both escalate from the same step-4 site"*). This is the single most
important fact for coherence: we are not adding a subsystem, we are **filling reserved rungs on one ladder**.

**The reachability-class ladder (NETWORK §10, Amendment 8 R5 — L1296–1304), from most to least reachable:**

| Class | Reach mechanism | Status |
|---|---|---|
| `full_duplex_listener` (`tcp`/`ws`/`webrtc`) | dial directly | **Active** (tcp/ws); `webrtc` **reserved, future** |
| `half_duplex_listener` (`http`) | POST an EXECUTE | **Active** |
| **`held_connection_client`** (NAT'd peer holding a duplex socket out to us) | **push down its held socket** | **Active — this already solves public↔NAT'd, both directions** (L1268/1302) |
| `pollable_client` (non-listening, no held socket) | poll fallback (R4) | **Deferred** (Phase-2, L1271) |
| `static_publisher` (`http-poll`) | consumer-fetch only | Active (never a dispatch target) |

**What is Active today (do not rebuild):**
- **Async floor — message any offline/NAT'd/browser peer + get a reply.** RELAY **Mode S** store-and-forward
  (`EXTENSION-RELAY.md` §3.2/§6.2.1) + INBOX `deliver_to`/`deliver_token` + a CONTINUATION at the reply path,
  whose correlation is **transport-independent** (`EXTENSION-RELAY.md` L433: *"it does not depend on how the
  result arrived"*). So the *messaging* substrate is done and survives whatever transport lands beneath it.
- **Live public↔NAT'd, both directions** — the `held_connection_client` class (the NAT'd peer's outbound socket
  is reused for server→client push). `GUIDE-NETWORKING-MODEL` marks this ✅ v1.

**The gap (the whole of "NAT/WebRTC"):** a **live, low-latency, direct link between two peers that both can't be
dialed** — NAT'd↔NAT'd with no shared live session, and browser↔browser. That is the only rung missing, and it
is exactly what NAT-punch + WebRTC + RELAY Mode C fill.

---

## Part B — The decomposition: connectivity is a composition of *separable services*

`GUIDE-NETWORKING-MODEL` L117–124 and the archived NAT proposal both insist: **NAT traversal is a composition,
not one extension.** The NAT proposal §5 sharpens this into a discipline that is the backbone of a coherent
design — **four separable services, each its own handler + cap, never conflated:**

1. **Reflection / observed-address (STUN role).** A peer R tells dialer A the public `IP:port` it observed →
   that is A's public mapping. Landable as a HELLO handshake field *or* an explicit `system/network:observe-address()`
   op (NAT proposal §3.1). Multi-reflector agreement yields **free NAT-type detection**. → **NETWORK.**
2. **Reachability self-knowledge (AutoNAT role).** `system/network:check-reachability() → {reachable,
   address_tested}` (§3.2), with the hard security rule: **dial back ONLY the requester's observed source
   address, never a body-supplied one** (amplification/DDoS guard). → **NETWORK.**
3. **Signaling (the punch/offer exchange *carrier*).** Gets the connect-request / SDP-offer from A to B *before*
   they can talk directly. A **separable service with pluggable carriers** — see Part D. → its own handler.
4. **Relay data-path (TURN role).** The last-resort relayed byte path: RELAY **Mode C** (live circuit) or
   **Mode S** (async). → **RELAY.**

**The load-bearing reframe (NAT proposal §1/§3.4):** a **punched connection *is* an ordinary `tcp`/`quic` live
transport.** No new entity-layer transport type for native punch. Only two things are genuinely new at the
transport layer: **establishment** (simultaneous-open) and **keepalive discipline** (punched mappings expire —
NETWORK §5 already has keepalive). This is why the design *fits*: reflection/dial-back are pure NETWORK facts,
signaling is a thin protocol, and the resulting pipe is something NETWORK already knows how to use.

---

## Part C — NAT and WebRTC are two substrates under **one** model (the unification)

This is the heart of "how it all fits." NAT-punch (native) and WebRTC (browser) are **not two features** — they
are **two punch substrates** behind the *same* four services, differing only where they must:

| | Reflection | Signaling | Punch mechanism | Resulting transport | Where |
|---|---|---|---|---|---|
| **Native, cone NAT** | NETWORK observe-address | DCUtR entities over a carrier (Part D) | **TCP simultaneous-open** (buildable *now*) | ordinary `tcp` profile | NETWORK amendment |
| **Native, better** | NETWORK observe-address | same | **QUIC hole-punch** | `quic` profile *(we don't ship QUIC — Part E)* | NETWORK amendment |
| **Browser (↔ anything)** | **browser's built-in ICE/STUN** | **SDP + ICE candidates** over a carrier | **WebRTC ICE** (sandbox forbids raw sockets) | **`webrtc` profile** (reserved slot) | WEBRTC-TRANSPORT proposal |
| **Any↔any fallback** | — | — | — (relayed) | RELAY **Mode C** (live) / **Mode S** (async) | RELAY follow-on |

**What they *share* (the unification):** the reachability-facts layer (candidates), the **signaling-as-a-
separable-service** pattern, the relay-as-fallback ladder (Mode C/S), and the **transport-profile seam** —
`system/peer/transport/{peer}/*` resolved by reachability class (NETWORK §6.5/§10). Adding *either* substrate =
"a new profile type + wire its class"; the machinery already exists (the survey confirms the `webrtc` slot is
**reserved but unfilled** — no `system/peer/transport/webrtc` block yet).

**Why the browser is forced to WebRTC (the one hard asymmetry):** a browser peer *cannot* open raw sockets or
listen (NETWORK §14, L1426/1432 — *"browser peers are always connection initiators"*). Browser↔browser P2P is
**only** achievable via WebRTC data channels, which carry ICE/STUN/TURN *inside the browser runtime*. So WebRTC
isn't an alternative to NAT-traversal — for the browser it **is** the NAT-traversal-and-transport, bundled by the
platform. Native peers, by contrast, can run their own simultaneous-open. **The model unifies them by making the
*punch substrate* pluggable under one signaling+reflection+fallback frame** — the browser plugs in "WebRTC" where
a native peer plugs in "tcp-simopen."

---

## Part D — The signaling coherence (the crux, and where the two proposals must be reconciled)

Signaling is the one service the two archived proposals describe *differently*, and reconciling them is the
design's sharpest edge:

- The **NAT proposal** carries the native punch (DCUtR: `system/nat/connect-request` / `-response` /
  `-punch-sync {fire_at}`, §4.1) as *"a thin protocol over RELAY signaling."*
- The **WebRTC proposal** carries a *recorded operator decision* that its SDP/ICE signaling **must NOT ride RELAY
  Mode-S**; the carrier becomes one of (A) two-QR vanilla ICE, (B) QR + short-code back-channel, or (C) a thin
  native-peer **WebSocket rendezvous**.

**These reconcile cleanly under the §5 "separable services" discipline — and the reconciliation is the design
decision to pin:** signaling is **one pluggable-carrier service** with a message schema per substrate (DCUtR
entities for native; SDP/ICE for WebRTC) and a **carrier set** chosen by deployment:

- **relay-*forward* (Mode F)** for two peers that share a reachable relay — a *live signaling* path, explicitly
  **NOT** the Mode-S *mail inbox* (the NAT proposal §5 is emphatic: reflection, dial-back, signaling, and
  relay-data-path are four separable things; the Mode-S inbox is the async *data* path, never the live signaling
  carrier). This is the reconciliation: "over RELAY signaling" ≠ "over the Mode-S inbox."
- **rendezvous server** (a thin native-WS signaling endpoint) for peers with no shared relay — the WebRTC
  decision's option (C), which generalizes to native punch too.
- **out-of-band** (QR / short-code) for the human-present, zero-infrastructure case — the WebRTC decision's (A)/(B).

**The coherent rule:** *one* signaling service, *pluggable carriers* (relay-forward | rendezvous | out-of-band),
*per-substrate message schemas*. The Mode-S inbox is disqualified as a signaling carrier for *both* substrates
(it's async data, not live handshake). This unifies the two proposals instead of letting them contradict.

---

## Part E — The substrate decision (which we build first), framed not forced

**We build our own substrate** — this is not a buy-vs-build decision. The system and its infra are the point;
Iroh / libp2p / WebRTC are **reference designs we study** (their proven techniques — DCUtR punch coordination,
DERP-style relay, ICE, QUIC multiplexing), **not** dependencies we adopt. The genuine fork is *which substrate we
build first*:

- **TCP simultaneous-open** — buildable **now**, no new transport (rides the existing `tcp` profile), covers
  cone-NAT native↔native. The NAT proposal's §9 scorecard marks it the *one buildable substrate today*. **Build
  this first.**
- **Our own UDP/QUIC hole-punch** — the "better" native substrate (QUIC multiplexing; DCUtR is QUIC-shaped), but
  NETWORK ships no QUIC yet (`quic` is *"aspirational"*, §6.5.2/L797), so it requires **building our own QUIC
  transport** first — a large, separate undertaking, sequenced after the TCP-simopen slice proves the coordination
  protocol. *(Study Iroh's QUIC+traversal design as prior art here; implement our own on our substrate,
  entity-native and capability-gated.)*
- **Browser** — no fork: **WebRTC** (platform-mandated; the browser's ICE stack, driven by our signaling).

**Recommended posture:** spec the coordination protocol **substrate-agnostically** (the NAT proposal §9
recommendation), **build TCP-simopen as the first native substrate**, and **build our own QUIC transport as the
sequenced upgrade** (learning from Iroh's design, not depending on it) once the coordination protocol is proven.
WebRTC lands on its own track (browser demand), gated on the signaling schema (Part D). The state/resource
silhouette (reflect/signal/punch/relay-fallback) is **substrate-independent** — the same for tcp, our-QUIC, or
WebRTC — so the coordination + infra design carries across whichever substrate we build.

---

## Part F — The landing sequence (all rungs on the one §10.2 seam)

Ordered by *buildable-now → gated*, each an additive amendment/proposal, none a wire renumber:

1. **Reachability facts (NETWORK amendment) — landable now, independently useful.** Observed-address reflection
   (HELLO field, leaned) + `check-reachability()` dial-back-to-observed-only. Gives NAT-type detection + better
   dispatch *even before any punch* — worth landing on its own. (NAT proposal §9 "land first.")
2. **Signaling service (new handler) — the pluggable-carrier frame** (Part D): the carrier abstraction +
   per-substrate message schemas. Reconciles the two proposals.
3. **Native punch substrate — TCP-simopen** over the signaling service; the punched pipe registers as an ordinary
   `tcp` transport (Part B reframe). *(Our own QUIC transport is the sequenced upgrade after — study Iroh's design,
   build our own.)*
4. **WebRTC transport profile** (fills the reserved slot) + its SDP/ICE schema on the signaling service (Part D
   carrier decision). The browser P2P leg.
5. **RELAY Mode C** (live relayed circuit) as the last-resort live fallback when punch fails — or **subsumed by
   Iroh's relay** if that path is taken.

Everything from step 1 lands at / behind the **already-reserved** NETWORK §10.2 seam and the transport-profile
mechanism — which is *why* the current spec deliberately preserved `held_connection_client`, `webrtc (future)`,
and Mode-C-named-but-deferred: so this sequence lands **without rework**.

---

## Part G — Open design decisions (resolve before/at build)

1. **Which substrate we build first (Part E)** — TCP-simopen now, then our own QUIC transport as the sequenced
   upgrade (study Iroh's design as prior art; build our own). *The biggest call; not buy-vs-build — build order.*
2. **Signaling carrier set (Part D)** — confirm {relay-forward, rendezvous-WS, out-of-band QR/short-code} with
   Mode-S disqualified; decide which are v1.
3. **Reflection form** — HELLO handshake field (leaned, cheap) vs explicit `observe-address()` op.
4. **Unified service-advertisement** — do the four separable services (reflect / dial-back / signal / relay) get
   one discovery surface, or stay independent handlers+caps? (NAT proposal §5 leaves this open.)
5. **Consent-freshness / connectivity-check handshake** — *do not invent*; read RFC 8445 §7 + RFC 7675 (NAT
   proposal §9 🔬). Symmetric-NAT port prediction is out-of-scope-for-v1 research.

---

## Part H — Recommendation & next unit

The two archived DRAFTs are **decision-ready but were never folded forward** (the current specs reference them by
name — `EXTENSION-NETWORK` L54–56, `EXTENSION-DISCOVERY` L184 — but they are absent from this repo). Bring them
forward **reconciled under this architecture**, not verbatim. Recommended units, in order:

1. **`PROPOSAL-NETWORK-REACHABILITY-FACTS`** — Part F step 1 (reflection + dial-back), the landable-now wedge,
   independently useful. Small NETWORK amendment; ship first.
2. **`PROPOSAL-CONNECTIVITY-SIGNALING-AND-PUNCH`** — Parts B/D + Part F steps 2–3: the separable-services frame,
   the pluggable-carrier signaling service, and TCP-simopen. Carries the Iroh-eval decision (Part E/G1).
3. **`PROPOSAL-EXTENSION-WEBRTC-TRANSPORT` (brought forward + finished)** — the reserved profile + the resolved
   signaling schema (Part D). Browser leg.
4. **RELAY Mode C follow-on** — or its subsumption by an Iroh bridge, per G1.

Before authoring #2/#3, **G1 (substrate/Iroh) and G2 (signaling carriers) want an operator call** — they set the
build surface. This exploration is the "understand how it all fits" deliverable; the reconciled proposals are the
"pull it together" next step. Per the CDN-corridor meta-rule, none of it is validated until a cross-impl
conformance run exercises an actual punch between two conformant peers.

## References

- Current spec: `EXTENSION-NETWORK.md` §1.1/§6.5/§10/§10.2/§14 (L54–56, 786, 797, 1268, 1296–1304, 1325, 1334,
  1426–1434), `EXTENSION-RELAY.md` §3.2/§3.4/§6.2.1/§6.2.2/§11.1 (L196–203, 441, 460–467, 575–577),
  `EXTENSION-INBOX/CONTINUATION/SUBSCRIPTION.md` (transport-independent reply correlation), `EXTENSION-DISCOVERY.md`
  L184/325, `guides/GUIDE-NETWORKING-MODEL.md` L117–124/137/154.
- Archived DRAFTs (read-only, the internal legacy corpus (read-only)):
  `proposals/PROPOSAL-NAT-TRAVERSAL-AND-WAN-REACHABILITY.md` (§1/§3.1–3.4/§4.1–4.3/§5/§6/§9),
  `proposals/PROPOSAL-EXTENSION-WEBRTC-TRANSPORT.md` (profile §3, signaling gap G1/G4),
  `explorations/EXPLORATION-EXTENSION-TRANSPORT-IROH-BRIDGE-…`, `…-WEBRTC-BROWSER-ENTITY-TRANSFER-CASE-STUDY-…`.
- Surveys (this session, 2026-07-21): connectivity/messaging state; archived-proposal maturity; L5/compute
  connectivity-dependency (all three in the session record).
