# PROPOSAL — APP-CONVENTION-CHAT — conversational messaging as an L5 format convention (the proof-point app)

**Status:** DRAFT (2026-07-22)
**Target:** `specs/applications/APP-CONVENTION-CHAT.md` (new; the third `applications/` member after EMBED + SITE)
+ a `CHARTER.md` Members-table row.
**Provenance:** the operator ask for an "entity-chat example that shows how it all works," designed in
`docs/research/explorations/EXPLORATION-FULL-STACK-DISCOVERY-INFRA-AND-ENTITY-CHAT.md` (the five-layer pipeline)
+ `EXPLORATION-CONNECTIVITY-NAT-WEBRTC-UNIFIED-ARCHITECTURE.md` (the Connect layer). Zero precedent — no chat app
or convention exists anywhere in the ecosystem (surveyed).
**Why this is the right proof point.** A single chat message traverses **all five stack layers** (discover →
resolve → connect → message → render), the **grant/IDENTIFY trust seam**, and **every infra role** — so building
it is the integration test that the whole coherent pattern actually coheres. It is buildable **today** on the
shipping layers (mDNS + registry + INBOX/CONTINUATION + EMBED); only live low-latency P2P is a later upgrade.
**Charter compliance:** defines **format only** (§3 vocabulary + CDDL), **invents no protocol** (delivery,
transport, crypto all deferred to substrate — §5/§8, `[ASK-ARCH]` where a gap exists — §11), has a **valid floor**
(a signed append-only log — §7), **ships conformance vectors** (§10), and is **encoding-agnostic** (self-describing
`content-hash`, never a fixed width — §3).

---

## 1. Anatomy of a conversation `[read first]`

A conversation is a **distributed, append-only, content-addressed message log** — and it maps directly onto the
core's local-view authority model (V7 §1.4), which is what makes it correct without inventing anything:

- **Each participant authors their own messages** into *their own* namespace and is the authority for them
  (`/{author}/app/chat/{conversation_id}/messages/{message_hash}`). No shared-write conversation object exists —
  so there is no consensus/write-contention problem.
- **Each participant caches the others' messages** via the delivery layer (INBOX/RELAY). A participant's *view* of
  the conversation is the **union** of every participant's message entities for that `conversation_id` — complete
  for their own, cached (possibly stale) for others. This is the §1.4 "local view" applied verbatim.
- **The conversation identity is content-addressed and immutable** (§2); membership *evolves* via an append-only
  signed op-log (§6) — the growth hook that makes 1:1 and group **one model**.

Three layers, like SITE's document/embed/compute anatomy:
1. **Identity layer** — the immutable `conversation` genesis + the per-participant signed head pin (§2).
2. **Content layer** — `message` entities (body = an EMBED node), discovered by lazy `.list` + subscription (§4).
3. **Membership layer** — the `membership-op` append-only log; the roster is its fold (§6).

## 2. Conversation identity & the signed head pin `[LOCKED shape]`

The **conversation genesis** is an immutable ECF entity; its `content-hash` **is** the stable `conversation_id`,
forever. Participants agree on a conversation by agreeing on this hash (the same discipline as SITE's `site-root`).

```cbor
conversation = {                             ; type = app/chat/conversation
  "creator":              <peer_id>,         ; V7 §1.5 Base58 peer-id
  "created_at":           uint,              ; ms since epoch (informational; not an ordering authority — §4.3)
  "policy":               "closed" / "invite" / "open",
  "initial_participants": [+ peer_id],       ; the genesis roster (1:1 = exactly 2)
  ? "title":              tstr,
  ? "purpose":            <app/embed node>   ; optional rich description, reusing EMBED
}
; content_hash(conversation) == conversation_id  (stable, immutable)
```

The **head pin** is each participant's signed pointer, in *their own* namespace, to the latest message they have
authored in the conversation — the sync anchor a catching-up peer reads first:

```cbor
head-pin = {                                 ; type = app/chat/head  (at /{peer}/app/chat/{conversation_id}/head)
  "conversation_id": <content-hash>,
  "latest":          <content-hash>,         ; the participant's most recent message entity
  "updated_at":      uint
}
```
The head pin MUST carry a `system/signature` at the invariant-pointer path (V7 §3.5). A conversation with no head
pins is still valid — a reader falls back to a full `.list` (§4); the pin is an optimization, per the valid-floor
discipline.

## 3. The entity vocabulary & CDDL `[LOCKED shape — the cross-impl contract]`

All hashes are the **self-describing** `content-hash` `(format_code, digest)` (V7 §1.2) — **no fixed width**
(CHARTER discipline 6; the EMBED `hex33` mistake is not repeated).

```cbor
message = {                                  ; type = app/chat/message
  "conversation_id": <content-hash>,         ; binds the message to its conversation
  "author":          <peer_id>,              ; MUST equal the namespace it is authored under
  "body":            <app/embed node>,       ; rich content — the whole point of the L5 format layer
  "sent_at":         uint,                   ; author-clock ms since epoch (best-effort ordering only — §4.3)
  ? "reply_to":      <content-hash>,         ; causal link to a prior message (any author)
  ? "attachments":   [* content-hash],       ; content-addressed blobs, deduped by the store
  ? "prev":          <content-hash>          ; this author's previous message (per-author causal chain)
}
; signed by `author`; signature at system/signature/{hex(content_hash(message))}

membership-op = {                            ; type = app/chat/membership-op  (§6)
  "conversation_id": <content-hash>,
  "op":              "add" / "remove",
  "subject":         <peer_id>,              ; who is added/removed
  "actor":           <peer_id>,              ; who performed it (MUST be authorized per policy — §6)
  "at":              uint,
  ? "prev":          <content-hash>          ; the actor's previous membership-op (per-actor chain)
}
; signed by `actor`

receipt = {                                  ; type = app/chat/receipt  (OPTIONAL tier)
  "message_ref":     <content-hash>,
  "by":              <peer_id>,
  "kind":            "delivered" / "read",
  "at":              uint
}
; signed by `by`; correlates to the CONTINUATION reply at the sender's reply path
```

**Dispatch keys / paths (convention, per V7 §3.5 "paths are convention"):**
`/{author}/app/chat/{conversation_id}/messages/{message_hash}`,
`/{actor}/app/chat/{conversation_id}/membership/{op_hash}`,
`/{peer}/app/chat/{conversation_id}/head`.

## 4. Discovery, ordering & history `[the conversation entity holds no message-collection — same as SITE §4.2]`

- **History discovery is lazy `.list` + reactive subscription**, never a downloaded index (SITE §4.2 pattern).
  A participant, for each other participant P, `.list`s `/{P}/app/chat/{conversation_id}/messages/` and
  **subscribes** to that prefix (EXTENSION-SUBSCRIPTION) — new messages pop into the local view; the UI reacts.
  The conversation genesis (§2) deliberately carries **no message list** — messages are found by enumerating each
  participant's namespace, not by a central mutable collection (no write-contention, per §1).
- **§4.3 Ordering (the honest floor — ties to the CLOCK non-goal).** There is **no total order** across authors:
  `sent_at` is an *author-clock* wall-clock stamp, and peer clocks are **not synchronized** (`EXTENSION-CLOCK.md`
  makes sync a non-goal). The v1 ordering floor is: **(a)** per-author total order via the `prev` causal chain
  (an author's own messages are strictly ordered); **(b)** cross-author *causal* order where `reply_to` links
  exist; **(c)** `sent_at` as a **display heuristic only**, never a correctness authority. Renderers MUST NOT
  assume `sent_at` gives a consistent global order across authors. *(This is the message-layer instance of the
  W7 Knob-3 clock-skew reality; it is stated once here, generally.)* A stronger causal order (vector clocks per
  `EXTENSION-CLOCK`) is an optional tier, not the floor.

## 5. Delivery — a pointer, not a mechanism `[invents no protocol]`

entity-chat defines **no** delivery machinery. A message is delivered by the shipping stack: the author EXECUTEs
the `message` write toward each other participant with `deliver_to` (the author's reply path) + `deliver_token`
(EXTENSION-INBOX/CONTINUATION); NETWORK reachability-class dispatch pushes it live, or — when a participant is
offline/NAT'd — `dispatch_fallback` stores it at that participant's inbox-relay (RELAY Mode S, resolved via the
registry MX). The recipient polls/receives; a `receipt` may ride back on the CONTINUATION correlation. **All of
this is layer-4 substrate that already ships** — entity-chat only specifies the *entities*, never their transport.

## 6. Membership & the 1:1→group growth path `[the stable-growable core]`

**1:1 and group are one model.** A 1:1 conversation is `initial_participants: [a, b]` with an empty membership-op
log. A group is the same genesis plus an append-only `membership-op` log. The **current roster** = `fold(
initial_participants, membership-op log ordered per §4.3)`. This is why the v1 model **grows into the full feature
set without a rework**: adding groups, invites, and leaves is *filling the op-log*, not changing the entities.

**Authorization is by `policy`, and the authority substrate already exists — `[ASK-ARCH-CHAT-1]` is RESOLVED
(§11): compose `EXTENSION-ROLE` + `EXTENSION-GROUP`; the chat convention invents no authority of its own.** A
membership-op's `actor` is authorized iff it holds the conversation-scoped capability grant the `policy` requires;
those grants come from the existing extensions, not a convention-layer invention:

- **`closed`** — no post-genesis membership authority is granted (pure 1:1 or a fixed group). **No ROLE/GROUP
  needed** — ships immediately.
- **`invite`** — the `add` authority is a **conversation-scoped `EXTENSION-ROLE` grant** (`system/role/{conversation_id}/...`
  — an `admin`/`member` role is a named capability-grant bundle, ROLE §1). The `actor` MUST hold the role-derived
  capability for `add`. `remove` is self-only (leave — needs no grant) **or** admin (holds the remove capability).
  For a full group-identity conversation, **`EXTENSION-GROUP`** supplies the *governance* that mints those grants
  (founder-K-of-N / admin-set / all-members, GROUP §3.2) and the `system/group:add_member`/`remove_member`
  lifecycle — the chat roster is then a projection of `system/group/member` state, or the chat's own op-log with
  GROUP/ROLE supplying authority.
- **`open`** — a broad self-`add` grant (any authenticated peer holds the add-self capability).

A `membership-op` whose `actor` is not authorized under `policy` is **invalid and MUST be ignored** by every
conformant reader (fail-closed) — so a roster fold is deterministic given the same observed op set (the same
convergent-input discipline as revocation, V7 §5.10). The **offline-first floor is preserved**: the op-log is a
signed content-addressed log (§7); ROLE/GROUP supply the *authorization predicate*, not a live-handler dependency —
a conversation is valid and syncable with no group handler online (local-view authority). **The only remaining
work is a fold-time convention detail**, not a substrate gap: pin the exact `policy` → role-grant mapping (which
role name covers `add`/`remove`) in a conformance vector.

## 7. Tiers & the floor `[capability adds, never assumed]`

- **Floor (any peer):** a conversation is a **signed, content-addressed, append-only message log** in a tree
  namespace. With *zero* live connectivity it is readable, verifiable, and syncable via store-and-forward —
  **chat is offline-first by construction.** No live peer, no relay, no encryption is required to *have a valid
  conversation*.
- **+ Live delivery:** NETWORK direct / held-socket / (later) NAT-punch — upgrades store-and-forward to real-time.
- **+ Reactive UI:** SUBSCRIPTION — messages stream in.
- **+ Confidentiality:** §8.
- **+ Receipts / read-state:** the optional `receipt` tier.
Each tier **adds**; a peer with fewer tiers is a valid participant, not a degraded one.

## 8. Confidentiality & access `[orthogonal — do not conflate]`

**Access control and confidentiality are separate axes.** entity-chat gates *access* and defers *confidentiality*:

- **Access is gated by connectivity + trust**, not encryption: the DISCOVERY grant-prompt + IDENTIFY admission
  decides *who you talk to*; the capability model + `policy` (§6) decide *who may participate/read*. A peer that
  is not admitted and not authorized never receives the messages.
- **Confidentiality is a three-rung ladder, first two available today:**
  1. **Signed, not encrypted** — the **v1 floor** (author signature + content-addressing = authenticity +
     integrity; relay operators can see content, as with any store-and-forward).
  2. **E2E-encrypted via *base* `EXTENSION-ENCRYPTION`** (v1 peer-mode, *available now* — the `message` body is
     encrypted end-to-end; "untrusted intermediaries can't read content"). **No session extension required.**
  3. **Forward-secret via `EXTENSION-ENCRYPTED-SESSION`** (planned; Signal/MLS) — *greater* security (PFS/ratchet),
     an upgrade, **not** a prerequisite.

entity-chat's entities are **encryption-agnostic**: an encrypted `message` is the base-ENCRYPTION envelope
carrying the `app/chat/message` as plaintext; the convention is unchanged whether rung 1, 2, or 3 is in use.

## 9. Security `[gates named with owners — never deferred]`

- **Author-spoofing:** `message.author` MUST equal the namespace it is authored under AND match the signature's
  key (V7 §5.2 target-matching). *(owner: convention + core verify.)*
- **Membership forgery:** a `membership-op` with an unauthorized `actor` is fail-closed-ignored (§6). *(owner:
  the resolved authority model — the `actor` MUST hold the conversation-scoped `EXTENSION-ROLE`/`EXTENSION-GROUP`
  grant `policy` requires; §6, §11 `[ASK-ARCH-CHAT-1]` resolved.)*
- **Replay:** a relayed `message` is replayable within its delivery window — the same non-interactive-freshness
  reality as W7; mitigated by content-addressing (a replayed message is the *same* entity, deduped) + handler
  idempotency. entity-chat needs **no** anti-replay of its own (content-addressing makes a chat message naturally
  idempotent). *(owner: W7 / the substrate; noted, not reinvented.)*
- **Metadata leakage:** with rung-1/2 confidentiality, a relay sees participant peer-ids + timing. Rung-3 and
  relay-opacity narrow it. *(owner: ENCRYPTION family; stated honestly.)*
- **DoS via history:** a reader MUST bound `.list` fan-out (per-namespace page limits) — the SITE §4.1 navigation-
  safety discipline applies. *(owner: convention renderer contract.)*

## 10. Conformance vectors `[REQUIRED before ratification]`

Ships example entities + expected `content-hash`es (the CHARTER discipline-5 gate; the PRIMER meta-rule — not
validated until a cross-impl run exercises it): a genesis `conversation`; a signed `message` with an EMBED body;
a `reply_to` causal pair; a `membership-op` add + the resulting roster fold; an out-of-order-`sent_at` pair
asserting the §4.3 ordering floor (per-author `prev` order holds; cross-author `sent_at` does not); a rung-2
base-ENCRYPTION-wrapped `message` decrypting to the plaintext vector. **The best integration test the whole stack
has:** a real chat exchange between two conformant peers (discover → resolve → connect → message → render).

## 11. Open / deferred + `[ASK-ARCH]`

- **`[ASK-ARCH-CHAT-1]` — group membership authorization. ✅ RESOLVED (2026-07-22): the substrate exists; no new
  primitive.** The `invite`/admin authority model (§6) composes the **already-Active** `EXTENSION-ROLE`
  (conversation-scoped role = named capability-grant bundle at `system/role/{conversation_id}/...`) + `EXTENSION-GROUP`
  (governance that mints the grants — founder-K-of-N / admin-set / all-members, GROUP §3.2 — and the
  `add_member`/`remove_member` lifecycle). A membership-op is authorized iff its `actor` holds the required
  role-derived capability; fail-closed otherwise (§6). The offline-first log floor is preserved — ROLE/GROUP supply
  the *authorization predicate*, not a live-handler dependency. 1:1 + `closed`/`open` need neither and ship first.
  **What remains is a fold-time convention detail** (pin the exact `policy` → role-grant mapping in a conformance
  vector), not a substrate dependency — so this ASK no longer blocks group chat.
- **`[ASK-ARCH-CHAT-2]` — subscription fan-out at scale.** Subscribing to N participants' prefixes is fine for
  small groups; large groups want an aggregate (RELAY Mode A, deferred). Flag, don't solve.
- **Open/unsolicited delivery ("message anyone") — designed at the substrate (2026-07-24).** "Receive from a
  stranger" is not a chat mechanism; it is an **INBOX open-delivery admission mode** — a published, narrow-scoped
  `deliver_token` into a rate-limited requests quarantine — specified in `PROPOSAL-INBOX-OPEN-DELIVERY-ADMISSION`
  (the delivery-layer twin of the connection node's open admission mode). Chat consumes it via a `chat-requests`
  sub-inbox + accept-to-promote; the convention itself still invents no delivery machinery (§5).
- **Deferred tiers:** typing indicators, presence, edits/deletes (an append-only supersession op), threaded
  replies beyond `reply_to`, reactions. All additive `message`/op tiers; none changes the floor.
- **Forward secrecy** (rung 3) rides `EXTENSION-ENCRYPTED-SESSION` when it lands.

## 12. Provenance

- Design record: `docs/research/explorations/EXPLORATION-FULL-STACK-DISCOVERY-INFRA-AND-ENTITY-CHAT.md` (§C),
  `EXPLORATION-CONNECTIVITY-NAT-WEBRTC-UNIFIED-ARCHITECTURE.md`.
- Template + discipline: `specs/applications/CHARTER.md`, `specs/applications/APP-CONVENTION-SEMANTIC-CONTENT-SITE.md`.
- Substrate deferred to: EXTENSION-INBOX/CONTINUATION/RELAY (delivery), EXTENSION-NETWORK (transport),
  EXTENSION-SUBSCRIPTION (reactive history), APP-CONVENTION-EMBED (message body), EXTENSION-ENCRYPTION /
  -ENCRYPTED-SESSION (confidentiality ladder), EXTENSION-ROLE + EXTENSION-GROUP (membership authorization —
  `[ASK-ARCH-CHAT-1]` resolved, §11).
- Operator decisions folded (2026-07-22): group-capable stable-growable core (1:1 = degenerate group); signed
  floor + optional base-ENCRYPTION + SESSION-later, access gated by connectivity/trust.
