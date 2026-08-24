# PROPOSAL — NETWORK reachability facts (observed-address reflection + dial-back + candidate gathering)

**Status:** **RATIFIED + FOLDED (2026-07-29)** — landed as `specs/extensions/EXTENSION-NETWORK.md` **v1.5,
Amendment 13**, new **§6.7.1–§6.7.5** (+ §3.1 manifest, §3.2 capability model, §12.1/§12.3/§12.4 conformance,
§13 types). Superseded as the normative home: **the spec is source of truth from here**; this document is the
design record and the rationale that did not fold. Cohort findings against §6.7 fix the spec in place.
*Was: DRAFT (2026-07-22).*
**Not validated:** the §6.7.5 cross-impl gate (one peer reflects the other's real observed address; a dial-back
across a real NAT) **has not run.** Folded ≠ proven — see §7.
**Target:** amendment to `specs/extensions/EXTENSION-NETWORK.md` — a new reachability-facts section
(proposed **§6.7**), one optional field on the §6.3 HELLO handshake, two new capabilities
(`system/capability/network-reflect`, `system/capability/network-dialback`), and a note wiring the gathered
candidates into the §10 dispatch loop. No V7/wire renumber; no new required dispatch behavior.
**Provenance:** brought forward — *reconciled, not verbatim* — from the archived DRAFT
`PROPOSAL-NAT-TRAVERSAL-AND-WAN-REACHABILITY.md` §2/§3.1–§3.3/§5/§8/§9 (read-only, in
`entity-lab-legacy-meta/entity-core-architecture/docs/architecture/v7.0-core-revision/proposals/`), landed under
the architecture of `EXPLORATION-CONNECTIVITY-NAT-WEBRTC-UNIFIED-ARCHITECTURE.md` (Part F step 1 + Part H unit #1:
"the reachability-facts wedge — landable now, independently useful").
**Scope:** **additive; the landable-now wedge.** The *facts* a peer needs to know how it is reachable — how it
looks from outside (reflection), whether it is publicly dialable (dial-back), and what addresses it might be
reached at (candidate gathering). Independently useful — NAT-type detection and better §10 dispatch **even before
any hole-punch exists.** The punch-coordination protocol that *exchanges and acts on* these facts is deliberately
**out of scope** here (it is the follow-on `PROPOSAL-CONNECTIVITY-SIGNALING-AND-PUNCH`, which carries the open
carrier/substrate operator calls — §8 below). This unit has **no such open dependency** and is decision-ready.

---

## 0. Motivation — why land the facts first

`EXTENSION-NETWORK.md` §10 (Amendment 8) already dispatches by **reachability class**: full-duplex listener /
half-duplex listener / held-connection client / pollable client / static publisher (L1296–1304). The class table
is the spine of outbound delivery. But a peer today has **no protocol way to learn its own reachability facts**:

- It cannot learn its **public-facing `IP:port`** (only its NAT router knows; the router reveals it only
  implicitly, on outbound packets).
- It cannot learn **whether it is publicly dialable** or sits behind NAT.
- It therefore cannot honestly *populate or order* its own transport profiles for a peer on the far side of a NAT.

These are **pure transport facts** — the entity layer never sees `IP:port`s (load-bearing invariant: addressing
below the model; V7 §1.4 local-view authority), which is exactly why they belong **in NETWORK, below the model**,
not in an application extension. They are useful the moment they land, independent of any punch:

- **NAT-type detection for free** — agreement across several reflectors ⇒ a stable mapping (endpoint-independent
  NAT, punchable); disagreement ⇒ symmetric NAT (mapping differs per destination, punch will likely fail, prefer
  relay). A peer knows this **before** attempting anything.
- **Better dispatch** — a peer that knows it is NAT'd stops advertising unreachable direct profiles and leans on
  its held-outbound socket (`held_connection_client`, §1 below) / relay, cutting failed dial attempts.
- **The on-ramp for everything after** — the punch (unit #2), WebRTC (unit #3), and the registry
  service-advertisement's `reflector` service (`PROPOSAL-REGISTRY-SERVICE-ADVERTISEMENT.md` §1, which advertises a
  reflector endpoint but does **not** define what it does — *this* proposal defines it) all consume these facts.

## 1. What NETWORK already does — the accounting (half the problem is already solved)

Before adding anything, the honest baseline (this is unchanged from the archived DRAFT §2, re-verified against
current NETWORK):

- **The asymmetric-NAT case is already handled.** NETWORK §10's `held_connection_client` class (L1300): a
  non-listening peer behind NAT that holds an **outbound** duplex socket to a public peer *receives pushes down
  that socket* — no inbound dial. This is exactly how RELAY Mode S (§6.2.1) delivers to a NAT'd peer. **So a NAT'd
  peer ↔ a *public* peer works in v1 today, both directions**, no traversal.
- **The `webrtc` slot is already reserved** in the §10 full-duplex-listener row (L1298) — the browser leg (unit #3)
  lands there without a table change.
- **The one genuinely missing case** (out of scope here, it is unit #2) is *two peers both behind NAT wanting a
  direct connection* — the held-socket trick has no public party in the middle to hold a socket *to*, so it needs
  either a relayed data path (RELAY) or a punched hole (the coordination protocol).

This proposal adds **only the facts** that both the improved dispatch *and* the future punch require. It does not
add the punch.

## 2. Observed-address reflection (STUN-role) — proposed NETWORK §6.7.1

**The fact:** when peer A dials any peer R, R can tell A the **source `IP:port` it observed** on that connection.
That observed source address *is* A's public NAT mapping. This is STUN's binding, expressed in our protocol.

### 2.1 Two mechanisms — the handshake field (v1) and the explicit op

**(a) HELLO handshake field — the v1 default.** The §6.3 connection handshake (HELLO) gains an **optional**
responder-filled field:

```
observed_address: string   ; OPTIONAL. The source IP:port the responder observed on THIS connection.
                           ; When present, it is the transport-layer source of the request connection —
                           ; never a value echoed from the requester's body. Absent ⇒ responder does not
                           ; offer reflection; the requester proceeds (tries another reflector).
```

Cheapest possible: every handshake yields the fact for free, one field, no new op, no extra round-trip.

**(b) Explicit op — for periodic re-checks.** `system/network:observe-address() → {observed_address}`. A peer can
ask *without* a full handshake (NAT mappings drift; a peer re-checks periodically). Slightly more surface than the
field; both MAY be offered.

> ~~**Leaned (a) for v1** (matches the archived DRAFT §3.1 lean); (b) is the re-check convenience.~~
>
> **RULED 2026-07-29 — (b) is the v1 path; (a) routes upstream.** The lean toward (a) was made on cost (it is
> free with every handshake) without noticing that **(a) is not this repo's to ratify.**
> `system/protocol/connect/hello` is normatively defined in
> `entity-core-protocol/specs/ENTITY-CORE-PROTOCOL.md` (L1238) — adding a field to it is a **core-protocol
> change**. This repo may propose it; it may not land it, because the spec is upstream and implementations
> implement it rather than define it (AGENTS-STANDARD, "working across the polyrepo").
>
> **(b) carries no such *ownership* dependency.** `system/network:observe-address() → {observed_address}` is an
> ordinary NETWORK extension op, gated by §5's `system/capability/network-reflect`: **no core-protocol edit and
> no handshake change** — the surface it adds is this repo's to ratify. It yields the *same fact* — the
> transport-layer source address of an established connection — and satisfies §2.3's agreement-across-reflectors
> discipline identically. Arguably better: it can re-measure a drifted mapping without forcing a fresh
> handshake, which is what (b) existed for in the first place.
>
> **Correction (2026-07-29, `entity-core-rust`): "no shared-core file touched in any impl" was false, and it was
> asserted without checking any impl.** Ownership and cost are different claims and this ruling conflated them.
> Rust reported that its dispatch seam extracts only `remote_peer_id` from the inbound connection while
> `conn.remote_addr` sits on the same struct at the same expression and is dropped, and that its `HandlerContext`
> doc explicitly rejects convenience additions. **Verifying that report against the spec found the real shape of
> the cost, and it is a spec fact, not an impl shortcoming — see §2.1.1.** The ruling itself (b over (a)) is
> **unchanged**: it rests on ownership, which the correction does not touch.
>
> **So (b) is what the v1 punch stands on and what the cohort builds.** (a) stays the eventual cheap default and
> routes to `entity-core-protocol` as its own proposal, on its own schedule, **gating nothing here.**
>
> A deployment's `reflector` service (`PROPOSAL-REGISTRY-SERVICE-ADVERTISEMENT.md`) is *this* op/field offered by
> a public peer.

### 2.1.1 The observed address is responder-side and has no durable home (MUST) — added 2026-07-29

Verifying Rust's cost finding produced the load-bearing fact neither this proposal nor the ruling had:
**an observed source address is a responder-side fact, and every existing place an address is written is
dialer-side dialable-endpoint state.**

- `system/connection/{peer_id}` (ENTITY-CORE-PROTOCOL.md §3.13) has an `address` field and is keyed by peer —
  it is the registry an implementer will reach for first. It is **the wrong one twice over.** Its `address` means
  *the endpoint I dial to reach this peer*, and it is written **dialer-side**: the responder — the party that
  holds the observed source — records nothing, correctly, because an ephemeral source port is not a dialable
  endpoint. (`entity-core-rust` `connection_state.rs` states exactly this discipline, and it is right.)
- `system/peer/transport/{peer}/*` profiles are durable published endpoints — ruled out by §4.1 already.
- Keying by `peer_id` cannot express the fact anyway: §4.2 makes the mapping **per socket**, and a peer may hold
  more than one connection.

> **MUST.** The observed source address is read **from the live connection the request arrived on** and returned
> in the op response. An implementation **MUST NOT** write it to `system/connection.address`, to a
> `system/peer/transport/*` profile, or to any other durable per-peer address field. It is not connection-state,
> not a transport profile, and not a peer attribute.

**Why this is normative.** The plumbing is genuinely missing — that part of Rust's finding stands — and the
cheapest way to "solve" it is to widen a per-peer address record that already exists and already reaches the
handler. Doing so **corrupts dispatch for every other reader**: `system/connection.address` is what §10 and
`system/peer/status` consume as a dialable endpoint, and an ephemeral source port written there is a routable-
looking value that routes nowhere. That is §4.1's durable-vs-ephemeral error one layer down, on a field with
more consumers. Pinning the prohibition costs nothing and forecloses the attractive wrong fix.

**What it therefore costs, honestly.** One accept-side seam per impl, carrying the connection's source address
to the handler that answers `observe-address` — a narrow, NETWORK-scoped addition, not a general
`HandlerContext` widening (which its own doctrine rejects, and which this rule is designed to avoid needing).
Where that seam sits in a given tree's layering is the impl's call; in Rust's it is core-side, which is why
**§2.1's "no shared-core file touched" was wrong as stated.**

### 2.2 The native / browser split (do not conflate)

Reflection **splits by world**, and collapsing the two is a cross-peer-seam error:

- **Native peers doing our own punch:** reflection is the entity-protocol fact above — any peer that speaks our
  protocol can reflect. `system/capability/network-reflect` gates it (§5).
- **Browser peers:** a browser's *own* ICE agent gathers `srflx` candidates by talking to a **standard STUN
  server over the STUN/UDP wire protocol**, which our entity-protocol peers do **not** speak. **A peer cannot be a
  STUN reflector *for a browser* just by offering our op.** Browser reachability uses standard STUN/TURN infra
  (e.g. coturn); what *we* contribute on the browser leg is **signaling carriage** (unit #2/#3), not reflection.

### 2.3 NAT-type detection falls out for free

A peer collects `observed_address` from **several** reflectors. **Agreement** across reflectors ⇒ a stable,
endpoint-independent mapping (punchable). **Disagreement** ⇒ the mapping differs per destination ⇒ symmetric NAT ⇒
punch will likely fail ⇒ prefer relay. This is NAT-type detection with no extra mechanism — a direct input to the
prefer-cheap-path selection the infra design relies on.

## 3. Reachability self-knowledge (dial-back / AutoNAT) — proposed NETWORK §6.7.2

**The fact:** "Am I publicly dialable, or behind NAT?" A peer learns this by **asking another peer to dial it
back** at its observed address and seeing whether the dial-back arrives.

- **Op:** `system/network:check-reachability() → {reachable: bool, address_tested: string}`. The asked peer
  attempts an inbound dial to the requester's observed source address and reports success/failure.

### 3.1 The load-bearing security rule (MUST — a reflection/amplification vector)

**This is not optional to get right; it is a MUST because a body-supplied target turns every dial-back peer into
a DDoS reflector.** The rule (mirrors STUN binding + libp2p AutoNAT "dial-back to observed addr only"):

> **MUST:** the dial-back targets **the requesting peer's own observed source address** — the address the asked
> peer *itself observed* on the request connection — and **never an address supplied in the request body.** The
> asked peer MUST rate-limit dial-backs per requester and keep the dial-back payload small and fixed-size (no
> amplification factor).

This is a **cross-peer-seam MUST**, not a SHOULD: two conformant readings of "dial back the requester" — one using
the observed source, one honoring a body field — diverge into a security hole at the peer boundary and would pass
prose review. It is pinned here once, generally. (Full audit: §6.)

## 4. Candidate gathering + ICE-style typing — proposed NETWORK §6.7.3

To be reachable, a peer knows **all the addresses it might be reached at** — its *candidates* — typed and ordered.
**This section defines gathering and typing only.** The *exchange* of candidates between two peers (the
`connect-request`/`connect-response` messages) is the punch-coordination protocol and is **out of scope** — unit
#2. Gathering is a local fact; exchanging is a protocol. Keeping them apart is the clean seam.

- **Candidate types** (origin of the address; ICE priority order):
  - `host` — a local/LAN address (works if peers share a network). Highest priority (cheapest).
  - `srflx` (server-reflexive) — the reflection-observed public mapping (§2). The hole-punch target.
  - `relay` — a public relay address (RELAY) — the always-works fallback. Lowest priority.
- **Ordering:** `host` → `srflx` → `relay`; first pair that completes a connectivity check wins. This is the
  **session-scoped** extension of §10's existing "try profiles in `(priority asc, profile-id lex)` order" loop —
  the same try-in-order idea, applied to ephemeral candidates instead of durable profiles.

### 4.1 Candidates are session-scoped and ephemeral — NOT durable `system/peer/transport` profiles (MUST)

**A load-bearing distinction, pinned as MUST to prevent a cross-impl bug:**

> **MUST NOT** model a candidate as a durable `system/peer/transport/{peer}/{profile-id}` profile entity
> (§6.5.1). §6.5 profiles are **stable published endpoints** (a TCP listener URL, an `http-poll` CDN prefix);
> candidates **change per session and per NAT mapping** and exist for one connection attempt. A candidate written
> as a durable profile goes **stale instantly** and mis-routes every later dispatch that reads it.

Candidates therefore travel *inside* the (unit-#2) coordination messages, **never** as published tree state. An
impl that persists a `srflx` candidate as a transport profile is non-conformant to this rule. This is the
durable-vs-ephemeral seam that, uncaught, produces the classic "worked for the issuer, stale for everyone else"
cross-peer failure.

### 4.2 A `srflx` candidate MUST be the mapping of the socket the peer will punch from (MUST) — added 2026-07-29

A NAT allocates a mapping **per local socket**. An observed address is therefore only meaningful *for the socket
that produced it*, and neither this proposal nor the punch proposal said so.

> **MUST.** A peer that publishes a `srflx` candidate MUST punch from the **same local endpoint whose mapping was
> observed** — in practice, binding the reflector connection and the punch socket to the same local port with the
> platform's address/port-reuse options. A `srflx` gathered on one ephemeral socket and punched from another **is
> not the peer's address**: it describes a hole that will never open.

**Why this is normative and not an impl detail.** The socket options are the impl's business. The *binding
between the candidate and the socket* is not — a peer that gets it wrong sends its counterparty an address that
is a **lie**, and the counterparty punches at a mapping that does not exist. It never appears on the wire, yet it
is cross-peer observable in its effect, which is precisely the class this corpus keeps having to pin.

**And it fails wearing someone else's costume.** The symptom is "the punch didn't land" — indistinguishable from
a `fire_at` timing miss. The natural response is to retune the delay, and retuning is *sanctioned*
(`PROPOSAL-CONNECTIVITY-SIGNALING-AND-PUNCH` §4.1 makes `d` a local tunable), so an implementer can spend the
entire debugging budget inside the one knob guaranteed not to be the problem. **Bisect against this before the
timing** on the first cross-impl punch failure.

Applies identically to §2.1 mechanism (a) and (b) — the mapping belongs to the connection's socket either way.

**Its cost is substrate-dependent, and that is a G1 input** (`entity-core-rust`, 2026-07-29). §4.2 + §2.1(b)
(the mapping is observed on an established connection) + `PROPOSAL-CONNECTIVITY-SIGNALING-AND-PUNCH` §5
(TCP simultaneous-open) **jointly** require punching from the reflector connection's own local port. On
UDP/QUIC that is trivial. On TCP it requires `SO_REUSEADDR`/`SO_REUSEPORT` plus an explicit bind on both the
reflector dial and the punch dial — **it is not satisfiable by discipline**, and no impl gets it by writing
careful code. This does not change §4.2, which is substrate-independent; it changes what §5's "chosen first
because it reuses what ships" is worth. Routed to that proposal's §5 as a recorded cost correction, and to the
operator as a G1 input — not re-decided here.

## 5. Capabilities (proposed)

Two new capabilities, matching NETWORK's `system/capability/*` naming (kebab namespace, snake data keys):

| Capability | Gates | Default posture |
|---|---|---|
| `system/capability/network-reflect` | who may ask for observed-address reflection (§2, op form; the field rides the ordinary handshake) | **Broad default grant is reasonable** — it only echoes the source address the peer itself saw (low-risk, no amplification). Still **rate-limited**. |
| `system/capability/network-dialback` | who may ask a peer to dial them back (§3) | **Restricted** — a peer SHOULD grant it to peers it is actively connecting with (so setup-time reachability checks work) and **rate-limit**; SHOULD NOT be an open grant to arbitrary peers. |

Neither capability exists in NETWORK today (verified: no `network-reflect`/`network-dialback` grep hit) — net-new,
no collision. No other new cap: the handshake field reuses the existing HELLO flow.

## 6. Security (baseline audit — not deferred)

Per the ecosystem's "security audits are never deferred" discipline, the new surface, audited now:

| Surface | Threat | Mitigation |
|---|---|---|
| **Dial-back (§3)** | **Reflection/amplification DDoS** — attacker asks N peers to "dial back" a victim | **§3.1 MUST:** dial-back to the requester's **own observed source** only, never a body address; per-requester rate-limit; small fixed-size payload (no amplification factor). Cap-gated (`network-dialback`). |
| **Observed-address (§2)** | A lying reflector feeds a peer a wrong public mapping ⇒ wasted/failed punches, or steering toward an attacker | Collect from **multiple** reflectors and require **agreement** (§2.3); a single reflector is **advisory, not trusted**. No security decision rests on one observed address. |
| **Reflection op (§2)** | Amplification via the reflect op | Response is a single small address echo (no amplification factor); rate-limited; `network-reflect`-gated. |

**Deliberately NOT re-derived here (flagged for the build, not invented — §8):** the exact connectivity-check /
consent-freshness handshake (RFC 8445 §7 ICE connectivity checks + RFC 7675 consent freshness) and symmetric-NAT
port prediction. These are detailed, well-studied, and security-sensitive; mine the RFCs at build time rather than
inventing. They gate the *punch* (unit #2), not these facts.

## 7. Conformance & v1 posture

- **v1 floor unchanged; these additions are roadmap, not v1-blocking.** A NAT'd peer already *receives* via
  store-and-forward (RELAY Mode S) and talks to public peers via its held outbound socket (§1). This proposal adds
  facts that *improve* dispatch and *enable* the later punch; nothing here is required for the current release
  shape (content sites, CDN, offline delivery).
- **No wire-format change to the entity model.** Everything new is a transport-layer fact (`observed_address`,
  candidate types) or an ordinary op (`observe-address`, `check-reachability`). No V7 bump, no new error code
  beyond capability-denied (403) reuse.
- **The cross-impl-observable surface to pin (MUST), per "pin what diverges across a peer boundary":**
  1. **Dial-back target = observed source only** (§3.1) — divergence is a security hole. **MUST.**
  2. **`observed_address`, when present, is the transport source, never a body echo** (§2.1) — divergence is the
     same amplification seam on the reflection side. **MUST.**
  3. **Candidates MUST NOT be persisted as durable transport profiles** (§4.1) — divergence is a stale-routing
     cross-impl bug. **MUST.**
  Everything else (which mechanism a peer offers — field vs op; how many reflectors it consults; its rate-limit
  constants; its idle re-check cadence) is **local and MAY diverge** without a cross-peer effect — left to
  converge, not pinned.
- **The CDN-corridor meta-rule applies:** none of these facts is *validated* until a cross-impl conformance run
  exercises them — two conformant peers where one reflects the other's real observed address, and a dial-back that
  correctly reports reachable/not across a real NAT. That run is the gate; prose review is not. The natural
  carrier is the same cohort run that gates unit #2's first punch.

## 8. What this proposal deliberately defers (and to where)

| Deferred item | Goes to | Why not here |
|---|---|---|
| The **punch-coordination protocol** (`connect-request`/`-response`/`punch-sync`, the DCUtR dance) | `PROPOSAL-CONNECTIVITY-SIGNALING-AND-PUNCH` (unit #2) | It *exchanges and acts on* these facts; a protocol, not a fact. |
| The **signaling carrier set** ({relay-forward, rendezvous-WS, out-of-band QR}, Mode-S disqualified) | unit #2 — **open operator call G2** | Genuinely open; wants an operator decision (exploration Part D/G2). Does **not** gate the facts. |
| The **native substrate build order** (TCP-simopen → our own QUIC) | unit #2 — **operator call G1** (build-order already leaned in the handoff) | Substrate choice is additive; the facts are substrate-independent. |
| **Browser / WebRTC** reflection + transport profile | `PROPOSAL-EXTENSION-WEBRTC-TRANSPORT` (unit #3) | Browser uses standard STUN + its own ICE (§2.2); a distinct leg. |
| **RELAY Mode C** live relayed circuit (last-resort live fallback) | RELAY follow-on | Gated on a real-time driver (live chat/call between two NAT'd peers). |
| **Consent-freshness handshake + symmetric-NAT port prediction** | research (RFC 8445 §7, RFC 7675) at unit #2 build | Don't invent; read the references. |

**The point of the split:** this unit is decision-ready with **zero open operator dependency** — it can land now.
Unit #2 carries the two open calls (G1/G2), so it is authored *after* those are confirmed.

## 9. Open items (for this proposal specifically) — **all closed at fold, 2026-07-29**

- ~~**Observed-address shape** — MUST-offer or MAY?~~ **CLOSED as leaned.** NETWORK §12.3 makes §6.7 **MAY-offer
  as a whole**; §12.1 makes every rule inside it a **MUST when offered**. A requester meeting a peer that does
  not offer it proceeds to another reflector — the same path as an unreachable reflector, so nothing new to
  handle. The cohort sanity-check this bullet wanted is now the §6.7.5 conformance gate.
- ~~**Candidate `substrate` tag** — declare here or defer to unit #2?~~ **CLOSED as leaned:** the enum
  (`tcp`/`quic`/`webrtc`) is declared in `system/network/candidate` (NETWORK §6.7.3) and consumed in unit #2.
- ~~**Exact §6.7 home**~~ **CLOSED as leaned:** one cohesive §6.7 "Reachability Facts" after §6.6, split
  §6.7.1 reflection / §6.7.2 dial-back / §6.7.3 candidates / §6.7.4 capabilities / §6.7.5 §10 composition.

**Carried out of this proposal at fold (not open items here — they belong to their owners):**

- **Mechanism (a), the HELLO `observed_address` field** — routes to `entity-core-protocol` as its own proposal
  (§2.1 ruling). Gates nothing; NETWORK §6.7.1 is complete without it.
- **`system/connection.address` direction semantics** — ENTITY-CORE-PROTOCOL.md §3.13 does not say whether the
  field is dialer-side-only. Implementations read it as dialer-side and are right to (§2.1.1), but the spec is
  silent. **Route as a spec-ambiguity note upstream**, not a change request — NETWORK §6.7.1 no longer depends
  on the answer, because it forbids using the field either way.
- **The accept-side seam** carrying the observed source to the handler — per-impl layering, NETWORK §12.4
  implementation-defined. Real work, not free (§2.1.1); not a spec unknown.

## 10. References

- **Brought forward from (read-only):**
  `entity-lab-legacy-meta/entity-core-architecture/docs/architecture/v7.0-core-revision/proposals/PROPOSAL-NAT-TRAVERSAL-AND-WAN-REACHABILITY.md`
  §2 / §3.1–§3.3 / §5 / §8 / §9 (the "🎯 close-out #1: land the reachability facts first").
- **Reconciled under:** `docs/research/explorations/EXPLORATION-CONNECTIVITY-NAT-WEBRTC-UNIFIED-ARCHITECTURE.md`
  Part F step 1, Part H unit #1.
- **Current NETWORK surface amended:** `EXTENSION-NETWORK.md` §6.3 (HELLO handshake), §6.5 (transport profiles —
  the durable/ephemeral contrast, §4.1), §10 reachability-class table (L1296–1304 — `held_connection_client`,
  reserved `webrtc`), §5 (keepalive — relevant to the punched connection in unit #2, not here).
- **Complementary (advertises the reflector this proposal defines):** `PROPOSAL-REGISTRY-SERVICE-ADVERTISEMENT.md`
  §1 (`reflector` / `signaling` service set).
- **Reference systems (decomposition we follow, not depend on):** libp2p Identify / AutoNAT; WebRTC ICE
  (RFC 8445), STUN (RFC 5389), consent freshness (RFC 7675). `[K]` industry-typical, not spec-pinned.
