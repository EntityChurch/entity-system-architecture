# PROPOSAL — NETWORK live-establishment seam (§10.3)

**Status:** **FOLDED 2026-07-31 — and folded *before* this proposal existed. See "Process deviation" below.**
Landed as `specs/extensions/EXTENSION-NETWORK.md` **v1.6, Amendment 14** (new §10.3; new §10 step 3b; a
correction to §10.2's forward-looking paragraph).
**Target:** `EXTENSION-NETWORK.md` — one new seam, one new ladder step, two MUSTs on the returned connection,
one correction. No V7/wire renumber, no new capability, no new error code.
**Scope:** the dispatch-ladder slot a NAT-traversal policy plugs into. **Not** the traversal protocol itself —
that is `EXTENSION-SIGNALING.md` §7.
**Cohort review: HAD, 2026-07-31 — two independent builds, all four open items closed, the seam survives with
four additions (`ctx`, stream semantics, the identity-check placement, the retry composition).** See §6. The
process deviation below stands as recorded; the review it was written to invite has now happened.

---

## 0. Process deviation — recorded, not hidden

**This change was folded directly into a landed spec without a proposal first.** `AGENTS.md` is explicit:
*"Significant or normative/protocol changes are proposal-first, not a direct edit (wording-only hygiene can go
direct)."* A new dispatch-ladder step with a normative ordering MUST, binding all three implementations, is not
wording-only hygiene. It should have been a DRAFT first.

This document is therefore written **after the fact** as the design record the fold should have had, so that:

- the reasoning is reviewable rather than buried in a spec diff and a commit message;
- the cohort has something to disagree with **before building it** (§6);
- the deviation is on disk rather than in a transcript.

**It is not back-dated and it is not presented as prior authorization.** If the cohort's review rejects the
shape, the spec edit comes back out — the fold does not get to stand on having happened first.

## 1. The gap

`EXTENSION-NETWORK.md` §10's dispatch ladder resolves **durable** transport profiles (step 3). A peer behind NAT
has no dialable durable profile — its published endpoints are unreachable from outside — so the ladder falls
straight through to step 4's store-and-forward terminal, **even when the peer is reachable right now** by a hole
punch.

Nothing in §10 offered a place to try. That is the gap.

## 2. Why §10.2's existing seam does not fill it

§10.2 (Amendment 11) named `dispatch_fallback(peer_id, execute) → {ok, result} | null`, and its forward-looking
paragraph claimed the punch would ride the same site:

> *"Naming the seam now gives both deferred axes a single home at §10 step 4: the asynchronous floor … and,
> later, the live-connection punch … A mature dispatcher tries live first and store-and-forward last; both
> escalate from the **same** step-4 site rather than as parallel systems."*

**That paragraph predates the punch's design and does not survive it.** Three independent reasons:

1. **Wrong return type.** `dispatch_fallback` returns a **delivered result**. Traversal produces a **connection**,
   after which the ladder must re-enter and dispatch normally. Forcing traversal through this signature means the
   policy punches, dispatches once, and returns a result — **a fresh hole per message**, with the connection
   invisible to §10 step 1's active-connection check and to the pool.
2. **Wrong position.** The escalation belongs where the *reachability* question is answered — after durable
   profiles fail (step 3), not at the delivery terminal (step 4).
3. **Self-contradiction.** "Tries live first and store-and-forward last" is unachievable from **one seam consulted
   once at one site**. The paragraph asserts an ordering its own mechanism cannot express.

## 3. The proposal

```
establish_live(peer_id) → connection | null
```

- Consulted **once, at a new §10 step 3b** — after durable-profile resolution has failed to connect, and
  **before** §10.2's `dispatch_fallback`.
- `null` when no traversal policy is registered ⇒ the ladder falls through to step 4, **byte-identical to the
  pre-seam behavior**. Additive, no regression, no v1 floor change.
- Returns a **connection**, so the ladder re-enters ordinary dispatch and the connection is pooled and reused.

**Ordering is normative (MUST):** `establish_live` before `dispatch_fallback`. A dispatcher consulting them in the
other order store-and-forwards to a peer it could have reached directly — correct in outcome, wrong in cost, and
it silently defeats the traversal path entirely.

**Two obligations on the returned connection (MUST):**

1. **It is an ordinary transport** — indistinguishable to the entity layer from a dialed `tcp` connection; slots
   into the §10 full-duplex-listener class; **MUST NOT** be published as a durable `system/peer/transport/*`
   profile, because the mapping behind it is session-scoped (§6.7.3).
2. **It MUST run keepalive (§5).** A punched NAT mapping expires on silence and the hole closes, so an idle
   punched connection **dies silently** — a "worked, then dropped" bug that no same-host test reproduces.

## 4. Layering

Identical to §10.2's, and to the relay→routing `resolve_next_hop` precedent (`EXTENSION-ROUTE.md` §4): **NETWORK
is below the traversal extension and cannot call into it, so the substrate exposes the slot and the extension
plugs the algorithm.** In the strictest implementation this is a compile-time constraint (an inline "if signaling
present" guard would invert the crate dependency graph), not a preference.

NETWORK owns: the slot, the ordering, and the returned connection's obligations. It owns **nothing** about how the
connection was obtained — candidate exchange, simultaneous-open timing, carrier selection, and connectivity checks
all belong to the policy.

## 5. What this is not

- **Not the traversal protocol.** That is `EXTENSION-SIGNALING.md` §7, which registers behind this seam.
- **Not a replacement for §10.2.** The two are complementary and both are consulted: §10.3 answers *can I reach
  this peer now*, §10.2 answers *how do I deliver when I cannot*.
- **Not a new transport type.** See §3 obligation 1.

## 6. Open items — **CLOSED 2026-07-31 by cohort review**

Review happened the same day this record was written, and it happened the strongest way available: **two
implementations built the seam** (`entity-core-go` and `entity-core-rust`) and reported against the committed
text, independently of each other. All four items are answered. **The seam survives** — no item came back as a
rejection of the shape, and the spec edit does not come out.

| # | Item | Resolution |
|---|---|---|
| 1 | **The seam name and signature.** | **Confirmed, with `ctx` added.** Both implementations report `connection` fell out of their existing peer layer with nothing invented — so leaving it undefined is *correct layering*, not under-specification, and that is now stated in §10.3 rather than merely practiced. **But the two builds put the handshake on opposite sides of the seam** (Go returns a post-handshake connection; Rust returns pre-handshake and runs the handshake in its shared adopt-path). That divergence is **internal to one peer and invisible on the wire — both interop**, so it is blessed as impl-idiomatic rather than pinned; forcing either factoring would make one implementation restructure for zero interop benefit. Both raised the missing **`ctx`/deadline** independently: a seconds-long seam with no cancel is indistinguishable from a hang. Signature is now `establish_live(ctx, peer_id)`. |
| 2 | **Is one call enough?** | **Yes — confirmed, and now on better grounds than "simple."** Both implementations independently rejected racing the seam against §10.2: the two seams live at different sites with different signatures, and racing them forces a layering regression. One call, strict ordering, unchanged. |
| 3 | **Interaction with `maintain-peer` (§4.1).** | **The one item that produced new normative text.** Rust found what Go's own review had missed: §7.2's 3-attempt punch budget composes *multiplicatively* with §4.1's reconnect backoff, so a symmetric-NAT peer's reconnect loop spends *retries × 3* punches — each a fresh reflector round trip and carrier exchange against **third-party** infrastructure. Ruled: **exactly one retry authority per escalation path, never nested** — maintain-peer-driven ⇒ one attempt, §4.1 owns retry; standalone dispatch ⇒ §7.2's budget. §10.3 obligation 4 + `EXTENSION-SIGNALING.md` §7.2. |
| 4 | **§10.2's paragraph.** | **Framing confirmed**, no objection from either implementation; corrected-in-place stands. |

**One item review added that this proposal did not anticipate:** the **identity check's placement**. Granting
freedom over the handshake boundary (item 1) silently grants freedom to pool an unverified connection, which is a
security regression the freedom was never meant to include. §10.3 obligation 3 now pins it — verification MUST
complete before the connection enters the pool, wherever an implementation chooses to run it. *This is the shape
the exploration catalog calls an equivalence-collapse: two factorings that are equivalent for the issuer, with a
distinction that springs apart the moment something else depends on the ordering.*

## 7. Validation

**Build state (peer-reported, observed 2026-07-31, post-review):** `entity-core-go` has built the seam
(`core/peer.tryEstablishLive` + `ext/signaling/peerwiring`, driven end-to-end with a `RemoteExecute` landing over a
punched connection and a failed punch falling through to relay); `entity-core-rust` has built its `LiveEstablish`
counterpart; Python has not. **Both are loopback / in-process — the choreography is proven, NAT traversal is
not.** The gate — two NAT'd peers establishing a direct live transport through this seam that carries ordinary
operations and survives idle — **has not run**, and needs live infrastructure rather than more specification.

*(The pre-review line here read "the seam is present in no implementation," written the morning of the day both
builds were reported. Left visible rather than silently swapped: this proposal's whole subject is a fold that ran
ahead of its review, and the build-state claim ran ahead of its evidence the same way.)*

## 8. References

- **Folded into:** `specs/extensions/EXTENSION-NETWORK.md` v1.6 §10.3 (Amendment 14), §10 step 3b, §10.2.
- **The policy that registers behind it:** `specs/extensions/EXTENSION-SIGNALING.md` §7.
- **Layering precedent:** `EXTENSION-NETWORK.md` §10.2 (Amendment 11); `EXTENSION-ROUTE.md` §4.
- **Process rule deviated from:** `AGENTS.md` — "Significant or normative/protocol changes are proposal-first."
