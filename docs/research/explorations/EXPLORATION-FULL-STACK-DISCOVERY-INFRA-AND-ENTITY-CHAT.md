# EXPLORATION — the coherent full stack: discovery → managed infrastructure → entity-chat (the proof point)

**Status:** Exploration / architecture synthesis — 2026-07-22. **NOT** a proposal (it maps the coherent pattern
and names the ratifiable units — Part F). Companion to `EXPLORATION-CONNECTIVITY-NAT-WEBRTC-UNIFIED-ARCHITECTURE.md`
(the transport layer beneath this). Author: arch workspace. **Goal (operator steer):** *discovery mechanism + an
L5 "entity-chat" example that shows how it all works + managed infrastructure connected with the registry — pulled
into one coherent pattern that works with the system.*

**The finding in one line.** The system is **one five-layer pipeline** — *discover → resolve → connect → message
→ render* — of which **four layers already ship** (discover/mDNS, resolve/registry, message/INBOX+CONTINUATION,
render/EMBED) and only the *connect* layer's hard case (NAT/WebRTC) is deferred; the **"managed infrastructure"**
is not a special tier but **ordinary peers running handlers, tied together by the registry serving A+MX(+services)
in one signed zone**, seeded as a *coral reef* the distribution ships; and **entity-chat** is the missing
vertical slice that exercises every layer at once — the third `APP-CONVENTION`, net-new, and the proof point.

**Provenance (read-only; four surveys this session + the connectivity exploration).** `EXTENSION-DISCOVERY.md`
(v1.0 Active), `EXTENSION-REGISTRY.md` (v1.2 Active, "v1 not finished"), `EXTENSION-RELAY.md` (§3.5 MX, §12
configs), `EXTENSION-NETWORK.md` (§10 reachability), `EXTENSION-INBOX/CONTINUATION.md`, `applications/CHARTER.md`
+ `APP-CONVENTION-{EMBED,SEMANTIC-CONTENT-SITE}.md`, `guides/GUIDE-{NETWORKING-MODEL,CROSS-PEER-MESSAGING,PEER-COMPOSITIONS}.md`.
Claims pinned `(§/line, file)`; line numbers current-state.

---

## Part A — The five-layer pipeline (what actually happens when two peers interact)

Every peer-to-peer interaction — including a chat message — is the same pipeline. Naming it is the coherence:

| # | Layer | Question it answers | Mechanism | Status |
|---|-------|---------------------|-----------|--------|
| 1 | **Discover** | "what peers exist that I don't know?" | DISCOVERY mDNS (`_entity-core._udp.local.`); QR / registry-assisted / DHT (deferred) | **mDNS ships (native)**; browser can't mDNS (physics) → uses layer 2 |
| 2 | **Resolve** | "given a name I know, where is this peer?" | REGISTRY `resolve(name) → (peer_id, endpoints, MX, ttl)` | **Ships v1** (local-name + peer-issued, 3-way green) |
| 3 | **Connect** | "open a channel to this peer" | NETWORK reachability-class dispatch (tcp/ws/http/held-socket); **NAT-punch/WebRTC/Mode-C deferred** | direct + public↔NAT'd **ship**; symmetric-NAT/browser-P2P **deferred** (connectivity exploration) |
| 4 | **Message** | "send it something; get a reply" | INBOX `deliver_to`/`deliver_token` + CONTINUATION reply-correlation; RELAY Mode-S store-and-forward when offline | **Ships** (transport-independent reply correlation) |
| 5 | **Render** | "show it to a human" | L5 `APP-CONVENTION-EMBED` typed nodes | **Ships (Draft, 3-way locked)** |

**The load-bearing observation:** the *only* not-shipping layer is **Connect's hard case**. Discovery (local),
resolution, messaging, and rendering are all Active. So a working distributed app is *buildable today* on the
async floor (store-and-forward), and gets *live/low-latency* when Connect's NAT/WebRTC rung lands. This is why
entity-chat can be built **now** and *demonstrate the whole stack* — its live-P2P polish is the one thing gated.

**The trust seam that spans layers 1–3:** DISCOVERY never silently connects (`EXTENSION-DISCOVERY.md` §1 —
"Discovery never silently connects you to strangers"). Admission is the **grant-prompt flow + IDENTIFY
handshake** (DISCOVERY §2/§5.3), fail-closed. Whether a peer arrives via mDNS, registry, or QR, it is *admitted*
by the same human-in-the-loop grant + cryptographic IDENTIFY. One trust seam, every discovery path.

---

## Part B — The managed-infrastructure pattern (no special tier; the registry is the tie)

**The governing principle (elegant, and already in the spec):** there is **no "infrastructure" role** — infra is
ordinary peers running a system handler. `EXTENSION-REGISTRY.md` §1: *"Registry is just a peer… No special
infrastructure role."* `EXTENSION-RELAY.md`: *a static-CDN-hosted peer "IS Mode S relay… same wire shape."* So
"managed infrastructure" = **a curated set of always-on peers, each running one handler, advertised through the
registry.**

**The infra role set (each = a peer running one handler):**

| Role | Handler | Ships? | What it provides |
|------|---------|--------|------------------|
| **Registry / directory** | `system/registry` | **v1** | `name → (peer_id, endpoints, MX, ttl)`; the coral-reef seed |
| **Relay — inbox (MX)** | `system/relay` Mode S | **v1** | store-and-forward mailbox for offline/NAT'd peers |
| **Relay — forward** | `system/relay` Mode F | **v1** | live message forwarding |
| **Relay — circuit (TURN)** | `system/relay` Mode C | deferred | live relayed data path for NAT'd↔NAT'd |
| **Reflector (STUN)** | `system/network:observe-address` | proposed | tells a peer its public address |
| **Rendezvous / signaling** | signaling handler | proposed | carries the punch/SDP offer between peers |
| **Static host / CDN** | storage-substitute-http | in-flight | hosts a peer's tree as static bytes |

**The tie — the registry serves everything in one signed zone.** The MX pattern already does this:
`EXTENSION-RELAY.md` §3.5 — a peer's `system/peer/inbox-relay` (its fallback relays, MX-priority-ordered) is
**"served by the registry… the A+MX-in-one-zone pattern"** (resolution returns the name→peer binding *and* the
MX declaration together). **The coherent generalization (the net-new piece):** extend the zone to advertise the
*other* infra services too — the reflector, the rendezvous, the relay-set a deployment offers — so a fresh peer
that resolves the well-known registry name learns, in one fetch, **who to reach + how to reach them + what infra
is available**. Call it **A+MX+SRV-in-one-zone** (the SRV-analog for infra services). This is what "connected
with the registry to pull it into a coherent pattern" concretely means.

**The seed — a coral reef the distribution ships.** `EXTENSION-REGISTRY.md` §7.4: the release preloads a **pinned
registry identity** (trust root, "like a root CA shipped with an OS") + **resolver-config** (with the registry's
static `http-poll` endpoint) + **precedes** (pre-cached signed bindings — the deployed **"CDN trio"**: Bill's
Lab, entitycoreprotocol.org, entitychurchfoundation.org). Its defining property: *"No live registry peer is
required — the registry is itself a coral reef (a static publisher in dormancy)."* A fresh peer resolves the
well-known name over static http-poll, verifies against the pinned identity, and is connected — **zero live
infrastructure required for the async floor.** Live infra (punch reflector, rendezvous, Mode-C circuit) is the
*upgrade*, not the *floor*.

**The gaps (what makes this "partly designed"):** (1) there is **no single reference-deployment doc** — the roles
are specced one-per-extension in scattered `§Configurations` sections + `GUIDE-PEER-COMPOSITIONS` (deployment
recipes, not one runbook); (2) the **A+MX+SRV service-advertisement** beyond MX is unspecified; (3) the
reflector/rendezvous handlers themselves are proposed-not-specced (connectivity exploration).

---

## Part C — entity-chat: the worked vertical slice (the proof point)

**No chat/messaging app or `APP-CONVENTION-CHAT` exists anywhere** (confirmed: zero hits across all repos +
archive). entity-chat is **net-new**, and it is the ideal demonstrator because a single chat message **traverses
all five layers + the trust seam + touches every infra role** — exactly "show how it all works."

### C.1 — The convention (following the CHARTER + SITE template)

Per `applications/CHARTER.md`, an `APP-CONVENTION-*` **defines FORMAT and only format** (the `{type,data}`
vocabulary + CDDL + dispatch keys + on-wire bytes), **invents no protocol** (gaps become `[ASK-ARCH]`), has a
**valid floor**, and **ships conformance vectors**. entity-chat, mirroring SITE's `manifest + pages + signed
root-pin + lazy .list` shape:

- **`app/chat/conversation`** — the conversation identity (SITE-manifest analog): `participants: [peer_id]`,
  `created_at`, optional `title`, `policy` (open/invite). **Signed-root-pinned** like SITE's `site-root` — the
  conversation is a signed pointer its participants agree on.
- **`app/chat/message`** — `conversation` (ref), `author` (peer_id), **`body` = an `app/embed` node** (reuse
  EMBED for rich content — the whole point of the L5 format layer), `sent_at`, optional `reply_to` (ref),
  optional `attachments` (content hashes). Content-addressed; **signed by `author`**.
- **`app/chat/receipt`** (optional) — delivery/read markers, keyed to the CONTINUATION reply.
- **Discovery of history:** **lazy one-level `.list`** over the conversation namespace (SITE's §4.2 pattern —
  no downloaded index), **reactive via SUBSCRIPTION** (new messages pop into the tree; the UI subscribes).

- **The valid floor (CHARTER discipline 4):** with *zero* live connectivity, a conversation degrades to a
  **signed, content-addressed, append-only message log in a tree namespace** — readable, verifiable, and
  syncable via store-and-forward. Chat *works offline-first by construction*; live delivery is the upgrade.

- **What it defers (invents no protocol):** delivery → INBOX/CONTINUATION + RELAY (layer 4, ships); transport →
  NETWORK/connectivity (layer 3); crypto → the **ENCRYPTION** family. **The confidentiality ladder is three
  rungs, and the first two ship or exist today:** (1) **signed, not encrypted** — the v1 floor (author signature
  + content-addressing = authenticity + integrity); (2) **E2E-encrypted via *base* `EXTENSION-ENCRYPTION`**
  (v1 peer-mode, *available now* — "end-to-end through relays; untrusted intermediaries can't read content"; no
  session extension required); (3) **forward-secret via the planned `EXTENSION-ENCRYPTED-SESSION`** (Signal/MLS,
  §35) — *greater* security, not a prerequisite. **Access is gated by connectivity + trust** (the grant/IDENTIFY
  admission + the capability model), *not* by encryption — confidentiality and access-control are orthogonal.
  entity-chat *names* these as `[ASK-ARCH]`-style dependencies, never reimplements them.

### C.2 — How a message flows through every layer (the demonstration)

Alice sends Bob a message — this is the whole system in one trace:

1. **Discover / Resolve (layer 1–2).** LAN: Alice `:scan`s mDNS, Bob's candidate appears. Wide-area: Alice
   `resolve("bob.entitychurch.org")` at the registry → Bob's `peer_id` + transports + **MX** (his inbox-relays),
   verified against the pinned registry identity.
2. **Admit (trust seam).** Grant-prompt + IDENTIFY handshake over the channel — Alice admits Bob (or already
   has). Never silent.
3. **Connect (layer 3).** NETWORK dispatches to Bob's advertised transport: direct (tcp/ws) if reachable;
   public↔NAT'd via Bob's held socket; **(when landed)** NAT-punch/WebRTC via the reflector+rendezvous+Mode-C
   infra.
4. **Message (layer 4).** Alice writes an `app/chat/message` (body = EMBED node), signs it, and EXECUTEs it to
   Bob's conversation namespace with `deliver_to` (her reply path) + `deliver_token`. **If Bob is offline/NAT'd
   → `dispatch_fallback` → RELAY Mode-S** stores at Bob's MX inbox (resolved via the registry in step 1). Bob
   polls/receives on reconnect; INBOX `receive` fires; a CONTINUATION at Alice's reply path correlates the
   delivery receipt — **transport-independent** (`EXTENSION-RELAY.md` L433: "does not depend on how the result
   arrived").
5. **Render (layer 5).** Bob's front-end (web/Tauri/Godot/terminal) renders the message's EMBED body — the same
   bytes render the same everywhere (the L5 format contract).

**Infra touched in one message:** registry (resolve Bob + MX), relay (store-and-forward if offline),
reflector+rendezvous (only if a live NAT punch is needed), discovery (only on LAN). **entity-chat is the test
that the coherent pattern actually coheres.**

---

## Part D — Build/gap map (per layer, shippable-now vs gated)

| Layer / piece | Ships now | Gated / net-new |
|---|---|---|
| Discover — mDNS LAN | ✅ native | browser (physics → uses registry); QR / registry-assisted / DHT (deferred) |
| Resolve — registry | ✅ v1 (curated + peer-issued) | live self-registration (in-flight); did-web/dns-txt/dht backends |
| Connect — direct + public↔NAT'd | ✅ v1 | NAT-punch / WebRTC / Mode-C (connectivity exploration) |
| Message — INBOX/CONTINUATION/Mode-S | ✅ v1 | — |
| Render — EMBED | ✅ Draft, locked 3-way | conformance vectors → ratify |
| **Infra tie — A+MX via registry** | ✅ v1 (MX) | **A+MX+SRV service advertisement (net-new)** |
| **Reference deployment doc** | — | **net-new (no single runbook)** |
| **entity-chat convention** | — | **net-new (this exploration → APP-CONVENTION spec)** |
| Chat confidentiality | signed floor ✅; **E2E via base ENCRYPTION peer-mode ✅ (v1, available)** | forward secrecy → `EXTENSION-ENCRYPTED-SESSION` (planned, *greater* security not a gate) |

**The honest headline:** you can **build entity-chat end-to-end today** on the async floor + LAN mDNS + registry
resolution + EMBED rendering. What's gated is *live low-latency P2P* (Connect) and *E2E encryption* — both
additive upgrades the convention's valid-floor discipline already accommodates.

---

## Part E — The coherent build sequence

1. **entity-chat `APP-CONVENTION` spec** (Part C) — the demonstrator; buildable now on shipping layers; the
   forcing function that proves the pattern coheres. **Highest-leverage: it's the proof point and it exercises
   every other piece.**
2. **A+MX+SRV registry service-advertisement** — the net-new infra tie (Part B): extend the registry zone to
   advertise reflector/rendezvous/relay-set alongside A+MX. Small REGISTRY amendment; makes "managed infra
   connected with the registry" real.
3. **Reference-deployment guide** — `GUIDE-REFERENCE-DEPLOYMENT` (or similar): assemble the infra roles (Part B
   table) into one operator runbook anchored on the coral-reef registry + CDN trio. Consolidation, not invention.
4. **Discovery consolidation + the connectivity track** — the mDNS→registry→QR→(future) backends story as one
   discovery narrative; then the connectivity exploration's reachability-facts wedge feeds the *live* Connect
   layer that upgrades entity-chat from store-and-forward to live P2P.

**Doc hygiene to fix in passing (survey-flagged drift):** `SYSTEM-ARCHITECTURE.md` §13.1 still lists Discovery +
Relay as "spec gap / not authored" (both are now **Active**); `EXTENSION-NETWORK.md:52` overstates "seed peers —
DISCOVERY, shipped" (DISCOVERY ships **only** mDNS; seed/bootstrap lives in REGISTRY §7 + `system/config/bootstrap`).
Correct both — wording-only hygiene.

---

## Part F — Recommendation & next units

**entity-chat first** — it is the proof point the operator asked for, it is buildable on today's shipping layers,
and authoring its convention forces every other piece to cohere (if a chat message can't cleanly traverse the
five layers, the pattern isn't right — that's the test). Recommended units, in order:

1. **`PROPOSAL-APP-CONVENTION-CHAT`** (or `specs/applications/APP-CONVENTION-CHAT.md` Draft) — the entity
   vocabulary + CDDL + `.list`/subscription discovery + the valid floor + conformance vectors (Part C).
2. **`PROPOSAL-REGISTRY-SERVICE-ADVERTISEMENT`** — the A+MX+SRV one-zone infra advertisement (Part B / E.2).
3. **`GUIDE-REFERENCE-DEPLOYMENT`** — the managed-infrastructure runbook (Part E.3).
4. Feed the connectivity exploration's reachability-facts wedge for the live-Connect upgrade.

Before authoring #1 in full, two small confirmations worth an operator call: the **conversation model**
(1:1 vs group-from-day-one — affects the participant/policy shape) and whether **v1 floor is signed-not-encrypted**
(recommended: yes, with `EXTENSION-ENCRYPTED-SESSION` composing later). Per the CDN-corridor meta-rule, none of
this is validated until a cross-impl run exercises an actual chat exchange between two conformant peers — which is
*also* the best integration test the whole stack has.

## References

- Companion: `EXPLORATION-CONNECTIVITY-NAT-WEBRTC-UNIFIED-ARCHITECTURE.md` (the Connect layer).
- Specs: `EXTENSION-DISCOVERY.md` §1/§2/§3/§5, `EXTENSION-REGISTRY.md` §1/§3.5(via RELAY)/§6a/§7.4,
  `EXTENSION-RELAY.md` §3.5/§12, `EXTENSION-NETWORK.md` §10, `EXTENSION-INBOX/CONTINUATION.md`,
  `applications/CHARTER.md`, `APP-CONVENTION-{EMBED,SEMANTIC-CONTENT-SITE}.md`, `EXTENSION-ENCRYPTION.md` §35.
- Guides: `GUIDE-NETWORKING-MODEL.md`, `GUIDE-CROSS-PEER-MESSAGING.md` §6, `GUIDE-PEER-COMPOSITIONS.md`.
- Surveys (this session, 2026-07-22): discovery+registry mechanism; infra-roles + chat-precedent (none);
  connectivity state; L5/compute dependency.
