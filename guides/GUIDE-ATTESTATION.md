# Attestation: User Guide

**Status**: Active
**Audience:** Application developers and consumer-extension authors building on the substrate. Assumes familiarity with V7 (peers, signatures, the tree) and capabilities (GUIDE-CAPABILITIES.md).
**Spec reference:** `EXTENSION-ATTESTATION.md` v1.1.
**Related guides:** GUIDE-QUORUM.md (the K-of-N companion that consumes this), GUIDE-IDENTITY.md (the largest current consumer).

---

## 1. What attestations are

An attestation is **"Peer A asserts that B has property X."** It's the **edge type in the system's signed graph**: hash-addressable entities are nodes; signed claims are edges.

```
system/attestation := {
  attesting:   <peer hash>     ; who's making the claim
  attested:    <hash>          ; what's being claimed about (any hash-addressable
                               ;   entity: peer, quorum, content, software binary, ...)
  properties:  {kind: "...", ...}   ; consumer-extensible payload; `kind` dispatches
                                    ;   interpretation
  supersedes:  <hash | null>   ; previous attestation this one replaces
  not_before:  <epoch_ms | null>
  expires_at:  <epoch_ms | null>
}
```

Every attestation is signed by `attesting` (single-sig default). Consumers requiring multi-sig (e.g., quorum K-of-N, identity dual-sig rotation) compose `verify_specific_signer` per their topology.

**Why a single primitive.** Identity, quorum, group, VC issuance, reputation, provenance, cluster, governance, audit — they all needed "Peer A asserts that B has X." Each was reinventing the same mechanics: signature validation, supersession chains, liveness checks, revocation lookups, attesting-chain walks. The substrate provides those mechanics once. Consumers add semantic interpretation via `properties.kind` conventions, topology rules, and storage paths.

## 2. Issuing an attestation

`system/attestation:create` is the canonical issuance op. It validates structural invariants and persists the entity; **signature gathering is the caller's responsibility** (per V7 envelope patterns or local signing).

```
EXECUTE
  resource: system/attestation/{computed_hash_hex}        ; path-as-resource MUST
  operation: system/attestation:create
  params: {
    attesting:  <peer P_a's hash>
    attested:   <peer P_b's hash>
    properties: {kind: "vc-issuance", credential_id: "...", claims: {...}}
    expires_at: <epoch_ms>
  }
  ; signature on the attestation's content_hash, signed by P_a's key,
  ; arrives in envelope.included; dispatcher binds it at the V7 invariant
  ; pointer path /{P_a_peer_id}/system/signature/{att_hash_hex}
```

**Path-as-resource is MUST** (per V7 §3.2 + spec §6). The resource targets the tree binding location; the params carry the attestation fields. Calling `:create` without a resource returns `path_required`.

**Storage paths are consumer-defined.** The substrate doesn't mandate a path. Identity uses `system/identity/{tier}/cert/{h}`; quorum uses `system/quorum/{q}/event/{h}`; a VC issuer might use `system/vc/issued/{h}`. The substrate stores and indexes by field — `attesting`, `attested`, `properties.kind`, `supersedes` — so graph operations work regardless of where consumers place attestations.

## 3. Verifying an attestation

The substrate's `:verify` op is a basic gate: signature-validates and runs liveness (`is_attestation_live`).

```
EXECUTE
  resource: system/attestation/{att_hash_hex}
  operation: system/attestation:verify
  params: {attestation_hash: <h>, as_of?: <epoch_ms>}
  ; result: {valid: bool, reason?: string}
```

`:verify` does NOT validate consumer-specific rules (topology, function, authority-revocation). Use it as the first gate; layer consumer-specific validation on top — this is what identity's `identity_verify_cert` does.

**Liveness has three components** (per spec §4.3):

1. **Expiration** — `not_before` ≤ now < `expires_at`.
2. **Supersession (transitive)** — no live successor in the forward supersedes graph. Walks the entire forward DAG up to `max_chain_depth` (default 256), cycle-safe.
3. **Self-revocation** — `attesting` has not issued a `kind="revocation"` attestation targeting this one.

**Authority-revocation is consumer-specific.** The substrate exposes `find_revocations_for(att_hash, ctx)`; the consumer applies its own rule. Identity says "the quorum has authority to revoke its own certs"; reputation might say "self-revocation only"; cluster might say "any sufficient quorum of voters."

## 4. The `properties.kind` open vocabulary

`properties` is a consumer-extensible map. The substrate doesn't interpret keys.

**Strongly recommended convention: include `kind`** so consumer dispatch is path-independent and so generic tooling (graph explorers, cross-extension dashboards) can dispatch optionally.

The substrate defines exactly **one** kind: `"revocation"` (universal mechanism). Every other kind is consumer-owned. The kind-ownership table in spec §3.2 is the registry; current entries:

| Kind | Owner |
|---|---|
| `"revocation"` | EXTENSION-ATTESTATION |
| `"quorum-update"`, `"quorum-publish"` | EXTENSION-QUORUM |
| `"identity-cert"`, `"identity-rotation-handoff"`, `"identity-rotation-recovery"`, `"identity-retirement"` | EXTENSION-IDENTITY |

**Naming convention.** Prefix kind names with the owning extension's domain (`"identity-cert"`, not `"cert"`). Within an extension's storage paths, kinds MUST be unambiguous; across extensions, kinds MAY collide (path scoping is the actual correctness mechanism), but namespacing is friendlier to generic tooling.

**Defining a new kind.** Pick a namespaced name. Document in your extension spec: required `properties` shape; signature topology (single-sig / dual-sig / K-of-N); storage path convention; handler operations that produce attestations of that kind. No central registry — coordinate via the kind-ownership table.

## 5. Revocation: a kind, not a separate type

Revoking attestation X means producing a new attestation:

```
system/attestation/{rev_hash} := {
  attesting:  <revoker's peer hash>
  attested:   <X's content hash>            ; X is the SUBJECT of the revocation
  properties: {kind: "revocation", reason: "..."}
}
```

`system/attestation:revoke` is a convenience wrapper:

```
EXECUTE
  resource: system/attestation/{rev_hash_hex}
  operation: system/attestation:revoke
  params: {target_hash: <X's hash>, reason?: "compromised", attesting: <revoker_hash>}
```

Two flavors of revocation, distinguished by **who** is revoking:

- **Self-revocation** (`rev.attesting == X.attesting`). Always valid; the substrate's `is_attestation_live` honors it directly.
- **Authority-revocation** (`rev.attesting != X.attesting`). Substrate doesn't endorse it on its own — it just stores and indexes the revocation. The consumer extension applies its rule when checking liveness:

```
def consumer_check_authority_revocations(att, ctx):
    revocations = ATTESTATION.find_revocations_for(att.content_hash, ctx)
    for rev in revocations:
        if not ATTESTATION.is_attestation_live(rev, ctx):
            continue
        if consumer_revoker_has_authority(rev.attesting, att, ctx):
            return REVOKED
    return OK
```

Identity's `identity_is_authorized_revoker` walks the cert chain back to the quorum and accepts revocations where `revoker == quorum_id`. Other consumers compose their own predicate.

## 6. Supersession and predecessor-revival

Supersession is **separate from revocation**. They're orthogonal signals:

- **Revocation:** "this attestation is invalid."
- **Supersession:** "there's a newer version."

An attestation is live iff (no live descendant supersedes it) **AND** (no live revocation targets it).

**Predecessor-revival (normative, spec §4.3).** Consider chain `a0 → a1` (a1 supersedes a0). If a1 is revoked and no other descendant exists, **a0 becomes live again**. The semantics of "live" is "current authoritative version"; if the rotation that replaced a0 is itself rolled back, a0 is what stands.

This is intentional. It matches the natural mental model: rotating a key, then discovering the rotation was a mistake, leaves the original intact.

If you want **once-superseded-stays-dead** semantics, two options:

1. Issue an explicit `revocation` against the predecessor at supersession time (sign with the same authority that issued the supersession).
2. Use a retirement-style kind that itself stays live in the chain — identity's `identity-retirement` is the canonical example. A retirement caps a function's authority dead; if the retirement is later revoked, the predecessor revives correctly because retirement stayed live until that point.

## 7. Walking chains

`walk_attesting_chain` walks back through the `attesting` relation until a consumer-supplied `terminate_predicate` matches. Identity uses it to walk cert chains back to the quorum:

```
chain = ATTESTATION.walk_attesting_chain(
  start = some_agent_cert,
  terminate_predicate = identity_is_quorum_link,   ; "is this attesting a quorum_id?"
  ctx = ctx,
  max_depth = 32                                   ; bounded walk
)
; chain[-1].attesting is the quorum_id (predicate target value)
; chain[0] == start (the cert we walked from)
```

**The default `find_authorizing_fn` is normative** (spec §5.1):

1. Find live attestations where `attested == peer_hash`.
2. Resolve to the live head of each supersedes chain.
3. If multiple distinct live heads remain: deterministic tie-break by lowest content_hash.

**When to override `find_authorizing_fn`.** Multi-context peers (a peer that's a controller in identity A AND an agent in identity B simultaneously). The default tie-break picks one deterministically, but the chain-walk may then traverse the "wrong" identity's chain. Pass a predicate that filters by your context — e.g., by storage-path prefix, by `properties.kind`, or by quorum membership of `attesting`. (See spec §5.1 multi-context-peers note + TV-A11.)

## 8. Time-traveling validation (`as_of`)

Liveness checks accept an optional `as_of` parameter (epoch ms) for validating against historical state:

```
ATTESTATION.is_attestation_live(att, ctx, as_of = some_past_timestamp)
```

Used to validate cap chains against historical attestations: "was this cert live when the cap was issued?" Without `as_of`, you can't tell whether a cert that's now revoked or expired was valid at the time the chain link was made.

The check covers expiration AND supersession AND self-revocation, all evaluated at `as_of`. Authority-revocation, layered by the consumer, also accepts `as_of`.

**Storage retention.** Implementations don't garbage-collect superseded or revoked attestations — they remain in the tree and indexes (per index invariant I4). `find_attestations_*` returns them; consumers use `is_attestation_live` to filter. This is what makes time-travel work.

## 9. Indexes (impl-side, observable)

Implementations MUST maintain indexes on `attesting`, `attested`, `properties.kind`, and `supersedes` (spec §5.7). Observable invariants:

- **Write-then-read consistency** within a handler invocation.
- **Atomicity with handler completion** — failed handler ⇒ entity not in any index.
- **Liveness is not indexed** — entries persist after revocation/supersession. Apply `is_attestation_live` at query time.

If your code needs `find_attestations_with_kind("foo")` or `find_attestations_with_supersedes(prev_hash)`, those are spec'd helpers; they use the required indexes.

## 10. Pitfalls

- **Don't validate attestations with `verify_capability_chain`.** Attestations are not capability tokens (three-parallel-mechanisms invariant). Use `verify_attestation_signature` + consumer-specific topology rules.
- **Don't grant raw `tree:put` to attestation paths.** Bypasses the handler. Grant `system/attestation:create` / `:supersede` / `:revoke` instead (Coherent Capability principle).
- **Don't mistake supersession for revocation.** They're orthogonal. If you want "superseded means dead forever," issue an explicit revocation against the predecessor.
- **Authority-revocation is your problem, not the substrate's.** The substrate finds revocations; your consumer rule decides which ones count.
- **Multi-context peers need custom `find_authorizing_fn`.** Default tie-break is deterministic but kind-agnostic; if a peer has live attestations from multiple contexts, the wrong chain may be walked.

## 11. Reference

**Helpers (spec §4):**
- `verify_attestation_signature(att, ctx)` — single-sig from `attesting`.
- `verify_specific_signer(att, expected_signer, ctx)` — used for dual-sig and K-of-N composition.
- `is_attestation_live(att, ctx, as_of?)` — composite expiration + supersession + self-revocation.

**Graph operations (spec §5):**
- `walk_attesting_chain(start, terminate_predicate, ctx, find_authorizing_fn?, max_depth)` — chain walk back via `attesting`.
- `walk_supersedes_chain(start, ctx)` — chain walk back via `supersedes`.
- `find_live_head(start, ctx)` — forward walk to current live successor.
- `find_attestations_targeting(entity_hash, predicate, ctx)` — by `attested` (indexed).
- `find_attestations_by(peer_hash, predicate, ctx)` — by `attesting` (indexed).
- `find_revocations_for(attestation_hash, ctx)` — convenience wrapper.
- `find_attestations_with_supersedes(predecessor_hash, ctx)` — by `supersedes` (indexed).
- `find_attestations_with_kind(kind_value, ctx)` — by `properties.kind` (indexed).

**Handler operations (spec §6, all path-as-resource MUST):**
- `system/attestation:create` — issue.
- `system/attestation:supersede` — issue with `supersedes = previous_hash`.
- `system/attestation:revoke` — convenience for `kind="revocation"`.
- `system/attestation:verify` — first-gate signature + liveness; layer consumer rules on top.

**Cross-impl test vectors:** TV-A1 through TV-A11 in `VALIDATION-MATRIX-IDENTITY-FOUNDATIONS.md`. Implementations MUST pass all of them.
