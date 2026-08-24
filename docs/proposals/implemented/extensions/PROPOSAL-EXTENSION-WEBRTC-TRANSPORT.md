# PROPOSAL — EXTENSION-WEBRTC-TRANSPORT — the browser P2P leg (workstream D / S2)

**Status:** **FOLDED 2026-08-02 — S3 unblocked; open item A (offerer determination) resolved 2026-08-02.** Ratified into
`EXTENSION-SIGNALING.md` §6.5 (the `system/signaling/webrtc/*` coordination schema + the DTLS MUST) and
`EXTENSION-NETWORK.md` §6.5.2d (the `system/peer/transport/webrtc` profile) + §6.5.1b (reserved slot un-futured).
**Reviewed by all three cohort impls:** core-rust (protocol content + the S5-native-end finding), Go (deferred
the fold to arch), and **browser-rust (the S4 implementer, 2026-08-02) — its substantive review surfaced four
browser-substrate findings** (§6 open items). Three are **folded in place** (candidate structure, DTLS-MUST
restatement, `session_id` randomness) plus the missing rendezvous-driven flow; the fourth — **who sends the WebRTC
offer** — is now **resolved** (§6 item A: the lower-sorting `peer_id` offers, §6.5), so **S3 is unblocked**. This is the
Amendment-14 fold pattern (fix findings in place, no rev bump), and per `EXTENSION-SIGNALING.md` §11.5.1 **nothing
here is validated until the S5 gate** (§7) — everything anyone has run is still one box. This document is the
design rationale; the spec sections above are source of truth. Native↔native punch never waited on any of this
(`EXTENSION-SIGNALING.md` §7.3.1).

**What changed since the 2026-07-24 draft (why this revision exists):**
- **Seam corrected §10.2 → §10.3.** The prior draft routed WebRTC through §10.2's `dispatch_fallback` ("both
  escalate from the same step-4 site"). **Amendment 14 (landed 2026-07-31) corrected exactly that claim** and
  created the separate **§10.3 `establish_live(ctx, peer_id) → connection` seam** — it returns a *connection*,
  not a delivered result, so the connection is pooled and reused. WebRTC establishes through §10.3, the same
  seam Rust's `LiveEstablish` and Go's `tryEstablishLive` already implement.
- **Namespace resolved (draft open item #2 → RESOLVED).** `system/webrtc/*` → **`system/signaling/webrtc/*`**,
  under the one signaling owner-segment, per `EXTENSION-SIGNALING.md` §13 item #1 and
  `PROPOSAL-NAMESPACE-CLEANUP-AND-BROWSER-LEG` §5.2.
- **Reachability class resolved (draft open item #1 → RESOLVED).** No new class — WebRTC is a **punch substrate
  at the §10.3 seam**, already pinned by `EXTENSION-SIGNALING.md` §7.3.1 and now an *implementation fact*: S1
  showed `LiveEstablish` is already `#[async_trait(?Send)]` on `wasm32`, so a `!Send` RTCPeerConnection
  establisher implements it without touching the trait (S3 is one impl, not a refactor).
- **DTLS fingerprint pinning promoted to a body MUST** (was open item #3). A security MUST does not wait for a
  cross-impl run to surface it.

**Target:** fill the **reserved `webrtc` transport slot** (`EXTENSION-NETWORK.md` §6.5.1b — `webrtc (future)`)
with a `system/peer/transport/webrtc` profile entity-type + its SDP/ICE **signaling message schema** riding the
landed `system/signaling` carrier (`EXTENSION-SIGNALING.md` §6). It defines **no new wire message and no V7
renumber** — a punched WebRTC data channel is an *ordinary live transport* once open — so this is "a new profile
type + its §10.3 wiring + one per-substrate signaling schema," nothing more.
**Provenance:** `EXPLORATION-CONNECTIVITY-NAT-WEBRTC-UNIFIED-ARCHITECTURE.md` Parts C/D/F/H. Consumes the landed
reachability facts (`EXTENSION-NETWORK.md` §6.7) and the landed signaling carrier (`EXTENSION-SIGNALING.md`).
**Scope:** the **browser↔anything P2P leg** — the one rung where a peer *cannot* run a native simultaneous-open
because the sandbox forbids raw sockets, so the platform's own ICE stack is the punch-and-transport, bundled. It
is **not** an alternative to `EXTENSION-SIGNALING`'s native punch; it is the **same model with the browser's
punch substrate plugged in**. All spec-only; nothing built anywhere.

---

## 1. The one-paragraph design (why this is small)

WebRTC and native NAT-punch are **two substrates under one model** (`EXTENSION-SIGNALING.md` §7.3.1): the *same*
reachability facts (`EXTENSION-NETWORK.md` §6.7), the *same* separable signaling carrier
(`EXTENSION-SIGNALING.md` §6 — rendezvous key, opaque coordination blobs, `fire_at`, roles), the *same*
relay-as-fallback ladder — differing only in the **punch mechanism** (the browser's built-in ICE vs native
TCP-simultaneous-open) and the **transport profile** (`webrtc` vs `tcp`). The current spec already **reserved the
slot**: `EXTENSION-NETWORK.md` §6.5.1b lists `webrtc (future) | full-duplex | NAT-traversing | ONE connection,
reused both ways`, and §10 resolves transports by reachability class. So the whole of "WebRTC support" is:
**(a)** define the `system/peer/transport/webrtc` profile that fills the reserved slot; **(b)** define the
**SDP/ICE signaling schema** that rides the `system/signaling` carrier as new `system/signaling/webrtc/*`
coordination entities; **(c)** wire the resulting data channel into §10 through the **§10.3 `establish_live`
seam** as an ordinary full-duplex live transport. No new entity-layer transport primitive.

**The one hard asymmetry it exists to serve.** A browser peer *cannot* open raw sockets or listen
(`EXTENSION-NETWORK.md` §14 — *browser peers are always connection initiators*). Its only peer-to-peer transport
is a WebRTC data channel, which carries ICE/STUN/TURN *inside the browser runtime*. So for the browser, WebRTC is
not an alternative to NAT-traversal — it **is** the NAT-traversal-and-transport, mandated by the platform. Native
peers plug in `tcp-simopen`; browser peers plug in `webrtc`; the signaling + reflection + fallback frame above
them is identical.

**The reflection non-collapse `[cross-peer seam — do not conflate]`.** The browser's `srflx` candidates come from
its **own ICE stack speaking STUN/UDP to a standard STUN server** — a wire protocol our peers do not speak. A
peer offering `EXTENSION-NETWORK.md` §6.7.1 `observe-address` **is not** a STUN reflector for a browser
(`EXTENSION-NETWORK.md` §6.7, the explicit non-collapse). What this ecosystem contributes on the browser leg is
**signaling carriage, not reflection**. This proposal touches only the carriage (the `system/signaling/webrtc/*`
schema) and the resulting transport; it introduces no reflection path.

## 2. The `system/peer/transport/webrtc` profile `[fills the reserved §6.5.1b slot]`

A durable transport-profile entity per `EXTENSION-NETWORK.md` §6.5.1 (the dispatcher in §10 consults it by
reachability class). It is the **advertisement** that this peer accepts a WebRTC data channel — and, unlike a
dialable live profile, it carries **no dial `endpoint`**: a WebRTC channel is never dialed, it is *negotiated*
through signaling, so the profile declares the capability + negotiation parameters, not an address.

**Distinguish the profile from the channel `[the load-bearing distinction]`.** Two different things, both named
`webrtc`, must not be conflated:

- **The published profile** (`system/peer/transport/webrtc`, this section) is a **durable advertisement** — "a
  direct WebRTC path to me exists." It is a normal published entity, discovered like any transport profile.
- **The established data channel** is a **session-scoped §10.3 connection** — the result of actually negotiating
  and punching. Per Amendment 14 it is an *ordinary transport*, **never itself published** as a durable
  `system/peer/transport/*` profile (the negotiated channel, like a punched NAT mapping, is session-scoped and
  `MUST` run keepalive). The profile advertises that a punch is *possible*; the channel is the punch's *result*.

```
type: "system/peer/transport/webrtc"          ; fills the §6.5.1b reserved slot
data: {
  transport_type:   "webrtc",                 ; matches the profile-name suffix (§6.5.1)
  supported_ops:    ["EXECUTE"],              ; + reserved SUBSCRIBE (descriptive, never a grant — §6.5.1)
  freshness:        "live",                    ; a negotiated data channel is a live transport (§6.5)
  ? priority:       uint,                       ; §6.5.1a selection (lower = preferred; default 100)
  negotiation: {
    signaling_schema: "webrtc-sdp-ice/1",      ; the §3 message schema this peer speaks (pins the schema version)
    ? ice_policy:     "all" / "relay",          ; "relay" = force TURN (privacy / no host candidates); default "all"
    ? dtls_role:      "auto",                    ; DTLS role negotiation; "auto" = the SDP offer/answer decides
  },
  ; NO endpoint.url — a WebRTC channel is negotiated, not dialed (contrast §6.5.1a D4 live-endpoint rule)
}
```

- **Why no `endpoint`.** §6.5.1a D4 requires a scheme-prefixed `url` on every *dialable* live profile (`tcp`,
  `websocket`, `http`). `webrtc` is reached by **rendezvous + negotiation**, never by dialing an address. The
  profile's presence is the advertisement; the address is discovered per-session as ICE candidates (§3).
- **Reachability class.** A peer that publishes a `webrtc` profile is reachable as a **punch-substrate class** —
  the same rung native-punch fills at the §10.3 `establish_live` seam, not a `full_duplex_listener` (nothing is
  listening). Resolving such a peer, the §10 dispatcher escalates to §10.3 and drives the WebRTC negotiation
  rather than attempting a dial. No new dispatch branch beyond "this class negotiates rather than dials."
- **Once open, it is ordinary.** After negotiation the data channel carries **bare ECF envelopes** exactly like
  any live transport; framing is the data-channel's own message boundaries (like `http`, and unlike `tcp`, it
  does **not** apply the `ENTITY-CORE-PROTOCOL.md` §1.6 TCP length prefix — the channel is message-oriented). The
  §10 dispatcher, the held-capability model (§6.6), and session state (`system/peer/session/*`) all apply
  unchanged.

## 3. The `system/signaling/webrtc/*` schema `[the only per-substrate new shape]`

WebRTC negotiation rides the **landed signaling carrier** (`EXTENSION-SIGNALING.md` §6 — `offer`/`collect` over
a rendezvous key) — the carrier is **opaque** (it never decodes the message, and byte-preserves it so the
detached signature stays valid, §6.2), so WebRTC needs only to define *what signed entity travels through it*.

These are the **WebRTC-substrate analogue of the native `system/signaling/{connect-request,connect-response}`
coordination types** (§6.1). They are **separate types**, not those types with an added field: the native
coordination entities carry a `candidates: array_of system/network/candidate` list and no SDP; WebRTC negotiation
is an **SDP** exchange (DTLS fingerprint, ICE ufrag/pwd, data-channel params — more than a candidate list), so it
gets its own shapes under the substrate sub-namespace. All three are signed by their author exactly as
`EXTENSION-SIGNALING.md` §6.3 requires; the signature is the origin proof and the carrier stays byte-opaque.

```
system/signaling/webrtc/offer = {       ; the initiator's SDP offer + initial candidates
  session_id:  <bytes>,                 ; ties offer/answer/candidates together at one rendezvous key
  sdp:         tstr,                     ; the RFC 8866 SDP offer (opaque to the carrier)
  ? candidates: [* <ice_candidate>],    ; trickle-ICE: initial candidates inline, more via -candidate
}
system/signaling/webrtc/answer = {      ; the responder's SDP answer
  session_id:  <bytes>,
  sdp:         tstr,                     ; the SDP answer
  ? candidates: [* <ice_candidate>],
}
system/signaling/webrtc/candidate = {   ; a trickled ICE candidate (either direction, post-offer/answer)
  session_id:  <bytes>,
  candidate:   <ice_candidate>,         ; a RAW RFC 8445 candidate line + its sdpMid/sdpMLineIndex
}
```

- **`<ice_candidate>` is a raw RFC 8445 line, NOT `system/network/candidate` `[non-collapse — MUST]`.** The
  native candidate type (`EXTENSION-NETWORK.md` §6.7.3, `host`/`srflx`/`relay` with a substrate + endpoint) is
  entity-core's own reachability fact. A WebRTC ICE candidate is the **browser's** ICE format, produced and
  consumed by the browser's ICE stack and passed through this carrier **verbatim**. They are different shapes and
  MUST NOT be translated into one another — the reflection non-collapse (§1) at the candidate layer.
- **`session_id` is the correlation key** *within* a rendezvous key — a `lobby` or `tag` key may host several
  concurrent pairings, so offer/answer/candidate tuples are grouped by `session_id`, not by the shared key alone.
  This is the WebRTC analogue of the native `nonce` echo (§6.4); it does not replace the §6.4 bucket-read rules
  (skip-own, skip-undecodable), which apply to these entities unchanged.
- **Trickle ICE is the default** (RFC 8838): candidates flow as `-candidate` entities as they are gathered,
  rather than blocking the offer on full gathering — critical for the seconds-matter handshake window. A peer MAY
  inline its first candidates in the offer/answer for a one-round-trip fast path.
- **STUN/TURN reflexive candidates come from the browser's own ICE stack** (§1 non-collapse), seeded by the
  reflector/relay pools the peer already knows. A `data_relay` pool doubles as the TURN pool for
  `ice_policy: "relay"` — no separate infra noun.
- **DTLS fingerprint pinning `[security — MUST]`.** The SDP carries a DTLS fingerprint. Because the offer/answer
  entity is signed by its author, that fingerprint is **bound to the peer identity**. A receiving peer **MUST
  verify the negotiated DTLS fingerprint matches the fingerprint in the signed offer/answer** before treating the
  channel as the peer, and **MUST tear the channel down on mismatch.** Without this, a signaling MITM that cannot
  forge the signature can still substitute *its own* channel by relaying a different fingerprint. This is the
  browser-leg analogue of the native §7.4 connectivity-check identity binding — the same "confirm the thing you
  punched to is who you meant" MUST, one layer over.
- **The carrier disqualification carries over.** The Mode-S async inbox is **not** a signaling carrier for
  WebRTC: SDP/ICE is a live, seconds-bounded handshake, so it rides the live `rendezvous` carrier (v1) or
  `relay-forward` (RELAY Mode-F, opt-in), never async mail. Same rule the native punch pinned; WebRTC does not
  reopen it.

## 4. Wiring into §10 (the dispatcher) and §14 (the browser rule)

- **§10.3 `establish_live` seam.** Resolving a peer whose best profile is `webrtc` consults the §10.3
  `establish_live(ctx, peer_id)` seam (step 3b — after profile resolution fails to find a dialable endpoint,
  before the §10.2 delivery fallback). The seam's policy drives the offer/answer/candidate exchange over
  signaling, opens the data channel, runs the §7.4 identity + DTLS-fingerprint check, and returns it as an
  **ordinary session-scoped connection** that §10 pools and reuses. The two Amendment-14 obligations apply
  unchanged: it is an ordinary transport (never a new transport type, never a published durable profile), and it
  **MUST run keepalive** (a negotiated channel over a NAT mapping dies on silence).
- **§14 browser-initiator rule holds and is *why* this exists.** A browser peer is always the connection
  initiator; it cannot be a `full_duplex_listener`. WebRTC is the *only* way it participates in P2P. A native
  peer **MAY** also publish `webrtc` (to be punchable by browsers) alongside its `tcp`/`websocket` listener
  profiles — the §6.5.1a `(priority asc, profile-id lex)` selection picks per-peer, so "native reaches browser
  over `webrtc`, browser reaches native over `webrtc`, native reaches native over `tcp`" is the ordinary
  asymmetric case, not a special one.
- **Relay fallback underneath (unchanged).** When ICE fails (symmetric NAT / restrictive firewall), the peer
  falls to the relay ladder — RELAY **Mode-C** (live circuit) or **Mode-S** (async) — exactly as native punch
  does (`EXTENSION-SIGNALING.md` §10). This proposal does not define that fallback; it inherits it. A substrate
  mismatch is a punch that fails at candidate selection, never a dispatch error (§7.3.1 point 1).

## 5. Build posture — the cheapest end-to-end exercise of the connectivity arch

The browser's ICE stack is **provided by the platform** — we do not implement STUN/TURN/ICE, we *drive* it with
our signaling. So this unit exercises the whole connectivity architecture (reflection → signaling → punch → live
transport → §10.3 dispatch) with the **least new native code** — the punch itself is the browser's. S1 confirmed
the carrier half already compiles for `wasm32` and that `LiveEstablish` is already `?Send`-shaped, so S3 is one
`LiveEstablish` impl plus a `Connector` at existing seams (`transport.rs`'s `?Send` split, with
`MessagePortListener` as the non-socket-transport precedent), **not** a trait refactor.

- **Owner (S3, the transport crate):** `entity-core-rust`, `wasm32`-gated (`cfg(target_arch = "wasm32")`), the
  shape `core/crypto` already uses — a transport is protocol, so it lives in core-rust, not in the app host.
- **Consumer (S4):** `entity-browser-rust` (the WASM/browser peer) consumes it — the repo whose §14
  browser-initiator constraint motivates the whole leg. **See open item #1 on what, if anything, it publishes.**
- **The gate (S5):** **two browser peers** negotiating a real data channel to each other via a real
  `system/signaling` node — see §7. **Not browser↔native** (core-rust's review finding): a native end would need a
  `webrtc-rs`-class data-channel terminator, which §4 defers — so browser↔native has *no native WebRTC end* to
  punch to. Browser↔browser needs none, and it is the case where WebRTC is a genuine *unblock* (two browsers reach
  each other no other way), not the mere optimization browser↔native is over the already-working WebSocket path
  (§7.3.1). The signaling node is native but carries only opaque coordination blobs — it needs no WebRTC.
  **Browser↔native-direct is a *later* gate that lands with native WebRTC**, and it is that gate — not this one —
  that needs open item #1 resolved.

## 6. Open items (fold-time / cohort)

Resolved before fold: **~~reachability class~~** (no new class — §10.3 substrate, S1-confirmed, §7.3.1);
**~~namespace~~** (`system/signaling/webrtc/*`); **~~DTLS pinning~~** (a §6.5 MUST).

**S4-implementer review (browser-rust, 2026-08-02), folded in place:** the trickled candidate is now a
**structured** `{candidate, sdp_mid, sdp_mline_index, ?username_fragment}` (a bare line is rejected by
`addIceCandidate()`); the DTLS MUST is restated in browser-achievable form (**verify signature → feed that SDP
verbatim → never `setRemoteDescription` on unverified SDP**; RFC 8827 discharges the fingerprint binding);
`session_id` is now MUST-random-≥16B; and the missing **rendezvous-driven** establishment trigger (the
browser↔browser gate) is documented (§6.5). browser-rust also **confirmed** item 1's lean against its shipping
code (a browser publishes no `webrtc` profile) and the S3-core-rust / S4-consumes split (a `BrowserWebRtcConnector`
slots into the SDK `Connector` seam). **One finding is not fixable in place — it blocks S3:**

**A. ✅ Offerer/answerer determination — RESOLVED 2026-08-02 (was blocking S3).** WebRTC glare (both offer) is
fatal, so the offerer MUST be deterministic. **Rule: W3C perfect negotiation keyed to §3.2's byte-wise
peer-id sort** — a peer posts an offer optimistically and the collector answers; on a **glare** (both offered)
the lower-sorting `peer_id` (`lo`) wins and `hi` rolls back to answerer; `pair` mode pre-assigns (`hi` waits for
`lo`). Deterministic in every mode, and not a new convention (it reuses the §3.2 sort). The analysis that clears the
`b3ff6ad` constraint (`ROUTING-2026-07-31`): the retracted `lower-dials/higher-listens` split was non-traversing
because a *listening* peer never fires outbound and never opens its NAT hole — but the offer/answer role is the
**distinct SDP-negotiation layer** (the role layer §7.4.1 governs, which that same routing note explicitly
separates from the dial layer), and **both** ICE agents still send connectivity checks regardless of who offered,
so **both still dial**. So the peer-id-sorted offerer is consistent with *both* §7.4.1 and the retraction; the
offerer also chains to the §7.4.1 initiator (same peer runs the HELLO client). Folded at `EXTENSION-SIGNALING.md`
§6.5. A cross-peer MUST — validated at S5 like every §6.5 claim.

**S3-scoping finding (core-rust, 2026-08-03), folded in place — split-peer byte preservation.** Core-rust's S3
scoping surfaced that the browser peer is **execution-split**: `RTCPeerConnection` is main-thread-only, while the
signing identity + carrier live in a Worker, so the signed offer/answer/candidate crosses an **intra-peer**
boundary — between production and signing (send), and between verification and `setRemoteDescription` /
`addIceCandidate` (receive). §6.5's channel-identity MUST assumed *verify* and *use* touch the same bytes; in a
split peer they do not, so an interposed re-encode verifies one SDP and consumes another — silently defeating the
fingerprint bind. Fixed with a **split-peer byte-preservation MUST** (`EXTENSION-SIGNALING.md` §6.5):
signed-bytes == produced-bytes and consumed-bytes == verified-bytes across the crossing, carried verbatim, no
re-encode; the signed `sdp` is the peer's finalized **local-description** SDP (the fingerprint it actually
presents). This **confirms** core-rust's architecture (Worker = trust anchor, main thread = SDP conduit) rather
than changing it — **S3 has an explicit go-ahead**, and the §7.4 obligation stays discharged by the §6.5
channel-identity binding (no separate native check). The worker↔main `Request`/`Response` variants + version bump
this needs are core-rust's **internal** boundary protocol (out of spec scope); the spec pins only the byte
invariant across it. Single-impl-invisible `[§11.5.1]` — exercised at S5.
`docs/status/ROUTING-2026-08-03-webrtc-worker-boundary-and-s3-go-ahead.md`.

Remaining (non-blocking):

1. **Does a browser peer publish a `webrtc` profile at all? — CONFIRMED: no** (browser-rust, verified against
   shipping code; it publishes no `system/peer/transport/*` and reaches peers as a WebSocket-initiating client).
   Retained here as the record and because it still gates the *later* browser↔native-direct milestone. `[was: cross-peer seam]` §2 and §4
   pull opposite ways and the prior draft did not reconcile them. A browser is a **permanent initiator** (§14):
   it opens the channel outbound, and the channel is full-duplex and **reused both ways for that session**
   (§6.5.1b), so the native side never needs to *initiate to* the browser. On that reading a browser peer
   publishes **no** transport profile — its reachability is rendezvous/session-scoped, and only the **native**
   peer publishes a `webrtc` profile (its §4 MAY). The alternative — a browser advertises `webrtc` via
   REGISTRY/manifest so a native peer can find it and open a rendezvous — needs a story for how the two meet at a
   rendezvous when the browser is offline (they can't; rendezvous needs both present). *Leaned: browser publishes
   nothing; native-side profile only.* This is **not** about a browser being unable to hold a tree — it holds a
   full tree — it is about the channel's session-scoped properties and the initiator rule. **Confirm with
   browser-rust + core-rust, because it decides what S4 builds.**
2. **Consent-freshness / ICE restart.** Long-lived channels need ICE consent-freshness (RFC 7675) and restart on
   network change. *Do not invent* — read RFC 8445 §7 + RFC 7675 at build. Cohort at build.
3. **`ice_policy: "relay"` privacy mode.** Forcing TURN hides host candidates (no LAN-IP leak) — confirm whether
   v1 ships it or defers to the RELAY Mode-C track.

## 7. Honesty / validation gate `[§11.5.1 — substrate-scoped]`

Per the CDN-corridor meta-rule: **none of this is validated until a cross-impl conformance run exercises an
actual WebRTC data channel** negotiated between **two browser peers** via a real `system/signaling`
node — SDP/ICE schema bugs and DTLS-pinning gaps do not surface in prose review. Per `EXTENSION-SIGNALING.md`
§11.5.1 the browser leg **does not inherit the native punch's green**: a same-origin / loopback browser test is
blind in exactly the way that section's blindness class describes (no NAT in path ⇒ every packet arrives
regardless of mapping), so **S5 — two real browser peers establishing a real data channel across real networks
via a real signaling node — is the only real evidence, and a compile (S1) or a loopback/same-origin run is not
it.** Everything anyone has run so far is one
box. Every "typical" WebRTC number (ICE success rate, gathering latency) is `[K]` until measured on that gate.

## References
- Design: `docs/research/explorations/EXPLORATION-CONNECTIVITY-NAT-WEBRTC-UNIFIED-ARCHITECTURE.md` (Parts A/C/D/F/H).
- Landed surface it consumes: `EXTENSION-SIGNALING.md` §6 (coordination carrier), §7.3.1 (substrate model), §7.4
  (connectivity check), §10 (fallback ladder), §11.5.1 (substrate-scoped gate); `EXTENSION-NETWORK.md` §6.5.1/
  §6.5.1b (reserved slot), §6.7 (reflection facts + the non-collapse), §10.3 (`establish_live` seam), §14
  (browser-initiator rule).
- Board: `docs/status/HANDOFF-2026-08-02-namespace-split-and-connectivity-board.md` (workstream D / S0–S5);
  `PROPOSAL-NAMESPACE-CLEANUP-AND-BROWSER-LEG` §5 (the S-order).
