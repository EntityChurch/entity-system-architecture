# GUIDE: Cross-Peer Messaging (Delivery, Durability, Tracking)

**Status**: Active

**Audience:** application developers and peer operators wiring cross-peer requests where the result may come back later, reliably, or through a peer too small to run extensions.

**What this guide is — and is not.** This is *one pattern*: cross-peer messaging with delivery and durability options. It is **not** the peer-compositions concept. "Peer compositions" is the broader structural idea — you define a composition by the *set of extensions you install* plus the *topology/network shape* you wire peers into, and that framework accommodates *any* coupling pattern, of which this is one. The composition concept, the substrate kernel properties (liveness / path-scoped dispatch / bounded cascade), the named-composition catalog and the no-reactive-cycle topology discipline live in `core-protocol-domain/explorations/EXPLORATION-PEER-COMPOSITIONS.md` (v2) and the (deferred) `PROPOSAL-PEER-COMPOSITIONS-AND-DELIVERY-CLASSES.md` thread, whose own guide deliverable is separate. This guide stays in its lane: the delivery/durability mechanics.

---

## 1. The problem

Synchronous request-response does not survive a slow, busy, or disconnected receiver, and a request you wanted preserved can vanish silently if the receiver does not handle it. Cross-peer messaging is the set of options that fix that: get the result back, get it back later, get it back reliably, get it through an embedded peer — and *be told what actually happened* instead of silence.

There are **two layers** to "send a message to another peer," and this guide covers both:
- **Reachability / addressing (§3A, the SMTP-shaped layer).** *How do you get the message to the target in the first place* when the target may be named-but-unknown, offline, or a stranger? This is resolve-by-name (REGISTRY) + find-where-to-store (the inbox-relay / MX-equivalent) + store-and-forward (RELAY Mode S). It is the email-delivery analog, validated against SMTP in the SMTP-model and generic-message-passing exploration.
- **Delivery / durability mechanics (§1–§8 here).** *Once the message can reach the target* (directly, or via an embedded-capable peer), how does the result come back reliably and how is it preserved? This is the inbox / continuation / `deliver_to` machinery.

The scenario matrix (§3) is the *delivery* layer and assumes the sender can already reach the receiver. §3A is the layer beneath it.

## 2. The pieces (plain terms)

- **The universal namespace.** Every path is peer-namespaced. `/peer-B/system/inbox/...` held on another peer is *that peer's cached/synced copy of B's data*. Authority is B's signature and content addressing — not which peer physically holds the bytes. (Today — ENTITY-CORE-PROTOCOL.md §1.4: :159-166, :171, :177, :179, :185.)
- **The inbox: a passive durable mailbox.** A delivered request is stored in the tree. If a continuation is installed at that path it is processed; if not, it waits. (Today — EXTENSION-INBOX.md:31, :73, :148, :150.)
- **The continuation: the active adapter.** A continuation step is a general request dispatch — its own `target`/`operation`/`dispatch_capability` — independent of how the result arrived. (Today — EXTENSION-CONTINUATION.md:38, :68-78.)
- **Subscription: freshness, not retention.** A subscription keeps a cached remote namespace *current*; it does not by itself *retain* anything. Retention is a storage choice the hosting deployment makes. (Today — V7 §1.4:179.)
- **REGISTRY: name → peer (the DNS analog).** Resolve a recipient name to a `peer_id`. (Today — `EXTENSION-REGISTRY.md` v1.0.)
- **The inbox-relay declaration: where to store when I'm offline (the MX analog).** A peer publishes a signed `system/peer/inbox-relay` naming the relay(s) that hold its mail. Self-certifying (signed by the peer's own key), served always-on by REGISTRY. (Today — `EXTENSION-RELAY.md` v1.0 §3.5; closed RELAY Q2.)
- **RELAY Mode S: store-and-forward (the SMTP queue / mailbox analog).** Put an envelope at a namespace; the receiver polls it on reconnect. (Today — `EXTENSION-RELAY.md` v1.0 §3.2/§4.2/§6.2.1.)
- **The three knobs on a request.** All optional, all independent; set none → today's behavior unchanged.
  - **wait or don't** — block until the receiver answers, or fire and move on.
  - **`deliver_to`** — return the result to me, or route it onward (the receiver proxies it with the capability I gave it). A sender's "send onward" *is* the next peer's "something arrived": the sender's outbox and the receiver's inbox are one store from two ends. (A field — V7 §3.2.) A peer that does not implement inbox delivery silently ignores this field and returns a normal EXECUTE_RESPONSE — that is V7 v7.46's behavior; the briefly-introduced no-silent-ignore rule (v7.47) was reverted with the durability retraction.
  - **durable** *(exploratory, not in V7)* — store this reliably and let me find it again, by request id: `(author, request_id)` already uniquely identifies a request (V7:661) and the stored inbox delivery already carries `original_request_id` (EXTENSION-INBOX:84). The supporting machinery lives in the optional, exploratory `EXTENSION-DURABILITY.md`; it is not required by V7 or by any other extension.

## 3. The scenario matrix (read this first)

Every sender/receiver/knobs combination, concrete paths, and a verdict. Lifted from `core-protocol-domain/explorations/EXPLORATION-DELIVERY-DURABILITY-SCENARIO-MATRIX.md` §3 — the file:line-grounded backing **independently verified by entity-core-go and entity-workbench-go**. The exploration is the proof; this is the readable copy.

**Legend:** **OK** = works today · **EXPL** = behavior described by the exploratory, optional `EXTENSION-DURABILITY.md`; not normative for V7 or for any peer that does not install that extension · **GUIDE** = a convention a deployment must arrange (not a spec gap).

| # | Sender | Receiver | Knobs | What happens | Verdict |
|---|--------|----------|-------|--------------|---------|
| 1 | core-only | core-only | sync, no `deliver_to`, not durable | A→B, B does the op, returns the result on the response. | **OK** — today, unchanged |
| 2 | core-only | inbox | sync, `deliver_to`, not durable | B does the op; forwards the result as an EXECUTE to the `deliver_to` URI; B's sync return to A is the **202 accepted** receipt, not the result. | **OK** — existing 202-ack (EXTENSION-INBOX:389) |
| 3a | core-only | inbox | sync, `deliver_to` (default op) | The delivered result is an EXECUTE with `operation: "receive"` at `system/inbox` — the inbox extension's conformance handler (EXTENSION-INBOX:53, :306, :420, :426). A core-only target has no such handler → the result is lost. **A core-only peer cannot accept the default delivered result.** | **GUIDE/precondition** — target needs the inbox extension |
| 3b | core-only | core-only + matching domain handler | sync, `deliver_to` with `system/delivery-spec.operation` overridden | The override (optional, V7 §3.11; defaults to `receive`, EXTENSION-INBOX:441) names a domain op the target *does* implement → a plain EXECUTE to that handler; no inbox needed. Works **only if that handler exists**; else silently lost. | **GUIDE** — state the precondition |
| 3c | core-only (embedded) | inbox + continuation on a **capable** peer | `deliver_to` → an inbox on a capable peer, continuation installed there | The canonical embedded-participation pattern — see §5. | **GUIDE pattern** (grounded; not a spec gap) |
| 4 | core-only | inbox | sync, durable, no `deliver_to` | B preserves A's request at `/peer-B/system/inbox/{rid_A}` (B-namespace, B-signed). B's response says **what durability was actually applied vs. asked, and why if less**. A's later handle = that path in the universal namespace. | **EXPL** for the response; storage + handle **OK** |
| 5 | core-only | inbox + durable peer C | async, durable, `deliver_to` set | The durable A→B→C path — see §4-walkthrough. | **OK** addressing/sync; **EXPL** verdict |
| 6 | core-only / embedded | full | any | Embedded peer emits durable/`deliver_to` (understands the semantics, offloads realization) but as a *receiver* only does plain sync request/response. **Not-required durability:** it now *says* "no durable store; handled normally" (`200`, `applied: none, reason: no_durable_store`) instead of silently ignoring the ask. **Required durability:** an embedded receiver with no durable store cannot meet it → **`412` refused** (operation not performed), not a `200` — same refuse-at-acceptance rule as any other receiver. | **OK** + **EXPL** (response says so, both the not-required and required paths) |
| 7 | full | inbox + continuation | async, durable, `deliver_to` = inbox path with a continuation | Stored at the inbox path (write-ahead); a continuation exists there → inbox hands off to continuation `advance` (EXTENSION-INBOX:148). Inbox owns "stored"; continuation owns "interpret." | **OK** — two existing extensions compose |
| 8 | full | inbox | durable; A wants to track/replay | A subscribes to `/peer-B/system/inbox/{prefix}` — subscription keeps the cache current (V7:179). Inbox = store; subscription = freshness; replay = re-read retained entries. | **OK** — three extensions compose |

**Wait vs. don't-wait, when `deliver_to` is set.** Both return a 202-class receipt, not the operation result. Waiting blocks the caller until the receiver returns the 202 — i.e. until the receiver has *accepted responsibility for the forward*. Not-waiting means the caller does not even learn that much. That confirmation-of-acceptance is the meaningful difference; it is **not** confirmation the final target received it — no synchronous channel through an intermediary can give that. (EXTENSION-INBOX:324, :389.)

## 3A. Reaching an offline or unknown peer — the addressing & store-and-forward layer

The matrix above assumes A can already reach B. This section is the layer beneath:
*how A reaches B in the first place* when B is named-but-unknown, offline, or a
stranger. This is the email-delivery shape — and the SMTP cross-check
confirmed our stack reproduces it, with one composition (not a new primitive).

**The mapping (why "nobody owns it but it works" works for us too):**

| Email layer | Our analog | State |
|---|---|---|
| IP / BGP packet routing ("the network figures out the path") | underlying transport — we run over IP and inherit it | inherited, free |
| DNS A-record (name → server) | REGISTRY (name → peer) | landed v1.0 |
| DNS **MX** record (where to deliver mail for a recipient) | `system/peer/inbox-relay` (signed, self-certifying) | landed (EXTENSION-RELAY §3.5) |
| SMTP store-and-forward + backup MX + mailbox | RELAY Mode S + INBOX | landed v1.0 |
| DKIM/SPF (trust the sender, not the relay) | V7 signatures + cap chains | core |
| DSN / bounce (status back to sender) | CONTINUATION `deliver_to` reply path | landed |

**The composition** (`send_message`, best-effort, to a possibly-offline recipient):

```
peer_id  = REGISTRY.resolve(name) or name            ; resolve (DNS A-record)
envelope = sign(msg, deliver_to=my_reply_path,        ; sign + return-path (DKIM)
                deliver_token=cap)
if NETWORK.reachable(peer_id):                         ; try direct (the common case)
    dispatch(peer_id, envelope)                         ; → lands in B's inbox (§1–§8 take over)
else:
    mx = fetch(peer_id, "system/peer/inbox-relay")      ; where to store (MX lookup)
    verify_signed_by(mx, peer_id)                        ; self-certifying (better than DNSSEC)
    RELAY.put(mx.relay, mx.namespace, envelope)          ; store-and-forward (SMTP queue)
; B reconnects, polls its own peer_id namespace (RELAY §5.5 self-poll grant),
; fetches two-hop, verifies the inner signature directly — relay never trusted.
; A reply lands at my_reply_path → CONTINUATION advances (DSN).
```

**Touch points — the whole surface:** REGISTRY (resolve), the inbox-relay declaration
(where-to-store), NETWORK (try direct), RELAY Mode S (store-and-forward), INBOX
(mailbox + the §1–§8 delivery mechanics), V7 (sign/verify), CONTINUATION (reply). **All
landed** — the inbox-relay declaration (the MX-equivalent) is `EXTENSION-RELAY.md` §3.5,
which closed RELAY's Q2 fallback-target gap.

**This is a composition, not a primitive — and that is correct.** "Send a message to
anyone" is the SMTP submission/delivery split expressed in our terms: the sender's
local agent does *submission* (resolve + sign + try-direct-or-store); the recipient's
relay/inbox does *delivery* (hold + serve on poll). No new core surface; it is one
named pattern over landed handlers.

**Adversarial-default, cooperative-open (both available).** We are born where SMTP
painfully ended up: a relay is cap-gated (`relay-forward/-put/-poll` root at the
relay). But the open-cooperative posture is a knob — an open relay grants
`relay-*: include all` and just-works, SMTP-1985-style, with the abuse risk now the
operator's informed choice. The default is safe; the openness is reachable.

**Where it does NOT work (honest parity with SMTP).** If neither A↔B nor a
mutually-reachable relay exists (both behind NAT, no shared relay), the message is
undeliverable — **email fails here too** (no IP route = no delivery). Entity-layer
multi-hop routing (RELAY Mode C / smart Mode F) is the *beyond-email* fix, deferred
until a mesh/DTN driver appears (RELAY §11.1a). Not a regression vs SMTP.

**Bridging to the real world.** A `BRIDGE-SMTP` peer (bridge-family member, BRIDGE-HTTP
shape) translates entity-messages ↔ real email: outbound via real DNS/MX, inbound
wrapping received mail into a peer's INBOX. Interop is a peer-you-run, zero core change.
See `proposals/PROPOSAL-EXTENSION-BRIDGE-SMTP.md`.

### 3A.1 Worked example — stranger sends to an offline named peer (the canonical SMTP flow)

Alice knows only the name `bob.example`; Bob is offline.
1. Alice resolves `bob.example` → Bob's `peer_id` (REGISTRY) — *DNS A-record.*
2. Alice fetches Bob's `system/peer/inbox-relay` → `[{relay R, namespace Bob, priority 10}]`;
   verifies it is signed by Bob's key — *MX lookup, better-than-DNSSEC.*
3. Alice signs the message with `deliver_to` = her reply path — *DKIM + return-path.*
4. Bob unreachable → Alice `RELAY:put` at `(R, Bob)` — *store-and-forward to the MX.*
5. Bob reconnects, polls R at his own peer_id namespace, fetches two-hop, verifies
   Alice's inner signature directly (R untrusted) — *IMAP fetch + DKIM verify.* From
   here the §1–§8 delivery/durability mechanics take over (e.g. a continuation at Bob's
   inbox path advances).
6. Bob's reply lands at Alice's `deliver_to`; her continuation advances — *DSN / reply.*

Every step maps to a landed mechanism except step 2 (the inbox-relay declaration).

## 4. Worked example — the durable A→B→C path (Scenario 5)

1. **A → B**, durable, `deliver_to` set, don't-wait.
2. **B preserves** A's request at `/peer-B/system/inbox/{rid_A}` — B's namespace, B-signed-authoritative. A's lookup handle is the *origin* id `(author=A, request_id=rid_A)`, stable regardless of who serves it.
3. **C caches/syncs B's inbox namespace.** The entry appears on C, structurally identical, authority = B's signature (V7:185). Nothing moved into a "special composition" — this is the universal namespace operating exactly as specified.
4. **Sync is freshness; the deployment provides retention.** V7:179 keeps C's copy *current* — it does not decide C *keeps* it. "C is the durable host" is a deployment role; retention is C's storage choice, not a property sync grants.
5. **The response.** If the exploratory `EXTENSION-DURABILITY` is installed on B, B reports durability via status + the `durability` field per that extension's §5 (strengths B self-determines at acceptance → `200` with `applied` = that level and `handle` = the path where the entry can be read; replication-class → `202` accepted, `committed` = the replication target, `handle` = where to observe it propagate; the response never claims a durability it does not yet have). If `EXTENSION-DURABILITY` is not installed, B's response is V7 v7.46's normal EXECUTE_RESPONSE — no `durability` field — and the durability marker, if sent, is silently ignored. (Exploratory — `EXTENSION-DURABILITY.md`; PROPOSAL-DELIVERY-AND-DURABILITY.md Status: RETRACTED.)

## 5. Worked example — an embedded peer in a durable async flow (Scenario 3c)

The canonical pattern for a peer too small to run the inbox extension to still take part.

1. The embedded peer is core-only: a plain handler for some op, no inbox, no continuation.
2. It sends with `deliver_to` → an **inbox on a capable peer** that has a **continuation installed** at that path.
3. The result lands in the capable peer's inbox; the inbox hands off to the continuation's `advance` (EXTENSION-INBOX:31, :148).
4. The continuation is independent of how the result arrived (EXTENSION-CONTINUATION:38) and is a general dispatch (:68-78). Its step dispatches a **plain EXECUTE back to the embedded peer's own op**, authorized by a `dispatch_capability` rooted at the embedded peer's conferred authority (cross-peer provenance spec-closed; EXTENSION-CONTINUATION §4.2/§4.3).
5. The embedded peer receives an ordinary request for an op it implements — **pure core protocol, no inbox extension on it**.
6. **Composes with durability:** the durable record persists in the capable peer's inbox (addressable by request id). The re-dispatch is the active push. If the embedded peer is unreachable the push fails into the continuation error model (`on_error` is best-effort compensation and can itself fail with no observable surface — EXTENSION-CONTINUATION §3.4 lost-error marker, :546; the `on_error` field is :96/:98) **but the record persists** and can be pulled later. The Kafka shape — store it, keep it, deliver/read it back by id.

**Preconditions (state them, do not paper over):** the embedded peer implements the callback op (a plain handler, *not* the inbox extension) and confers the dispatch capability. With none of: the inbox handler, a `delivery-spec.operation` override to an implemented domain handler, or this composition — a delivered result is **silently lost**, the failure class this whole line of work exists to kill.

## 6. Deployment roles for durable messaging

Which peer plays which role so the pattern realizes a durability tier:

- **Originator** — holds the origin `(author, request_id)`; the stable lookup handle forever.
- **Preserver** — the receiver that writes the durable entry into its own namespace, signed-authoritative.
- **Durable host / replica** — a peer that caches/syncs the preserver's inbox namespace and is *assigned by the deployment to retain it*. Sync = freshness; retention is this host's storage choice.
- **Resolver** — whichever peer serves the freshest copy is a routing/partial-projection concern; the address and content hash are stable across hosts.

These are deployment configuration. The same messaging pattern under different role assignments yields best-effort, retained-on-one-peer, or replicated durability — which is exactly why the durability strength list stays illustrative and is **not** frozen into an enum (PROPOSAL-DELIVERY-AND-DURABILITY.md Decision 2). (How these roles are assigned and wired is a *peer-compositions/topology* question — that broader concern lives in the peer-compositions thread, not here.)

## 7. The durability contract — see EXTENSION-DURABILITY (exploratory, optional)

The durability surface previously described in this section was retracted from the spec. The architecture team concluded the thread had broached too far into user-space semantics — it pattern-matched on log-system conventions (durability levels, status branching, advertisement) without an actual deployment driver. The lifted material is preserved as a standalone exploratory extension at `core-protocol-domain/specs/extensions/standard-peer-extensions/EXTENSION-DURABILITY.md`. It is **not normative for V7 or for any other extension**; no deployment is required to install it; EXTENSION-INBOX does not depend on it.

If a peer installs `EXTENSION-DURABILITY`, that extension's §5 defines the response shape (status code + `durability` field, with `applied` = physically-in-place-now and `committed` = the async target on 202), §6 defines the lookup handle, and §8 defines the conformance obligations. If a peer does not install it, durability markers on requests are silently ignored — V7 v7.46's default behavior.

The open question on whether anything narrower (e.g. a one-line V7 rule requiring an explicit refusal status when a peer cannot honor a request-side marker) belongs in the core protocol is left for a future session. See `proposals/PROPOSAL-DELIVERY-AND-DURABILITY.md` (Status: RETRACTED) for the design history and the cross-impl validation record.

## 8. Pitfalls

- **A `deliver_to` target with no handler for the delivered op loses the result silently.** The handler is the inbox `receive` (needs the inbox extension — 3a), or a domain handler named via `system/delivery-spec.operation` (3b), or the §5 embedded composition (3c). State the precondition or results vanish. (This is the silent-loss class the retracted durability thread tried to address; V7 v7.46's default is still silent-ignore. A future narrower V7 amendment may make this explicit.)
- **Sync with `deliver_to` gets you acceptance, not the result.** See §3's wait-vs-don't-wait note.
- **Sync is freshness, not durability.** "C synced it" ≠ "C will keep it." Retention is a deployment role (§6).
- **If you do install `EXTENSION-DURABILITY`, don't freeze the strength list.** "accepted / stored-and-retried / replicated" is illustrative; a fixed durability enum repeats the mistake that started — and ended — that thread.
- **A continuation with no `on_error` whose forward target persistently returns non-2xx is observable, not invisible.** Forward continuations are fire-and-forget by design — handler-level non-2xx responses are classified as completed forward dispatches and decrement `remaining_executions` rather than triggering retry or error routing. If you want compensation logic on failure, configure `on_error`. If you just want to *see* failures (the common case during dev/debug or during operator triage of a mis-scoped dispatch cap → repeated 403), implementations bind a `system/runtime/chain-error-lost` marker at `system/runtime/chain-errors/lost/{chain_id}/{step_index}` with `reason: "forward_dispatch_non2xx"` on every non-2xx occurrence (idempotent re-binding; SHOULD-bind per EXTENSION-CONTINUATION v1.13 §3.4). Same marker shape covers the A.1 case (`on_error` dispatch itself fails — `reason: "on_error_dispatch_failed"`). Aggregate from `system/runtime/chain-errors/lost/` for diagnostics; the marker is informational, never reactive.
- **A chain step receives the prior result *double-wrapped* — decode twice.** When a continuation step (or a trampoline handler bridging two compute steps) is delivered the previous step's result, its `params` is a `system/protocol/inbox/delivery` entity carrying `.status` and `.result` (EXTENSION-INBOX §receive — `extracted_result = message_entity.data.result`). And `.result` is the **full entity envelope** of the prior result (`{type, data, content_hash}`), *not* the bare value — so reaching the value is two decodes: `params → .result` (an entity) `→ .data` (or the value field). This is easy to get wrong (workbench-go surfaced it building a multi-step `compute → trampoline → compute` chain — `compute_chain_multistep_test.go`); the shape isn't obvious from reading `EXTENSION-CONTINUATION` alone. SDKs SHOULD provide an `UnwrapChainStepDelivery(params) → (status, innerValue)` helper that does both decodes (SDK-EXTENSION-OPERATIONS §8 E7.2); a hand-written trampoline must replicate the two-level peel.

## 9. Cross-references

- `proposals/PROPOSAL-DELIVERY-AND-DURABILITY.md` — design history (Status: RETRACTED).
- `core-protocol-domain/specs/extensions/standard-peer-extensions/EXTENSION-DURABILITY.md` — the exploratory standalone extension carrying the lifted §10 material.
- `core-protocol-domain/explorations/EXPLORATION-DELIVERY-DURABILITY-SCENARIO-MATRIX.md` — file:line-grounded, independently verified backing for §3. (Still useful for the delivery-scenario portion; the durability verdicts in it correspond to the retracted spec text and now describe `EXTENSION-DURABILITY` behavior rather than V7 behavior.)
- `core-protocol-domain/explorations/EXPLORATION-PEER-COMPOSITIONS.md` (v2) and the deferred `PROPOSAL-PEER-COMPOSITIONS-AND-DELIVERY-CLASSES.md` — the **broader peer-compositions concept** (extension sets + topology; kernel properties; named-composition catalog).
- `EXTENSION-INBOX.md`; `EXTENSION-CONTINUATION.md`; `ENTITY-CORE-PROTOCOL.md` §1.4 / §3.2 / §3.3.
- **Addressing/reachability layer (§3A):** the SMTP-model and generic-message-passing exploration (the SMTP cross-check); `EXTENSION-RELAY.md` v1.0 §3.5 (the MX-equivalent, folded; design record at `proposals/implemented/PROPOSAL-PEER-INBOX-RELAY-MX-EQUIVALENT.md`); `EXTENSION-RELAY.md` v1.0 (Mode S store-and-forward); `EXTENSION-REGISTRY.md` v1.0 (name→peer); `proposals/PROPOSAL-EXTENSION-BRIDGE-SMTP.md` (real-email interop, stub).

---

*First pass. Written before the spec merge as the design accuracy test. Scope is the delivery/durability/cross-peer-messaging pattern only — the peer-compositions concept is a separate thread and a separate guide.*
