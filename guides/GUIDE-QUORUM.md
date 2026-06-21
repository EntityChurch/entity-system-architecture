# Quorum: User Guide

**Status**: Active
**Audience:** Application developers and consumer-extension authors who need K-of-N entity authorization. Assumes familiarity with V7 (peers, signatures, the tree), GUIDE-ATTESTATION (the attestation substrate), and GUIDE-CAPABILITIES.
**Spec reference:** `EXTENSION-QUORUM.md` v1.1.
**Related guides:** GUIDE-ATTESTATION.md (the substrate this composes with), GUIDE-IDENTITY.md (the largest current consumer), GUIDE-MULTISIG.md (cap-layer K-of-N — different layer; see §8 arbiter rule for when to use which).

---

## 1. What a quorum is

A `system/quorum` is the **K-of-N node primitive** — a special node in the signed graph whose "signature" is K signatures from N constituent peers, rather than a single keypair's signature.

```
system/quorum := {
  signers:           [<peer_hash>, ...]      ; N constituents
  threshold:         <K>                     ; K-of-N
  signer_resolution: "concrete" | "identity-resolved" | <custom>?
                                             ; default "concrete" — entries are
                                             ; specific peer hashes
                                             ; "identity-resolved" — entries are
                                             ; references to identities; resolver
                                             ; looks up the current controller
  name:              <string>?               ; optional
  metadata:          <any>?                  ; optional
}
```

Stored at `system/quorum/{quorum_id_hex}` (where `quorum_id` is the entity's own content_hash). The quorum entity is structural and not itself signed — authorization for the quorum's role flows from its constituents collectively K-of-N-signing other entities (top-level identity certs, quorum-update events, cluster votes, transaction approvals, etc.).

**Why a substrate primitive.** Identity, group, cluster, transaction, governance, and any K-of-N-needing extension all needed the same machinery: a roster + threshold; a K-of-N validator over an entity hash; a way to walk the chain of membership changes; a way to publish current state for off-peer consumers. EXTENSION-QUORUM provides those once. Identity is the largest current consumer (its recovery quorum); group consumes via `identity-resolved` mode; cluster will consume.

**Quorum vs cap-layer multi-sig.** Don't confuse the two. The cap-layer multi-sig primitive (V7 + GUIDE-MULTISIG) provides K-of-N at the **capability-issuance layer** — joint authority where each participant's cap stays locally rooted. EXTENSION-QUORUM provides K-of-N at the **entity layer** — non-cap entities (attestations, votes, snapshots) signed K-of-N. They never mix; see §8 for the arbiter rule on which to use when.

## 2. Creating a quorum

```
EXECUTE
  resource: system/quorum/{quorum_id_hex}
  operation: system/quorum:create
  params: {
    signers:           [<peer_a_hash>, <peer_b_hash>, <peer_c_hash>]
    threshold:         2                       ; 2-of-3
    signer_resolution: "concrete"              ; default; can omit
    name:              "alice-recovery-quorum"
  }
  ; result: {quorum_id: <hash>}
```

**Path-as-resource is MUST.** The resource targets the canonical storage path. Calling `:create` without a resource returns `path_required`.

**The quorum entity is structural and not signed.** Authorization for `:create` is per Coherent Capability — application-level grants cover `system/quorum:create`, not raw `tree:put` to `system/quorum/...`.

## 3. Choosing a signer-resolution mode

**`concrete` mode (default).** Each entry in `signers` is a specific peer-identity hash. K-of-N verification matches against those exact identities. If a constituent rotates their keypair (their peer hash changes), their slot is **invalidated** until an explicit `quorum-update` swaps the new hash in.

**`identity-resolved` mode.** Each entry is an identity reference. The resolver (registered by EXTENSION-IDENTITY) looks up the identity's **controller** at the validation moment (or at `as_of` for time-travel), and that controller's peer hash participates in the K-of-N count. Constituent key rotation is **transparent** — the quorum's signer slot keeps pointing at the same identity; identity's internal cert chain handles rotation.

**Which to choose** (lifted from `GUIDE-MULTISIG.md` §6.1, updated for v3.3 vocabulary — same trade-off applies at the entity-layer K-of-N):

> **Trade-off: identity-resolved vs concrete in cross-quorum contexts.** Identity-resolved buys transparent rotation: a constituent's keypair rotation is invisible to the quorum (resolver picks up the identity's current controller at issuance). The cost: the quorum's constituency is coupled to each constituent's identity continuity. If a constituent loses their identity (quorum-below-threshold; identity dies), this quorum has lost a member — even though the constituent's recovery process is independent. Concrete buys isolation: the quorum points at specific keypairs, independent of constituents' identity flows. The cost: each constituent's keypair rotation requires an explicit `quorum-update` to swap the new hash in.
>
> **Recommendation:** identity-resolved for federations and dynamic-membership groups (constituents rotate keys frequently; explicit quorum-update friction outweighs coupling risk); concrete for high-stakes recovery quorums (deliberate update on rotation is acceptable; isolation from constituents' identity flows is desired). Identity's own recovery quorum uses `concrete`.

**Other modes.** Extensions can register custom modes via `system/quorum:register_resolver(mode_name, resolver_handler)`. The resolver is `(signer_ref, ctx, as_of?) → resolved_peer_hash`; deterministic and side-effect-free. Bounded recursion: `max_resolver_depth = 8` (per spec §5.2) with cycle detection (`identity_resolver_cycle` error).

## 4. Updating a quorum

`system/quorum:update` produces a `quorum-update` attestation — the substrate edge that records "the quorum's membership changes to this":

```
EXECUTE
  resource: system/quorum/{quorum_id}/event/{update_hash_hex}
  operation: system/quorum:update
  params: {
    quorum_id:      <Q>
    new_signers:    [<peer_d>, <peer_b>, <peer_c>]   ; replacing peer_a with peer_d
    new_threshold:  2
    supersedes:     <prev_quorum_update_hash>?       ; or null on first update
  }
  ; result: {update_hash: <hash>}
  ; signatures gathered K-of-N from the CURRENT effective quorum constituents
  ; (NOT from new_signers); arrive in envelope.included
```

A `quorum-update` is a `system/attestation` per `EXTENSION-ATTESTATION` with `properties.kind = "quorum-update"`. Stored at `system/quorum/{q}/event/{hash_hex}`. Per-quorum supersedes chain — the new update supersedes the previous quorum-update (or carries `supersedes: null` for the initial one). The effective signer set after a `quorum-update` is `(new_signers, new_threshold)`.

**Identity is a consumer, not a definer.** When an identity's quorum membership changes, identity's controller (or a tool acting on its behalf) calls `system/quorum:update` on the identity's `trusts_quorum`. Identity does NOT define its own quorum-update kind. Same for group, cluster, etc. — all consumers route through the substrate ops.

## 5. Publishing quorum state for contact-side discovery

`system/quorum:publish` produces a `quorum-publish` attestation — a contact-visible snapshot of current quorum state used by external peers as a trust anchor (e.g., for compromise-recovery validation):

```
EXECUTE
  resource: system/quorum/{q}/event/{publish_hash_hex}
  operation: system/quorum:publish
  params: {
    quorum_id:        <Q>
    signers:          [<peer_a>, <peer_b>, <peer_c>]
    threshold:        2
    published_handle: <peer_a_hash>?              ; consumer-extension hook
                                                  ; (identity sets to controller's
                                                  ;  key in 3-key default)
    supersedes:       <prev_publish_hash>?
  }
```

**The "previous pinning" rule (normative, spec §3.3).** Initial publish: signed K-of-N by the **current** quorum constituents. Supersede: signed K-of-N by the **previous** quorum (the prior publish's `properties.signers` snapshot). This pins the supersedes chain to cached prior state — a contact who cached `publish_v1` can validate `publish_v2 supersedes publish_v1` against the snapshot they have, independent of intervening `quorum-update` events.

This is generic, not identity-specific. Any consumer that caches prior `quorum-publish` state externally (federations, public registries, etc.) needs the cryptographic continuity to validate successor publishes.

**`published_handle` and other extra `properties` keys are consumer-extension hooks.** EXTENSION-QUORUM doesn't interpret them. Identity sets `published_handle` to the contact-side handle (the controller's key in 3-key default; the identifier's key in 4-key advanced). Other consumers may set their own keys for their own purposes.

## 6. Validating K-of-N over an entity

```
QUORUM.verify_k_of_n_signatures(entity_hash, signer_set, threshold, ctx) → bool
```

Counts valid signatures from `signer_set` over `entity_hash`. Returns true once `threshold` valid signatures are counted. Composes substrate-level `find_signature_by_signer` + `verify_signature` (per `EXTENSION-ATTESTATION` §4.0).

You typically don't call this directly — you call `current_signer_set(quorum_id, ctx)` first to get `(signers, threshold)`, then pass that to the verifier. Or you call `system/quorum:verify`, the helper-wrapping op:

```
EXECUTE
  resource: system/quorum/{q}/event/{publish_hash_hex}
  operation: system/quorum:verify
  params: {entity_hash: <h>, quorum_id: <Q>}
  ; result: {valid: bool, signed_by: [<hash>, ...]}
```

## 7. Walking the signer set with `current_signer_set`

`current_signer_set(quorum_id, ctx, as_of?)` returns `(signers, threshold)` at a moment in time. The walk:

1. Reads `system/quorum/{q}` for initial signers + threshold.
2. Looks up live `quorum-update` attestations targeting `quorum_id` (via ATTESTATION's `find_attestations_targeting`).
3. Walks to the live head (per `as_of` if provided) — the most recent quorum-update whose chain remains live.
4. Applies the signer-resolution mode: `concrete` returns hashes as-is; `identity-resolved` calls the registered resolver per signer (resolver receives `as_of` — historical resolution required).

**Time-travel matters for cap chain validation.** A cap signed under `quorum_v1` should remain validatable after `quorum_v2` lands. Pass `as_of = cap_issuance_timestamp`; the resolver returns the controller live at that time. Without `as_of` propagation, identity-resolved quorums break in mid-flight signing across rotations (this is exactly the bug the v3.3 cross-impl batch closed).

**Cache invalidation contract** (spec §4.2.1, normative — implementations MUST satisfy):

| Trigger | Invalidate cache? |
|---|---|
| Successful local `system/quorum:update` or `:publish` for `quorum_id` | YES |
| Validated `quorum-update` / `quorum-publish` arrival via sync (passes K-of-N) | YES (on validate-accept moment, NOT on raw tree-write) |
| Live revocation of a previously-seen `quorum-update` / `quorum-publish` | YES |
| Failed validation (insufficient sigs, wrong signer set) | NO |
| Raw `tree:put` bypassing handler validation | NO |
| Activity on a different `quorum_id` | NO |

Two implementation patterns satisfy this (spec §4.2.1):
- **Sync-hook:** a hook fires on every tree-write of a relevant attestation, validates K-of-N inline, invalidates on success.
- **Routed-handler:** sync arrivals and ingestion paths route through a `process_quorum_attestation`-equivalent op that validates + invalidates atomically.

Cross-impl conformance verifies the post-state via TV-QF12–TV-QF15 regardless of mechanism.

## 8. Identifying a quorum: `is_quorum_id`

```
QUORUM.is_quorum_id(hash, ctx) → bool
```

Returns true iff `hash` refers to a `system/quorum` entity bound at `system/quorum/{hex(hash)}` on the local tree. **Path-based lookup, not content-store-only** — a quorum is "known" only if it's bound at its canonical path locally.

Used by identity's chain-walk terminate predicate (`identity_is_quorum_link`) and topology dispatch. **Race during bootstrap:** if `is_quorum_id` is called before the relevant quorum entity has been written (sync catch-up; bootstrap ordering), it returns false and dispatch falls through. Cert validation surfaces an appropriate failure (e.g., `chain_to_quorum_not_found`); implementations MAY retry validation after observing the relevant write but MUST NOT cache "not a quorum" status across writes. The primitive is stateless.

## 9. When to use entity-layer quorum vs cap-layer multi-sig (arbiter rule)

Both provide K-of-N, but at different layers. Pick by use case:

| Use case | Pick |
|---|---|
| Joint account (two parties co-sign every transfer) | **Cap-level multi-sig** — see GUIDE-MULTISIG §4.1 |
| Escrow / dual control (M-of-N humans approve a release) | **Cap-level multi-sig** — GUIDE-MULTISIG §4.2 |
| Threshold approval (board votes on funds movement) | **Cap-level multi-sig** — GUIDE-MULTISIG §4.4 |
| Login / per-operation approval (short-TTL fresh issuance) | **Cap-level multi-sig** — GUIDE-MULTISIG §4.6 |
| Recovery cluster (deployed identity-installed) | **Entity-layer quorum** — this guide; identity composes |
| Recovery cluster (V7-only, no identity) | **Cap-level multi-sig** under Architecture A1 — GUIDE-MULTISIG §4.3 |
| Identity attestation: certify a controller or agent | **Entity-layer quorum** — identity uses K-of-N over `identity-cert` attestations |
| Identity rotation-recovery (compromise-driven) | **Entity-layer quorum** — identity uses K-of-N over the recovery attestation |
| Quorum membership change | **Entity-layer quorum** — `system/quorum:update` (this guide) |
| Group governance (federation, admin-set, all-members) | **Entity-layer quorum** — group uses K-of-N over governance attestations |
| Cluster vote (planned) | **Entity-layer quorum** — cluster will use K-of-N over vote attestations |

**Two heuristics:**

- **Are you signing a capability token?** → cap-layer (V7 + GUIDE-MULTISIG). The local-peer-in-signers rule applies.
- **Are you signing a non-cap entity (attestation, snapshot, vote)?** → entity-layer (this guide). No local-peer-in-signers rule.

The two never mix in the same chain. Identity's recovery quorum is entity-layer; if Alice also wants per-transaction co-sign on funds movement, that's a separate cap-layer multi-sig cap.

## 10. Pitfalls

- **Don't validate quorum entities via `verify_capability_chain`.** Three-parallel-mechanisms invariant: `system/quorum` entities have their own validator (`verify_k_of_n_signatures` over their self-events). Quorum entities never enter cap-chain verification.
- **Don't grant raw `tree:put` to `system/quorum/...`.** Bypasses the handler. Grant `system/quorum:create` / `:update` / `:publish` instead (Coherent Capability principle).
- **Don't skip `as_of` when validating historical signatures.** A cap-chain link signed under a prior quorum must validate against THAT quorum's signer set (identity-resolved mode resolves the controller live AT issuance, not now). Pass `as_of`.
- **Don't fall back from unknown `signer_resolution` to `concrete`.** Fail-closed required (spec §5.3.1). A typo'd or future mode name with no resolver MUST return `quorum_resolver_unavailable`, not silently re-interpret identity references as peer references.
- **Don't expect `quorum-publish` supersedes to be signed by the current quorum.** It's signed by the **previous** quorum (the prior snapshot's signer set). Continuity-of-chain is what lets contacts validate later publishes against state they cached earlier.
- **Don't rely on `is_quorum_id` returning true during sync catch-up.** It's stateless and path-based. If the quorum entity hasn't synced yet, `is_quorum_id` returns false; consumers retry after the relevant write.

## 11. Reference

**Helpers (spec §4):**
- `verify_k_of_n_signatures(entity_hash, signer_set, threshold, ctx)` — K-of-N count over an entity hash.
- `current_signer_set(quorum_id, ctx, as_of?)` — walk the chain; returns `(signers, threshold)`. Time-travel via `as_of`.
- `is_quorum_id(hash, ctx)` — path-based lookup at `system/quorum/{hex(hash)}`.

**Resolver hook (spec §5.2):**
- `system/quorum:register_resolver(mode_name, resolver_handler)` — register a custom signer-resolution mode. Handler is `(signer_ref, ctx, as_of?) → peer_hash`. Bounded by `max_resolver_depth = 8` with cycle detection.
- Built-in mode: `concrete` (default; signers ARE peer hashes).
- Identity-registered mode: `identity-resolved` (signers are identity references; resolves to current controller).

**Handler operations (spec §6, all path-as-resource MUST):**
- `system/quorum:create` — instantiate a `system/quorum` entity.
- `system/quorum:update` — issue a `quorum-update` attestation; signed K-of-N by the current signer set.
- `system/quorum:publish` — issue a `quorum-publish` attestation; initial signed by current quorum, supersede signed by **previous** quorum (the cached snapshot, not the current resolution).
- `system/quorum:verify` — convenience wrapper: `current_signer_set` + `verify_k_of_n_signatures`.

**Path layout (spec §7):**
```
system/quorum/
   {quorum_id_hex}                    ← quorum entity (structural; not signed)
   {quorum_id_hex}/event/{hash_hex}   ← quorum-update + quorum-publish attestations
                                        (kind-discriminated; same path subtree)
```

**Cross-impl test vectors:**
- TV-Q1–TV-Q5 (signer-resolution mode dispatch + fail-closed)
- TV-Q6–TV-Q9 (`is_quorum_id` semantics)
- TV-QF12–TV-QF15 (cache invalidation contract)
- TV-Q-V16a–c (historical-state resolution with `as_of`)
- TV-Q-V-IDENTITY-2 + cycle vector (resolver max_depth + cycle detection)

All in `VALIDATION-MATRIX-IDENTITY-FOUNDATIONS.md`. Implementations MUST pass every vector for conformance.
