# PROPOSAL — cross-algorithm identity migration (F-PQ)

**Status:** DRAFT (2026-07-21) · **foundational, NOT before-freeze** (long-horizon; scopes the problem +
design direction, does not yet pin wire).
**Target:** `specs/extensions/EXTENSION-IDENTITY.md` (§4.4 rotation-recovery, §3 quorum reference) +
`specs/extensions/EXTENSION-QUORUM.md`.
**Provenance:** keystone **F-PQ** (`HANDOFF-TO-ARCH-2026-07-19-convergence-and-substrate-review`), independently
re-derived by keystone's convergence survey (a did:plc-style rotation op-log as a second valid design).
Triaged in `docs/research/reviews/ABSORPTION-keystone-findings-F31-F46-and-named-handoffs.md §3` ("EXTENSION,
not core"). Supersedes the keystone-named placeholder "PROPOSAL-MULTIKEY-MULTIHASH-ALIGNMENT" (which had no
file in this repo).
**Scope:** identity-layer only. Core already provides crypto-agility (V7 §9.3 `key_type`/`hash_type`, a MAY);
this proposal specifies how an **identity** (a quorum-rooted cert graph) migrates *across* algorithms — the
piece agility alone does not give.

---

## 1. Problem — agility ≠ recoverability under a broken primitive

V7 crypto-agility (§9.3) lets *a peer* use a chosen `key_type`. It does **not** answer what happens to a
**multi-peer identity** when the algorithm its keys use is **broken** (the post-quantum threat, or any future
break). Two concrete gaps keystone surfaced:

1. **The recovery quorum is not required to sit on an unbroken algorithm.** `EXTENSION-IDENTITY §4.4`
   (`identity-rotation-recovery`) lets a quorum recover a lost/compromised controller. But nothing requires the
   **quorum's own keys** to be on a *different, unbroken* algorithm from the keys it recovers. If an identity's
   quorum constituents and its controllers are **all Ed25519**, and Ed25519 is broken, the recovery path is
   broken too — the quorum can be forged exactly when it is needed. Recovery has no algorithmic diversity
   requirement.
2. **Cross-algorithm rotation is unspecified.** There is no defined ceremony to migrate an identity from
   algorithm A to algorithm B — rotate every cert, re-root the quorum on B-keys, and preserve continuous
   cross-peer recognition (contacts must still recognize the identity across the cut). `§4.3`
   (`identity-rotation-handoff`) rotates *keys*; it does not contemplate rotating the *algorithm*.

Neither is a core-protocol change — the wire already carries `key_type` per key; this is identity-layer
topology + ceremony.

## 2. Design direction (not yet pinned)

Two invariants + one ceremony, to be demonstrated before landing:

- **RM-1 — recovery-algorithm diversity (the load-bearing invariant).** An identity's **recovery path** MUST be
  anchored on at least one algorithm **distinct from** the algorithm(s) protecting its hot/controller keys — so
  a break in the hot algorithm does not break recovery. Concretely: a K-of-N recovery quorum SHOULD hold ≥1
  constituent on a PQ/unbroken `key_type`, or a designated recovery cert MUST be on such a key. State the
  invariant generally (agility gives the *mechanism*; this gives the *topology requirement* agility omits).
- **RM-2 — continuity across the cut.** Cross-algorithm rotation MUST preserve identity **recognition**: the
  post-migration identity is provably the same identity to any contact that held the pre-migration one — via a
  signed migration event that both the old-algorithm and new-algorithm keys attest (a dual-signed handoff), so
  a verifier with only the old view can still authenticate the transition.
- **The ceremony — a cross-algorithm `identity-rotation-handoff` variant.** Extend `§4.3` with an
  algorithm-migration mode: the quorum (or a recovery cert satisfying RM-1) issues a handoff whose new certs
  are on algorithm B and which is **co-signed by both A and B keys**; contacts fold it exactly like a normal
  handoff. Keystone's convergence survey independently arrived at a **did:plc-style rotation op-log** as a
  second valid shape (an append-only, self-certifying log of rotation ops) — evaluate both; the op-log gives
  stronger auditability, the dual-signed handoff reuses the existing cert machinery.

## 3. Open questions

1. **Where RM-1 is enforced** — as an `EXTENSION-QUORUM` validity rule (a recovery quorum MUST declare its
   algorithm set) vs. an `EXTENSION-IDENTITY` topology rule (`identity_topology_for`). Lean quorum-level (it is
   a property of the signer set), with identity imposing the recovery-specific requirement.
2. **PQ `key_type` allocation** — which PQ algorithm(s) get canonical `key_type`/`hash_type` allocations
   (V7 §1.5 table already reserves the space; the identity layer consumes it). Coordinate the allocation with
   core, but the *identity* migration model is independent of *which* PQ algorithm.
3. **Retirement interaction** — how `identity-retirement` (§4.5) composes with a migration (retire-the-old vs.
   supersede-in-place). Lean supersede (RM-2 continuity).
4. **Hash-agility parallel** — the same diversity argument applies to `hash_type` (a broken hash breaks
   content-addressed identity binding). Note it; the keystone "multikey-multihash-alignment" framing is exactly
   this parallel — a migration model MUST cover both the signature algorithm and the hash.

## 4. Non-goals

- **Not a core-protocol change.** Core provides the agility mechanism (§9.3); this is identity topology +
  ceremony. Any core touch (a new PQ `key_type` allocation) is a separate, coordinated core item.
- **Not before-freeze.** Foundational and long-horizon; it does not gate the core freeze. Landing waits on a
  worked demonstration (a migration ceremony exercised across a break simulation) per the conformance bar.

## References

- `specs/extensions/EXTENSION-IDENTITY.md` §3 (quorum reference), §4.3 (rotation-handoff), §4.4
  (rotation-recovery), §4.5 (retirement); `specs/extensions/EXTENSION-QUORUM.md` (K-of-N, signer resolution).
- Determination: `docs/research/reviews/ABSORPTION-keystone-findings-F31-F46-and-named-handoffs.md §3`.
- Keystone: `HANDOFF-TO-ARCH-2026-07-19-convergence-and-substrate-review.md` (F-PQ + the did:plc-style
  op-log re-derivation).
