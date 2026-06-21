# System Identity Composition — Navigation Doc

**Status**: Active (promoted Reference → Stable; §6 marked normative for cross-impl ownership)
**Authoritative scope:** §6 (ownership boundaries) is normative for cross-impl conformance. Other sections are informative orientation.
**Audience:** New readers needing to understand how identity, attestation, and quorum compose; cross-impl reviewers; spec authors of new extensions that consume the identity stack
**Scope:** What the three-extension identity stack is; how to read its specs in order; what each extension owns and what it doesn't

---

## 1. Why this document exists

The identity layer is composed of three extensions (per the landed identity proposal stack):

- **EXTENSION-ATTESTATION v1.2** — substrate primitive (signed-claim entity type + helpers)
- **EXTENSION-QUORUM v1.2** — K-of-N node primitive
- **EXTENSION-IDENTITY v3.5** — structured composition layer over the substrate primitives

Reading three specs cold without orientation is a cost. This doc is the orientation: what each extension is, why it exists, how they compose, and what to read first depending on what you're trying to do.

If you only have time for one paragraph: **the system is a signed-graph substrate. `system/attestation` is the edge type. `system/quorum` is a special node type with K-of-N signature semantics. Identity is a structured composition layer that uses both to express recognizable, recoverable, rotatable presence — but identity is just one consumer; group, VC, reputation, cluster, transaction, governance, audit, and provenance are or will be other consumers of the same primitives.**

---

## 2. The three extensions, in one sentence each

**EXTENSION-ATTESTATION:** "Peer A signs a claim that B has property X." Provides the entity type (`system/attestation`), single-sig validation, supersedes-chain walks, liveness checks, generic graph operations (parameterized by consumer-supplied predicates), and revocation as `properties.kind = "revocation"` attestations. Substrate.

**EXTENSION-QUORUM:** "These N peers; K must agree on this entity." Provides the K-of-N signing roster entity (`system/quorum`), the K-of-N validation primitive callable on any entity hash, self-maintenance events (`quorum-update`, `quorum-publish` as attestation conventions), and pluggable signer-resolution (concrete by default; identity-resolved registered by EXTENSION-IDENTITY at install).

**EXTENSION-IDENTITY:** "This handle, with these authority chains, recoverable through this quorum." A structured composition layer over attestation + quorum: function discrimination (controller / agent / identifier / app-defined), cert-chain walking, handle-bearing distinction, contact-side caching, recovery flows, publication modes for agent certs, sub-controller chains. Adds essentially zero new entity types — just `peer-config` (per-agent local state) and `identity-binding` (helper inner type).

---

## 3. The dependency stack

```
ENTITY-CORE-PROTOCOL
        ↑
   EXTENSION-ATTESTATION (the signed-graph substrate)
        ↑
   EXTENSION-QUORUM (depends on attestation)
        ↑
   EXTENSION-IDENTITY (depends on both)
        ↑
   EXTENSION-GROUP v1.5+ (depends on all three)
        ↑
   Future: EXTENSION-CLUSTER, EXTENSION-TRANSACTION, EXTENSION-GOVERNANCE,
           EXTENSION-VC, EXTENSION-REPUTATION, EXTENSION-PROVENANCE,
           EXTENSION-AUDIT
   (each depends on attestation + quorum + their own logic; do NOT need identity)
```

Future consumers can depend on attestation alone, on attestation + quorum, or on the full stack. Nothing requires identity unless the consumer needs identity-specific semantics.

---

## 4. The graph framing

Conceptually, the system is a directed graph:
- **Nodes:** anything hash-addressable. Mostly peers (Ed25519 keypairs); also quorums (K-of-N signing rosters); could be entities of other kinds (a piece of content, a service endpoint, a software binary).
- **Edges:** `system/attestation` entities. Direction `attesting → attested` with a `properties` payload.
- **Special node type:** `system/quorum` — a node whose "signature" is K signatures from N constituents. Otherwise behaves like any node.
- **Identity:** a convention. "An identity is the subgraph rooted at a quorum, with edges of specific shapes (function-discriminated certs), validated by specific rules (chain walks, topology requirements), with recognition semantics layered on top."

Identity adds essentially zero new entity types because it doesn't need them — it uses the substrate primitive (`system/attestation`) with identity-specific `properties` shape and identity-specific validation rules. Other consumers do the same: VC uses `system/attestation` with `properties.kind = "vc"`; reputation uses `system/attestation` with `properties.kind = "reputation"`; etc.

---

## 5. Three parallel mechanisms (the load-bearing invariant)

After the substrate split, three structurally distinct entity classes exist with three structurally distinct validators:

1. **V7 capability tokens** (validated by `verify_capability_chain` in ENTITY-CORE-PROTOCOL.md §5.5). Includes single-sig caps and multi-sig caps (per multi-sig core primitive). Cap-issuance layer.
2. **`system/attestation` entities** (validated by EXTENSION-ATTESTATION's helpers + consumer-specific topology rules). Used by identity, group, future VC, reputation, provenance, audit, etc.
3. **`system/quorum` entities** (validated by EXTENSION-QUORUM's `verify_k_of_n_signatures`). Used by identity, group, future cluster, transaction, governance.

**No shared validator.** Cap-chain machinery NEVER opens an attestation or quorum entity. Identity validators NEVER call into V7's `verify_capability_chain`. Quorum's K-of-N validator NEVER processes a cap or an identity-context attestation as anything other than the entity hash + signers being validated.

The mechanisms share signature-counting at a low level (any K-of-N reduces to "do K of N hashes have valid signatures over this target?") but the validation paths are structurally distinct. This is intentional — the type-level distinction must be preserved.

**Future K-of-N-needing extensions MUST pick one of the three existing mechanisms; do NOT introduce a fourth.**

---

## 6. What each extension owns

**§6 is normative for cross-impl ownership.** When a question arises about which extension owns a kind, type, validator, or storage path, §6 is the authority. Per-spec ownership claims that contradict §6 are spec bugs; surface them to the architecture team.

### 6.1 EXTENSION-ATTESTATION owns:
- The `system/attestation` entity type
- Generic graph operations (signature verify, supersedes walks, liveness checks, attesting-chain walks with consumer-supplied terminate predicates, attestation lookups by predicate)
- The revocation mechanism (`properties.kind = "revocation"`)
- The kind-ownership table (open vocabulary; consumers register their kinds; path-scoping disambiguates collisions)

EXTENSION-ATTESTATION does NOT own:
- Storage paths for attestations (consumers choose where their attestations live)
- Kind interpretation (consumers define the meaning of their kinds)
- Signature topology rules (consumers impose K-of-N, dual-sig, single-sig, etc. rules on top)
- Authority-revocation rules (consumers define who has authority to revoke their attestations)

### 6.2 EXTENSION-QUORUM owns:
- The `system/quorum` entity type
- The K-of-N validation primitive (`verify_k_of_n_signatures`)
- The current-signer-set resolver (`current_signer_set` walks the live quorum-update chain)
- Quorum self-event conventions (`"quorum-update"` and `"quorum-publish"` as `properties.kind` values)
- The pluggable signer-resolution interface (`concrete` mode built in; other modes registered by other extensions)

EXTENSION-QUORUM does NOT own:
- The attestation entity type itself (that's EXTENSION-ATTESTATION)
- Identity-specific resolution (EXTENSION-IDENTITY registers the `identity-resolved` mode)
- What entities get K-of-N validation (consumers decide; quorum just provides the validator)

### 6.3 EXTENSION-IDENTITY owns:
- The `system/identity/peer-config` entity type (per-agent local state)
- The `system/identity/identity-binding` helper inner type
- Identity-specific `properties.kind` values: `"identity-cert"`, `"identity-rotation-handoff"`, `"identity-rotation-recovery"`, `"identity-retirement"`
- Identity-specific `properties.function` values: `"controller"`, `"agent"`, `"identifier"`, app-defined
- Identity-specific `properties.mode` values for cert audience tier: `"internal"`, `"public"`, `"per-relationship"`, `"embedded"`
- Topology dispatch (which kind requires K-of-N from quorum, single-sig from controller, dual-sig for rotation, etc.)
- Cert-chain walking (walk via `attesting` back to a quorum)
- Handle-bearing distinction (which cert is the contact-facing handle)
- Contact-side caching of `system/quorum-publish` for compromise-recovery validation
- Recovery flows (quorum-driven `identity-rotation-recovery`; fail-closed validation against cached `quorum-publish`)
- Sub-controller chain semantics
- Publication-mode dispatch for agent certs
- Storage path conventions (`system/identity/internal/cert/...`, `system/identity/public/cert/...`, `system/identity/relationships/{c}/cert/...`)
- The `identity-resolved` signer-resolution mode (registered against EXTENSION-QUORUM's hook)

EXTENSION-IDENTITY does NOT own:
- The attestation entity type (uses `system/attestation` directly)
- The quorum entity type (uses `system/quorum` directly)
- K-of-N validation (calls `EXTENSION_QUORUM.verify_k_of_n_signatures`)
- Generic attestation mechanics (calls `EXTENSION_ATTESTATION.verify_attestation_signature`, `walk_attesting_chain`, etc.)

---

## 7. Reading order — what to read first

**If you're a new reader trying to understand the identity stack:**
1. This doc (you're here)
2. `core-protocol-domain/specs/ARCHITECTURE-IDENTITY-INFRASTRUCTURE.md` v1.0 — architecture overview with the layered picture (graduated from the v4.2 identity-infrastructure proposal)
3. `EXTENSION-ATTESTATION.md` v1.2 — the substrate (originating proposal: `proposals/implemented/PROPOSAL-EXTRACT-ATTESTATION-PRIMITIVE.md`)
4. `EXTENSION-QUORUM.md` v1.2 — K-of-N node primitive (originating proposal: `proposals/implemented/PROPOSAL-EXTRACT-QUORUM-PRIMITIVE.md`)
5. `EXTENSION-IDENTITY.md` v3.5 — the structured composition layer (originating proposal: `proposals/implemented/PROPOSAL-MINIMIZE-EXTENSION-IDENTITY.md`)

**If you're authoring a new extension that needs signed claims:**
1. This doc §6.1 (what attestation owns) and §6.2 (what quorum owns)
2. `EXTENSION-ATTESTATION.md` v1.0 for the API
3. (if you need K-of-N) `EXTENSION-QUORUM.md` v1.0
4. Pick your `properties.kind` values; document them in your spec; coordinate with the kind-ownership table in EXTENSION-ATTESTATION.md §3.2

**If you're doing a cross-impl review:**
1. `VALIDATION-MATRIX-IDENTITY-FOUNDATIONS.md` — consolidated test vectors (~50 across categories A/Q/I/X/TPM/R/M)
2. The three landed substrate specs' Conformance sections (`EXTENSION-ATTESTATION.md` §9, `EXTENSION-QUORUM.md` §9, `EXTENSION-IDENTITY.md` §10)
3. The originating proposals at `proposals/implemented/` for design rationale
4. Per-impl scope inventory in each originating proposal's §10 (cross-impl impact)

**If you're updating EXTENSION-GROUP, EXTENSION-ROLE, or another consumer:**
1. This doc §6 to understand what each extension owns
2. The dependency stack in §3
3. Your extension's spec text — focus on cross-references to identity v3.0 / v2.2 that need updating
4. `PROPOSAL-EXTENSION-GROUP-V1.5.md` (when authored) for the group-side reference pattern

### 7.4 Use-case and deployment-shape references

The architecture-side has accumulated multiple exploration documents. This table pins which is canonical for each topic so a cold reader doesn't have to guess.

| Topic | Canonical doc | Status |
|---|---|---|
| Conceptual identity model (synthesis) | `EXPLORATION-PEER-IDENTITY-MANAGEMENT.md` | Current (synthesis; supersedes USER-AND-ROLE-LIFECYCLE + RECOVERY-CLUSTER-MECHANICS) |
| Continuum of deployment shapes (8 shapes from solo to public consensus) | `EXPLORATION-DEPLOYMENT-SHAPES.md` | Current |
| Concrete single-user deployment | `EXPLORATION-ALICE-INFRASTRUCTURE.md` | Spatial layout current; **vocabulary stale** (uses pre-v3.0 names — see banner on doc); cross-reference DEPLOYMENT-SHAPES §3 for post-v3.x rendering |
| Identity design rationale + lenses | `EXPLORATION-IDENTITY-LENSES-AND-CONVERGENCE.md` | Current (latest synthesis pass) |
| Identity design review (v3.0 prep) | `EXPLORATION-IDENTITY-CONSOLIDATED-REVIEW.md` + `EXPLORATION-IDENTITY-DESIGN-REVIEW.md` | Current |
| Per-peer attestation rationale | `EXPLORATION-IDENTITY-TOPOLOGY-AND-DISCLOSURE.md` | Current |
| Identity simplification rationale | `EXPLORATION-IDENTITY-SIMPLIFICATION.md` | Superseded — banner-flagged; conclusions absorbed by CONSOLIDATED-REVIEW |
| User/role lifecycle | superseded — see `EXPLORATION-PEER-IDENTITY-MANAGEMENT.md` | Banner-flagged; structural lifecycle gaps absorbed into EXTENSION-ROLE v1.3+ |
| Recovery cluster mechanics | superseded — see `EXPLORATION-PEER-IDENTITY-MANAGEMENT.md` + v3.3 substrate | Banner-flagged; "recovery cluster" → "identity quorum" |
| Use cases by role-extension shape | `GUIDE-ROLE.md` §14 (Acme small business; OSS collaborator network; vacation delegation; adversarial) | Current (v0.2; §14.1 Acme is the cross-impl validation anchor) |
| Cross-impl test vectors | `reviews/VALIDATION-MATRIX-IDENTITY-FOUNDATIONS.md` | Current (Amendments 1-3 landed; Amendment 4 in flight per `proposals/PROPOSAL-ROLE-V2.0-PRODUCTION-READINESS.md`) |

When a new exploration lands, add a row. When a doc is superseded, mark the row and update the banner on the doc itself.

---

## 8. Common questions

**Q: Where does revocation live?**
A: It's a generic mechanism in EXTENSION-ATTESTATION. A revocation IS an attestation with `properties.kind = "revocation"`. Self-revocation (issuer revokes own) is handled by the primitive's liveness check. Authority-revocation rules (e.g., identity says quorum can revoke its own certs) live in the consumer extension.

**Q: How do I rotate a quorum member?**
A: Call `EXTENSION_QUORUM.system/quorum:update(quorum_id, new_signers, new_threshold)`. Gather K-of-N signatures from current constituents. Submit. This is a quorum-extension operation, not identity-specific. Identity is one consumer of this operation; cluster, transaction, governance are others.

**Q: Can two extensions use the same `properties.kind` value?**
A: Across extensions, kinds MAY collide because path-scoping disambiguates (each consumer's entities live under its own paths; each consumer's validators only process entities at its own paths). Within an extension's storage paths, kinds MUST be unambiguous. Namespacing (e.g., `"identity-cert"` instead of bare `"cert"`) is RECOMMENDED for global tooling clarity but not REQUIRED for correctness.

**Q: Why is multi-sig (cap-layer) separate from EXTENSION-QUORUM?**
A: They're at different layers. V7 multi-sig is at the cap-issuance layer (root cap with polymorphic `granter`); EXTENSION-QUORUM is at the entity-attestation layer (K-of-N validation over any entity hash). Same underlying signature-counting; different validation paths (per the three-parallel-mechanisms invariant in §5).

**Q: Does identity have its own attestation entity type?**
A: No. v3.2 uses `system/attestation` directly with identity-specific `properties.kind` and `properties.function` values. v3.0 (paper-only, never shipped) had `system/identity/attestation` as a wrapper type; v3.2 removes the wrapper.

**Q: How do I issue a controller cert for an identity?**
A: Call identity's `create_attestation` handler (which wraps `EXTENSION_ATTESTATION:create` with the right properties shape: `kind="identity-cert"`, `function="controller"`, `mode="public"` (3-key default) or `"internal"` (4-key advanced), `attesting=quorum_id`, `attested=controller-peer-hash`). Gather K-of-N signatures from the quorum. Submit.

**Q: What's the relationship between `peer-config.trusts_quorum` and `system/quorum`?**
A: `peer-config.trusts_quorum` is a content hash referencing a `system/quorum` entity at `system/quorum/{trusts_quorum_hex}`. The peer-config is identity-specific (per-agent local state); the quorum entity it references lives in EXTENSION-QUORUM's namespace.

---

## 9. What this doc is NOT

- **Not a tutorial.** For tutorials, see `GUIDE-IDENTITY.md` and related user guides.
- **Not a complete spec.** For specs, read each substrate proposal / spec.
- **Not a long-term roadmap.** For that, see `PLAN-EXTENSION-LANDSCAPE.md`.
- **Not a discussion of trade-offs or design rationale.** For that, see `EXPLORATION-IDENTITY-LENSES-AND-CONVERGENCE.md`.
- **Not authoritative on cap-layer multi-sig.** For that, see `PROPOSAL-MULTISIG-CORE-PRIMITIVE.md`.

---

## 10. Document history

- **Stable; §6 normative:** Status promoted Reference → Stable per the identity v3.2 migration-fixes proposal (SI-18). §6 explicitly marked normative for cross-impl ownership boundaries (it is the load-bearing single-page summary that implementations cite when deciding which extension owns a kind, type, validator, or storage path). Other sections remain informative orientation.

- **Initial:** Created as part of the substrate cross-impl pass (round 5 of identity foundations work). Per `EXPLORATION-IDENTITY-LENSES-AND-CONVERGENCE.md` §4a.16.6 round-4 recommendation #3 ("author SYSTEM-IDENTITY-COMPOSITION.md as single-entry-point navigation doc"). Reflects the post-split architecture (attestation + quorum + identity v3.2).
