# Multi-Signature Capabilities: User Guide

**Status**: Active
**Audience:** Application developers and operators using K-of-N multi-signature root capabilities. Assumes familiarity with capabilities (GUIDE-CAPABILITIES.md) and basic V7 concepts (peers, signatures, the chain root rule).
**Spec reference:** PROPOSAL-MULTISIG-CORE-PRIMITIVE.md (M1–M12), ENTITY-CORE-PROTOCOL.md §3.6 (capability schema), §5.5 (chain verification), §5.6 (`is_attenuated`).
**Related guides:** GUIDE-CAPABILITIES.md, GUIDE-IDENTITY.md.

---

## 1. What multi-sig caps are (and what they aren't)

A multi-signature capability is a root cap whose `granter` field names a *group* of K-of-N peers instead of a single peer. The cap is valid only when at least K signatures from that group are present. Conceptually:

> "This authority is so important it required K-of-N approval to come into existence."

It's structurally equivalent to Bitcoin's P2SH multisig (BIP 16, deployed 2012) — same K-of-N signing requirement, applied at the capability-issuance layer instead of the transaction layer. Established practice; not invention.

**Two layers of K-of-N exist in this system; pick the right one for your use case.** This guide covers **cap-layer** multi-sig (K-of-N at the capability-issuance layer; signatures sit on the cap token; validated by V7's `verify_capability_chain`). The other layer — **entity-layer K-of-N** (signatures sit on non-cap entities like attestations, snapshots, votes; validated by `EXTENSION-QUORUM`'s `verify_k_of_n_signatures`) — is the right choice for identity, group, cluster, governance, etc. They never mix. See §6.2 for an arbiter table; for the entity-layer side, see GUIDE-QUORUM.md.

**What multi-sig caps (cap-layer) serve well:**
- Joint accounts (Alice and Bob jointly hold authority on something).
- Escrow (a buyer, a seller, and an escrow agent jointly authorize release).
- Dual control (two engineers must both sign off on a change).
- Threshold approval (any 3 of 5 board members can authorize).
- Multi-device user authority (any 2 of [laptop, phone, hwtoken] required to issue new authority).
- Login / per-operation approval via short-TTL fresh issuance (each sensitive op requires fresh K-of-N consent; see §4.6).
- Personal recovery clusters in **core-only deployments** (your warm device + cold backups jointly hold authority over your operator key, under Architecture A1). **For identity-installed deployments, use the quorum substrate instead** — `EXTENSION-IDENTITY` v3.3 routes recovery through entity-layer `system/quorum` + K-of-N attestation; see GUIDE-IDENTITY.md §11 and GUIDE-QUORUM.md. The §6 composition section below explains the boundary.

**What a single long-lived multi-sig cap doesn't serve directly:**
- "Every operation under this *standing* authority requires K-of-N approval at use time." A long-lived multi-sig cap's K-of-N is checked once at issuance; once issued, the grantee uses it like any other cap. **Per-operation K-of-N is reachable** by issuing a fresh, narrowly-scoped, short-TTL multi-sig cap per operation (§4.6) — that's the recommended pattern when you want 2FA-style fresh consent per action *and* you want cap-machinery uniformity (audit, attenuation, revocation). Handler-level K-of-N (§3.2) is the alternative when no cap artifact is wanted.
- Caps used cross-peer from anywhere by anyone holding K-of-N keys. The primitive has a local-peer constraint that makes joint authority work via *each participant having their own cap* rather than one shared cap. See §2 below.
- Mid-chain multi-sig (a multi-sig cap with a multi-sig parent). Multi-sig caps are root-only by construction. Mid-chain patterns are a different mechanism (deferred indefinitely; see PROPOSAL-MULTISIG-CORE-PRIMITIVE §13.4).

---

## 2. The local-peer-in-signers rule

This is the most important thing to internalize. **A multi-sig root cap is valid at peer P only if P is in the cap's `signers` AND P signed.**

V7 §5.5 line 1974 says "root capabilities MUST be granted by the local peer." For single-sig caps, this means the granter equals the peer using the cap. The multi-sig primitive generalizes this: if the granter is a group, the group must include the local peer, and the local peer must have contributed one of the K signatures.

### 2.1 What this means in practice

If you're coming from a Bitcoin mental model — "any K-of-N participants can use this multisig output from anywhere" — that's not how this primitive works.

Each participant has their **own** multi-sig cap. The cap is locally rooted at *their* runtime peer; their runtime peer is one of the `signers`; and their runtime peer signed it. The other K-1 signatures come from the other participants' keys.

```
Joint account between Alice and Bob (3-of-3 [alice_peer, bob_peer, escrow_agent]):

  Alice's cap                          Bob's cap
  ────────────                         ──────────
  granter:                             granter:
    signers:                             signers:
      [alice_peer,                         [alice_peer,
       bob_peer,                            bob_peer,
       escrow_agent]                        escrow_agent]
    threshold: 2                         threshold: 2
  grantee: alice_peer                  grantee: bob_peer
  parent: null                         parent: null
  signatures: 2 of [alice, bob,        signatures: 2 of [alice, bob,
                    escrow_agent]                        escrow_agent]
                  (incl. alice)                       (incl. bob)
```

The same logical authority — a 2-of-3 between Alice, Bob, and the escrow agent — is expressed as **two distinct caps**, one rooted at Alice's peer and one at Bob's. Alice uses her cap on Alice's peer; Bob uses his cap on Bob's peer. Neither can use the other's.

### 2.2 Why the rule exists

V7's root-granter rule is what makes the cap system local: each peer is the sole root authority for capabilities used at that peer. Without this, a multi-sig group with no participation by the verifying peer could anchor chains the peer treats as authoritative — which would let three external parties grant Alice authority on Alice's own resources without her consent.

Generalized for multi-sig: the local peer must be a participant in any cap it accepts as a root. For joint authority to work at peer P, P must be one of the K-of-N signers and must have signed.

### 2.3 What this constraint enables

- **Each participant retains sole control over their own peer's resources.** Alice can't unilaterally bind Bob; Bob can't unilaterally bind Alice. The K-of-N requirement just means each participant's own cap was jointly authorized by the group.
- **Revocation is local.** When Alice wants to leave the joint account, she revokes her cap on her peer (V7 standard tree-binding revocation). Bob's cap is unaffected unless Bob also revokes his.
- **No global state.** There's no "the joint account" entity that all parties must agree on. Each participant has their own cap; consistency across participants is application convention, not protocol.

### 2.4 What this constraint forbids

- **Shared caps used from anywhere.** The Bitcoin mental model — "the multisig output exists; whoever has K-of-N keys can spend it" — doesn't apply.
- **Caps issued to a peer not in the signers.** A 2-of-3 cap with grantee = some peer not in `signers` is technically allowed (grantee and signers are separate fields), but the cap is only usable at peers that ARE in signers. So issuing a cap to an external grantee means that grantee has no peer where the cap roots — the cap is unusable.
- **Architecture A2 for recovery clusters.** "Cold cluster issues a delegation to a separately-running warm device" doesn't fit. The warm device must be in the cluster's signer set. See GUIDE-IDENTITY.md §11 for how the identity extension handles recovery instead (entity-level K-of-N, not multi-sig caps).

---

## 3. Threshold attenuation: issuance only

When you issue a single-sig delegation from a multi-sig cap, the K-of-N requirement does **not** propagate to the child.

```
Multi-sig 2-of-3 root cap (grants X)
   │
   │ grantee Alice receives this cap
   │ Alice delegates to Bob:
   ▼
Single-sig cap (granter=Alice, grantee=Bob, grants ⊆ X)
   ; This is a normal V7 single-sig cap. Bob uses it without K-of-N anything.
```

This is correct as "the K-of-N approved the existence of the authority; once created, downstream delegation uses standard V7 semantics." The threshold applied at issuance (the moment the K-of-N parties agreed to bring this authority into being); after that, the grantee delegates normally.

### 3.1 What this serves

> *"This authority is so important it requires K-of-N approval to come into existence. Once it exists, the grantee can use and delegate it normally."*

That's the use-case multi-sig serves. Bringing the authority into existence is the gated moment.

### 3.2 What this does NOT *automatically* serve (and how to reach it anyway)

> *"Every operation under this authority requires K-of-N approval at use time."*

This isn't what a single long-lived multi-sig cap gives you. Once a multi-sig cap is issued, the grantee uses it normally — no per-operation re-signing.

**But you can reach per-operation K-of-N via the primitive** by issuing a fresh, narrowly-scoped, short-TTL multi-sig cap **per operation**. The primitive's `expires_at` field plus tight `grants` scope make the issuance event itself the per-operation approval gate. See §4.6 for the full walkthrough.

So there are two ways to do per-operation K-of-N:

1. **Short-TTL fresh-issuance multi-sig** (this primitive, §4.6 below). One cap per operation; K signers sign per-op; cap expires shortly after. Uniform with V7 cap machinery; benefits from chain walk, attenuation, revocation, audit.

2. **Handler-level K-of-N validation.** A handler that, at request time, requires K signatures attached to the request before processing it. No cap is issued. Pure handler logic on top of single-sig caps. The identity extension uses an entity-level version of this for attestation entities (`verify_k_of_n_signatures`).

When to pick which:

| Pattern | When it fits | Cost |
|---|---|---|
| Long-lived multi-sig cap | Standing authority used many times | One issuance, many uses |
| **Short-TTL fresh-issuance multi-sig (§4.6)** | Per-op approval where cap-machinery uniformity and audit help (login, sensitive admin, deploy approval) | One issuance per op (~5 entities); plus expiry GC |
| Handler-level K-of-N | Per-request decisions where no persistent authority artifact is wanted (board vote on one proposal; one-shot consensus check) | Inline sig validation per request; no cap lifecycle |

The simplest mental model: a multi-sig cap's K-of-N condition gates **the cap's existence boundary**. Whether the cap is long-lived (standing authority) or short-TTL-per-op (fresh per use) depends on what the issuance boundary corresponds to in your application.

---

## 4. Use case walkthroughs

### 4.1 Joint account

Alice and Bob run a small business. Both should be able to authorize charges to the business account. Neither should be able to do so unilaterally.

Setup:

1. They agree on a 2-of-2 (or 2-of-3 with their accountant as the third) multi-sig group: `signers: [alice_peer, bob_peer, accountant_peer]`, `threshold: 2`.
2. Alice constructs a multi-sig cap on her peer:
   ```
   granter: { signers: [alice_peer, bob_peer, accountant_peer], threshold: 2 }
   grantee: alice_peer
   parent: null
   grants: <business-account scope>
   ```
3. The cap is signed by 2 of [alice, bob, accountant]. Alice + Bob is sufficient. Bob signs with his keypair when Alice presents the proposed cap.
4. The cap is persisted at `system/capability/grants/multi-sig-root/{cap_hash}` on Alice's peer.
5. Bob does the same on his peer: constructs his version (same `signers` and `threshold`, but `grantee: bob_peer`), gets it signed by 2 of [alice, bob, accountant], persists at his own multi-sig-root path.

**Sigs flow both directions during setup.** Alice's cap_alice needs Bob's signature; Bob's cap_bob needs Alice's signature. The pairing is symmetric: each participant signs the other's cap as part of the joint-issuance ceremony. In practice the application orchestrates this — when Alice initiates "set up joint account with Bob," the app prepares both cap_alice and cap_bob, has Alice sign both, sends them to Bob for his signatures, returns each finalized cap to its respective owner. Two caps, one ceremony.

Each now has their own cap. Either can use theirs for business-account operations on their own peer. If they leave the partnership, each revokes their own cap.

### 4.2 Escrow / dual control

Buyer pays seller; an escrow agent holds release authority until the parties agree.

The simplest pattern: a 2-of-3 cap among [buyer, seller, escrow_agent], threshold 2, grantee = escrow_agent. Whichever pair signs (buyer+agent, seller+agent, or buyer+seller) makes the cap valid; the agent uses it to release funds.

**A real escrow design choice — does the agent always have to participate?** With plain K=2 of 3, the answer is *no*: buyer + seller alone could resolve without the agent. That may be what you want (true 2-of-3 mediation) or it may not. Two ways to enforce "agent always participates":

1. **K=N=3.** All three must sign. Strongest, but if the agent is unavailable everything stalls.
2. **Two asymmetric caps:** `cap_release_to_seller` (signers: [buyer, agent], threshold: 2) and `cap_return_to_buyer` (signers: [seller, agent], threshold: 2). Each resolution path is its own cap; the agent is in both signer sets. Cleaner separation of resolution intents; closer to typical escrow contracts.

Most escrow flows want option 2 (asymmetric caps). It also lets the two resolutions have different `grants` scope — release-to-seller and return-to-buyer might touch different accounts or trigger different downstream effects.

When the chosen pair signs the corresponding cap, the cap is valid; the agent uses it to perform the resolution. The losing-side cap is left unsigned and eventually expires (use `expires_at` to bound how long the escrow can sit unresolved).

### 4.3 Recovery cluster (Architecture A1)

GUIDE-IDENTITY.md §11.4 covers this in detail. The summary:

A recovery cluster under Architecture A1 is "warm device + N-1 cold keys" jointly authorizing the operator key. The warm device is one of the cluster constituents. K=2 of [warm, cold1, cold2] is operationally "warm + 1 cold" (since the warm device's signature is always required by the local-peer rule).

Note: this is the **multi-sig primitive** version of recovery clusters. The **identity extension** (EXTENSION-IDENTITY.md) takes a different approach — entity-level K-of-N validation on non-cap attestation entities, no involvement of the multi-sig primitive. The two are parallel mechanisms; see §6 below.

### 4.4 Threshold approval

A 5-person board, any 3 of whom can authorize a class of operations.

Each board member has their own multi-sig cap with `signers: [m1, m2, m3, m4, m5]`, `threshold: 3`. The cap is signed by 3 of the 5; the member's own peer is in the signers and signed.

Any board member can use their cap for the authorized operations. Other board members are unaffected. If a member leaves, their cap is revoked individually; the others continue using theirs.

**M6 wrinkle.** Because Site 3 requires the local peer to be in `signers` AND signed, "any 3 of 5 sign" is operationally "the issuing member + any 2 others." Each member's cap is signed by their own peer plus 2 of the other 4. Different members end up with caps signed by different K=3 subsets — same logical authority, distinct caps.

If you want **per-operation** approval (each board action requires fresh 3-of-5 signatures), see §4.6 for the short-TTL fresh-issuance pattern, or §3.2's handler-level K-of-N alternative.

### 4.5 Multi-device user authority

Alice has a laptop, a phone, and a hardware token. She wants any 2 of these 3 to be required to issue new authority. No single compromised device can issue caps unilaterally.

**This is not per-operation MFA.** Multi-sig caps don't make every login or every operation require 2 devices to approve at use time — the K-of-N condition gates the cap's existence, not its uses. (For per-operation 2-device approval, see §4.6.)

What it does serve: "issuing new authority requires 2 of my 3 devices." Once the cap exists, the device holding it uses it normally.

Two operational shapes depending on whether one device or all devices should be able to *use* the cap.

**Case A — single-device use (cap rooted on laptop, hwtoken is cold backup):**

1. On laptop, draft cap_laptop with `signers: [laptop_peer, phone_peer, hwtoken_peer]`, `threshold: 2`, `grantee: laptop_peer`, `parent: null`, `grants: <full personal scope>`.
2. Laptop signs. Phone signs (Alice opens the phone app, reviews the proposed cap, taps approve). Hwtoken stays in the safe.
3. Two signatures present → cap valid → persisted at laptop's `multi-sig-root` path. Laptop uses the cap freely.
4. **If laptop is compromised:** revoke cap_laptop on laptop (or recover phone+hwtoken first if laptop is fully lost). Generate fresh laptop_v2 keypair on a new device. Draft cap_laptop_v2 with signers `[laptop_v2, phone, hwtoken]`. Phone + hwtoken sign (the warm device is gone, so the cold pair comes online). New laptop now has its own multi-sig root cap; old one is revoked.

This is structurally the recovery cluster pattern (§4.3) — the laptop is the warm constituent that must always sign caps it will later use.

**Case B — any-device use (laptop OR phone can use the cap independently):**

Each device holds its own multi-sig cap, joint-account-style.

1. On laptop: cap_laptop with `grantee: laptop_peer`, signers `[laptop, phone, hwtoken]`, threshold 2. Laptop + phone sign (hwtoken stays cold).
2. On phone: cap_phone with `grantee: phone_peer`, same signers and threshold. Phone + laptop sign.
3. Each device persists its own cap. Either can use independently for the granted scope.

Hwtoken is in both signer sets but doesn't sign either cap at issuance — it's the cold backup for recovery scenarios. It earns its threshold position at the moment the warm device is lost (Case A's recovery flow), where the cold pair (any 2 cold) issues a new cap for the new warm device.

**Setup ergonomics.** First-run wizard: Alice runs setup on laptop, app drafts the cap(s), prompts for phone approval (push notification, in-app review, tap), prompts for hwtoken signature (insert + tap). Multi-step but one-shot — once provisioned, her devices use the caps without further multi-sig involvement.

### 4.6 Login / per-operation approval (short-TTL fresh issuance)

Alice wants to authorize sensitive operations — bank transfers, deploy commands, admin actions — with fresh 2-of-3 device approval per action. Standing authority isn't enough; she wants *this transfer right now* to require fresh consent from a second device.

**The pattern: issue a fresh, narrowly-scoped, short-TTL multi-sig cap per operation.** The cap's K-of-N existence condition becomes the per-operation approval gate.

1. Alice initiates "transfer $5000 to Bob" on her laptop.
2. Laptop drafts:
   ```
   cap_transfer: {
     granter: { signers: [laptop, phone, hwtoken], threshold: 2 },
     grantee: laptop_peer,
     parent: null,
     grants: <transfer-funds scope, narrow: amount=5000, dest=bob_account, single-use intent>,
     expires_at: now + 300,        ; 5 minutes
     not_before: now
   }
   ```
3. Laptop signs. Push notification fires on phone: "Approve $5000 transfer to Bob?" Alice opens the phone app, reviews the scope (the proposed cap entity is shown to her in human terms), taps approve. Phone signs.
4. Two valid signatures present → cap valid → laptop uses it immediately to execute the transfer.
5. Operation completes. Cap expires after 5 minutes; whether or not it's GC'd, it can no longer authorize anything.

**Why this is per-operation K-of-N at the cap layer:**

- Each operation requires fresh issuance because previous caps are scoped to previous operations and have expired.
- Alice's phone-tap is per-operation consent; the K-of-N condition is satisfied freshly per op.
- The cap's tight `grants` scope ensures it can't be reused for a different operation — even if captured during its 5-minute lifetime.

**Why multi-sig primitive instead of handler-level K-of-N:**

- Audit trail: each op leaves a cap entity + signatures, content-addressed and persistable. Same audit shape as long-lived caps.
- Uniformity: the same primitive serves Alice's 2-of-3 joint account *and* her 2-of-3 login flow. One mental model.
- Composes with chain walk, attenuation, revocation. If Alice wants to delegate the transfer authority transiently to a script (`grantee = script_peer`), standard V7 delegation works.

**Costs and ergonomics:**

- Per-op overhead: 1 cap entity + K signature entities + storage churn. For K=2, ~5 entities per op (with identity entities cached). Bounded but real — pick this pattern for actions where the cost is tolerable, not for high-frequency operations.
- Coordination: K signers must reach consent within the TTL window. App orchestrates the UX (push notifications, hardware-token prompts).
- Storage: expired caps need GC; standard housekeeping but real.
- Site 3 still applies — the cap is rooted on the device that runs the op. For "any of my devices can perform login," each device drafts its own short-TTL cap with the others as co-signers. (The per-op nature makes Case B from §4.5 less burdensome — each cap is throwaway.)

**When this fits:**

- Login flows where 2FA-style "tap your phone to approve" is the UX.
- Sensitive admin actions (deploy to prod, drop database).
- Per-transfer approval in financial workflows.
- Anything where fresh consent per action matters and the audit trail of "this specific operation was authorized by these K parties at this time" has value.

**When handler-level K-of-N (§3.2) fits better:**

- Per-request board votes or one-shot consensus checks where a cap artifact has no value.
- High-frequency operations where per-op cap issuance overhead is too much.
- Domain-specific validation rules that exceed plain K-of-N (weighted, role-aware, time-window constraints) — though some of those compose with standard caps via attenuation.

#### Idempotency / at-most-once is handler logic, not cap logic

A natural follow-up question for the per-op pattern: *can the cap layer enforce that the operation runs at most once?* No — and this is by design.

**Why caps can't enforce at-most-once.** Caps are stateless by construction:
- Content-addressed and immutable (a "used" flag would change the hash).
- Verification is a pure function of `(envelope, included, local_peer_id)` — same envelope → same result, every time. The verifier doesn't track usage history.
- The chain walk has no place to record "this cap has been spent."

V7's existing fields bound *when* and *what* the cap can authorize (`expires_at`, `not_before`, `grants`, `resource_limits`). None of them are a use-count. Adding one would make verification stateful (the verifier would need a "consumed caps" registry), which breaks the content-addressed-immutable model the rest of the system rests on.

**What handlers can do.** Three patterns compose with the short-TTL multi-sig flow:

**Pattern 1 — Idempotency key + receipt entity (the standard).** EXECUTE params include an idempotency key (typically a UUID generated when the user initiates the action). The handler:
1. Looks up `<handler-domain>/receipts/{idempotency_key}` — not found on first run.
2. Performs the operation.
3. Writes a receipt entity recording the result.
4. Returns the result.

On retry with the same key: lookup hits the receipt, handler short-circuits and returns the cached result without re-executing. Idempotent on retry, exactly-once on effect. The handler picks the dedup window (forever / 24h / per-account TTL).

**Pattern 2 — Handler revokes the cap after execution.** Push at-most-once into the cap layer through revocation:
1. Multi-sig cap issued (K-of-N signed, narrow scope, short TTL).
2. Handler verifies chain, performs the operation.
3. Handler calls `tree:put(system/capability/grants/multi-sig-root/{cap_hash}, null)` — revokes the cap.
4. Subsequent EXECUTEs with the same cap fail `is_revoked` (V7 §5.1).

The handler can do this because the cap's grantee is the local peer, and the handler runs on the local peer with authority over its own cap-grants tree. Note: enforcement is still handler bookkeeping — the handler is the one calling `tree:put`. It's just that the resulting state lives in the cap-revocation index rather than a handler-private receipt store.

**Pattern 3 — Combine both (production pattern).** For account transfer or similar critical operations:
- Short-TTL multi-sig cap (per-op K-of-N + time bound + tight scope).
- Idempotency key in params (idempotent on retry — returns same result without re-executing).
- Handler revokes the cap after execution (defense-in-depth — even if the idempotency-key check is somehow bypassed, the cap is gone).

Three independent layers, no overlap in concern: cap layer authorizes existence of the operation, idempotency key handles retry semantics, cap revocation prevents stale-cap reuse. None of these can be moved into the cap primitive without taking opinionated defaults that wouldn't fit every domain.

**Why this division of labor is right.** Idempotency semantics are domain-specific:
- "Same idempotency_key + different params → reject" (transfer system).
- "Same idempotency_key → return cached result" (read-mostly system).
- "Idempotent within a 24h window, then key can be reused" (event ingestion).
- "Idempotent forever, key collision = bug" (financial settlement).

These can't be captured by a single cap-layer primitive without either taking opinionated defaults or extensive parameterization. Handler-level idempotency lets each domain pick its semantics; multi-sig caps stay a pure cryptographic-authorization primitive.

(Aside on nonce caps: some capability systems — zcap-LD variants, biscuit — add nonce-cap patterns where the cap includes a unique nonce and the verifier checks a consumed-nonce registry. V7 deliberately doesn't. Trade-off: uniform "at-most-once" mechanism vs. stateful verifier with unboundedly-growing consumed-nonce registry, registry pruning policy that's itself domain-specific, and doesn't compose with handler-specific idempotency semantics. V7's call: keep the cap layer stateless; let handlers own idempotency where they can express it natively.)

---

## 5. Async signature gathering

Multi-sig signing isn't synchronous. The K signers don't all need to be online at the same moment. The convention (informative — see PROPOSAL-MULTISIG-CORE-PRIMITIVE.md §14):

1. The proposer constructs the multi-sig cap entity with the desired `signers`, `threshold`, `grants`, and `grantee`. This produces a content_hash.
2. The proposer publishes the cap at a tree path (e.g., `system/capability/grants/multi-sig-root/proposed/{hash}`) and notifies the signers.
3. Each signer reviews, signs, publishes their signature entity (targeting the proposed cap's content_hash). Signing is asynchronous — each signer does it on their own time.
4. Once K signatures are present, an application-level continuation or the proposer themselves moves the cap from the proposed path to the final path (`system/capability/grants/multi-sig-root/{hash}`) and the cap becomes usable.

The protocol doesn't normatively specify where partial signatures live or who triggers the move-to-final step. Implementations may use different paths or workflows; the threshold-based completion model (signatures accumulate; once K present, cap is valid) is the underlying semantic.

For UI: each signer's interface should show "pending signatures from [list]" and let the user view the proposed cap before signing. Hardware-token UX (each signer on their own YubiKey) is per-signer signing, orchestrated by the application.

---

## 6. Composition with the identity / quorum extensions

The identity and quorum extensions also use K-of-N signing — but at the **entity level**, not the cap level. The two layers exist in parallel.

| Aspect | Multi-sig primitive (this guide) | Entity-layer K-of-N (`EXTENSION-QUORUM` v1.1) |
|---|---|---|
| What's signed | Capability tokens (cap chains) | Non-cap entities — `system/attestation`, `quorum-update`, `quorum-publish`, governance attestations, votes |
| Validated by | V7's `verify_capability_chain` (M4 multi-sig branch) | `QUORUM.verify_k_of_n_signatures` over an entity hash |
| Local-peer rule | Local peer MUST be in signers AND signed (M6) | No local-peer constraint (attestations propagate to contacts who validate against cached state) |
| Quorum mutability | Signers fixed in the cap shape; rotation = re-issue cap | Signers tracked by `system/quorum` + `quorum-update` chain; identity-resolved mode absorbs constituent rotation |
| Use cases | Joint accounts, escrow, threshold approval, multi-device authority, login / per-op approval (via short-TTL fresh issuance), recovery clusters under A1 (V7-only) | Identity recovery + cert chain (v3.3 `system/quorum`); group governance (admin-set, federation); cluster vote sets (planned); transaction approvals (planned) |
| Storage path | `system/capability/grants/multi-sig-root/{cap_hash}` | `system/quorum/{quorum_id_hex}` (entity); `system/quorum/{q}/event/{hash_hex}` (self-events); identity certs at `system/identity/{tier}/cert/{h}` |
| Spec / guide | `PROPOSAL-MULTISIG-CORE-PRIMITIVE.md` + this guide | `EXTENSION-QUORUM.md` v1.1 + GUIDE-QUORUM.md |

**They never mix.** The identity extension's recovery quorum is NOT implemented as a multi-sig cap (the local-peer-in-signers rule wouldn't fit — Bob's runtime peer isn't in Alice's quorum). Conversely, multi-sig caps are NOT implemented as `system/quorum` entities (caps are V7 capability tokens, not non-cap entities). Each mechanism is appropriate for its own use cases.

When you read about "K-of-N" in entity-core, ask which layer is in play:
- Cap-level K-of-N (capability tokens; local-peer rule applies) → this guide.
- Entity-level K-of-N (attestations, votes, snapshots; no local-peer rule) → GUIDE-QUORUM.md.

### 6.1 The signer-resolution trade-off lives in GUIDE-QUORUM

The identity-resolved-vs-concrete trade-off (transparent rotation vs isolation; coupling vs explicit update friction) applies to entity-layer quorums only — multi-sig caps don't have signer-resolution modes. See `GUIDE-QUORUM.md` §3 for the trade-off and recommendation.

### 6.2 Arbiter — when to pick which layer

A common decision point. Compact form here; the comprehensive matrix is at `GUIDE-QUORUM.md` §9.

| Use case | Pick |
|---|---|
| Joint account / escrow / dual control | **Cap-level multi-sig** (this guide) |
| Threshold approval (board votes on a single action) | **Cap-level multi-sig** §4.4 |
| Login / per-op fresh consent (short-TTL fresh issuance per action) | **Cap-level multi-sig** §4.6 |
| Recovery cluster — V7-only deployment (no identity) | **Cap-level multi-sig** §4.3 (Architecture A1) |
| Recovery cluster — identity-installed deployment | **Entity-layer quorum** (GUIDE-QUORUM + GUIDE-IDENTITY §11) |
| Identity cert chain, rotation, retirement | **Entity-layer quorum** (identity composes) |
| Group governance, federation membership | **Entity-layer quorum** (group composes) |
| Cluster vote sets (planned) | **Entity-layer quorum** (cluster composes) |

**Two heuristics:**
- **Are you signing a capability token?** → cap-layer (this guide). Local-peer rule applies.
- **Are you signing a non-cap entity (attestation, vote, snapshot)?** → entity-layer (GUIDE-QUORUM).

---

## 7. Pitfalls and antipatterns

### 7.1 Treating multi-sig as transaction-time approval

The most common misconception. "We need every transaction to require K-of-N approval, so we use multi-sig caps." No — multi-sig caps are issuance-time. Once issued, the grantee uses the cap normally. Per-transaction approval is handler-level K-of-N.

If your spec says "every funds transfer requires the board to vote," reach for a handler that requires K signatures attached to each `transfer` request. Not for multi-sig caps.

### 7.2 Issuing multi-sig caps to grantees not in signers

Allowed by the schema, but the cap will be **unusable**. The grantee can hold the cap, but the local-peer-in-signers rule means the cap is only valid at peers that are in the `signers` set. If the grantee's peer isn't in `signers`, no peer accepts the cap as a root.

The only sensible grantee for a multi-sig cap is one of the `signers` themselves (or, mechanically, a different peer ID held by the same operator that controls one of the signers).

### 7.3 Trying to do mid-chain multi-sig

A multi-sig cap with a non-null `parent` is rejected at content validation (M3). Multi-sig caps are root-only.

Common attempt: "I want to delegate from a multi-sig root to another multi-sig group." This requires either polymorphic grantee or a cluster-identity mechanism, neither of which the primitive supports. Defer until the use case forces a separate amendment, or use single-sig delegation from the multi-sig grantee.

### 7.4 Expecting constituent-identity revocation to invalidate the cap

The multi-sig verifier is **identity-hash-based, not identity-status-based.** It consults the keypair hashes embedded in the cap's `signers` field and the K signatures. It does NOT check whether constituent signer identities have been independently rotated, attested-against, or "revoked" outside the cap (M11).

This means: if Bob's keypair is compromised and he rotates his personal Op, an existing 2-of-3 cap signed by [Alice, Bob] still validates — Bob's old signature on that specific cap remains a valid Ed25519 signature against the keypair hash recorded in `signers`. The cap is locked to the keypairs that signed it, not to the live identity status of those keypairs' owners.

Operational recourse for compromised constituents is **revoke-and-reissue at the cap level**: the cap's root tree binding is removed (V7 §5.1 standard mechanism), and a new multi-sig cap is issued with the rotated keypairs. Plan for this when designing cap lifecycles — don't assume constituent rotation propagates to outstanding caps.

### 7.5 N too small or too large

- **N=1 with K=1.** Use a single-sig cap. The schema rejects N=1 multi-sig as malformed.
- **K=N (all-or-nothing).** Valid, but operationally fragile — losing any single key invalidates all caps. Consider K<N for tolerance (a 2-of-3 survives one loss).
- **N>32.** The recommended ceiling. Implementations MAY refuse N>32 caps for DoS resistance. If your use case genuinely needs more, document and verify both implementations support it.

### 7.6 Conflating with threshold cryptography

FROST, BLS threshold signatures, and similar cryptographic schemes produce a **single** signature output that's cryptographically threshold-secured. They look like single-sig at the protocol level — one signature, one signer (the conceptual group key).

Multi-sig caps in this primitive are NOT threshold cryptography. The K signatures are individual Ed25519 signatures from each constituent, all carried in the cap's signature entities. Each is independently verifiable and individually attributable. The protocol sees K signatures, not one.

If you need threshold cryptography, build it as an application-layer extension that produces single-sig output (the threshold-signed group key acts as a single signer). It's orthogonal to this primitive — they don't conflict, and either or both can be used in a deployment.

---

## 8. Reference

### 8.1 Schema (V7 §3.6 with primitive applied)

```
system/capability/token := {
  fields: {
    granter:  { union_of: [
      { type_ref: "system/hash" },                         ; single-sig (CBOR bstr)
      { type_ref: "system/capability/multi-granter" }      ; multi-sig (CBOR map)
    ]}
    grantee:  { type_ref: "system/hash" }                   ; single-hash, unchanged
    parent:   { type_ref: "system/hash", optional: true }   ; null when granter is multi-granter (M3)
    grants:   ...
    ...
  }
}

system/capability/multi-granter := {
  fields: {
    signers:    { array_of: { type_ref: "system/hash" } }   ; constituent peer IDs, N≥2, no dups
    threshold:  { type_ref: "primitive/uint" }              ; K, in [2, N]
  }
}
```

### 8.2 Validity constraints (M3)

- N ≥ 2 (use single-sig form for N=1).
- K ∈ [2, N] (K=0 invalid, K=1 invalid, K>N invalid).
- No duplicate `signers` entries.
- `parent` MUST be null for multi-sig (root-only restriction).

### 8.3 Verification rules (V7 §5.5 with primitive applied)

- **Site 1 (per-link):** Branch on granter shape. Multi-sig path finds K valid signatures from `signers` (identity-hash-based — verifier does not consult constituent identity status; M11); deduplicates defensively.
- **Site 3 (root):** Local peer MUST be in `signers` AND must have signed (M6).
- **Site 4 (`check_creator_authority`):** Strict-with-signature — writer matches single-sig granter, OR writer is in multi-sig signers AND signed at this link (M7).

### 8.4 Wire encoding (M8)

- Single-sig granter: CBOR byte string (major type 2).
- Multi-sig granter: CBOR map (major type 5) with `signers` and `threshold`.
- **No CBOR tags.** The encoding spec (ENTITY-CBOR-ENCODING.md §11) prohibits data-field tags. Decoders branch on major type.

### 8.5 Storage path (M12)

```
system/capability/grants/multi-sig-root/{cap_hash}
```

Standard tree-binding revocation applies — `is_revoked` checks the binding at this path.

### 8.6 Spec pointers

- PROPOSAL-MULTISIG-CORE-PRIMITIVE.md — full proposal, items M1–M12, conformance, test vectors
- ENTITY-CORE-PROTOCOL.md §3.6 — capability schema (post-primitive)
- ENTITY-CORE-PROTOCOL.md §5.5 — chain verification, four sites
- ENTITY-CORE-PROTOCOL.md §5.6 — `is_attenuated` (unchanged; M10 attenuation rule)
- ENTITY-CBOR-ENCODING.md §11 — CBOR tag prohibition (why M8 uses structural distinction)
- EXPLORATION-MULTISIG-CORE-PRIMITIVE.md — established-practice framing, three-architecture distinction for downstream consumers
- EXPLORATION-RECOVERY-CLUSTER-MECHANICS.md — cluster Architecture A1/A2/A3 framings
- GUIDE-IDENTITY.md §11.4 — recovery cluster as a downstream consumer (under Architecture A1)
- GUIDE-CAPABILITIES.md — single-sig cap fundamentals (this guide assumes you've read it)
