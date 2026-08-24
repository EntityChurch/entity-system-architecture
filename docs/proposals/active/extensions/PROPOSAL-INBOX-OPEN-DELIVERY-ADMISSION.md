# PROPOSAL — INBOX open-delivery admission — the "message-me" capability, done inside the capability model

**Status:** DRAFT (2026-07-24)
**Target:** an amendment to `specs/extensions/EXTENSION-INBOX.md` (a new open-admission mode + §2.3 companion) +
a one-line consumer note in `PROPOSAL-APP-CONVENTION-CHAT` §5. No wire change.
**Provenance:** the operator's "chat with anyone, anywhere" steer; the gap isolated in
`HANDOFF-2026-07-23-basic-chat-scope-isolation` §"THE ONE REAL GAP — open/unsolicited delivery." Resolves that
gap. Unifies with the connectivity node's two admission modes (`PROPOSAL-CONNECTIVITY-SIGNALING-AND-PUNCH` §2.3).
**Scope:** how a peer lets **strangers** deliver it a message (a chat message, an open-feed follow, …) **without
pre-arranging a capability with each one** — while keeping the protocol's "capability-gated end to end" invariant
intact. This is a **delivery-layer** admission concern (INBOX), not a chat concern; chat is its first consumer.

---

## 1. The gap (precise)

The protocol is capability-gated end to end: to `receive`-deliver into peer B's inbox, the sender's EXECUTE must
carry a `deliver_token` — the hash of a `system/capability/token` authorizing delivery to that inbox URI — or B
rejects with `400 missing_deliver_token` (`EXTENSION-INBOX` §2.3). So today B can only be messaged by a peer B has
**already** handed a token to. "Message me, whoever you are" has no path.

**The wrong fix** is a capability-less "open handler" that bypasses the gate — it puts a hole in the one invariant
the whole model rests on, and every other layer would have to reason about the exception.

## 2. The fix — a deliberately open, deliberately narrow `deliver_token`

Keep the gate; **change what the token authorizes and who may hold it.** The "message-me capability" is a real
`system/capability/token`, published openly, scoped as narrowly as possible:

- **Narrow scope (the safety):** it authorizes **only** `receive` of **one message type** (`app/chat/message`, or
  a `chat-request` wrapper — §4) to **one path**, B's **requests sub-inbox** (`system/inbox/chat-requests`, §4).
  Not arbitrary EXECUTE, not the main inbox, not any other handler. "Open" is tightly bounded to "a stranger may
  drop one chat request in the quarantine tray."
- **Open audience (the reach):** the token is **published with B's contact** — B's shareable identity becomes
  `peer_id + transports + open-chat token` (it rides the registry binding, `EXTENSION-REGISTRY`). Anyone who
  resolves B obtains it. Resolving B *is* getting permission to message B.

Because the token is public, **holding it proves nothing about the holder** — exactly the connectivity §2.2 rule
("the key introduces, it does not authorize"). So the token cannot be the abuse control; §3 is.

**This is the delivery-layer instance of the two admission modes** the connection node already defines (§2.3):

| Mode | Connectivity node (§2.3) | INBOX delivery (this proposal) |
|---|---|---|
| **Open** | reflect/offer/collect, **rate-limit only** | `receive` a chat-request, **rate-limit + quarantine**, via the published narrow token |
| **Capability-gated** | same surface + one capability check | the ordinary per-sender `deliver_token` (accepted contacts, friends-only) |

Same shape, same interface, admission differs — one coherent pattern across the connect layer and the deliver
layer. A deployment picks per surface: open message-me by default, or friends-only by requiring a per-sender token.

## 3. Abuse control (what makes "open" safe — the v0 minimum)

Because the token is bearer/public, admission for the open mode is **not** the token; it is, at B's inbox handler:

1. **Authenticated sender, always.** Open ≠ anonymous. The delivering EXECUTE is a signed envelope; `message.author`
   MUST equal the authoring namespace and match the signature (V7 §5.2 target-matching). So a sender has a stable,
   unspoofable id to rate-limit and block. Identity is required; a *pre-shared grant* is not.
2. **Rate-limit per sender-id** (per authenticated peer, per window) + a **global open-intake ceiling** — the same
   resource-hygiene the connection node applies. Over-limit → `429`, dropped, not queued.
3. **Block-list.** A per-peer deny that B controls; a blocked sender's requests are refused before storage.
4. **Quarantine by construction (§4).** An unknown sender's message lands in the **requests sub-inbox**, never the
   main conversation — so "open" cannot flood the timeline; it fills a tray B chooses to look at.

No global anti-spam, no proof-of-work, no reputation system in v0 — those are the deferred open-overlay economics
(`EXPLORATION-P2P-CONNECTIVITY-AND-OVERLAY` §7), not the first release.

## 4. The requests sub-inbox + accept-to-promote (the chat consumer)

Two inbox paths, one accept action:

- **`system/inbox/chat-requests`** — the open, rate-limited, quarantined tray. The published token authorizes only
  this path. Messages here are *pending*: shown as "message requests," not in the conversation, until accepted.
- **`system/inbox/chat`** (or the accepted-conversation path) — the ordinary gated inbox; only accepted senders
  reach it.
- **Accept = promote the sender.** When B accepts a request, B (a) admits the sender to the conversation and (b)
  issues that sender an ordinary per-sender `deliver_token` for the main path (or adds them to an accept-list). A
  reply implicitly accepts. **Ignore** leaves it in the tray; **block** adds the deny (§3.3). This is both the
  abuse control *and* the "who do I talk to" UX — one mechanism.

Accepting is the moment the relationship graduates from open (bearer, quarantined) to gated (per-sender,
revocable) — the same open→gated spectrum, walked per contact.

## 5. The two mechanisms for the open token (one call to confirm)

How B produces the published token — pick one; both keep the gate:

- **(a) Bearer open token (recommended for v0).** One narrow token (§2), published with the contact, held by
  anyone. Simplest; zero per-sender state. Revocation is coarse (rotate the token → old contact links stop
  working) — acceptable because the token only reaches the quarantine tray, and per-sender revocability arrives on
  accept (§4).
- **(b) Auto-mint on resolve.** B's resolve path mints a per-requester scoped token when a peer resolves B. More
  machinery, but per-sender revocation from first contact.

**The one open item to confirm against the capability spec:** whether a **bearer / open-audience**
`system/capability/token` (no bound grantee) is permitted, or whether every grant must name a grantee (forcing
(b)). If bearer grants are disallowed, (b) is the mechanism and the design is otherwise unchanged. *Recommend (a)
if the model allows it — the narrow scope + quarantine make a bearer grant safe here.* `[ASK-ARCH-CAP-1]`

## 6. Why this belongs at INBOX, not in chat

Chat's Charter says it **invents no delivery machinery** (`PROPOSAL-APP-CONVENTION-CHAT` §5, §8). Open delivery is
a **substrate** capability any "receive from strangers" app needs — chat now, open feeds / open follows next
(`EXPLORATION-P2P-CONNECTIVITY-AND-OVERLAY` §6 "open-follow discovery"). So it lands as an INBOX admission mode
that chat *consumes*, keeping chat a pure format convention. The chat-specific parts (the `chat-requests` path,
accept-to-promote) are a thin consumer convention on top.

## 7. Security

- **Author-spoofing** — prevented by V7 §5.2 target-matching (§3.1); the open mode still authenticates the sender.
- **Token leakage is a non-event** — the open token is *meant* to be public; it grants only quarantined chat-request
  delivery, nothing more. This is why the narrow scope is load-bearing (§2).
- **Amplification / flood** — bounded by the per-sender rate-limit + global intake ceiling + quarantine (§3); a
  flood fills a tray B can ignore or block, never the conversation or arbitrary handlers.
- **Metadata** — B's registry binding advertising "open-chat enabled" reveals that B accepts open messages; that
  is the intended semantics, not a leak.
- **Revocation** — coarse for bearer (rotate), per-sender on accept; rides the standard dispatch-layer
  `is_revoked` check (`EXTENSION-INBOX` §2.3 revocation note), no inbox-specific machinery.

## 8. Conformance & posture

- **No wire change.** Reuses `deliver_token` (§2.3) unchanged; adds an admission mode + the requests-path
  convention + rate-limit/quarantine behavior.
- **The cross-impl-observable surface to pin (MUST):** (1) the open token's **narrow scope** (type + path) is
  enforced — an open token MUST NOT authorize delivery beyond `chat-requests`; (2) an unknown-sender delivery lands
  in `chat-requests`, never the accepted path; (3) rate-limit/`429` on over-limit open intake. Accept/ignore/block
  UX is local.
- **Validation gate (CDN-corridor discipline):** not real until a cross-impl run exercises a **stranger round-trip**
  — peer A resolves B (no prior relationship), obtains the open token, delivers a chat request into B's quarantine,
  B accepts and replies, A receives. That round-trip is the v0 gate (basic-chat handoff §"minimal proof").

## 9. Open / deferred

- `[ASK-ARCH-CAP-1]` (§5) — bearer vs. bound grant for the open token. The one substrate confirm.
- Global anti-spam / proof-of-work / reputation — **deferred** to the open-overlay economics
  (`EXPLORATION-P2P-CONNECTIVITY-AND-OVERLAY` §7); not v0.
- Generalization to open feeds/follows — named, not specified here (same mode, different consumer path).

## 10. References

- `EXTENSION-INBOX` §2.3 (the `deliver_token` gate + revocation), §3.2 (write-ahead receive).
- `PROPOSAL-CONNECTIVITY-SIGNALING-AND-PUNCH` §2.2/§2.3 (the two admission modes + "introduces ≠ authorizes" —
  the pattern this mirrors).
- `PROPOSAL-APP-CONVENTION-CHAT` §5 (delivery-deferred-to-substrate), §7 (offline-first floor).
- `HANDOFF-2026-07-23-basic-chat-scope-isolation` (the gap + the minimal proof).
- `EXPLORATION-P2P-CONNECTIVITY-AND-OVERLAY` §6 (open-follow), §7 (deferred abuse economics).

*The one sentence: "message me, whoever you are" is a **published, narrow-scoped `deliver_token`** — the delivery-
layer twin of the connection node's open admission mode — that keeps the capability gate intact (the token
introduces, it does not authorize), lands strangers in a rate-limited quarantine tray, and graduates a contact
from open-bearer to gated-per-sender the moment you accept them.*
